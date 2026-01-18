# Virtual Home Energy Management System (vHEMS)

## 1. Scope and Location
- **Primary market:** Netherlands
- **Default timezone:** Europe/Amsterdam
- **Default currency:** EUR

This document defines a vHEMS that can evaluate and control (where possible) residential energy assets (PV, battery, EV, heat pump), and can also **simulate** alternative asset sizes and configurations using a combination of **real smart-meter (P1) data** and **virtual assets**.

Key requirement: support **mixed mode** so you can:
- Read **real P1** consumption/import/export for yourself, your parents, and your friend.
- Overlay **virtual PV/battery/EV/heat-pump** scenarios to estimate savings and optimal sizing.
- Optionally switch to “real-control mode” later if assets/integrations exist.

---

## 2. Operating Modes

### 2.1 Measured Mode (Real Data)
Uses measured telemetry.
- Primary input: **P1 smart meter** (15-min or finer)
- Optional inputs (if available): inverter/battery/EV/heat-pump telemetry

Purpose: establish a high-quality baseline and validate forecasts.

### 2.2 Virtual Mode (Synthetic Assets + Profiles)
Uses user-defined parameters and generated profiles.
Purpose: evaluate “what if I buy X kWp PV / Y kWh battery / EV / heat pump?”.

### 2.3 Mixed Mode (Measured Baseline + Virtual Overlay) — REQUIRED
This is the default research workflow.
- Use **measured P1** data for household net load.
- Add **virtual PV** to create a net-load-after-PV signal.
- Add **virtual battery/EV/heat pump** to simulate dispatch and costs.

This enables:
- You: P1-only today → simulate PV and/or battery and EV.
- Parents: P1 + existing PV/heat pump → simulate adding battery.
- Friend: P1 + PV + EV → simulate adding battery and alternative charging strategies.

---

## 3. Physical Assets (Examples + Parameterized Models)

The system must support:
- **Example templates** (quick start)
- **Fully parameterized virtual assets** (for sizing studies)

### 3.1 Example Template: PV
- Panels: 5× ~400 Wp
- Inverters: Enphase IQ-series microinverters (1:1 panel mapping)
- Orientation: South-East
- Peak AC Power: ~2.0 kW

### 3.2 Example Template: Battery
- System: Huawei LUNA2000
- Usable Capacity: 10 kWh
- Max Charge/Discharge Power: 5 kW
- Round-trip Efficiency: 90% (assumed)
- SOC Limits: 10–100%

### 3.3 Example Template: EV
- Battery Capacity: 60 kWh
- Max Charging Power: 11 kW (3-phase)

### 3.4 Virtual Asset Parameter Schemas

#### PV (virtual)
- `kwp`
- `orientation_deg`, `tilt_deg`
- `inverter_ac_kw`
- `losses_pct`
- `curtailment_export_limit_kw` (optional)

#### Battery (virtual)
- `usable_kwh`
- `max_charge_kw`, `max_discharge_kw`
- `roundtrip_eff_pct`
- `soc_min_pct`, `soc_max_pct`
- `reserve_soc_pct` (for reliability / flex programs)
- `degradation_cost_eur_per_kwh_throughput` (optional)

#### EV (virtual)
- `charger_max_kw`
- `kwh_per_100km` or `kwh_per_km`
- `km_per_week` (or a schedule)
- `plug_in_windows` (availability)
- `must_have_soc_by_time` (optional)

#### Heat Pump (virtual)
- `annual_heat_kwh` or `monthly_heat_kwh`
- `cop_profile` (seasonal COP)
- `thermal_buffer_kwh` (optional)
- `comfort_constraints` (optional)

---

## 4. Sign Conventions (Critical) (Critical)
All power values are in **W**, energy in **Wh**.
- **Grid Power:**
  - Positive = import from grid
  - Negative = export to grid
- **Battery Power:**
  - Positive = discharge to home
  - Negative = charging
- **PV Power:**
  - Always positive (generation)

**Energy balance (per timestep):**
```
PV + Battery_discharge - Battery_charge - Load - Grid_export + Grid_import = 0
```

---

## 5. Data Sources

### 5.1 Smart Meter (P1) — Primary Input
Required for measured and mixed modes.

### 5.1.1 Supported P1 sources (pluggable adapters)
The ingestion layer must support multiple P1 implementations via adapters that normalize to a common schema. Each adapter is responsible for polling, authentication (if any), unit conversion, and interval aggregation.

Supported adapters:

**A) HomeWizard P1 (local API)**
- Mode: local network polling
- Typical update rate: ~1 second (latest values)
- Usage in vHEMS:
  - Poll latest measurements
  - Locally aggregate to 15‑minute intervals
- Required configuration:
  - `base_url`
  - `poll_interval_seconds`
- Limitations:
  - No long-term history unless vHEMS stores it

**B) DSMR-Reader API**
- Mode: REST API against local DSMR-Reader instance
- Typical data: historical readings already aggregated
- Usage in vHEMS:
  - Fetch historical import/export per interval
  - Preferred source for backfilling long periods
- Required configuration:
  - `base_url`
  - `api_token` (if enabled)

**C) P1 Monitor (zTatz) Internet API**
- Mode: HTTP API (Internet API must be enabled in P1 Monitor)
- Typical data: historical and recent readings
- Usage in vHEMS:
  - Fetch historical import/export
  - Suitable for remote households (parents/friends)
- Required configuration:
  - `base_url`
  - `api_key`

The system must allow selecting the source per household:
`p1_source: homewizard | dsmr_reader | p1monitor`

Each adapter must expose the same internal interface:
- `get_latest()` (optional, realtime dashboards)
- `get_range(start, end)` (required for simulation)
- `capabilities`: `{ has_export, has_power_w, native_resolution }`

### 5.1.2 What to ingest
- Import energy (kWh) per interval
- Export energy (kWh) per interval (if present)
- Optional instantaneous power (W) for higher-resolution control

### 5.1.3 Normalization
All sources must be normalized into the same internal time-series model.

Canonical internal representation (15‑minute resolution):
```yaml
p1_interval:
  timestamp:
  import_kwh:
  export_kwh:
```

### 5.1.4 Interval aggregation rules
- Native resolutions finer than 15 minutes must be **summed** into fixed 15‑minute buckets.
- If only instantaneous power (W) is available:
  - Integrate power over time to energy (Wh → kWh).
- Missing intervals:
  - Short gaps (<1 interval): interpolate linearly.
  - Long gaps: mark as invalid; exclude from simulations or require user confirmation.

### 5.1.5 Household separation
Each household (you / parents / friend) is treated as a separate dataset with:
- Its own P1 source and credentials
- Its own tariff definition
- Its own virtual asset overlays

```yaml
p1_interval:
  timestamp:
  import_kwh:
  export_kwh:
```

### 5.2 Weather & Solar Model Inputs (NL)
- Provider: KNMI / EU NWP model (licensed API) or local irradiance model
- Horizon: 48 h
- Resolution: 15 min

### 5.3 Load Forecast
In mixed mode, load forecast is derived from measured P1 history (seasonality + weekday effects).
Forecast stored as **power**; energy derived by integration.
```yaml
load_forecast:
  timestamp:
  power_w_p10:
  power_w_p50:
  power_w_p90:
```

### 5.4 PV Generation Series
- Real mode: from inverter telemetry (if available)
- Mixed/virtual mode: generated from location + PV parameters
```yaml
pv_series:
  timestamp:
  pv_power_w_p50:
```

### 5.5 Tariffs
Support:
1) Fixed price
2) Time-of-use (2–3 rates)
3) Dynamic (hourly/15-min import/export series)

Store as time series:
```yaml
tariff:
  timestamp:
  import_price_eur_per_kwh:
  export_price_eur_per_kwh:
```

---

## 6. Simulation Engine (Baseline + Virtual Overlay)

### 6.1 Timeline and Resolution
- Default: 15-minute timesteps
- Sim horizon: 1 year preferred; fallback: representative weeks scaled

### 6.2 Mixed-Mode Signal Construction
From measured P1 interval data:
- Baseline net grid energy: `import_kwh - export_kwh`
- Convert to baseline power if needed.

Overlay virtual PV:
- Compute virtual PV power series.
- Reduce baseline net import by PV generation, create **virtual export** when PV exceeds load.

Overlay virtual battery/EV/heat pump:
- Optimizer dispatches battery and EV charging to minimize cost.
- Heat pump (if modeled) adds controllable demand within comfort constraints.

### 6.3 Baselines (for savings claims)
Selectable baselines:
1) **P1-only baseline:** no PV/battery/EV control (your current situation)
2) **PV-only baseline:** PV added, no battery/EV optimization
3) **PV + battery self-consumption baseline:** common inverter default behavior
4) **PV + EV dumb charging baseline** (charge immediately on plug-in)

Savings are computed relative to the chosen baseline.

### 6.4 KPIs
- Annual import/export (kWh)
- Net energy cost (€)
- Self-consumption (%), self-sufficiency (%)
- Battery throughput (kWh), equivalent full cycles
- EV charging cost (€)
- Peak import power (kW) (optional)

---

## 7. Control Strategy (Rolling Optimization)

### 7.1 Objective Function
Minimize net cost:
```
Σ(import_kWh[t] × import_price[t])
− Σ(export_kWh[t] × export_price[t])
+ battery_degradation_cost
```

### 7.2 Decision Variables
- Battery power
- EV charging power
- Optional deferrable loads / heat pump flexibility

### 7.3 Constraints
- Battery SOC and power limits
- EV availability and required energy
- Export limitation (if applicable)

Recalculate every 15 minutes using latest forecasts and prices.

---

## 8. Imbalance / Flexibility Participation (Onbalansmarkt)

### 8.1 Market Access
- Participation **only via an aggregator** (not direct household trading)

### 8.2 Technical Requirements (Program-Dependent)
- Minimum controllable power
- Response time
- Metering and verification method

### 8.3 Risk Handling
- Maintain SOC reserve to avoid non-delivery
- Physical constraints override signals

---

## 9. Business Case & Sizing Optimization

### 9.1 What the system must answer
For each household dataset (you / parents / friend):
- Optimal PV size (kWp) and/or battery size (kWh) for given tariff
- Marginal benefit curves (extra € saved per added kWh battery or kWp PV)
- EV impact (with/without smart charging)

### 9.2 Sizing Search
Support parameter sweeps:
- PV: 0 → N kWp
- Battery: 0 → M kWh
- EV: off/on with charger limits
Return ranked configurations by:
- Minimum annual cost
- Payback period (if capex provided)

---

## 10. Validation & Fallback
- Forecast accuracy tracked
- Fallback to last-known values
- Safety layer always enforces physical limits

---

## 11. Open Items
- Tariff source selection
- Aggregator program selection (if any)
- Battery degradation model
 (Rolling Optimization)

### 5.1 Objective Function
Minimize daily net cost:
```
Σ(import_kWh[t] × import_price[t])
− Σ(export_kWh[t] × export_price[t])
+ battery_degradation_cost
```

### 5.2 Decision Variables
- Battery charge/discharge power
- EV charging power
- Optional deferrable loads

### 5.3 Constraints
- Battery power and SOC limits
- EV availability window and required energy
- No grid export above contractual limit

This is a **rolling-horizon optimization** (not full MPC), recalculated every 15 minutes.

---

## 6. Imbalance / Flexibility Participation (Onbalansmarkt)

### 6.1 Market Access
- Participation **only via an aggregator** (not direct household trading)
- Aggregator provides:
  - Activation signals
  - Baseline definition
  - Settlement and revenue sharing

### 6.2 Technical Requirements
- Minimum controllable power: defined by aggregator (typically ≥1 kW)
- Response time: <5 minutes
- Metering: certified smart meter (15-min or finer)

### 6.3 Risk Handling
- SOC reserve maintained to avoid non-delivery penalties
- Hard constraints override market signals

---

## 7. Business Case & Simulation

### 7.1 Baseline Metrics
- Annual import (kWh)
- Annual export (kWh)
- Self-consumption (%)
- Battery cycles/year

### 7.2 Scenarios Compared
1. No optimization (default inverter logic)
2. HEMS with PV + load forecast
3. + EV smart charging
4. + Aggregated flexibility program

### 7.3 Outputs
- Annual energy cost (€)
- Annual savings vs baseline (€)
- Payback period (years)

---

## 8. Validation & Fallback
- Forecast accuracy tracked (MAE for PV/load)
- Fallback to last-known or climatology forecast
- Safety layer always enforces physical limits

---

## 9. Open Items
- Select exact tariff provider
- Confirm aggregator availability and contract terms
- Battery degradation cost model

