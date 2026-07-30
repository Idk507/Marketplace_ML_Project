# Advanced Marketplace ML Dataset Blueprint

This document defines the exact datasets, joins and feature engineering strategy for the 10 marketplace ML projects.

## Common Data Lake

| Layer | Dataset | Official Source | Format | Update |
|---|---|---|---|---|
| Trips | NYC TLC Trip Records | https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page | Parquet | Monthly |
| Zones | TLC Taxi Zone Lookup | https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page | CSV | Rare |
| Road Network | Geofabrik OpenStreetMap | https://download.geofabrik.de | PBF | Regular |
| Weather | Open-Meteo | https://open-meteo.com | JSON | Live |
| H3 | Uber H3 | https://h3geo.org | Library | Stable |
| Routing | OSRM | https://project-osrm.org | API | Live |
| Restaurants | Yelp Open Dataset | https://business.yelp.com/data/resources/open-dataset | JSON | Periodic |
| Places | Foursquare Open Data | https://location.foursquare.com/products/open-data | CSV | Periodic |
| Traffic | TomTom Traffic API | https://developer.tomtom.com/traffic-api | JSON | Live |
| Holidays | Nager.Date | https://date.nager.at | JSON | Annual |

## Project Mapping

### 1. Marketplace Intelligence
Datasets:
- NYC Yellow Taxi
- NYC Green Taxi
- NYC High Volume FHV
- Taxi Zone Lookup
- Open-Meteo
- TomTom
- H3
- OSM

Join:
Trip -> Zone -> H3 -> Weather -> Traffic -> Holiday

Models:
ETA, Demand Forecasting, Pricing, Dispatch, Fraud.

### 2. Driver Dispatch
Primary: High Volume FHV
Supporting: OSM, H3, OSRM

### 3. Demand Forecasting
Aggregate TLC trips into 15-minute H3 cells.
Supporting: Weather + Holidays.

### 4. Recommendation
Primary: Yelp (business.json, review.json, user.json, tip.json)
Supporting: Foursquare.

### 5. Dynamic Pricing
Primary: NYC TLC
Supporting: Weather, Traffic, Holidays.

### 6. Search Ranking
Primary: Yelp.
Generate click labels for learning-to-rank.

### 7. Fraud Detection
Primary: NYC TLC.
Inject synthetic fraud scenarios.

### 8. Fleet Optimization
Primary: High Volume FHV.
Supporting: OSM, OSRM, H3.

### 9. Multi-objective Optimization
Primary: Chicago Taxi.
Supporting: Weather and Routing.

### 10. Urban Mobility Digital Twin
Primary: SUMO + NYC TLC + OSM + Open-Meteo.

## Recommended Download Order

1. NYC TLC
2. Taxi Zone Lookup
3. Geofabrik OSM
4. Open-Meteo
5. Uber H3
6. OSRM
7. Yelp
8. Foursquare
9. TomTom
10. SUMO
