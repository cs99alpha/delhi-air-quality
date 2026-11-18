# delhi-air-quality
# Delhi air quality pipeline

Work in progress. A scheduled pipeline on Databricks Free Edition that collects PM2.5 readings from Delhi-area air quality monitors and checks the sources against each other.

**Sources:** OpenAQ, which carries government monitors (CPCB/DPCC), a US government feed and low-cost sensors. Planned: Open-Meteo weather and modelled air quality.

**Layers:** bronze (raw API responses), silver (parsed, de-duplicated, hourly averages in IST).

**Why cross-check sources:** reports have questioned how reliable Delhi's official monitors are. This project measures station completeness and disagreement between nearby stations instead of assuming any one source is right.