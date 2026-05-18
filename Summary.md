# Jego EV Fleet Telematics Analysis Report
**Dataset:** 8,640 records across 3 vehicles (24 hours, 30-second intervals)  
**Location:** Lagos, Nigeria — VI–Lekki, Lagos Island–Mainland, Surulere–Ikeja corridors

---

## 1. Introduction

I analyzed telematics data from a three-vehicle electric fleet (JEGO-V001, JEGO-V002, JEGO-V003) operating in Lagos over 24 hours. The raw data came in as JSON with GPS coordinates, speed, battery level (SoC), and motor current readings every 30 seconds.

I built a full pipeline to answer four questions:

1. **Is the data reliable?** — Clean duplicates, sensor errors, and GPS gaps.
2. **How far did vehicles actually travel?** — Compare GPS distance vs. speedometer distance.
3. **What's the carbon impact?** — How much CO₂ did these EVs save vs. petrol vehicles?
4. **Are there hidden problems?** — Detect abnormal motor or battery behavior.

### Quick Results

- Cleaned 3 duplicates, 3 sensor errors, and 40 GPS gaps from the raw data.
- Fleet traveled **1,845 km** in 24 hours, saving **323.73 kg of CO₂** vs. petrol equivalents.
- **JEGO-V002** was the most efficient vehicle.
- I found **40 anomalous events** — mostly routine, but a few worth investigating.

---

## 2. Methodology

### 2.1 Data Cleaning

The raw dataset had 8,643 records. I cleaned it down to 8,640 valid records:

| Problem | What I Did | How Many |
|---------|------------|----------|
| Duplicate records | Removed duplicate (vehicle, timestamp) pairs | 3 removed |
| SoC sensor errors | Set bad readings to NaN and filled gaps using surrounding values | 3 fixed |
| GPS signal loss | Interpolated missing coordinates during gap events | 40 filled |

**My assumption:** 30 seconds is short enough that battery level and location change roughly in a straight line between readings. This isn't perfect, but it's the best we can do with the data we have.

---

### 2.2 Route Mapping & Distance

I compared two ways to measure distance:

1. **GPS Distance (Haversine)** — Straight-line distance between consecutive GPS points.
2. **Speedometer Distance** — Speed × time for each 30-second interval.

| Vehicle | Trip | Speed Distance (km) | GPS Distance (km) | Difference |
|---------|------|---------------------|-------------------|------------|
| JEGO-V001 | T01 | 28.57 | 2.00 | 14.3× |
| JEGO-V001 | T02 | 39.01 | 2.68 | 14.6× |
| JEGO-V001 | T03 | 32.73 | 2.51 | 13.0× |
| JEGO-V001 | T04 | 33.48 | 2.33 | 14.4× |
| JEGO-V001 | T05 | 36.49 | 2.43 | 15.0× |

Lagos traffic is stop-and-go. In 30 seconds, a vehicle might barely move — so GPS points cluster together and straight-line distance massively underestimates actual travel. The speedometer captures every inch of movement, even in traffic.

**I used speed-based distance for all calculations** because it reflects reality.

---

### 2.3 Carbon & Efficiency

I calculated energy use from battery level changes:

- **Energy consumed** = change in SoC × 26.5 kWh (battery capacity)
- **EV emissions** = energy used × 0.430 kg CO₂/kWh (Nigeria's grid)
- **Petrol equivalent** = distance × 0.192 kg CO₂/km
- **CO₂ saved** = petrol emissions − EV emissions

| Vehicle | Distance (km) | Energy (kWh) | EV CO₂ (kg) | Petrol CO₂ (kg) | CO₂ Saved (kg) | Efficiency |
|---------|---------------|--------------|-------------|-----------------|-----------------|------------|
| JEGO-V001 | 609.92 | 24.37 | 10.48 | 117.10 | 106.63 | 96.1% |
| JEGO-V002 | 617.12 | 22.94 | 9.86 | 118.49 | 108.63 | 103.3% |
| JEGO-V003 | 617.73 | 23.58 | 10.14 | 118.60 | 108.47 | 100.6% |
| **Fleet** | **1,844.77** | **70.89** | **30.48** | **354.19** | **323.73** | **100.0%** |

**Key finding:** The EV fleet produced only **8.6%** of the CO₂ that petrol vehicles would have emitted. 

---

### 2.4 Anomaly Detection

I trained an **Isolation Forest** model to flag unusual motor behavior. 

**Features I used:**
- `motor_current_amps` — how hard the motor is working
- `current_speed_ratio` — current divided by speed (catches strain at low speeds)
- `soc_pct` — battery level for context

**How I validated it:**
- Split data 80/20 (train/test)
- Normalized features with `StandardScaler`
- Train anomaly rate: **1.00%** | Test anomaly rate: **1.71%**

The small gap between train and test tells me the model generalizes reasonably well, though the test rate is slightly higher than expected.

Instead of arbitrary thresholds, I used percentiles from the anomaly data itself:
- **High Load** — `current_speed_ratio` in the top 10% (motor struggling relative to speed)
- **Battery Drain** — `motor_current_amps` in the top 10% (drawing power fast)
- **Default** — everything else the model flagged

| Alert Type | Count | What It Means |
|------------|-------|---------------|
| Default | 32 | Unusual but not critical |
| Battery Drain | 4 | High current draw — check electrical systems |
| High Load | 4 | Motor working hard — inspect drivetrain or route |

**By vehicle:**

| Vehicle | High Load | Battery Drain | Default | Total |
|---------|-----------|---------------|---------|-------|
| JEGO-V001 | 1 | 3 | 10 | 14 |
| JEGO-V002 | 1 | 0 | 16 | 17 |
| JEGO-V003 | 2 | 1 | 6 | 9 |

**What I noticed:**
- **JEGO-V001** had the most battery drain events (3) — worth checking its electrical system.
- **JEGO-V003** had the most high-load alerts relative to its total — its route (Surulere–Ikeja) might be more demanding.
- **JEGO-V002** had the most total anomalies but nearly all were default — just normal variance, nothing alarming.

An interactive operator dashboard (`jego_operator_dashboard.html`) visualizes all 40 anomalies with filterable alerts by vehicle and severity, providing maintenance teams with actionable guidance.

**Important assumption:** I treated all three vehicles as identical (same model, battery, condition). This lets me use one model for the whole fleet. If they're actually different vehicles, I'd need to train separate models for each one.

---

## 3. Conclusion

### The Good

- The fleet is well-maintained. Only **0.46%** of readings were anomalous.
- Electric vehicles in Lagos save **323 kg of CO₂ per day** vs. petrol — that's over **117 tonnes per year** for just 3 vehicles.
- Speed-based distance is the only reliable way to measure travel in stop-and-go traffic. GPS distance underestimates by ~14×.

### The Limitations

1. **Map matching isn't perfect** — When GPS drifts more than 50 meters, or roads aren't on OpenStreetMap (common in Lagos industrial areas), routes look wrong. New construction shows up as straight-line artifacts.

2. **Regenerative braking is invisible** — Even during confirmed braking events, the battery level still shows net consumption over 30 seconds. Why? The sampling window is too slow to catch short regen bursts, and constant loads (AC, lights, electronics) drown them out.

3. **Energy numbers look unrealistically low** — The fleet averages ~3.8 kWh/100km. Real EVs use 15–20 kWh/100km. This could be due to the fact that the dataset being worked with is a mock dataset and not real EV data.

4. **One day of data isn't enough** — The anomaly model was trained on a single day. It will drift as seasons change, temperatures shift, and drivers vary. It needs regular retraining to stay accurate.

5. **One model for all vehicles** — I assumed all three vehicles are identical. If they differ in motor specs, age, or typical payload, the shared model might miss vehicle-specific issues or flag normal behavior as abnormal.

### What I'd Do Next

| Next Step | Why |
|-----------|-----|
| Real-time pipeline | Move from batch analysis to live streaming (Kafka → Spark) so alerts hit the dashboard within seconds |
| Driver notifications | Wire anomaly detection to SMS or PagerDuty so drivers get alerts within 2 minutes |
| Weekly model retraining | Auto-retrain the anomaly model every week to account for changing conditions |
| Per-vehicle models | If the fleet grows or vehicles differ, train individual models for each one |

---

## Appendix: Technical Details

- **Tools:** Python, Pandas, NumPy, Scikit-learn, Folium, OSMnx
- **Earth radius:** 6,371 km (Haversine formula)
- **Battery capacity:** 26.5 kWh
- **Nigeria grid emission factor:** 0.430 kg CO₂/kWh
- **Petrol vehicle emission factor:** 0.192 kg CO₂/km
