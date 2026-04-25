# Battery EMS – Node-RED Setup & Reference Guide
**EMS v3.7 / Planner v2.8** — Updated 25 April 2026

---

## My Setup

This flow was built around a **Growatt inverter** for the solar array and a **Deye inverter** for the battery bank. The battery inverter connection is not included in this flow — how the DC amps, charging, and discharging booleans are wired to your inverter depends entirely on your hardware. The entity names used here reflect my installation; adjust them to match yours.

**Why Frank Energie for pricing?** Frank Energie provides open API access to the actual day-ahead market prices (APX/EPEX) used in the Netherlands. Even if you are on a different energy provider, these price curves accurately reflect the real market variation throughout the day — all Dutch dynamic tariff providers use the same underlying wholesale market, just with different markups on top. Using the raw market curve gives the best signal for when to charge and discharge.

**After importing the flow into Node-RED**, several nodes will show a red triangle — this is expected. Those nodes reference Home Assistant entities that need to be mapped to your specific installation. Go through each red node and update the entity ID to match what appears in your **Developer Tools → States**. The entity names documented in this guide are the ones used in my setup; yours may differ, especially for the DSMR meter, battery SoC sensor, and solar inverter.

---

## Prerequisites

Install in Node-RED via **Manage palette**:
- `node-red-contrib-home-assistant-websocket` — all Home Assistant nodes

---

## Home Assistant Integrations

All integrations must be installed and working before the EMS flow can function.

| Integration | Source | Purpose in EMS |
|---|---|---|
| **Frank Energie** | [github.com/HiDiHo01/home-assistant-frank_energie](https://github.com/HiDiHo01/home-assistant-frank_energie) | Hourly dynamic electricity prices — primary price source for planner and EMS |
| **DSMR Smart Meter** | [home-assistant.io/integrations/dsmr](https://www.home-assistant.io/integrations/dsmr/) | Real-time grid import/export (kW) and per-phase load (kW) via P1 port |
| **Growatt ESPHome** | [github.com/WaarlandIT/ESPHOME-Growatt](https://github.com/WaarlandIT/ESPHOME-Growatt) | Live solar AC output per phase (PAC1/2/3 in W) |
| **Forecast.Solar** | [home-assistant.io/integrations/forecast_solar](https://www.home-assistant.io/integrations/forecast_solar/) | Solar production forecast — used by planner to reduce grid charge hours when solar will cover part of the charge |
| **ha-solarman** | [github.com/davidrapan/ha-solarman](https://github.com/davidrapan/ha-solarman) | Battery state of charge (%) from BMS via Solarman protocol |
| **EnergyZero** | [home-assistant.io/integrations/energyzero](https://www.home-assistant.io/integrations/energyzero/) | Used only as an hourly trigger — not the price source |

> Frank Energie is the authoritative price source since Planner v2.4. EnergyZero is kept only as a trigger to fire the flow at the start of each new price hour.

---

## Step 1 – Home Assistant Helpers

Create these in **Settings → Devices & Services → Helpers** before importing the flow:

| Type | Entity ID | Min | Max | Step |
|---|---|---|---|---|
| `input_number` | `input_number.battery_dc_amps` | 0 | 200 | 1 |
| `input_boolean` | `input_boolean.battery_charging` | — | — | — |
| `input_boolean` | `input_boolean.battery_discharging` | — | — | — |

These are the three outputs the EMS writes every cycle. Your battery inverter integration reads from these.

---

## Step 2 – Sensor Entity Name Mapping

Verify these in the relevant `api-current-state` nodes after importing. Use **Developer Tools → States** to find your exact entity names.

### Frank Energie (price source) — [GitHub](https://github.com/HiDiHo01/home-assistant-frank_energie)
| Node | Entity ID | Output |
|---|---|---|
| Frank Energie prijzen | `sensor.frank_energie_prijzen_huidige_elektriciteitsprijs_all_in` | `msg.frankPrices` (attributes.prices array) |

> The planner calculates `currentPrice` and `avgPrice` directly from this array, overriding anything from EnergyZero sensors.

### DSMR Smart Meter — [HA Docs](https://www.home-assistant.io/integrations/dsmr/)
| Node | Entity ID | Output |
|---|---|---|
| Grid import kW | `sensor.dsmr_reading_electricity_currently_delivered` | `msg.gridImport` (kW) |
| Grid export kW | `sensor.dsmr_reading_electricity_currently_returned` | `msg.gridExport` (kW) |
| Phase L1 kW | `sensor.dsmr_reading_phase_currently_delivered_l1` | `msg.phaseL1` |
| Phase L2 kW | `sensor.dsmr_reading_phase_currently_delivered_l2` | `msg.phaseL2` |
| Phase L3 kW | `sensor.dsmr_reading_phase_currently_delivered_l3` | `msg.phaseL3` |

> Entity names vary by DSMR version. Search `dsmr` in Developer Tools → States to confirm yours.

### Growatt Solar (AC output per phase) — [GitHub](https://github.com/WaarlandIT/ESPHOME-Growatt)
| Node | Entity ID | Output |
|---|---|---|
| Solar PAC1 W | `sensor.growatt_pac1` | `msg.pac1` |
| Solar PAC2 W | `sensor.growatt_pac2` | `msg.pac2` |
| Solar PAC3 W | `sensor.growatt_pac3` | `msg.pac3` |

> AC watts per phase summed as `solarTotalW = pac1 + pac2 + pac3`.

### Forecast.Solar — [HA Docs](https://www.home-assistant.io/integrations/forecast_solar/)
| Node | Entity ID | Output |
|---|---|---|
| Solar forecast remaining | `sensor.energy_production_remaining_today` | `msg.solarForecastRemaining` (kWh) |

> Added in Planner v2.8. The planner deducts this from `kwhNeeded` to reduce the number of grid charge hours when solar will cover part of the charge. Falls back safely to 0 if unavailable.

### Battery SoC — [ha-solarman GitHub](https://github.com/davidrapan/ha-solarman)
| Node | Entity ID | Output |
|---|---|---|
| Battery SoC | `sensor.battery_state_of_charge` | `msg.batterySoC` (%) |

> Replace with your actual BMS/inverter SoC entity.

### Trigger nodes — [EnergyZero HA Docs](https://www.home-assistant.io/integrations/energyzero/)
| Node | Entity ID | Purpose |
|---|---|---|
| DSMR power change | `sensor.electricity_meter_energieverbruik` | Fire instantly on load change |
| Price hour change | `sensor.energyzero_today_energy_current_hour_price` | Fire on hourly price rollover |

---

## Step 3 – Node Chain Execution Order

**Battery SoC and Solar Forecast must both be read before the Price Planner.**

```
[Every 5 min inject]  ──┐
[DSMR power change]   ──┼──► [Frank Energie prices] ──► [Battery SoC] ──► [Solar forecast remaining] ──► [Price Planner]
[Price hour change]   ──┘                                                                                        │
                                                                                                       [Grid import kW]
                                                                                                       [Grid export kW]
                                                                                                        [Phase L1 kW]
                                                                                                        [Phase L2 kW]
                                                                                                        [Phase L3 kW]
                                                                                                       [Solar PAC1 W]
                                                                                                       [Solar PAC2 W]
                                                                                                       [Solar PAC3 W]
                                                                                                  [EMS Decision Engine]
                                                                                                       [Split outputs]
                                                                                      ┌──────────────┬──────────────┐
                                                                              [Set dc_amps]  [Set charging]  [Set discharging]
                                                                                                        [EMS Diagnostics]
```

---

## Step 4 – Trigger Configuration

| Trigger | Interval | Purpose |
|---|---|---|
| Every 5 min inject | 300 s | Baseline heartbeat |
| DSMR power change | Every DSMR update (~2–10 s) | React instantly to home load changes |
| Price hour change | Every hour | React immediately when price rolls to next hour |

In both `server-state-changed` nodes, ensure:
- **Only send if state changes** — enabled
- **Ignore unavailable/unknown** — enabled on both incoming and outgoing state

---

## Step 5 – Battery Specs (verify in both scripts)

| Spec | Value | Config key |
|---|---|---|
| Capacity | 40 kWh | `BATTERY_KWH` |
| Voltage | 48 V DC | `BATTERY_VOLTAGE` |
| Max charge/discharge | 200 A | `MAX_AMPS` |
| Max charge power | ~8.16 kW (200 A × 48 V × 0.85) | derived |
| Round-trip efficiency | 85% | `ROUND_TRIP_EFF` |
| Full charge time (from SOC_MIN) | ~4.2 h at max rate | derived |
| Grid connection | 3-phase, 20 A fuse | `GRID_MAX_A_PHASE` = 18 (2 A margin) |

---

## Price Planner (v2.8) — How It Works

The planner runs first each cycle and calculates today's optimal charge and discharge schedule before any sensor data is read.

### Price source
Prices are read from the Frank Energie `attributes.prices` array. `currentPrice` and `avgPrice` are calculated from this array and written to `msg`, overriding anything from EnergyZero.

### kWh needed and solar deduction (v2.8)

```
usableCapacity = 40 × ((95% − 10%) / 100)  = 34 kWh
currentKwh     = 40 × ((SoC% − 10%) / 100)
kwhNeeded      = usableCapacity − currentKwh
solarUsable    = solarForecastRemaining × SOLAR_FORECAST_EFF (0.90)
kwhFromGrid    = max(0, kwhNeeded − solarUsable)
hoursNeeded    = ceil(kwhFromGrid / 8.16 kW) + 1 safety hour
```

When solar is forecast to cover part of the charge, `kwhFromGrid` is smaller, `hoursNeeded` is fewer, and the grid charge window is shorter. The battery is expected to top up naturally during solar production hours. If `solarForecastRemaining` is unavailable it defaults to 0, keeping the planner conservative.

### Charge window — centered on cheapest hour (v2.6)

Rather than picking the cheapest N scattered hours, the planner:
1. Finds the single cheapest hour remaining today
2. Places it at the center of the window
3. Expands left and right, always picking the cheaper neighbour next, until `hoursNeeded` hours are filled

This produces a **contiguous time window** with the cheapest price in the middle. Example with `hoursNeeded = 5` and cheapest hour at 14h:

```
Step 1: [14h]
Step 2: left=13h (0.18) vs right=15h (0.16) → pick 15h  → [14h, 15h]
Step 3: left=13h (0.18) vs right=16h (0.20) → pick 13h  → [13h, 14h, 15h]
Step 4: left=12h (0.21) vs right=16h (0.20) → pick 16h  → [13h, 14h, 15h, 16h]
Step 5: left=12h (0.21) vs right=17h (0.28) → pick 12h  → [12h, 13h, 14h, 15h, 16h]
```

### Discharge hour selection (v2.7)

Hours are selected from morning band (06h–11h) and evening band (16h–23h). Since v2.7 the minimum filter is `avgPrice` — any hour below the daily average is excluded from discharge regardless of band. Top 50% by price from each qualifying band is selected. Any discharge hour overlapping a charge hour is removed and the threshold recalculated from the filtered set.

### Planner config constants

| Constant | Value | Description |
|---|---|---|
| `BATTERY_KWH` | 40 | Battery capacity |
| `SOC_MIN` | 10% | Minimum SoC (floor) |
| `SOC_MAX` | 95% | Maximum SoC (ceiling) |
| `SOC_NORMAL_THRESHOLD` | 80% | Above this SoC uses tight charge cap |
| `CHARGE_ABS_RATIO` | 0.55 | Tight charge cap = `avgPrice × 0.55` |
| `DISCHARGE_ABS_RATIO` | 0.55 | Kept for `dischargeAbsMin` reference; discharge band now uses `avgPrice` directly |
| `SOLAR_FORECAST_EFF` | 0.90 | 10% margin on solar forecast (forecast tends to be optimistic) |
| `SAFETY_BUFFER_H` | 1 | Extra hour added to `hoursNeeded` as safety margin |
| `ROUND_TRIP_EFF` | 0.85 | Used to calculate max charge kW rate |

### Planner outputs written to msg

| Property | Type | Description |
|---|---|---|
| `msg.inChargingWindow` | bool / null | true = current hour is in the planned charge window; null = no price data |
| `msg.inDischargeWindow` | bool | true = current hour is in the planned discharge window |
| `msg.dischargeThreshold` | float | Lowest price in selected discharge hours (EUR/kWh) |
| `msg.plannedChargeHours` | int[] | Sorted charge hours e.g. [12, 13, 14, 15, 16] |
| `msg.plannedDischargeHours` | int[] | Sorted discharge hours e.g. [20, 21, 22] |
| `msg.hoursNeeded` | int | Grid charge hours needed (after solar deduction) |
| `msg.kwhNeeded` | float | Total kWh needed to reach 95% SoC |
| `msg.kwhFromGrid` | float | kWh to be sourced from grid (kwhNeeded − solarUsable) |
| `msg.solarForecastKwh` | float | Usable solar kWh deducted from grid need (after efficiency) |
| `msg.cheapestPrice` | float | Cheapest remaining hour today |
| `msg.currentPrice` | string | Current price from Frank Energie |
| `msg.avgPrice` | string | Today's average price from Frank Energie |
| `msg.plannerVersion` | string | e.g. `'v2.8'` |
| `msg.plannerReason` | string | Human-readable summary including solar deduction |

---

## EMS Decision Engine (v3.7) — Configuration Reference

### CFG parameters

| Parameter | Value | Description |
|---|---|---|
| `MAX_AMPS` | 200 | Hard cap on DC output amps |
| `BATTERY_VOLTAGE` | 48 | Nominal DC bus voltage (V) |
| `BATTERY_KWH` | 40 | Usable capacity (kWh) |
| `SOC_MIN` | 10% | Discharge floor — never goes below this |
| `SOC_MAX` | 95% | Charge ceiling — stops charging above this (except negative price) |
| `SOC_DISCHARGE_MIN` | 15% | Discharge guard — will not discharge below this even in peak window |
| `SOC_CRITICAL` | 20% | At or below this SoC, charge at full amps in any below-average hour |
| `CHARGE_THRESHOLD` | 0.80 | Ratio threshold for price override and fallback charging |
| `DISCHARGE_THRESHOLD` | 1.20 | Fallback: discharge if price ≥ 120% of avg (no planner data) |
| `CHARGE_ABS_RATIO` | 0.55 | `chargeAbsMax` = `avgPrice × 0.55` |
| `DISCHARGE_ABS_RATIO` | 0.55 | `dischargeAbsMin` = `avgPrice × 0.55` |
| `DISCHARGE_OVERSHOOT` | 1.25 | 25% overshoot on discharge target to compensate for inverter lag |
| `EXPORT_BIAS_W` | 800 | Watts added to discharge target to push net flow toward export |
| `HYSTERESIS` | 0.05 | Price ratio band to prevent rapid toggling at thresholds |
| `SOLAR_SURPLUS_W` | 500 | Min solar export surplus (W) to start solar charging |
| `SOLAR_SURPLUS_EXIT_W` | 200 | Min solar export surplus (W) to keep solar charging going |
| `ROUND_TRIP_EFF` | 0.85 | Charging efficiency — applied to charge amps only, NOT discharge |
| `GRID_MAX_A_PHASE` | 18 | Usable amps per phase (20 A fuse − 2 A margin) |
| `GRID_VOLTAGE` | 230 | AC grid voltage |
| `SAFETY_MARGIN_A` | 2 | Additional per-phase headroom buffer |
| `MIN_DISCHARGE_A` | 35 | Minimum meaningful discharge current (A) |

### Decision priority (highest to lowest)

```
1. Negative price AND canCharge          → charge at maxAllowedChargeA

2. isPriceLow AND canCharge              → charge at scaled amps
   isPriceLow is true when ANY of:
     a) SoC <= SOC_CRITICAL (20%) AND price < avgPrice   → full amps, critical recovery
     b) inChargingWindow = true (planner window)         → scaled amps
     c) isPriceVeryLow: ratio <= 0.80 AND                → scaled amps, outside window
        price <= avgPrice × 0.55

3. Solar surplus > threshold AND         → absorb surplus at surplus amps
   canChargeSolar                          (always before discharge — free energy first)

4. isPriceHigh AND canDischarge          → discharge to cover import + export bias
   isPriceHigh requires price >= avgPrice (v3.6 guard)

5. None of the above                     → idle
```

### Charge reason strings

The reason string now distinguishes how the charge decision was made:

| Trigger | Reason string |
|---|---|
| `batterySoC <= SOC_CRITICAL` | `Critical SoC (19%) - charging at max 155 A (price 0.078 EUR below avg 0.192 EUR)` |
| `inChargingWindow = true` | `Planner: cheapest window (0.127 EUR) - charging at 94 A. Hours: 12h,13h,14h,15h,16h` |
| `isPriceVeryLow` override | `Price override: 41% of avg (0.078 EUR) below threshold - charging at 155 A outside planned window` |
| Negative price | `Negative price (-0.040 EUR/kWh) - charging at 106 A (worst phase headroom: 8.7 A)` |

### isPriceLow logic (v3.7)

```javascript
var isPriceVeryLow = priceRatio <= CFG.CHARGE_THRESHOLD        // price <= 80% of avg
                  && currentPrice <= chargeAbsMax;             // price <= avgPrice × 0.55

if (batterySoC <= CFG.SOC_CRITICAL && currentPrice < avgPrice) {
  isPriceLow = true;                                           // critical SoC — any below-avg hour
} else if (inChargingWindow === null) {
  isPriceLow = isPriceVeryLow || isNegPrice;                  // no planner — ratio fallback
} else {
  isPriceLow = inChargingWindow || isPriceVeryLow || isNegPrice; // planner + ratio override
}
```

The ratio override (`isPriceVeryLow`) ensures that dramatically cheap hours are never wasted even if they fall outside the planned window — for example, a cheap morning dip before the planned 12h–16h window.

### isPriceHigh guard (v3.6)

```javascript
if (msg.inDischargeWindow === true) {
  // Planner window alone is not enough — price must still be above average
  isPriceHigh = currentPrice >= avgPrice && currentPrice >= dischargeAbsMin;
} else {
  isPriceHigh = priceRatio >= dischargeRatioThreshold && currentPrice >= dischargeAbsMin;
}
```

This prevents the planner from triggering discharge in hours that fall below the daily average — a real scenario when prices are unusually flat or the morning band includes cheap hours.

### canDischarge guard (v3.5 fix)

```javascript
canDischarge = batterySoC > SOC_MIN && batterySoC >= SOC_DISCHARGE_MIN
// Effectively: SoC >= 15% (SOC_DISCHARGE_MIN is the binding constraint)
```

### Discharge amp calculation (v3.5)

```
grossTargetW     = gridImport_W + EXPORT_BIAS_W
dischargeTargetW = grossTargetW × DISCHARGE_OVERSHOOT    ← applied once only
dc_amps          = clamp(dischargeTargetW / BATTERY_VOLTAGE, MIN_DISCHARGE_A, MAX_AMPS)
```

`ROUND_TRIP_EFF` is NOT applied to discharge — efficiency only applies when charging. Applying it to discharge inflated amps by ~18% before v3.5.

Example (import 3698 W):
```
(3698 + 800) × 1.25 ÷ 48 = 117 A
```

### Charge amp calculation

```
worstPhaseHeadroomA = GRID_MAX_A_PHASE − maxPhaseLoad_A − SAFETY_MARGIN_A
gridHeadroomW       = worstPhaseHeadroomA × GRID_VOLTAGE × 3
maxAllowedChargeA   = min(MAX_AMPS, round(gridHeadroomW / BATTERY_VOLTAGE × ROUND_TRIP_EFF))
```

For scaled charging (planner window or ratio override, SoC > SOC_CRITICAL):
```
priceDiscount = 1 − priceRatio
fraction      = 0.50 + (priceDiscount × 2.5)    // 50%–100% of maxAllowedChargeA
targetAmps    = round(maxAllowedChargeA × fraction)
```

Example (phase headroom 12.71 A, price 41% of avg):
```
gridHeadroom    = 12.71 × 230 × 3 = 8,770 W
maxAllowedCharge = (8770 / 48) × 0.85 = 155 A
priceDiscount   = 1 − 0.41 = 0.59
fraction        = 0.50 + (0.59 × 2.5) = 1.0 → capped at 1.0
targetAmps      = 155 A
```

### State confirmation / debounce

| Trigger type | Confirm cycles | Effect |
|---|---|---|
| Price signal (cheap/high/negative/solar surplus) | 1 | Immediate state change |
| All other signals | 2 | One confirmation cycle required |

Reason string shows `[confirming N/2]` or `[pending change to X 1/2]` while debouncing. This is normal.

---

## Diagnostic Output Structure

```javascript
msg.payload = {
  version:     "EMS v3.7 / Planner v2.8",
  dc_amps:     155,
  charging:    true,
  discharging: false,
  reason:      "Price override: 41% of avg (0.078 EUR) below threshold - charging at 155 A outside planned window",

  planner: {
    inWindow:              false,
    inDischargeWindow:     false,
    dischargeThreshold:    0.304,
    hoursNeeded:           5,          // grid hours needed (after solar deduction)
    kwhNeeded:             29.2,       // total kWh to reach 95% SoC
    kwhFromGrid:           22.0,       // kWh to charge from grid (kwhNeeded − solar)
    solarForecastKwh:      7.2,        // usable solar kWh deducted (forecast × 0.90)
    cheapestPrice:         -0.101,
    plannedHours:          [12, 13, 14, 15, 16],
    plannedDischargeHours: [20, 21, 22],
    reason:                "Need 29.2 kWh total, solar covers ~7.2 kWh, 22.0 kWh from grid (5h). SoC 22% - cheapest hour: 14h (-0.101 EUR)..."
  },

  inputs: {
    currentPrice:        0.078,
    avgPrice:            0.192,
    priceRatio:          40.6,         // % of avg
    chargeAbsMax:        0.106,        // avgPrice × CHARGE_ABS_RATIO
    dischargeAbsMin:     0.106,        // avgPrice × DISCHARGE_ABS_RATIO
    lastState:           "charging",
    batterySoC:          22,
    solarTotal_W:        1374,
    netGrid_W:           685,          // positive = importing
    dischargeTarget_W:   777,          // gross import W (pre-bias reference)
    actualSurplus_W:     0,
    phaseL1_A:           3.29,
    phaseL2_A:           0,
    phaseL3_A:           0.09,
    maxPhaseLoad_A:      3.29,
    worstHeadroom_A:     12.71,
    gridHeadroom_W:      8772,
    maxAllowedCharge_A:  155,
    timestamp:           "2026-04-25T09:02:22.277Z"
  }
}
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Always idle, dc_amps 0 | Wrong entity names | Use Developer Tools → States to verify all entity IDs |
| `priceRatio` always 100% | `avgPrice` = 0 | Check Frank Energie entity has `attributes.prices` populated |
| `inChargingWindow: null` | Frank Energie array empty | Check planner node status — yellow dot = no price data |
| SoC not updating | Wrong SoC entity | Replace `sensor.battery_state_of_charge` with your BMS entity |
| Discharge amps ~18% too high | Old pre-v3.5 script | Check version string in debug output |
| Discharging below avg price | Old pre-v3.6 script | `isPriceHigh` avgPrice guard missing — update to v3.6+ |
| Not charging at cheap hours outside planner window | Old pre-v3.7 script | `isPriceVeryLow` override missing — update to v3.7 |
| Discharge not stopping at low SoC | Old `canDischarge` bug | Ensure EMS v3.5+ is deployed |
| Solar forecast not deducting from kwhNeeded | `solarForecastRemaining` = 0 | Check `get-solar-forecast` node is in chain; verify `sensor.energy_production_remaining_today` entity exists |
| Reason says "Planner: cheapest window" outside planned hours | Old pre-v3.7 reason string | Update to EMS v3.7 |
| `[confirming 1/2]` in reason | State pending confirmation | Normal — resolves next cycle |
| `maxAllowedCharge_A: 0` | Home load near grid limit | Large appliance consuming full phase headroom — EMS resumes when load drops |
| Charge amps fluctuate with appliances | Expected — phase headroom recalculated every cycle | Normal behaviour; EMS always stays within grid limit |

---

## Version History

| Version | Date | Change |
|---|---|---|
| **EMS v3.7** | 2026-04-25 | Added `isPriceVeryLow` ratio override — dramatically cheap hours now charge even outside the planned window; distinct reason strings for planner window vs ratio override vs critical SoC |
| **EMS v3.6** | 2026-04-25 | `SOC_CRITICAL` check changed `<` → `<=` (boundary SoC 20% was missed); `isPriceHigh` now requires `currentPrice >= avgPrice` — planner `inDischargeWindow` can no longer trigger discharge below avg price |
| EMS v3.5 | 2026-04-20 | Fixed `canDischarge` (`\|\|` → `&&`); removed `ROUND_TRIP_EFF` from discharge amp calc (was inflating amps ~18%) |
| EMS v3.4 | 2026-04-19 | All fixed price thresholds replaced with ratios of daily avg (`CHARGE_ABS_RATIO`, `DISCHARGE_ABS_RATIO`) |
| EMS v3.3 | 2026-04-15 | Solar surplus uses `canChargeSolar`; moved before discharge branch in priority order |
| EMS v3.2 | 2026-04-12 | `SOC_DISCHARGE_MIN` 15%; gross grid import as discharge base target |
| EMS v3.1 | 2026-04-10 | `SOC_DISCHARGE_MIN` 15%; `SOC_CRITICAL` 20% |
| EMS v3.0 | 2026-04-08 | P1 phase readings used directly for per-phase headroom |
| **Planner v2.8** | 2026-04-25 | Solar forecast (`sensor.energy_production_remaining_today`) deducted from `kwhNeeded`; `kwhFromGrid` and `solarForecastKwh` added to planner outputs; `get-solar-forecast` node required in chain |
| **Planner v2.7** | 2026-04-25 | Discharge band filter raised from `dischargeAbsMin` (avg × 0.55) to `avgPrice` — hours below daily average can never be discharge hours |
| **Planner v2.6** | 2026-04-25 | Charge window now centered on cheapest hour and expanded outward to adjacent hours — window is always contiguous in time |
| Planner v2.5 | 2026-04-20 | Price ratios aligned with EMS v3.5 |
| Planner v2.4 | 2026-04-19 | `currentPrice`/`avgPrice` from Frank Energie array; EnergyZero no longer price source |
| Planner v2.3 | 2026-04-17 | Version as single `PLANNER_VERSION` variable |
| Planner v2.2 | 2026-04-15 | `SOC_NORMAL_THRESHOLD` raised to 80% |
| Planner v2.1 | 2026-04-14 | Fixed `msg.plannerVersion` / `inWindow` undefined bugs |
| Planner v2.0 | 2026-04-13 | `dischargeThreshold` recalculated after conflict filter; discharge hours deduped |
