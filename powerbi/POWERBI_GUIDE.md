# Power BI dashboard: step-by-step build guide

This guide builds a one-page **Sales Forecast and Anomaly** dashboard from the two small CSV files in this folder. They hold about 630 rows in total, so the report stays light even on an older laptop.

| File | What it holds |
| --- | --- |
| `daily_sales.csv` | One row per day, Jan 2017 to Aug 2018: `date`, `orders`, `revenue`, `avg_order_value`, `anomaly_score`, `is_anomaly` (flags from the Python notebook) |
| `forecast_30d.csv` | The next 30 days from the Holt-Winters model: `date`, `forecast_revenue`, `lower_80`, `upper_80` |

Time needed: about 1.5 to 2 hours.

---

## 1. Load the data (Power Query)

1. **Home → Get data → Text/CSV** → `daily_sales.csv` → **Transform Data**.
2. In Power Query:
   - Check the column types: `date` = Date, `orders` = Whole number, `revenue`, `avg_order_value` and `anomaly_score` = Decimal number, `is_anomaly` = True/False.
   - Rename the query to `Sales`.
3. **New Source → Text/CSV** → `forecast_30d.csv`. Set `date` = Date and the other three columns = Decimal number. Rename the query to `Forecast`.
4. **Close & Apply**.

## 2. Date table

**Modeling → New table**, then paste:

```DAX
Date =
ADDCOLUMNS (
    CALENDAR ( DATE ( 2017, 1, 1 ), DATE ( 2018, 9, 30 ) ),
    "Year", YEAR ( [Date] ),
    "Month", FORMAT ( [Date], "MMM yyyy" ),
    "MonthKey", YEAR ( [Date] ) * 100 + MONTH ( [Date] ),
    "Weekday", FORMAT ( [Date], "ddd" ),
    "WeekdayNum", WEEKDAY ( [Date], 2 ),
    "Day Type", IF ( WEEKDAY ( [Date], 2 ) > 5, "Weekend", "Weekday" )
)
```

Then:
- Select the table → **Table tools → Mark as date table** → choose `Date`.
- Select `Month` → **Sort by column** → `MonthKey`. Select `Weekday` → **Sort by column** → `WeekdayNum`.
- **Model view:** drag `Date[Date]` onto `Sales[date]`, then onto `Forecast[date]`. Both relationships are one-to-many, single direction.

## 3. Measures

Create a blank table to hold the measures (**Home → Enter data** → name it `_Measures` → Load), then add each measure with **New measure**.

**Core KPIs**

```DAX
Total Revenue = SUM ( Sales[revenue] )

Total Orders = SUM ( Sales[orders] )

Avg Order Value = DIVIDE ( [Total Revenue], [Total Orders] )
```

**Trend**

```DAX
Revenue 7D Avg =
AVERAGEX (
    DATESINPERIOD ( 'Date'[Date], MAX ( 'Date'[Date] ), -7, DAY ),
    [Total Revenue]
)

Revenue Prev Month =
CALCULATE ( [Total Revenue], DATEADD ( 'Date'[Date], -1, MONTH ) )

Revenue MoM % =
DIVIDE ( [Total Revenue] - [Revenue Prev Month], [Revenue Prev Month] )
```

**Anomaly detection in DAX** (a rolling z-score: how unusual today is compared with the previous 28 days)

```DAX
Revenue 28D Mean =
AVERAGEX (
    DATESINPERIOD ( 'Date'[Date], MAX ( 'Date'[Date] ) - 1, -28, DAY ),
    [Total Revenue]
)

Revenue 28D StdDev =
STDEVX.P (
    DATESINPERIOD ( 'Date'[Date], MAX ( 'Date'[Date] ) - 1, -28, DAY ),
    [Total Revenue]
)

Revenue Z-Score =
DIVIDE ( [Total Revenue] - [Revenue 28D Mean], [Revenue 28D StdDev] )

Anomaly Revenue (DAX) =
IF ( ABS ( [Revenue Z-Score] ) > 3, [Total Revenue] )
```

**Anomalies from the Python model** (STL decomposition, more robust to the weekly cycle)

```DAX
Anomaly Days =
CALCULATE ( COUNTROWS ( Sales ), Sales[is_anomaly] = TRUE () )

Anomaly Revenue (Model) =
CALCULATE ( [Total Revenue], Sales[is_anomaly] = TRUE () )
```

**Forecast**

```DAX
Forecast Revenue = SUM ( Forecast[forecast_revenue] )

Forecast Lower 80 = SUM ( Forecast[lower_80] )

Forecast Upper 80 = SUM ( Forecast[upper_80] )
```

Formatting: set the revenue measures to Currency with 0 decimals (or a custom `"R$"#,0` format), `Revenue MoM %` to Percentage with 1 decimal, and `Revenue Z-Score` to 2 decimals.

> The 7-day, 28-day and z-score measures are meant for a **daily** axis. At month level they only look at the last days of each month.

## 4. Build the page

Canvas: 16:9. Use a light background and one accent colour (blue `#2A78D6`) with red `#E34948` kept for anomalies only.

```
┌──────────────────────────────────────────────────────────────┐
│ Title                                    [Date range slicer] │
├──────────┬──────────┬──────────┬──────────┬──────────────────┤
│ Revenue  │ Orders   │ AOV      │ Anomaly  │ Next 30 days     │
│ card     │ card     │ card     │ days     │ forecast card    │
├──────────┴──────────┴──────────┴──────────┴──────────────────┤
│ Line chart: daily revenue, 7-day avg, anomaly markers        │
├──────────────────────────────┬───────────────────────────────┤
│ Column chart: avg revenue    │ Line chart: forecast with     │
│ by weekday                   │ 80% range                     │
├──────────────────────────────┴───────────────────────────────┤
│ Table: anomaly days (date, orders, revenue, score)           │
└──────────────────────────────────────────────────────────────┘
```

1. **Title:** a text box with "Sales Forecast and Anomaly Detection | Olist e-commerce, 2017–2018".
2. **Date range slicer:** `Date[Date]`, style "Between".
3. **KPI cards:** `Total Revenue`, `Total Orders`, `Avg Order Value`, `Anomaly Days`. For the fifth card use `Forecast Revenue`, titled "Forecast, next 30 days".
4. **Main line chart**
   - X-axis: `Date[Date]`. Make sure it's the date field, not the Date Hierarchy: click the arrow next to the field and choose `Date`.
   - Y-axis: `Total Revenue` and `Revenue 7D Avg`. Add `Anomaly Revenue (Model)` as a third series.
   - Format the third series: line width 0, markers on, red, size 6. Only anomaly days show as red dots.
   - Revenue line grey and thin, 7-day average blue and thicker.
5. **Weekday column chart:** first add this measure, so the chart shows a typical day rather than a total:
   ```DAX
   Avg Daily Revenue = AVERAGEX ( VALUES ( 'Date'[Date] ), [Total Revenue] )
   ```
   Then X = `Date[Weekday]`, Y = `Avg Daily Revenue`.
   Colour weekend columns grey with **conditional formatting → Rules** on `Date[Day Type]`.
6. **Forecast chart:** a line chart with X `Date[Date]` and Y `Total Revenue`, `Forecast Revenue`, `Forecast Lower 80`, `Forecast Upper 80`.
   - Set a visual-level filter on `Date[Date]` to *is on or after* 1 Apr 2018.
   - Make the lower and upper lines light blue and dashed.
7. **Anomaly table:** columns `Sales[date]`, `Sales[orders]`, `Sales[revenue]` and `Sales[anomaly_score]`. Add a visual-level filter `is_anomaly = True`, sort by `anomaly_score` descending, and apply a red/blue diverging background to `anomaly_score`.

## 5. Use Power BI's built-in statistics (worth showing in interviews)

On a **copy** of the main line chart that has only `Total Revenue` on the Y-axis:

1. **Analytics pane (magnifying-glass icon) → Find anomalies → Add.** Set sensitivity to about 70%. Power BI marks unusual days itself and explains possible reasons when you click one.
2. **Analytics pane → Forecast → Add.** Settings: Forecast length 30 days, Confidence interval 80%, Seasonality 7 points. Compare it with the Python forecast in the other chart.

If either option is greyed out, check that the X-axis is the continuous `Date` field and that only one measure is on the Y-axis.

## 6. Finish

- Add **tooltips** on the main chart: `Total Orders`, `Avg Order Value`, `Revenue Z-Score`.
- **File → Save as** `sales_forecast_dashboard.pbix` in this folder.
- **File → Export → Export to PDF**. Also take a screenshot of the page, save it as `images/dashboard.png`, and upload both files to GitHub with the `.pbix`.

## Talking points

- *"I flagged anomalies two ways: a rolling z-score in DAX and an STL decomposition in Python. The DAX version is simple and live; the STL version handles the weekly cycle better. Black Friday 2017 stands out in both."*
- *"I backtested the forecast on 8 rolling 4-week windows instead of trusting one split. Holt-Winters beat a 'repeat last week' baseline, but only by about one percentage point, so I show the 80% range rather than a single number."*
