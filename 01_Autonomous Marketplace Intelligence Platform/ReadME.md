# Instructions.md
# Autonomous Marketplace Intelligence Platform

## 1. Project Overview

This project is an end-to-end machine-learning marketplace intelligence platform designed to simulate the intelligence layer of a large on-demand delivery or mobility marketplace.

The platform receives historical and simulated marketplace events, transforms them into point-in-time features, runs multiple specialized ML models, and feeds their outputs into a decision engine. The decision engine can estimate delivery ETA, forecast demand, rank available drivers, recommend prices, score fraud risk, and expose model health through APIs and dashboards.

The important engineering principle is that this is **not one ML model**. It is a coordinated ML system in which several models solve different parts of the marketplace optimization problem.

The recommended implementation uses a combination of real public datasets rather than pretending that one public dataset contains every field needed by a production marketplace. This distinction is important because real companies such as Uber, DoorDash, Instacart, Lyft, and food-delivery platforms have private operational data such as exact driver GPS traces, order preparation timestamps, cancellation reasons, dispatch offers, driver acceptance/rejection events, and counterfactual pricing experiments. Those fields are generally not available in public datasets.

Therefore, this project should be built as a **multi-dataset marketplace simulator backed by real observations**:

1. NYC TLC Trip Record Data for large-scale trip, time, geography, fare, and ETA/demand modeling.
2. Chicago Taxi Trips for persistent vehicle-level identity, fare/revenue information, trip outcomes, and driver/vehicle-ranking proxies.
3. IEEE-CIS Fraud Detection for a genuine labeled fraud problem.
4. Optional weather/event datasets can be joined later to improve exogenous demand and ETA features.

The resulting system is still based on real data. The simulator is used only where public data does not expose the operational event stream required by the original marketplace concept.

---

# 2. Dataset References

## 2.1 Primary Dataset: NYC TLC Trip Record Data

Use the official NYC Taxi and Limousine Commission Trip Record Data:

https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page

The TLC publishes monthly trip records for Yellow Taxi, Green Taxi, For-Hire Vehicles, and High Volume For-Hire Vehicles. The current official page provides Parquet files and data dictionaries.

The data contains fields such as pickup/drop-off timestamps, pickup/drop-off locations, trip distance, fares, payment information, passenger counts, and other operational attributes. For High Volume For-Hire Vehicle records, additional fields such as request time and pickup/on-scene information are available.

Official documentation:

https://www.nyc.gov/site/tlc/about/data.page

AWS public-data reference:

https://registry.opendata.aws/nyc-tlc-trip-records-pds/

Recommended initial training period:

- 2025-01 through 2025-12 for a manageable but recent baseline.
- Add 2026 months after the first pipeline is stable.
- For research experiments, use multiple years to capture seasonality and regime changes.

Do not download every month immediately. Start with 2-3 representative months, validate the pipeline, then scale out.

Recommended NYC tables:

- Yellow Taxi Trip Records
- High Volume For-Hire Vehicle Trip Records
- Taxi Zone Lookup Table
- Taxi Zone Shapefile/geometry data

The NYC TLC page states that trip data is published monthly and stored in Parquet because of dataset size. It also provides data dictionaries and taxi-zone lookup information.

### What NYC TLC gives this project

ETA prediction:
- pickup time
- drop-off time
- pickup zone
- drop-off zone
- trip distance
- trip duration
- traffic/time-of-day proxies

Demand forecasting:
- timestamp
- origin zone
- destination zone
- trip counts

Dynamic pricing:
- observed fare
- trip distance
- time
- location
- trip duration
- payment/revenue information

Cancellation modeling:
- this requires careful proxy construction because the public trip dataset does not provide a clean marketplace cancellation label.
- A better production-quality approach is to create a simulated request/accept/cancel event stream from historical trip arrivals.

Preparation/pickup-time modeling:
- for HVFHV data, use request-to-pickup/on-scene intervals where available as an operational preparation/fulfillment proxy.
- Do not claim this is restaurant kitchen preparation time.

---

## 2.2 Secondary Dataset: Chicago Taxi Trips

Official dataset:

https://data.cityofchicago.org/Transportation/Taxi-Trips-2024/ajtu-isnz

Data.gov reference:

https://catalog.data.gov/dataset/taxi-trips-2024

Historical dataset:

https://catalog.data.gov/dataset/taxi-trips-2013-2023

Chicago is particularly useful because the Taxi ID is consistent for a given taxi across trips. It therefore provides a persistent vehicle-level entity that can be used for ranking experiments.

Important limitation:

A Taxi ID is not the same thing as a driver identity. Therefore, the project should call this:

"Driver Ranking / Vehicle Ranking Proxy"

unless an actual driver-level dataset is introduced.

This is a much more honest and technically defensible design than pretending Taxi ID is a driver license.

Useful Chicago fields include:

- Trip ID
- Taxi ID
- Trip Start Timestamp
- Trip End Timestamp
- Trip Seconds
- Trip Miles
- Pickup Census/Community Area information where available
- Dropoff area
- Fare
- Tips
- Tolls
- Extras
- Trip Total
- Payment Type
- Company

The City of Chicago notes that timestamps are rounded to 15 minutes in the privacy-preserving public data and that not all trips are necessarily reported.

This makes Chicago suitable for:
- persistent entity performance features
- revenue/performance scoring
- ranking
- demand forecasting
- price modeling
- trip-duration modeling

but it introduces a coarser temporal resolution than raw marketplace event logs.

---

## 2.3 Fraud Dataset: IEEE-CIS Fraud Detection

Official Kaggle competition:

https://www.kaggle.com/c/ieee-fraud-detection/data

This is a real labeled fraud dataset. The target is:

`isFraud`

The dataset contains transaction and identity tables joined by `TransactionID`.

The official dataset includes:

- transaction features
- card-related features
- product code
- address-related information
- email domains
- device information
- network/identity attributes
- engineered Vesta features
- fraud label

The complete dataset is large, approximately 1.35 GB in the Kaggle representation, with hundreds of features.

Use this dataset specifically for the fraud component instead of trying to manufacture a fraud label from taxi data.

Important distinction:

IEEE-CIS is not a transportation dataset. It represents the fraud intelligence subsystem of the marketplace. This is intentional: a real marketplace typically has a payment/transaction risk system separate from its dispatch and ETA models.

---

# 3. Recommended Dataset Architecture

Do not force all datasets into one giant table.

Create a canonical marketplace schema and map each source dataset into it.

Conceptually:

```text
                       ┌──────────────────────┐
                       │ NYC TLC Trip Records │
                       └──────────┬───────────┘
                                  │
                       ┌──────────▼───────────┐
                       │ Canonical Trip Table │
                       └──────────┬───────────┘
                                  │
                 ┌────────────────┼─────────────────┐
                 │                │                 │
                 ▼                ▼                 ▼
             ETA Model      Demand Model       Pricing Model


                       ┌──────────────────────┐
                       │ Chicago Taxi Trips   │
                       └──────────┬───────────┘
                                  │
                                  ▼
                       Entity Performance
                                  │
                                  ▼
                         Ranking Model


                       ┌──────────────────────┐
                       │ IEEE-CIS Fraud Data  │
                       └──────────┬───────────┘
                                  │
                                  ▼
                         Fraud Risk Model
```

The canonical schema should conceptually contain:

```text
event_id
event_timestamp
request_timestamp
pickup_timestamp
dropoff_timestamp

origin_zone
destination_zone

entity_id
vehicle_id
driver_id

distance
duration
fare
tip
total_amount

payment_type
marketplace
service_type

weather_features
calendar_features
demand_features

eta_target
cancel_target
fraud_target
```

Fields that are not available in the source should remain null rather than being silently fabricated.

---

# 4. Project Objectives

The final system should implement the following intelligence services.

```text
Incoming Marketplace Event
            |
            v
     Feature Platform
            |
     +------+------+------+------+------+------+
     |      |      |      |      |      |
     v      v      v      v      v      v
    ETA  Demand  Prep  Cancel Rank  Pricing Fraud
     |      |      |      |      |      |      |
     +------+------+------+------+------+------+
                    |
                    v
             Decision Engine
                    |
          +---------+----------+
          |                    |
          v                    v
       REST API            Dashboard
```

The implementation should be modular enough that every model can be trained, evaluated, registered, deployed, monitored, and rolled back independently.

---

# 5. Data Engineering Pipeline

## 5.1 Raw Layer

Store downloaded files exactly as received.

Example:

```text
data/
  raw/
    nyc_tlc/
      yellow/
        year=2025/
          month=01/
          month=02/
      hvfhv/
        year=2025/
    chicago/
      taxi/
        year=2024/
    ieee_cis/
      train_transaction.csv
      train_identity.csv
```

Never modify raw files.

---

## 5.2 Bronze Layer

Convert source data into standardized Parquet datasets.

Use:

- PyArrow
- Polars or DuckDB for efficient processing
- partitioning by date
- compression
- schema validation

Example:

```text
data/
  bronze/
    trips/
      date=2025-01-01/
      date=2025-01-02/
```

---

## 5.3 Silver Layer

Clean and normalize the data.

Perform:

- timestamp parsing
- timezone normalization
- invalid-duration removal
- impossible-distance detection
- negative fare handling
- duplicate detection
- missing-value analysis
- categorical normalization
- geographic joins
- feature-level data quality checks

Example validity rules:

```text
trip_duration > 0
trip_distance >= 0
fare >= 0
pickup_timestamp <= dropoff_timestamp
```

Do not blindly delete anomalies. First classify them as:

```text
valid
invalid
suspicious
unknown
```

This matters for fraud and monitoring experiments.

---

## 5.4 Gold Layer

Create model-specific feature tables.

Example:

```text
features/
  eta/
  demand/
  ranking/
  pricing/
  fraud/
```

Each table must follow point-in-time correctness.

For a prediction made at time `t`, only information available at or before `t` may be used.

This is one of the most important requirements of the entire project.

---

# 6. Feature Engineering

## 6.1 Time Features

Create:

```text
hour
day_of_week
day_of_month
month
week_of_year
is_weekend
is_holiday
rush_hour
```

Encode cyclic features where appropriate:

```text
sin_hour
cos_hour
sin_day_of_week
cos_day_of_week
```

For tree models, raw categorical/time features can also work well, but cyclic encoding is useful for models that exploit smooth temporal relationships.

---

## 6.2 Geographic Features

Create:

```text
origin_zone
destination_zone
zone_pair
origin_demand
destination_demand
origin_to_destination_distance
```

If latitude/longitude or zone geometry is available, calculate:

```text
haversine_distance
zone_centroid_distance
```

For routing-aware ETA, road-network distance is preferable to straight-line distance.

---

## 6.3 Demand Features

For every zone and time window calculate:

```text
orders_last_5m
orders_last_15m
orders_last_30m
orders_last_60m
orders_last_6h
orders_same_hour_previous_day
orders_same_hour_previous_week
rolling_mean_1h
rolling_mean_6h
rolling_mean_24h
```

Do not calculate these using future records.

---

## 6.4 Entity Features

For ranking:

```text
completed_trips
average_trip_duration
median_trip_duration
average_fare
revenue_per_hour
trips_per_hour
recent_completion_rate
recent_cancellation_rate
recent_rating
acceptance_rate
```

The last three fields are not directly available in the public taxi data. If they are not observed, use a simulated marketplace event layer.

---

# 7. ETA Prediction

## 7.1 Problem Definition

The ETA model predicts how long a trip will take.

Target:

```text
eta_seconds = dropoff_timestamp - pickup_timestamp
```

For a real marketplace, the target should ideally be:

```text
predicted_delivery_time_from_request
```

which includes:

```text
dispatch delay
driver arrival
pickup/preparation
travel time
```

The public taxi data mostly gives trip-level duration, so implement the first version as travel-time ETA and later extend it with simulated dispatch and pickup components.

---

# 8. ETA Model: LightGBM

LightGBM is a gradient-boosted decision-tree framework optimized for speed and large datasets.

The model builds trees sequentially:

```text
Prediction_0
    |
    +-- residuals
          |
          v
       Tree_1
          |
          +-- residuals
                |
                v
             Tree_2
```

The final prediction is the sum of tree contributions.

For regression:

```text
ETA_hat = sum(tree_i(features))
```

Recommended features:

```text
trip_distance
origin_zone
destination_zone
hour
day_of_week
month
is_weekend
historical_zone_pair_duration
historical_origin_duration
recent_demand
recent_trip_duration
```

Recommended metrics:

```text
MAE
RMSE
Median Absolute Error
P90 Absolute Error
P95 Absolute Error
MAPE
```

MAE is especially important because ETA error has direct operational meaning.

A useful marketplace metric is:

```text
P90 absolute ETA error
```

because users care about severe underestimation as well as average performance.

---

# 9. ETA Model: CatBoost

CatBoost is another gradient-boosting algorithm with particularly strong handling of categorical variables.

This is useful because marketplace data contains many categorical features:

```text
origin_zone
destination_zone
zone_pair
company
payment_type
service_type
hour_bucket
```

Instead of manually one-hot encoding every high-cardinality category, CatBoost can learn target statistics in a leakage-aware manner.

Recommended comparison:

```text
LightGBM
    vs
CatBoost
```

Evaluate them on exactly the same temporal holdout.

Do not compare one model using random train/test splitting and the other using time splitting.

Recommended experiment:

```text
Train:      Jan-Oct
Validation: Nov
Test:       Dec
```

For multiple years:

```text
Train:      Year 1 + Year 2
Validation: first part of Year 3
Test:       final part of Year 3
```

---

# 10. Demand Forecasting

## 10.1 Problem Definition

Demand forecasting predicts the number of marketplace requests/trips in a future time interval.

Create a time series:

```text
timestamp
zone
demand
```

Example:

```text
2025-01-01 10:00, Zone_01, 83
2025-01-01 10:05, Zone_01, 91
2025-01-01 10:10, Zone_01, 88
```

Start with 15-minute or 30-minute buckets.

Do not immediately forecast thousands of zones at once. First create a single-zone model, then move to global/multi-zone modeling.

---

# 11. PatchTST

PatchTST is a Transformer-style time-series architecture that divides a long time series into patches.

Instead of treating every timestamp independently:

```text
x1 x2 x3 x4 x5 x6 x7 x8 ...
```

the model creates patches:

```text
[x1 x2 x3 x4]
[x5 x6 x7 x8]
...
```

The patches become tokens for the Transformer.

This reduces sequence length and allows the model to learn temporal patterns over local windows.

For marketplace demand:

```text
past 96 intervals
        |
        v
patching
        |
        v
Transformer
        |
        v
future 12 intervals
```

For 15-minute data:

```text
96 intervals = 24 hours
12 intervals = 3 hours
```

Useful input channels:

```text
demand
hour
day_of_week
holiday
weather
lag_1
lag_4
lag_96
lag_672
```

where:

```text
96  = previous day
672 = previous week
```

---

# 12. Temporal Fusion Transformer

TFT is designed for multi-horizon forecasting and combines:

- static features
- known future features
- observed historical features
- attention
- gating mechanisms
- variable selection

This is particularly appropriate when future calendar information is known.

Example:

```text
Static:
zone_id

Known future:
hour
day_of_week
holiday

Observed past:
demand
weather
recent trip volume
```

The model can learn which variables matter at different times.

TFT is useful for explaining which inputs contributed to forecast behavior through its variable-selection and attention mechanisms, although attention should not automatically be treated as causal explanation.

---

# 13. N-HiTS

N-HiTS is a hierarchical neural forecasting architecture that models different temporal scales.

The key idea is that forecasting can benefit from separating:

```text
short-term variation
medium-term structure
long-term trend
```

For marketplace demand:

```text
Short:
last few intervals

Medium:
daily pattern

Long:
weekly/seasonal pattern
```

N-HiTS can therefore be useful for strong multi-horizon forecasting without requiring the same Transformer structure as TFT or PatchTST.

---

# 14. Demand Model Evaluation

Compare:

```text
Seasonal Naive
LightGBM baseline
PatchTST
TFT
N-HiTS
```

Do not compare deep learning models without a strong baseline.

Recommended metrics:

```text
MAE
RMSE
WAPE
sMAPE
MASE
P50 error
P90 error
```

For marketplace operations, WAPE is often more interpretable than MAPE when demand contains zeros or very small counts.

---

# 15. Driver Ranking / Vehicle Ranking

## 15.1 Important Dataset Limitation

The public Chicago Taxi dataset provides persistent Taxi IDs, but Taxi ID does not necessarily identify the human driver.

Therefore, implement:

```text
Driver Ranking = Vehicle/Entity Ranking Proxy
```

If you later obtain an actual driver-level dataset, replace:

```text
vehicle_id
```

with:

```text
driver_id
```

without changing the ranking architecture.

---

# 16. Ranking Objective

At request time, suppose these vehicles are available:

```text
Driver A
Driver B
Driver C
Driver D
```

The ranking system produces:

```text
A -> 0.82
B -> 0.67
C -> 0.91
D -> 0.54
```

The dispatch engine selects:

```text
C
A
B
D
```

depending on constraints.

Features can include:

```text
distance_to_pickup
estimated_arrival_time
historical_completion_rate
historical_duration
revenue_per_hour
recent_trip_count
zone familiarity
recent performance
```

Some features must be simulated if unavailable in the public dataset.

---

# 17. LambdaMART

LambdaMART is a learning-to-rank algorithm based on gradient-boosted trees and ranking-specific optimization.

Instead of predicting:

```text
good_driver = 1
bad_driver = 0
```

the model learns:

```text
Which candidate should be ranked above another?
```

A training example can be represented as:

```text
request_id
candidate_driver_id
features...
relevance_label
```

Example:

```text
request_101, driver_A, ..., 2
request_101, driver_B, ..., 0
request_101, driver_C, ..., 3
```

Higher relevance means a better assignment.

A useful relevance function is:

```text
reward =
    completed_trip
    - pickup_distance_penalty
    - ETA_penalty
    + revenue_bonus
```

For an initial project, create labels from observed performance and then transition to a simulated dispatch environment.

---

# 18. Ranking Metrics

Use:

```text
NDCG@1
NDCG@3
NDCG@5
MRR
MAP@K
Recall@K
```

Also measure marketplace metrics:

```text
pickup ETA
completion rate
revenue per driver-hour
driver utilization
customer ETA
```

Offline ranking metrics alone are insufficient because a ranking system changes marketplace behavior.

---

# 19. Dynamic Pricing

Dynamic pricing is fundamentally different from supervised prediction.

ETA asks:

```text
What will happen?
```

Pricing asks:

```text
What action should I take?
```

Therefore, pricing is a decision-making problem.

---

# 20. Contextual Bandits

A contextual bandit receives:

```text
context
```

and selects:

```text
action
```

then observes:

```text
reward
```

Marketplace example:

```text
Context:
demand = high
available drivers = low
weather = bad
hour = 18:30
zone = Manhattan

Actions:
price multiplier = 1.0
price multiplier = 1.1
price multiplier = 1.2
price multiplier = 1.3
```

The system chooses an action and observes:

```text
acceptance
conversion
revenue
wait time
driver supply
cancellation
```

A simple reward can be:

```text
reward =
    revenue
    - cancellation_cost
    - wait_time_cost
    - supply_imbalance_cost
```

---

# 21. Offline Pricing Problem

Historical observational data does not tell us what would have happened if a different price had been offered.

This is called the counterfactual problem.

For example:

```text
Observed:
price = $20
accepted = yes
```

We cannot directly conclude:

```text
price = $25
would have been accepted
```

Therefore, do not train a contextual bandit by simply treating historical price as the optimal action.

Use one of these approaches:

### Approach A: Demand/acceptance model

Train:

```text
P(accept | context, price)
```

Then simulate expected reward for each price.

### Approach B: Offline contextual bandit evaluation

Use logged policies and inverse propensity scoring if historical action propensities are available.

### Approach C: Simulator

Build a marketplace simulator:

```text
demand
  +
supply
  +
price
  +
ETA
  |
  v
customer acceptance
  |
  v
marketplace reward
```

For this project, Approach A + C is the most practical.

---

# 22. PPO

PPO, Proximal Policy Optimization, is a reinforcement-learning algorithm.

It learns a policy:

```text
policy(action | state)
```

rather than a direct supervised prediction.

Marketplace state:

```text
demand
available_drivers
driver_utilization
average_wait_time
current_price
weather
hour
zone
```

Action:

```text
price_multiplier
```

Reward:

```text
revenue
- cancellation_cost
- waiting_cost
- supply_imbalance
- customer_penalty
```

The environment evolves:

```text
state_t
   |
   v
PPO policy
   |
   v
price action
   |
   v
market simulator
   |
   v
reward + state_t+1
```

Do not deploy PPO directly against a real marketplace.

Train PPO inside a simulator and validate it offline.

---

# 23. Fraud Detection

Fraud is a classification/risk-scoring problem.

Use the IEEE-CIS dataset for the initial implementation.

The target is:

```text
isFraud
```

The key challenge is class imbalance.

A model that predicts:

```text
not fraud
```

for almost every transaction can achieve high accuracy while being useless.

Therefore prioritize:

```text
PR-AUC
ROC-AUC
Recall
Precision
F1
Recall at fixed review rate
Expected fraud loss
```

---

# 24. Isolation Forest

Isolation Forest is an unsupervised anomaly-detection algorithm.

The intuition is that unusual observations are easier to isolate using random splits.

Normal observations:

```text
require many splits
```

Anomalies:

```text
require fewer splits
```

The resulting anomaly score can be used as:

```text
anomaly_score
```

Use Isolation Forest when:

- labels are unavailable
- fraud patterns change
- you want an independent anomaly signal
- you want a secondary model for model diversity

Do not assume:

```text
anomaly = fraud
```

An anomaly is simply unusual behavior.

A strong production design combines:

```text
supervised fraud probability
+
unsupervised anomaly score
+
rules
+
graph risk
```

---

# 25. Graph Fraud Detection

Fraud frequently involves relationships between entities.

Create a graph:

```text
Customer
   |
   +---- Device
   |
   +---- Card
   |
   +---- Email
   |
   +---- Address
   |
   +---- IP
```

Example:

```text
Customer A ---- Device X
Customer B ---- Device X
Customer C ---- Device X
```

A shared device across many unrelated customers may indicate coordinated behavior.

Possible graph features:

```text
device_degree
card_degree
email_degree
ip_degree
connected_fraud_count
neighbor_fraud_ratio
community_size
shortest_path_to_known_fraud
```

Start with graph-derived features and LightGBM.

Then move to:

```text
GraphSAGE
GAT
heterogeneous GNN
```

if the project needs a true graph neural network.

A heterogeneous graph is particularly appropriate because the nodes have different types:

```text
customer
card
device
email
address
transaction
```

and edges have different semantics.

---

# 26. Fraud Decision Engine

The fraud subsystem should not return only:

```text
fraud = true/false
```

Instead return:

```json
{
  "risk_score": 0.93,
  "decision": "review",
  "model_probability": 0.89,
  "anomaly_score": 0.77,
  "graph_risk": 0.91,
  "reason_codes": [
    "shared_device_cluster",
    "unusual_transaction_amount"
  ]
}
```

Example policy:

```text
risk < 0.30
    allow

0.30 <= risk < 0.75
    step_up_verification

risk >= 0.75
    manual_review/block
```

Thresholds must be calibrated against business costs rather than chosen arbitrarily.

---

# 27. Preparation Time Prediction

The original concept includes:

```text
Preparation Time Prediction
```

Public taxi datasets do not contain restaurant kitchen preparation time.

Therefore do not claim that taxi trip duration is food preparation time.

Use two phases.

### Phase 1

Model:

```text
request_to_pickup_time
```

using HVFHV request/on-scene/pickup information where available.

This represents fulfillment preparation/arrival delay.

### Phase 2

Introduce a synthetic restaurant-order event layer:

```text
restaurant_id
order_time
items
item_count
cuisine
restaurant_load
staff_capacity
prep_start
prep_complete
```

Generate realistic simulated prep labels conditioned on historical demand.

The architecture can then support:

```text
prep_time_model
```

without misrepresenting the public source data.

---

# 28. Cancellation Prediction

Cancellation is also not cleanly labeled in the NYC TLC trip records.

Build cancellation prediction using an event simulator.

Generate:

```text
request
driver_search
offer
accept/reject
pickup
completion
cancel
```

Then model:

```text
P(cancel | request context)
```

Features:

```text
ETA
price
wait_time
demand
supply
weather
distance
restaurant/zone load
driver availability
```

Later replace simulated labels with real marketplace request data if available.

---

# 29. Decision Engine

The decision engine combines model outputs.

Example:

```text
Demand Forecast
       |
       v
Supply/Demand Ratio
       |
       +---------> Dynamic Pricing
       |
       +---------> Driver Ranking
       |
       +---------> ETA
       |
       +---------> Cancellation Risk
```

A simplified decision function:

```text
score(driver) =
    w1 * ETA_score
  + w2 * completion_probability
  + w3 * driver_performance
  - w4 * pickup_distance
  - w5 * cancellation_risk
```

For pricing:

```text
expected_profit(price) =
    expected_conversion(price)
    * expected_margin(price)
    - expected_operational_cost(price)
```

The decision engine should remain separate from individual ML models.

This allows models to be independently replaced.

---

# 30. Real-Time Architecture

Recommended production-style architecture:

```text
                 Marketplace Events
                        |
                        v
                 Kafka / Redpanda
                        |
                        v
              Stream Processing
              Flink / Spark
                        |
             +----------+----------+
             |                     |
             v                     v
       Online Features        Data Lake
       Redis/Feast            S3/MinIO
             |
             v
        Model Serving
       FastAPI / KServe
             |
       +-----+------+
       |            |
       v            v
 Decision Engine   APIs
       |
       v
 Dashboard
```

For a local project, simplify to:

```text
Kafka/Redpanda
   |
Python consumers
   |
Redis
   |
FastAPI
   |
MLflow
   |
Streamlit
```

Do not introduce Kubernetes before the ML pipeline works locally.

---

# 31. Model Registry

Use MLflow.

Track:

```text
experiment
run_id
dataset_version
feature_version
git_commit
model_version
hyperparameters
metrics
artifacts
```

Example:

```text
eta-lightgbm-v1
eta-catboost-v1
demand-patchtst-v1
demand-tft-v1
ranking-lambdamart-v1
fraud-lightgbm-v1
fraud-isolationforest-v1
```

Use model aliases such as:

```text
candidate
staging
production
champion
challenger
```

---

# 32. Monitoring with Evidently

Use Evidently for data and model monitoring.

Monitor:

```text
feature drift
prediction drift
target drift
missing values
outliers
classification performance
regression performance
```

For ETA:

```text
MAE drift
P90 error drift
prediction distribution
feature drift
```

For fraud:

```text
PR-AUC
recall
precision
fraud rate
risk-score distribution
```

For demand:

```text
WAPE
MAE
forecast bias
forecast distribution
```

For ranking:

```text
NDCG
pickup ETA
completion rate
```

---

# 33. ML Monitoring Architecture

```text
                 Production Predictions
                         |
                         v
                 Prediction Store
                         |
             +-----------+-----------+
             |                       |
             v                       v
         MLflow                 Evidently
       Model Metadata         Drift Reports
             |                       |
             +-----------+-----------+
                         |
                         v
                      Grafana
                         |
                         v
                     Alerts
```

Store predictions with:

```text
prediction_id
model_version
timestamp
entity_id
prediction
actual
latency_ms
request_features_hash
```

Do not store sensitive raw payment information unnecessarily.

---

# 34. Data Drift

Examples:

```text
trip_distance distribution changes
fare distribution changes
zone demand changes
payment-type distribution changes
device distribution changes
```

Use:

```text
PSI
KS test
Jensen-Shannon divergence
Wasserstein distance
```

For categorical features use:

```text
population frequency divergence
```

Drift does not automatically mean model failure.

Drift means:

```text
input distribution changed
```

Performance degradation must be measured separately.

---

# 35. Model Drift

Model drift means predictive performance changes.

Example:

```text
ETA MAE

January: 4.8 minutes
February: 5.1 minutes
March: 7.3 minutes
```

This may indicate:

- traffic pattern change
- construction
- weather regime
- data pipeline change
- feature drift
- marketplace behavior change

Use both drift and performance monitoring.

---

# 36. Latency Monitoring

Every model endpoint should record:

```text
p50 latency
p95 latency
p99 latency
throughput
error rate
timeout rate
```

Example SLO:

```text
ETA API:
p95 < 100 ms

Fraud API:
p95 < 150 ms

Ranking API:
p95 < 200 ms
```

Exact targets should be measured against the chosen serving infrastructure.

---

# 37. API Design

Example:

```text
POST /v1/eta/predict
POST /v1/demand/forecast
POST /v1/ranking/rank
POST /v1/pricing/recommend
POST /v1/fraud/score
GET  /v1/models
GET  /v1/health
GET  /v1/metrics
```

ETA request:

```json
{
  "origin_zone": 161,
  "destination_zone": 237,
  "pickup_time": "2025-12-01T18:30:00",
  "trip_distance": 4.2
}
```

Response:

```json
{
  "eta_seconds": 1034,
  "model_version": "eta-catboost-v1"
}
```

---

# 38. Repository Structure

Recommended repository:

```text
marketplace-ml-platform/
│
├── README.md
├── Instructions.md
├── pyproject.toml
├── docker-compose.yml
├── Makefile
│
├── configs/
│   ├── data.yaml
│   ├── features.yaml
│   ├── models.yaml
│   └── monitoring.yaml
│
├── data/
│   ├── raw/
│   ├── bronze/
│   ├── silver/
│   └── gold/
│
├── src/
│   ├── ingestion/
│   ├── validation/
│   ├── features/
│   ├── models/
│   │   ├── eta/
│   │   ├── demand/
│   │   ├── ranking/
│   │   ├── pricing/
│   │   └── fraud/
│   │
│   ├── serving/
│   ├── decision_engine/
│   ├── simulation/
│   └── monitoring/
│
├── notebooks/
│   ├── 01_data_profiling.ipynb
│   ├── 02_eta_baseline.ipynb
│   ├── 03_demand_baseline.ipynb
│   ├── 04_ranking.ipynb
│   └── 05_fraud.ipynb
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── data/
│
├── dashboards/
│
└── deployment/
    ├── docker/
    └── k8s/
```

---

# 39. Recommended Development Sequence

Do not implement all models simultaneously.

## Phase 1: Data Foundation

Download:

```text
NYC TLC
Chicago Taxi
IEEE-CIS
```

Build:

```text
raw -> bronze -> silver
```

Validate schemas and data quality.

---

## Phase 2: ETA

Implement:

```text
baseline
LightGBM
CatBoost
```

Compare:

```text
MAE
RMSE
P90 error
latency
model size
```

Deploy the best model.

---

## Phase 3: Demand

Implement:

```text
Seasonal Naive
LightGBM
PatchTST
TFT
N-HiTS
```

Start with one zone.

Then:

```text
multi-zone
```

Then:

```text
hierarchical/global forecasting
```

---

## Phase 4: Ranking

Create candidate sets:

```text
request -> nearest N vehicles
```

Build ranking features.

Train LambdaMART.

Evaluate:

```text
NDCG@K
pickup ETA
completion proxy
```

---

## Phase 5: Fraud

Start with:

```text
LightGBM
```

Then add:

```text
Isolation Forest
```

Then:

```text
graph features
```

Finally:

```text
GraphSAGE/GAT
```

if needed.

---

## Phase 6: Pricing

First build:

```text
acceptance model
```

Then:

```text
pricing simulator
```

Then:

```text
contextual bandit
```

Only after the simulator behaves correctly should PPO be attempted.

---

## Phase 7: Monitoring

Integrate:

```text
MLflow
Evidently
Prometheus
Grafana
```

Monitor every production model.

---

# 40. Validation Strategy

The biggest ML mistake in this project would be random train/test splitting.

Marketplace data is temporal.

Use:

```text
Train
  |
  +---- past

Validation
  |
  +---- future

Test
  |
  +---- even later future
```

For example:

```text
January-August  -> Train
September       -> Validation
October         -> Test
```

For long-running experiments:

```text
Rolling Window Evaluation
```

Example:

```text
Train Jan-Mar -> Validate Apr
Train Jan-Apr -> Validate May
Train Jan-May -> Validate Jun
```

This is much closer to production.

---

# 41. Leakage Prevention

Never use:

```text
future trip count
future average fare
future driver performance
future cancellation
future target-derived statistics
```

when generating a prediction.

Bad:

```text
zone_average_duration = mean(all historical + future records)
```

Correct:

```text
zone_average_duration_at_t =
    mean(records where timestamp < t)
```

Use feature-generation timestamps and explicit point-in-time joins.

---

# 42. Experiment Tracking

Every training run should record:

```text
dataset_version
date_range
feature_version
model
hyperparameters
training_duration
CPU/GPU
metrics
git_commit
```

Example:

```text
run:
eta_catboost_2026_08_26_001

dataset:
nyc_tlc_2025

features:
eta_features_v3

MAE:
247 seconds

P90:
612 seconds
```

---

# 43. Model Comparison

Create a model leaderboard.

Example:

```text
Model          MAE     RMSE    P90 Error    Latency
-----------------------------------------------------
Baseline       420     650     880          1 ms
LightGBM       250     390     510          8 ms
CatBoost       245     380     500          14 ms
```

Do not choose the model based only on accuracy.

The production decision should consider:

```text
accuracy
latency
cost
stability
explainability
maintenance
```

---

# 44. Cost Optimization

For initial experiments:

```text
DuckDB + Parquet
```

is sufficient for many analytical operations.

Do not immediately deploy Spark.

Use Spark when:

```text
dataset size
parallel processing
distributed feature generation
```

actually require it.

For deep forecasting:

```text
GPU
```

is useful.

For LightGBM/CatBoost:

```text
CPU
```

is often sufficient for the first iteration.

Use mixed precision for neural forecasting if supported and validated.

---

# 45. Security

The fraud dataset contains security-relevant attributes.

Treat:

```text
device
IP
email
card-related
identity
```

as sensitive.

Do not expose raw identity features through public APIs.

Use:

```text
RBAC
encryption
secret management
network isolation
audit logging
data retention policies
```

For a portfolio project, demonstrate the architecture even if deployed locally.

---

# 46. Observability

Use structured logs.

Example:

```json
{
  "timestamp": "2026-08-26T18:30:00Z",
  "service": "eta-api",
  "model": "catboost",
  "version": "v1",
  "request_id": "abc123",
  "latency_ms": 18,
  "prediction_seconds": 1034
}
```

Use a request ID across:

```text
API
feature service
model
decision engine
```

This makes production debugging much easier.

---

# 47. Final Architecture

The completed platform should look like:

```text
                           DATA SOURCES
                               |
          +--------------------+---------------------+
          |                    |                     |
          v                    v                     v
      NYC TLC              Chicago Taxi          IEEE-CIS
          |                    |                     |
          +--------------------+---------------------+
                               |
                               v
                     Data Lake / Parquet
                               |
                               v
                    Data Validation Layer
                               |
                               v
                     Feature Engineering
                               |
        +----------+-----------+-----------+----------+
        |          |           |           |          |
        v          v           v           v          v
       ETA      Demand      Ranking     Pricing     Fraud
        |          |           |           |          |
   LightGBM/   PatchTST    LambdaMART   Bandit/    GBDT +
   CatBoost     TFT        /Ranking     PPO Sim    Isolation
                N-HiTS                              + Graph
        |          |           |           |          |
        +----------+-----------+-----------+----------+
                               |
                               v
                       Decision Engine
                               |
                +--------------+--------------+
                |                             |
                v                             v
             REST API                    Event Stream
                |                             |
                v                             v
           Dashboard                   Monitoring
                                              |
                                      +-------+-------+
                                      |               |
                                      v               v
                                   MLflow          Evidently
                                      |               |
                                      +-------+-------+
                                              |
                                              v
                                           Alerts
```

---

# 48. What Should Be Real vs Simulated

This distinction should be explicitly documented in the project README.

## Real

Use actual public observations for:

```text
trip timestamps
trip locations
trip distance
trip duration
fare
tips
payment type
vehicle/taxi identity where available
demand counts
fraud labels
transaction features
identity/network features
```

## Simulated

Only simulate fields that public datasets do not provide:

```text
restaurant preparation time
driver acceptance/rejection
driver availability
real-time driver GPS
customer cancellation
price elasticity
counterfactual pricing outcomes
dispatch events
```

The simulator should be calibrated using distributions learned from the real data.

This is significantly more credible than fabricating an entire dataset.

---

# 49. Suggested Final Portfolio Demonstration

The final dashboard should contain:

### Marketplace Overview

```text
Active Requests
Available Drivers
Predicted Demand
Average ETA
Cancellation Risk
Fraud Risk
Current Price Multiplier
```

### ETA

```text
Actual vs Predicted ETA
MAE
P90 Error
Error by Zone
Error by Hour
```

### Demand

```text
Forecast
Actual Demand
Forecast Error
Zone Heatmap
```

### Ranking

```text
Request
Candidate Drivers
Ranking Score
Predicted Pickup ETA
```

### Pricing

```text
Current Demand
Supply
Recommended Multiplier
Expected Conversion
Expected Revenue
```

### Fraud

```text
Risk Score
Fraud Probability
Anomaly Score
Graph Risk
Decision
```

### Monitoring

```text
Feature Drift
Prediction Drift
Model Performance
Latency
Throughput
Model Version
```

---

# 50. Minimum Viable Version

If the project becomes too large, the MVP should contain:

```text
1. NYC TLC data ingestion
2. ETA LightGBM
3. ETA CatBoost
4. Demand LightGBM baseline
5. Demand PatchTST
6. Chicago entity ranking
7. LambdaMART
8. IEEE-CIS fraud LightGBM
9. Isolation Forest
10. MLflow
11. Evidently
12. FastAPI
13. Streamlit dashboard
```

Then add:

```text
TFT
N-HiTS
Graph fraud
Contextual bandit
PPO simulator
Kafka
Redis
Kubernetes
```

in that order.

---

# 51. Final Engineering Recommendation

The strongest implementation is not the one containing the largest number of algorithms. It is the one that demonstrates that each algorithm solves a clearly defined marketplace decision problem and that the entire system remains temporally correct, observable, reproducible, and deployable.

The most important design choices are:

```text
Real public data
        +
Point-in-time features
        +
Temporal validation
        +
Model-specific objectives
        +
Decision engine
        +
Production monitoring
```

Do not present simulated fields as if they were collected by NYC TLC or Chicago. Explicitly label them as simulation-derived.

For the first production-quality milestone, build the complete data and serving pipeline around NYC TLC, use Chicago for persistent entity ranking experiments, and use IEEE-CIS for fraud. Once those three paths are stable, introduce the marketplace simulator for cancellations, driver availability, price elasticity, preparation time, and PPO.

This gives the project a defensible data provenance story while still allowing the architecture to represent a realistic autonomous marketplace intelligence platform.
