# TSA Complaint Forecasting

A time-series analysis of U.S. traveler complaints filed with the TSA. The
project forecasts monthly complaint counts, tests whether any complaint
category moves before the others, detects unusual spikes at individual
airports, and measures how the mix of complaints changed after the 2020 travel
disruption.

## Research Questions

1. Can monthly complaint volume be forecast from its own history?
2. Do any complaint categories rise before overall complaints rise?
3. Can a spike at an airport be detected against its own baseline?
4. Did the category mix change after 2020, and did it return to normal?

## Data

Three CSV files in `data/`, covering January 2015 to September 2025 (129
months):

| File | Description | Rows |
|------|-------------|------|
| `complaints-by-airport.csv` | Complaint count by airport and month | 49,045 |
| `complaints-by-category.csv` | Count by airport, month, and category | 291,752 |
| `complaints-by-subcategory.csv` | Count by airport, month, category, and subcategory | 628,628 |

All three files add up to the same total of 1,366,034 complaints, and the
notebook checks that they agree month by month.

**Source:** the [Data Liberation Project's TSA Complaint Counts](https://www.data-liberation-project.org/datasets/tsa-complaint-counts/),
a cleaned and standardized copy of reports the TSA released in its FOIA reading
room ([repository](https://github.com/data-liberation-project/tsa-complaint-counts)).
The TSA removed the original reports from its site on January 30, 2025, so the
CSVs are included here to keep the project reproducible. The data holds only
complaint counts, with no personal information.

## Methods

- **Cleaning and checks:** months converted to dates, blank airports relabeled
  `UNATTRIBUTED`, duplicate keys and negative counts checked, and the three
  files reconciled against each other. Airports that skip months are left out
  of airport-level work, since a missing month is not zero complaints.
- **Forecasting:** seven models, from a same-as-last-month forecast to a Random
  Forest, scored one month ahead over the last 24 months and on a 12-month
  holdout, always in time order so the future never leaks into training.
- **Leading indicators:** lagged correlations between each of the ten largest
  categories and everything else, then Granger causality tests with a
  Bonferroni correction.
- **Spike detection:** a robust z-score against the median of each airport's
  previous 12 months, run once on the airport alone and once relative to the
  nationwide level. With no labeled spikes, both versions are tested by
  injecting fake spikes into real airport-months.
- **Mix shift:** each month's category shares compared with the pre-2020 mix
  using Jensen-Shannon distance.

## Results

**Forecasting.** Over October 2023 to September 2025 (an average of 16,585
complaints a month), a damped seasonal Holt-Winters model was the most accurate
one month ahead.

| Model | One-month MAE | One-month MAPE | 12-month MAPE |
|-------|--------------:|---------------:|--------------:|
| Holt-Winters (damped, seasonal) | 956 | 6.2% | 7.7% |
| Naive (last month) | 1,293 | 8.2% | 7.7% |
| SARIMA (1,0,0)(1,0,0,12) | 1,299 | 8.2% | 8.2% |
| Seasonal naive | 2,658 | 15.4% | 20.6% |

Holt-Winters beat the naive forecast by two points one month ahead, but the two
tied over the 12-month holdout. At the airport level, forecast error ranged
from 20.3% at the 11 busiest airports to 65.2% at the 123 smallest.

**Leading indicators.** Categories move together in the same month (average
correlation 0.66) but barely across months (0.10). Two categories passed the
corrected Granger test, and neither is a reliable leading indicator.
Mishandling of Passenger Property is also predicted by the rest, so it moves
with them rather than ahead. PreCheck's correlation one month earlier is only
0.16, and it is the category most affected by changes in how complaints were
collected.

**Spike detection.** Comparing an airport to the nation worked better than
comparing it to itself. It flagged 25 airport-months instead of 84, only 5
during the 2021 to 2022 rebound instead of 60, and caught 73% of injected
spikes at three times the normal level, against 51%.

**Mix shift.** PreCheck rose from 40.4% of complaints before 2020 to 76.6% in
the recovery period, while Mishandling of Passenger Property fell from 20.7% to
1.9%. Every recovery month sits outside the normal pre-2020 range, even with
PreCheck removed.

## Limitations

- **Complaints are not service quality.** There are no passenger counts, so
  more complaints cannot be separated from more travelers.
- **Collection changes overlap with the recovery.** TSA attributes part of the
  increase to an easier PreCheck complaint form introduced in 2021. In October
  2022 the share of complaints tied to an airport also fell from 72.7% to
  17.2%, for reasons the source does not explain.
- **Spike detection was tested only on injected spikes,** not on confirmed real
  incidents.

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
