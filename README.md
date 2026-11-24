# Delhi air quality pipeline

Work in progress. A scheduled pipeline on Databricks Free Edition that collects PM2.5 readings from Delhi-area air quality monitors and checks the sources against each other.

**Sources:** OpenAQ (which carries government monitors (CPCB/DPCC)), a US government feed and low-cost sensors. Planned: Open-Meteo weather and modelled air quality.

**Layers:** bronze (raw API responses), silver (parsed, de-duplicated, hourly averages in IST).

**Why cross-check sources:** reports have questioned how reliable Delhi's official monitors are. This project measures station completeness and disagreement between nearby stations instead of assuming any one source is right.

## Data quality findings (work in progress)

Based on about two months of hourly PM2.5 from 64 stations and sensors in the Delhi area, as of 19 Nov 2026.

- **Inventory:** OpenAQ lists 97 stations within 25 km of central Delhi. About a third were not reporting on the day checked, and some stations appear under several IDs, with older feeds that stopped years ago. About 40 stations have two PM2.5 sensors, one of them retired, so picking one sensor per station can silently pick the wrong one.
- **Update frequency (early 3-day sample):** government "15-minute" feeds often repeat the same value across several slots. About 48% of consecutive 15-minute readings were identical, so the effective resolution at many stations is closer to hourly.
- **Timestamps:** the OpenAQ archive stamps each reading with the end of its period, while the API gives both start and end. The pipeline aligns the two.
- **Low-cost sensors and humidity:** relative to the median of the government stations in the same hour, 5 of the 7 low-cost sensors read progressively higher as humidity rises (for example 0.80x under 40% humidity to 1.61x above 80% at one sensor). This is consistent with a humidity effect on low-cost sensors but is not proof, since location and season also vary with humidity.
- **Co-located stations (early 3-day sample):** two stations listed at the same coordinates in Pusa reported levels 2 to 3 times apart while rising and falling together.

### Repeated values

About 75% of consecutive 15-minute readings are identical at CPCB stations (DPCC, UPPCB, HSPCB feeds). That is what a feed publishing one value per hour on a 15-minute grid produces (three of every four slots copy the previous one), so the data is effectively hourly for these stations and the pipeline works at hourly resolution.

Checked for stuck sensors: in hourly averages, no station has more than 5.5% identical consecutive hours (most about 2%), so no stations are excluded. Two stations (Rohini, Jahangirpuri) repeat slightly more at 15-minute level (84% and 80%); not explained yet, with no visible effect on hourly values.

### Error codes and impossible values

PM2.5 readings below 0, above 1000, or exactly 999.99 are excluded in the silver step. The data held sensor error codes (9999.9, -9999, 999.99), repeated stuck values (1588, 1786, 1496) and readings in the millions at one Gurugram station. A few hundred rows out of about 730,000, but enough to distort daily averages. Raw data is kept unchanged in bronze. The 1000 limit is a judgement call, not a verified instrument limit.

## Forecast (work in progress)

Task: predict PM2.5 six hours ahead at each station, using readings and weather up to now.

Split: train on Nov 2025 to Jan 2026 and Sep to Oct 2026; validate on Feb 2026; held-out check on Nov 1 to 22, 2026. There is a gap in the data from March to mid-September. A 6-hour gap separates train from evaluation periods.

| | Persistence MAE | Model MAE |
|---|---|---|
| Validation (Feb 2026) | 55.4 | 38.3 |
| Held-out (Nov 2026) | 76.3 | 55.1 |

Known limits: the model under-predicts peaks (mean error about -44 on hours above 150 ug/m3). The November check was looked at while the approach was being developed, so it is not a clean test. A clean test will come from data collected after Nov 22, 2026.

## How it runs

A Databricks job (delhi-air-pm25-refresh) runs every 6 hours with nine tasks:

1. ingest_openaq: last 24 hours of PM2.5 readings from the OpenAQ API into bronze
2. load_archive: rolling 11-day window from the OpenAQ archive (files appear about 72 hours late), skipping files already loaded
3. build_silver: parse and deduplicate readings; the archive version wins over the API version
4. build_silver_hourly: hourly average per sensor
5. ingest_openmeteo: weather and modelled PM2.5 from Open-Meteo into bronze
6. build_silver_openmeteo: parse weather into hourly tables
7. build_gold: join station hours with weather
8. build_features: lag and weather features with the 6-hour-ahead target (training table)
9. log_forecasts: frozen model (trained on data before Nov 1, 2026) predicts 6 hours ahead for the newest hours and appends to forecast_log

The frozen model means the logged forecasts form a clean test: nothing in them was seen during training or tuning. They will be scored against the readings that arrive later in time.