# delhi-air-quality
# Delhi air quality pipeline

Work in progress. A scheduled pipeline on Databricks Free Edition that collects PM2.5 readings from Delhi-area air quality monitors and checks the sources against each other.

**Sources:** OpenAQ, which carries government monitors (CPCB/DPCC), a US government feed and low-cost sensors. Planned: Open-Meteo weather and modelled air quality.

**Layers:** bronze (raw API responses), silver (parsed, de-duplicated, hourly averages in IST).

**Why cross-check sources:** reports have questioned how reliable Delhi's official monitors are. This project measures station completeness and disagreement between nearby stations instead of assuming any one source is right.

## Data quality findings (work in progress)

Based on about two months of hourly PM2.5 from 64 stations and sensors in the Delhi area, as of 19 Nov 2026.

- **Inventory:** OpenAQ lists 97 stations within 25 km of central Delhi. About a third were not reporting on the day checked, and some stations appear under several IDs, with older feeds that stopped years ago. About 40 stations have two PM2.5 sensors, one of them retired, so picking one sensor per station can silently pick the wrong one.
- **Update frequency (early 3-day sample):** government "15-minute" feeds often repeat the same value across several slots. About 48% of consecutive 15-minute readings were identical, so the effective resolution at many stations is closer to hourly.
- **Timestamps:** the OpenAQ archive stamps each reading with the end of its period, while the API gives both start and end. The pipeline aligns the two.
- **Low-cost sensors and humidity:** relative to the median of the government stations in the same hour, 5 of the 7 low-cost sensors read progressively higher as humidity rises (for example 0.80x under 40% humidity to 1.61x above 80% at one sensor). This is consistent with a humidity effect on low-cost sensors but is not proof, since location and season also vary with humidity.
- **Co-located stations (early 3-day sample):** two stations listed at the same coordinates in Pusa reported levels 2 to 3 times apart while rising and falling together.

## Forecast (work in progress)

Task: predict PM2.5 six hours ahead at each station, using readings and weather up to now.

Split: train on Nov 2025 to Jan 2026 and Sep to Oct 2026; validate on Feb 2026; held-out check on Nov 1 to 22, 2026. There is a gap in the data from March to mid-September. A 6-hour gap separates train from evaluation periods.

| | Persistence MAE | Model MAE |
|---|---|---|
| Validation (Feb 2026) | 55.4 | 38.3 |
| Held-out (Nov 2026) | 76.3 | 55.1 |

Known limits: the model under-predicts peaks (mean error about -44 on hours above 150 ug/m3). The November check was looked at while the approach was being developed, so it is not a clean test. A clean test will come from data collected after Nov 22, 2026.