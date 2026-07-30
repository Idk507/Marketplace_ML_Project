

---

# 1. Autonomous Marketplace Intelligence Platform

This project requires multiple datasets rather than a single dataset.

| Component  | Dataset                        | Why                        |
| ---------- | ------------------------------ | -------------------------- |
| Orders     | NYC Taxi Trip Records          | Millions of real trips     |
| Drivers    | NYC FHV Dataset                | Ride-hailing style data    |
| Weather    | Open-Meteo API                 | Live weather               |
| Maps       | OpenStreetMap                  | Road network               |
| Geospatial | Uber H3                        | Hexagonal spatial indexing |
| Holidays   | Nager.Date API                 | Public holidays            |
| Traffic    | TomTom Traffic API (free tier) | Live congestion            |

### Primary Dataset

**NYC Taxi & Limousine Commission (Official)**

Official Open Data

* [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page?utm_source=chatgpt.com)
* [NYC Open Data Taxi Portal](https://www.nyc.gov/site/tlc/about/raw-data.page?utm_source=chatgpt.com)

This dataset contains billions of real taxi trips and is continuously updated. ([NYC Government][1])

---

# 2. Intelligent Driver Dispatch System

### Dataset

Official NYC FHV (For-Hire Vehicle)

Contains

* pickup
* dropoff
* request time
* dispatch time
* trip duration
* distance

Official Source

* [NYC FHV Trip Records](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page?utm_source=chatgpt.com)

Additional Dataset

Chicago Taxi

* [Chicago Taxi Trips Open Data](https://data.cityofchicago.org/Transportation/Taxi-Trips-2013-2023-/wrvz-psew?utm_source=chatgpt.com)

Chicago's portal is openly available and regularly maintained. ([Chicago Data Portal][2])

---

# 3. Spatio-Temporal Demand Forecasting

Use

### NYC Taxi

Official

Millions of Trips

*

### OpenStreetMap

Road Network

* [OpenStreetMap Data Export](https://download.geofabrik.de/?utm_source=chatgpt.com)

-

### Uber H3

Hexagonal Grid System

* [Uber H3 Documentation](https://h3geo.org/?utm_source=chatgpt.com)

-

### Weather

* [Open-Meteo API](https://open-meteo.com/?utm_source=chatgpt.com)

This combination closely resembles the data inputs used in real mobility forecasting systems.

---

# 4. Personalized Food Recommendation Engine

Since Swiggy and Zomato data are proprietary, you can recreate the recommendation pipeline using open restaurant datasets.

Recommended datasets include:

### Yelp Open Dataset

* [Yelp Open Dataset](https://business.yelp.com/data/resources/open-dataset/?utm_source=chatgpt.com)

Contains

* Restaurants
* Ratings
* Reviews
* Categories
* Users

### OpenStreetMap

Restaurant locations

### Foursquare Open Places

* [Foursquare Open Data](https://location.foursquare.com/products/open-data?utm_source=chatgpt.com)

---

# 5. Dynamic Pricing Engine

Combine

NYC Taxi

*

Weather

*

Traffic

*

Holiday Calendar

Datasets

Weather

* [Open-Meteo API](https://open-meteo.com/?utm_source=chatgpt.com)

Traffic

* [TomTom Traffic API](https://developer.tomtom.com/traffic-api?utm_source=chatgpt.com)

Holiday

* [Nager.Date Public Holidays API](https://date.nager.at/?utm_source=chatgpt.com)

Events

* [Ticketmaster Discovery API](https://developer.ticketmaster.com/products-and-docs/apis/discovery-api/v2/?utm_source=chatgpt.com)

---

# 6. Restaurant Search Ranking

Recommended

Yelp

*

Foursquare

*

OpenStreetMap

*

Review embeddings

Official sources:

* [Yelp Open Dataset](https://business.yelp.com/data/resources/open-dataset/?utm_source=chatgpt.com)
* [Foursquare Open Data](https://location.foursquare.com/products/open-data?utm_source=chatgpt.com)

---

# 7. Marketplace Fraud Detection

There isn't a public "Swiggy fraud" dataset.

Instead

Generate fraud using simulation.

Base dataset

NYC Taxi

Inject

* fake GPS
* fake riders
* duplicate coupons
* abnormal routes
* impossible speeds

Additional graph datasets

IEEE Fraud Detection

* [IEEE-CIS Fraud Detection (Kaggle)](https://www.kaggle.com/c/ieee-fraud-detection?utm_source=chatgpt.com)

Credit Card Fraud

* [Credit Card Fraud Dataset](https://www.kaggle.com/datasets/mlg-ulb/creditcardfraud?utm_source=chatgpt.com)

---

# 8. Fleet Optimization

Datasets

NYC Taxi

*

OpenStreetMap

*

Weather

*

H3

*

Traffic

Road Graph

OpenStreetMap

Routing

* [Open Source Routing Machine (OSRM)](https://project-osrm.org/?utm_source=chatgpt.com)

---

# 9. Multi-objective Delivery Optimization

Datasets

NYC Taxi

Chicago Taxi

Weather

Traffic

Fuel

Elevation

OpenStreetMap

Fuel Prices

* [U.S. Energy Information Administration Data](https://www.eia.gov/opendata/?utm_source=chatgpt.com)

---

# 10. Urban Mobility Digital Twin

Use

### SUMO

Traffic Simulator

* [SUMO Simulation of Urban Mobility](https://sumo.dlr.de/?utm_source=chatgpt.com)

Maps

OpenStreetMap

Demand

NYC Taxi

Weather

Open-Meteo

Routing

OSRM

Traffic

TomTom

This allows you to simulate a realistic city-scale mobility system.

---

# Live APIs to Integrate

These APIs make your projects feel production-ready by providing continuously updated context.

| API                                                                                                                              | Use Case         |
| -------------------------------------------------------------------------------------------------------------------------------- | ---------------- |
| [Open-Meteo](https://open-meteo.com/?utm_source=chatgpt.com)                                                                     | Weather          |
| [OpenStreetMap / Geofabrik](https://download.geofabrik.de/?utm_source=chatgpt.com)                                               | Road Network     |
| [OSRM](https://project-osrm.org/?utm_source=chatgpt.com)                                                                         | Route Estimation |
| [Uber H3](https://h3geo.org/?utm_source=chatgpt.com)                                                                             | Spatial Indexing |
| [Nager.Date](https://date.nager.at/?utm_source=chatgpt.com)                                                                      | Holiday Features |
| [TomTom Traffic API](https://developer.tomtom.com/traffic-api?utm_source=chatgpt.com)                                            | Live Traffic     |
| [Ticketmaster Discovery API](https://developer.ticketmaster.com/products-and-docs/apis/discovery-api/v2/?utm_source=chatgpt.com) | Local Events     |

---

## The dataset combination I'd recommend

For the most realistic portfolio, use a unified data lake built from these sources:

* **Primary transactional data:** NYC TLC Trip Records (billions of trips)
* **Secondary validation:** Chicago Taxi Trips
* **Road network:** OpenStreetMap
* **Spatial indexing:** Uber H3
* **Weather:** Open-Meteo
* **Traffic:** TomTom Traffic API
* **Routing:** OSRM
* **Restaurant metadata:** Yelp Open Dataset
* **Points of interest:** Foursquare Open Data
* **Calendar features:** Nager.Date

This combination gives you historical, streaming, geospatial, environmental, and business-context data similar to what large mobility and delivery companies integrate internally, while remaining openly accessible or available through free tiers. ([NYC Government][1])

Since these datasets and APIs evolve over time, I can also prepare a **master dataset catalog** in Markdown that includes, for every source, the schema, update frequency, estimated size, licensing, API examples, and exactly which ML features each project should derive from it.

[1]: https://www.nyc.gov/site/tlc/about/raw-data.page?utm_source=chatgpt.com "Raw Data - TLC"
[2]: https://data.cityofchicago.org/Transportation/Taxi-Trips-2013-2023-/wrvz-psew?utm_source=chatgpt.com "Taxi Trips (2013-2023) | City of Chicago | Data Portal"
