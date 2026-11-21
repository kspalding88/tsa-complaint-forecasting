# TSA Complaint Forecasting

A time-series analysis of U.S. traveler complaints filed with the TSA. The
project forecasts monthly complaint counts, tests which complaint categories
move before the others, and measures how the mix of complaints changed after
the 2020 travel disruption.

## Overview

The TSA used to publish monthly complaint counts by airport and category. This
project asks four questions of that data:

1. **How well can next month's complaints be forecast?** Seven forecasting
   methods are compared on the same hold-out months.
2. **Does forecast accuracy depend on airport size?** Airports are grouped by
   typical monthly complaint volume and scored separately.
3. **Does any category lead the others?** A lagged-correlation check and a
   Granger causality test look for categories that move before the rest.
4. **Did the mix of complaints change after 2020?** Months are labeled Pre,
   Disruption, or Recovery and compared against the pre-2020 mix.

## Data

Three CSV files in `data/`, covering January 2015 to January 2024:

| File | Description | Rows |
|------|-------------|------|
| `complaints-by-airport.csv` | Complaint count by airport and month | 41,721 |
| `complaints-by-category.csv` | Count by airport, month, and category | 241,588 |
| `complaints-by-subcategory.csv` | Count by airport, month, category, and subcategory | 504,512 |

All three files add up to the same total of 1,021,281 complaints, and the
notebook checks that they agree month by month.

**Source:** the [Data Liberation Project's TSA Complaint Counts](https://www.data-liberation-project.org/datasets/tsa-complaint-counts/),
a cleaned and standardized copy of reports the TSA released in its FOIA reading
room ([repository](https://github.com/data-liberation-project/tsa-complaint-counts)).
The TSA removed the original reports from its site on January 30, 2025, so the
CSVs are included here to keep the project reproducible. No personally
identifiable information is in the data.

## Methods

- **Cleaning and checks:** months converted to dates, blank airports relabeled
  `UNATTRIBUTED`, duplicate keys and negative counts checked, and the three
  files reconciled against each other.
- **Forecasting:** one-month-ahead forecasts rolled forward over the last 24
  months, plus a 12-month hold-out. Models: Holt-Winters (damped, seasonal),
  two SARIMA specifications, a Random Forest on lagged values, a naive
  forecast, a seasonal naive forecast, and a 12-month average.
- **Metrics:** MAE, RMSE, and MAPE for the total series; weighted absolute
  percentage error (WAPE) for airport-level results.
- **Leading indicators:** lagged correlations between each of the ten largest
  categories and everything else, then Granger causality tests with a
  Bonferroni correction.
- **Mix shift:** each month's category shares compared with the pre-2020 mix
  using Jensen-Shannon distance.

## Results

**Total complaints.** Over the last 24 months (February 2022 to January 2024,
averaging 14,240 complaints a month), the damped seasonal Holt-Winters model
was the most accurate, both one month ahead and on the 12-month hold-out.

| Model | One-month MAE | One-month MAPE | 12-month MAPE |
|-------|--------------:|---------------:|--------------:|
| Holt-Winters (damped, seasonal) | 1,102 | 8.0% | 6.9% |
| Naive (last month) | 1,176 | 8.5% | 8.3% |
| Random Forest on lags | 1,443 | 10.1% | 7.4% |
| Seasonal naive | 4,092 | 29.3% | 9.0% |

The seasonal naive forecast does worst one month ahead because the level of
complaints roughly doubled between 2021 and 2022, so last year's pattern was a
poor guide.

**By airport size.** Across the 109 airports with at least 10 typical monthly
complaints, the plain 12-month average was the most accurate model (14.8%
pooled WAPE), ahead of Holt-Winters (17.5%). Accuracy falls steadily with size:
forecast error goes from 12.6% at the busiest complaint airports to 60.2% at
the smallest.

**Leading indicator.** Categories mostly move together within the same month
and barely across months. Only *Mishandling of Passenger Property* passes the
Granger test (1-month lag, p = .021 after correction). This shows the category
helps predict the rest, not that it causes them.

**Mix shift.** The Expedited Passenger Screening Program (PreCheck) rose from
40.4% of complaints before 2020 to 79.4% in the recovery period, while
Mishandling of Passenger Property fell from 20.7% to 1.7%. Every recovery
month sits outside the normal pre-2020 range of month-to-month variation.

## Limitations

- **Complaints are not service quality.** They reflect who chose to report, and
  the airport field is blank for 42.9% of complaints.
- **Changes in how complaints are collected affect comparisons over time.** The
  Data Liberation Project notes that the TSA attributes part of the increase to
  easier PreCheck complaint forms introduced in 2021, so the post-2020 shift in
  mix is not purely a change in traveler experience.
- **Small airports are hard to forecast.** Their monthly counts are small and
  noisy, which is why error is highest there.

## Repository Contents

```
tsa_complaint_forecasting.ipynb   Full analysis (cleaning, models, figures)
data/                             The three TSA complaint CSV files
requirements.txt                  Python dependencies
```

## How to Run

Tested on Python 3.13.

```bash
pip install -r requirements.txt
jupyter notebook tsa_complaint_forecasting.ipynb
```

Run all cells from the repository root. The notebook finds the CSVs in `data/`
and runs end to end in about two minutes with no errors.

## Tools

Python, pandas, NumPy, scikit-learn, statsmodels, SciPy, Matplotlib, Jupyter.
