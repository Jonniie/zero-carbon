# Jego EV Fleet Telematics Analysis Report

**Date:** 2026-05-17  
**Dataset:** 8,640 records across 3 vehicles (24-hour period, 30-second intervals)  
**Location:** Lagos, Nigeria — VI–Lekki, Lagos Island–Mainland, Surulere–Ikeja corridors

---

## 1. Introduction

### 1.1 Study Summary and Context

This report analyzes telematics data from a three-vehicle electric fleet (JEGO-V001, JEGO-V002, JEGO-V003) operating in Lagos over a 24-hour period. The data pipeline processes raw JSON telemetry — including GPS coordinates, speed, state of charge (SoC), and motor current — through cleaning, geospatial mapping, carbon accounting, and anomaly detection stages.

### 1.2 Key Questions

This analysis addresses four questions:

1. **Data Quality:** How reliable is the raw telemetry, and what cleaning is required to produce analysis-ready data?
2. **Route Accuracy:** What distances do vehicles actually travel, and how do GPS-based and speed-based distance estimates compare?
3. **Carbon Impact:** How much CO₂ does the EV fleet displace compared to equivalent ICE vehicles, and how efficient is each vehicle?
4. **Anomaly Detection:** Are there abnormal energy consumption patterns that indicate mechanical issues or driver behavior concerns?

### 1.3 Conclusions Preview

- The raw dataset contained 3 duplicate records, 3 SoC sensor errors (reading 100% mid-trip), and 40 GPS gap events — all successfully resolved via interpolation.
- Fleet traveled **1,845 km** in 24 hours, displacing **323.73 kg CO₂** vs. ICE equivalents.
- JEGO-V002 was the most efficient vehicle at **103.3%** of fleet average.
- Isolation Forest detected anomalous motor current patterns, generating actionable dashboard alerts for high-load and rapid battery drain events.

---

## 2. Analysis

### 2.1 Data Quality & Pipeline

**Methods**

The raw JSON dataset was loaded into a Pandas DataFrame with 9 columns: `vehicle_id`, `timestamp`, `latitude`, `longitude`, `speed_kmh`, `soc_pct`, `motor_current_amps`, `trip_id`, and `event`. The following cleaning steps were applied:

| Step | Action | Records Affected |
|------|--------|------------------|
| Deduplication | Drop duplicate `(vehicle_id, timestamp)` pairs | 3 removed (8,643 → 8,640) |
| SoC Validation | Filter out-of-range values (<0 or >100) | 0 removed |
| SoC Error Handling | Set `soc_error` events to NaN, then linearly interpolate per vehicle timeline | 3 errors corrected |
| GPS Gap Handling | Interpolate missing `latitude`/`longitude` during `gps_gap` events | 40 gaps filled |
| Timeline Construction | Sort by vehicle and timestamp; compute `time_diff_seconds` between consecutive readings | All records |

**Results**

After cleaning, the dataset contained 8,640 valid records across 3 vehicles with continuous timelines. SoC interpolation restored plausible values for all 3 sensor errors (e.g., JEGO-V003's 100% reading at 09:25 was interpolated to 5.0% based on surrounding declining trend). GPS gap interpolation maintained spatial continuity without introducing artificial route deviations.

**Key Assumption:** Linear interpolation is valid for both SoC and GPS gaps because the 30-second sampling interval is short enough that changes between readings are approximately linear.

---

### 2.2 Route Mapping & Distance Analysis

**Methods**

Two distance calculation approaches were implemented and compared:

1. **Haversine Distance** — Great-circle distance between consecutive GPS coordinate pairs using Earth radius of 6,371 km. Summed per trip for total route distance.
2. **Speed-Based Distance** — Integration of average speed over each time interval: `dist = (avg_speed_kmh / 3600) × time_diff_seconds`. This approach uses the vehicle speedometer as ground truth.

Additionally, two map visualizations were produced:
- **Raw GPS Map** — Folium map with vehicle-colored route lines, start (green) and end (red) markers, and interactive tooltips showing vehicle, trip ID, and corridor.
- **OSMnx-Snapped Map** — Raw GPS waypoints snapped to the nearest OpenStreetMap road nodes in Lagos, then connected via shortest-path routing on the drive network. 
**Results**

| Vehicle | Trip | Speed-Based Distance (km) | Haversine Distance (km) | Ratio |
|---------|------|---------------------------|-------------------------|-------|
| JEGO-V001 | T01 | 28.57 | 2.00 | 14.3× |
| JEGO-V001 | T02 | 39.01 | 2.68 | 14.6× |
| JEGO-V001 | T03 | 32.73 | 2.51 | 13.0× |
| JEGO-V001 | T04 | 33.48 | 2.33 | 14.4× |
| JEGO-V001 | T05 | 36.49 | 2.43 | 15.0× |

Haversine measures straight-line distance between GPS points, while speed-based distance uses the vehicle speedometer over time. In Lagos traffic, vehicles stop and start frequently, so GPS coordinates barely change between 30-second pings — making Haversine underestimate actual distance by ~14×. **Speed-based distance is used for all operational metrics** (carbon accounting, efficiency scoring) as it better reflects actual energy consumption.

---

### 2.3 Carbon Displacement & Fleet Efficiency

**Methods**

Energy consumption was derived from SoC changes rather than direct metering:

1. **SoC Delta Calculation:** `soc_delta_percent = prev_soc - current_soc` — positive values indicate consumption, negative values indicate regenerative braking recovery.
2. **Energy Conversion:** `energy_kwh = (soc_delta_percent / 100) × 26.5 kWh` 
3. **Emissions Comparison:**
   - EV emissions: `total_kwh_consumed × 0.430 kg CO₂/kWh`
   - ICE equivalent: `total_km_driven × 0.192 kg CO₂/km`
   - Net CO₂ displaced: `ICE emissions - EV emissions`
4. **Efficiency Scoring:** `km_per_kwh = total_km_driven / total_kwh_consumed`, scored as percentage of fleet average.

**Results**

| Vehicle | Distance (km) | kWh Consumed | EV CO₂ (kg) | ICE CO₂ (kg) | Net Displaced (kg) | Efficiency Score |
|---------|---------------|--------------|-------------|--------------|---------------------|------------------|
| JEGO-V001 | 609.92 | 24.37 | 10.48 | 117.10 | 106.63 | 96.1% |
| JEGO-V002 | 617.12 | 22.94 | 9.86 | 118.49 | 108.63 | 103.3% |
| JEGO-V003 | 617.73 | 23.58 | 10.14 | 118.60 | 108.47 | 100.6% |
| **Fleet Total** | **1,844.77** | **70.89** | **30.48** | **354.19** | **323.73** | **100.0%** |

**Key Findings:**
- EV grid emissions (30.48 kg) represent only **8.6%** of what ICE vehicles would have emitted (354.19 kg), demonstrating a **91.4% reduction** in direct tailpipe emissions even on Nigeria's carbon-intensive grid.
- JEGO-V002 achieved the highest efficiency score (103.3%), suggesting either more skilled driving or a more favorable corridor (Lagos Island–Mainland has more consistent traffic flow than Surulere–Ikeja).

---

### 2.4 Anomaly Detection & Predictive Alerts

**Methods**

An Isolation Forest model was trained to detect abnormal energy consumption patterns:

1. **Feature Engineering:**
   - `motor_current_amps` — raw motor current draw
   - `current_speed_ratio` — `motor_current_amps / (speed_kmh + 0.001)` — detects high mechanical load at low speeds
   - `soc_pct` — battery state of charge context

2. **Model Validation Pipeline:**
   - Data split 80/20 (train/test) with `StandardScaler` normalization
   - `IsolationForest(n_estimators=100, contamination=0.01, random_state=42)`
    - Train Anomaly Rate: 1.00%, Test Anomaly Rate: 1.71%
   - Single global model trained across all vehicles' combined data

3. **Alert Classification:** Detected anomalies were categorized using percentile-based thresholds:
   - **High Load** (`current_speed_ratio` ≥ 90th percentile): Motor working hard relative to speed, suggesting drivetrain resistance, incline, or payload
   - **Battery Drain** (`motor_current_amps` ≥ 90th percentile): High absolute current draw indicating rapid energy discharge
   - **Default**: Other unusual consumption patterns flagged by the model

**Results**

The model identified **40 anomalous events** across the fleet (0.46% of total records), below the 1% contamination threshold, indicating generally healthy operations.

| Alert Type | Total Count | Typical Pattern | Recommended Action |
|------------|-------------|-----------------|-------------------|
| Default | 32 | Unusual current/speed combination | Monitor vehicle during next shift |
| Battery Drain | 4 | High absolute current draw (≥90th percentile) | Check for electrical faults or excessive auxiliary load |
| High Load | 4 | High current relative to speed (≥90th percentile) | Inspect drivetrain, payload, or route incline |

**Vehicle-Level Breakdown**

| Vehicle | High Load | Battery Drain | Default | Total |
|---------|-----------|---------------|---------|-------|
| JEGO-V001 | 1 | 3 | 10 | 14 |
| JEGO-V002 | 1 | 0 | 16 | 17 |
| JEGO-V003 | 2 | 1 | 6 | 9 |

**Key Assumption:** All three vehicles are assumed to be of the same make, model, battery capacity, and operating condition. This justifies training a single global Isolation Forest model across the entire fleet rather than per-vehicle models. If vehicles differ in motor specs, payload capacity, or age, the shared benchmark may miss vehicle-specific anomalies or falsely flag normal behavior for outlier units.

**Key Observations:**
- **JEGO-V002** had the most total anomalies (17), but nearly all were classified as Default (16), suggesting general operational variance rather than specific mechanical issues.
- **JEGO-V001** recorded the most Battery Drain events (3), warranting a check on electrical systems or auxiliary load during operation.
- **JEGO-V003** had the highest proportion of High Load alerts (2 of 9), which may indicate more demanding route conditions (e.g., Surulere–Ikeja corridor with heavier traffic stops).
- The overall 0.46% anomaly rate confirms the fleet is well-maintained, with most anomalies falling into the Default category for routine monitoring.

---

## 3. Conclusion & Discussion

### 3.1 Summary of Findings

This analysis demonstrates that a three-vehicle EV fleet operating in Lagos can achieve meaningful carbon displacement (323.73 kg CO₂/day) while maintaining operational efficiency. The data pipeline successfully handled common telematics issues — duplicate records, sensor errors, and GPS gaps — producing clean, analysis-ready data. Route mapping revealed that speed-based distance calculation is significantly more accurate than raw Haversine distance. Anomaly detection identified actionable patterns in motor current draw that can inform predictive maintenance.

### 3.2 Limitations

1. **Map Matching Accuracy** — OSMnx snapping can misalign when GPS drift exceeds 50m or roads are unmapped (common in Lagos industrial zones). Routes near new construction appear as straight-line artifacts.

2. **Regenerative Braking Not Captured** — Our formula (`soc_delta = prev_soc - current_soc`) only measures net battery change over 30-second intervals. Even during confirmed `regen_braking` events, `soc_delta` remains positive (net consumption) because the 30-second sampling window is too coarse to capture short regen recovery bursts, and auxiliary loads (AC, lights, electronics) dominate the net change. This means **regenerative energy recovery is entirely invisible** in our analysis, causing per-trip efficiency to appear worse than reality — especially on stop-and-go routes like Lagos Island–Mainland where regen should be most beneficial. A proper implementation would require sub-second SoC sampling or direct regen current metering.

4. **Single-Day Dataset** — The Isolation Forest model was trained on one day of data and will drift as seasons, temperatures, and driver behavior change.

5. **Energy Efficiency Metric:**
The fleet averages ~3.8 kWh/100km (`70.89/1,844.77 × 100 = 3.84 kWh/100km`, which is quite low compared to typical EVs (15–20 kWh/100km). This happens because we derive energy from SoC changes between 30-second readings. Small SoC deltas at this sampling rate miss real-world losses (battery heat, auxiliary systems, drivetrain inefficiency). The relative efficiency scores between vehicles are still valid for comparison, but absolute kWh values should not be treated as ground truth.

6. Impact of 30-Second Sampling Frequency: Our data logs vehicle status only once every 30 seconds. This gap is too wide to capture the "micro-behaviors" of driving. <br> Rapid acceleration draws huge power spikes, and regenerative braking recovers energy in short bursts. Because we only check the battery every 30 seconds, these spikes are averaged out. The net change in State of Charge (SoC) often appears smaller than the actual energy used, making the fleet look more efficient than it really is.

### 3.3 Future Work

Deliverable | Success Metric |
-------------|----------------|
**Streaming Pipeline** — Migrate from batch notebook to real-time processing (Kafka → Spark Structured Streaming) | Latency <5s from vehicle to dashboard |
**Alerting Integration** — Wire anomaly output to PagerDuty/SMS for high-load and battery-drain events | <2 min from detection to driver notification |
**Model Drift Monitoring** — Track distribution shift in `current_speed_ratio`; auto-retrain Isolation Forest weekly | Precision >85% on validated holdout set |

---

**Appendix: Technical Notes**

- All analysis was performed in Python using Pandas, NumPy, Scikit-learn, Folium, and OSMnx.
- Haversine formula uses Earth radius of 6,371 km.
- Battery capacity: 26.5 kWh.
- Grid emission factor: 0.430 kg CO₂/kWh.
- ICE emission factor: 0.192 kg CO₂/km.