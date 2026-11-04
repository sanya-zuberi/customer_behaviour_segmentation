# Customer Behaviour Segmentation

Grouping EV charging customers by how often they charge, how many stations they use, how much energy they take and which chargers they prefer, then measuring each segment's revenue and usage.

## The Question

Who are the network's customers, and how much is each group worth? This project builds one profile per customer, clusters customers into segments, names them as personas, and compares their revenue and usage. It also tests the segments the brief expected (high-frequency, commuters, public, one-time) against the data.

## Repo Structure

```
├── notebooks/        → full analysis notebook
├── output/           → CSV outputs
├── dashboard/        → Power BI .pbix file + page screenshots
├── slides/           → presentation deck
└── documentation/    → insight summary
```

## Data

The source is the `sessions` sheet of the EV charging dataset: 294,024 sessions from 8,000 customers, January 2022 – December 2024, with no missing values. The raw file is not included in this repo. It is hosted as a Kaggle Dataset: https://www.kaggle.com/datasets/sanyazuberi/ev-charging-raw-dataset.

Output files (in `output/`):
- `customer_features_segments.csv`: one row per customer: all engineered features and the assigned segment
- `segment_profiles.csv`: average behaviour of each persona
- `segment_revenue_usage.csv`: customers, sessions, energy and revenue per segment, with each segment's share
- `monthly_revenue_by_segment.csv`: revenue by month and segment
- `hourly_profile_by_segment.csv`: share of each segment's sessions by hour of day
- `region_mix_by_segment.csv`: share of each segment's sessions by region
- `charger_mix_by_segment.csv`: share of each segment's sessions by charger type
- `model_validation.csv`: XGBoost accuracy against baseline (naive and controlled tests)

## How Segments Were Built

- **One row per customer.** Sessions were summarised into features for frequency (sessions, active months, recency), energy and value (kWh, duration, revenue), station preference (stations and regions used, share of visits to the favourite station) and timing (weekday, peak-hour and night share). Charger-type, payment-method and user-type mixes were added. Peak hours are 07:00–09:59 and 16:00–18:59.
- **Assumed segments were tested first.** Commuters would show weekday-heavy, peak-hour-heavy charging and a favourite station. A "public" segment would need a stable `user_type` per customer. Neither exists in the data (see Key Findings).
- **Five features were clustered:** log of total sessions, sessions per active month, stations used, average kWh per session, and the share of sessions on 300 kW chargers, all standardised. Adding the timing features lowered the silhouette score from 0.655 to 0.452, so they were left out.
- **K-Means with 6 clusters** was chosen because the silhouette score peaks at k = 6 (0.655, against 0.557 at k = 7). DBSCAN was run as a cross-check. At a tight setting (eps = 0.3) it matches K-Means closely (adjusted Rand index 0.93) but labels 122 customers as noise, and at wider settings it merges tiers (0.57).
- **Personas are named by rule, not by cluster number.** Repeat customers are ranked by sessions, and the one-time clusters are split by charger. Names stay correct if the cluster numbering changes between runs.
- **Validation with XGBoost.** A classifier trained only on behaviours not used for clustering (timing, payment, user type) shows whether they predict segment. A naive test and a controlled test (exactly 15 sampled sessions per customer) were compared.

## Key Findings

- **Customers split into six segments.** Three repeat tiers (Power Users, Frequent Chargers, Steady Regulars) and three one-time groups (Fast-Charge, Standard and Slow-Charge Visitors). Customers have between 1 and 302 sessions, with a median of 18.
- **Revenue is concentrated in a quarter of customers.** Power Users and Frequent Chargers are 25% of customers and earn 75% of revenue. The top 20% of customers generate 66%. A Power User earns 14 times as much as a Steady Regular (7,163 against 509).
- **The one-time groups are 25% of customers and 0.6% of revenue.** The 2,000 customers with exactly one session all charged between 1 January and 28 June 2022. They are one early cohort, not a continuous flow of new visitors.
- **The repeat tiers differ in how often and how widely customers charge, not in what they buy.** Energy per session (47 kWh), duration (35 minutes) and ticket (about 27.8) are almost identical, and so are their hourly profile, region mix and charger mix.
- **There is no commuter segment.** Weekday share (0.77) and peak-hour share (0.33) are the same in every frequency band. Among the 5,999 customers with more than 5 sessions, none sends over half their visits to one station, and the favourite station takes only 13% of visits on average.
- **There is no "public" segment.** Every repeat customer's sessions are spread roughly evenly across the four user types (the main type covers only 35%), so `user_type` describes sessions, not people.
- **Timing and payment method carry no segment signal.** The naive XGBoost test scored 0.90 against a 0.66 baseline, but that comes from noise: customers with many sessions have less noisy shares. In the controlled test, accuracy was 0.61 against a 0.62 baseline.
- **Growth comes from the repeat tiers.** Revenue rose from about 1.03 million in 2022 to 2.59 million in 2023 and 4.55 million in 2024. Within each year monthly revenue is almost flat, and it steps up at the start of the next year.

## Personas

| Persona | Customers | Sessions each | Stations used | Revenue per customer | Share of revenue |
|---|---|---|---|---|---|
| Power Users | 400 | 258 | 33 | 7,163 | 35.1% |
| Frequent Chargers | 1,603 | 72 | 28 | 2,012 | 39.5% |
| Steady Regulars | 3,987 | 18 | 13 | 509 | 24.8% |
| One-Time Fast-Charge Visitors | 497 | 1 | 1 | 39 | 0.2% |
| One-Time Standard Visitors | 713 | 1 | 1 | 26 | 0.2% |
| One-Time Slow-Charge Visitors | 800 | 1 | 1 | 19 | 0.2% |

Power Users average about 7.3 sessions a month across all 9 regions. Frequent Chargers average 2.6 and Steady Regulars 1.4. The three one-time personas differ only in the charger used for their one session (300, 150 or 50 kW), and their 2,010 customers include about 10 who charged twice. They are best treated as one group.

## Recommendations

1. **Protect the top 25% of customers.** They bring 75% of revenue, and 400 Power Users alone bring 35%, which is a concentration risk. Prioritise their uptime and loyalty benefits.
2. **Test ways to move Steady Regulars up a tier.** A Frequent Charger earns about 4 times as much as a Steady Regular. The data shows the gap, not what would cause a customer to charge more, so try incentives on a small group first.
3. **Treat win-back as a small experiment.** The one-time customers last charged in early 2022, so response is likely to be low.
4. **Do not target by time of day or payment method.** These did not distinguish segments.

## Dashboard

Built in Power BI, one page:
- **Segment slicer** that filters every visual
- **Five KPI cards**: customers, total revenue, revenue from the top two tiers, revenue from one-time visitors, and Power User vs Steady Regular revenue per customer
- **Share of customers vs share of revenue** by segment
- **Monthly revenue by segment**
- **Hour-of-day profile** and **region mix** and **charger mix** by segment
- **XGBoost accuracy vs baseline** (naive and controlled)
- **Persona table**

See `dashboard/` for the `.pbix` file and page screenshots.

## Reproducing This Project

1. Download the raw dataset from the Kaggle link above.
2. Run the notebook in `notebooks/` from top to bottom in a fresh session. It writes the CSV files and the `figures/` folder.
3. Open the `.pbix` in `dashboard/` and refresh the data sources if the CSV paths have changed.

## Tools Used

Python (pandas, NumPy, scikit-learn, XGBoost, Matplotlib, Seaborn) · Kaggle Notebooks · Power BI

## Limitations

- The dataset appears simulated. The group sizes are very round (400 Power Users, 2,000 single-session customers) and timing patterns are identical across customers, so the segments may reflect how the data was generated rather than real behaviour.
- The data is behavioural only, with no demographics or vehicle information.
- The one-time split into three groups reflects the charger used, not different intent.
- Total sessions and stations used are strongly correlated (0.97), so frequency is effectively counted twice in the clustering. Average kWh and 300 kW share also correlate (0.88).
- Revenue steps up at the start of each year rather than growing smoothly. This was not investigated.
- Revenue figures are in the dataset's price units, since no currency is given.
- Silhouette scores are computed on a 5,000-customer sample, so small differences may appear between library versions.
