# Battery EMS – Node-RED Setup & Reference Guide
**EMS v3.15 / Planner v2.13** — Updated 29 April 2026

![Node-red layout](NodeRed.png)

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
| **Forecast.Solar** | [home-assistant.io/integrations/forecast_solar](https://www.home-assistant.io/integrations/forecast_solar/) | Solar production forecast — used by planner to reduce grid charge hours and detect pre-discharge opportunities |
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

> Added in Planner v2.8. The planner deducts this from `kwhNeeded` to reduce grid charge hours when solar will cover part of the charge. Also used for pre-discharge detection. Falls back safely to 0 if unavailable.

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
| Max discharge power | 9.6 kW (200 A × 48 V / 1000) | `MAX_DISCHARGE_KW` |
| Round-trip efficiency | 85% | `ROUND_TRIP_EFF` |
| Usable discharge capacity | 32 kWh (SOC 95% → 15%) | derived |
| Full charge time (from SOC_MIN) | ~4.2 h at max rate | derived |
| Full discharge time (to SOC_DISCHARGE_MIN) | ~3.3 h at max rate | derived |
| Grid connection | 3-phase, 20 A fuse | `GRID_MAX_A_PHASE` = 18 (2 A margin) |

---

## Price Planner (v2.13) — How It Works

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

When solar is forecast to cover part of the charge, `kwhFromGrid` is smaller, `hoursNeeded` is fewer, and the grid charge window is shorter. If `solarForecastRemaining` is unavailable it defaults to 0, keeping the planner conservative.

### Charge window — centered on cheapest hour, with validity gate (v2.6 / v2.12)

The planner finds the single cheapest remaining hour and expands outward until `hoursNeeded` hours are filled — always picking the cheaper neighbour at each step. An **early-hour bias** (v2.9) prefers earlier hours when neighbour prices are within `EARLY_BIAS_EUR` (0.02 EUR) of each other, keeping grid charging before the solar peak.

Since v2.12, the charge window is only built if `cheapestPrice <= chargePriceCap`. If all remaining hours are above the cap (e.g. late evening), `plannedHours` is empty and the EMS falls back to ratio-based logic only — preventing the planner from scheduling charging at expensive hours.

```
chargeWindowValid = cheapestPrice <= chargePriceCap
if (!chargeWindowValid) plannedHours = []
```

Example with `hoursNeeded = 5`, cheapest hour at 14h, `EARLY_BIAS_EUR = 0.02`:
```
Start:          [14h]
right=15h(0.16) cheaper than left=13h(0.18) by 0.02 → within bias → pick left  → [13h, 14h]
right=15h(0.16) cheaper than left=12h(0.21) by 0.05 → exceeds bias → pick right → [13h, 14h, 15h]
left=12h(0.21) vs right=16h(0.20) → within bias → pick left                     → [12h, 13h, 14h, 15h]
left=11h vs right=16h(0.20) → pick right if cheaper                              → [12h, 13h, 14h, 15h, 16h]
```

### Discharge window — centered on most expensive hour (v2.13)

The discharge window is now sized from actual usable capacity and built symmetrically around the most expensive hour, with a late-hour bias to anchor the window in the evening peak band.

```
kwhToDischarge       = BATTERY_KWH × ((SOC_DISCHARGE_MAX − SOC_DISCHARGE_MIN_PL) / 100)
                     = 40 × ((95 − 15) / 100) = 32 kWh
dischargeHoursNeeded = ceil(32 / 9.6 kW) + 1 = 5h
```

The most expensive remaining hour above `avgPrice` becomes the center. The window expands outward preferring the more expensive neighbour. `LATE_BIAS_EUR` (0.02 EUR) means the later hour is preferred when prices are within that margin — keeping the window in the evening peak rather than drifting into the morning.

Any discharge hour overlapping a charge hour is removed and the threshold recalculated from the filtered set.

### Pre-discharge before negative price windows (v2.11)

When negative price hours are forecast later today, the planner detects them and flags hours before the negative window as pre-discharge candidates — provided those hours have prices above `dischargeAbsMin` and are not already negative. The EMS then discharges during those hours to free capacity for the upcoming negative price charging.

```
hoursUntilNegWindow  = hours from now until first future negative price hour
preDischargeHours    = hours before the neg window with price >= avgPrice × DISCHARGE_ABS_RATIO
inPreDischargeWindow = current hour is in preDischargeHours AND SoC > SOC_DISCHARGE_MIN
```

### Planner config constants

| Constant | Value | Description |
|---|---|---|
| `BATTERY_KWH` | 40 | Battery capacity |
| `SOC_MIN` | 10% | Minimum SoC (floor) |
| `SOC_MAX` | 95% | Maximum SoC (ceiling) |
| `SOC_NORMAL_THRESHOLD` | 80% | Above this SoC uses tight charge cap |
| `CHARGE_ABS_RATIO` | 0.55 | Tight charge cap = `avgPrice × 0.55` |
| `DISCHARGE_ABS_RATIO` | 0.55 | Discharge abs min = `avgPrice × 0.55` |
| `SOLAR_FORECAST_EFF` | 0.90 | 10% margin on solar forecast |
| `SAFETY_BUFFER_H` | 1 | Extra hour added to `hoursNeeded` as safety margin |
| `ROUND_TRIP_EFF` | 0.85 | Used to calculate max charge kW rate |
| `EARLY_BIAS_EUR` | 0.02 | Prefer earlier hour in charge window when price diff is within this margin |
| `LATE_BIAS_EUR` | 0.02 | Prefer later hour in discharge window when price diff is within this margin |
| `MAX_DISCHARGE_KW` | 9.6 | Max discharge rate (200 A × 48 V / 1000) for window sizing |
| `SOC_DISCHARGE_MAX` | 95% | SoC from which discharge is measured |
| `SOC_DISCHARGE_MIN_PL` | 15% | SoC floor used in discharge window sizing |

### Planner outputs written to msg

| Property | Type | Description |
|---|---|---|
| `msg.inChargingWindow` | bool / null | true = current hour is in the planned charge window; null = no price data |
| `msg.inDischargeWindow` | bool | true = current hour is in the planned discharge window |
| `msg.inPreDischargeWindow` | bool | true = current hour is a pre-discharge hour before a negative price window |
| `msg.preDischargeHours` | int[] | Hours flagged for pre-discharge |
| `msg.hoursUntilNegWindow` | int / null | Hours until first future negative price hour |
| `msg.futureNegHoursCount` | int | Number of future negative price hours today |
| `msg.negPriceHoursRemaining` | int | Remaining negative price hours including current (for spread charging) |
| `msg.dischargeThreshold` | float | Lowest price in selected discharge hours (EUR/kWh) |
| `msg.plannedChargeHours` | int[] | Sorted charge hours e.g. [12, 13, 14] |
| `msg.plannedDischargeHours` | int[] | Sorted discharge hours e.g. [19, 20, 21, 22, 23] |
| `msg.hoursNeeded` | int | Grid charge hours needed (after solar deduction) |
| `msg.kwhNeeded` | float | Total kWh needed to reach 95% SoC |
| `msg.kwhFromGrid` | float | kWh to be sourced from grid (kwhNeeded − solarUsable) |
| `msg.solarForecastKwh` | float | Usable solar kWh deducted from grid need (after efficiency) |
| `msg.cheapestPrice` | float | Cheapest remaining hour today |
| `msg.currentPrice` | string | Current price from Frank Energie |
| `msg.avgPrice` | string | Today's average price from Frank Energie |
| `msg.plannerVersion` | string | e.g. `'v2.13'` |
| `msg.plannerReason` | string | Human-readable summary including solar deduction and pre-discharge status |

---

## EMS Decision Engine (v3.15) — Configuration Reference

### CFG parameters

| Parameter | Value | Description |
|---|---|---|
| `MAX_AMPS` | 200 | Hard cap on DC output amps |
| `BATTERY_VOLTAGE` | 48 | Nominal DC bus voltage (V) |
| `BATTERY_KWH` | 40 | Usable capacity (kWh) |
| `SOC_MIN` | 10% | Discharge floor — never goes below this |
| `SOC_MAX` | 95% | Hard charge ceiling — never charges above this regardless of price |
| `SOC_DISCHARGE_MIN` | 15% | Discharge guard — will not discharge below this even in peak window |
| `SOC_CRITICAL` | 20% | At or below this SoC, charge at full amps if price is genuinely cheap |
| `CHARGE_THRESHOLD` | 0.80 | Ratio threshold for price override and fallback charging |
| `DISCHARGE_THRESHOLD` | 1.20 | Fallback: discharge if price ≥ 120% of avg (no planner data) |
| `CHARGE_ABS_RATIO` | 0.55 | `chargeAbsMax` = `avgPrice × 0.55` |
| `DISCHARGE_ABS_RATIO` | 0.55 | `dischargeAbsMin` = `avgPrice × 0.55` |
| `DISCHARGE_OVERSHOOT` | 1.25 | 25% overshoot on discharge target to compensate for inverter lag |
| `EXPORT_BIAS_W` | 800 | Watts added to discharge target to push net flow toward export |
| `HYSTERESIS` | 0.05 | Price ratio band to prevent rapid toggling at thresholds |
| `SOLAR_SURPLUS_W` | 500 | Min solar export surplus (W) to start solar charging |
| `SOLAR_SURPLUS_EXIT_W` | 200 | Min solar export surplus (W) to keep solar charging going |
| `SOLAR_SUPPRESS_DISCHARGE_W` | 500 | Suppress minimum discharge when solar surplus exceeds this |
| `ROUND_TRIP_EFF` | 0.85 | Charging efficiency — applied to charge amps only, NOT discharge |
| `GRID_MAX_A_PHASE` | 18 | Usable amps per phase (20 A fuse − 2 A margin) |
| `GRID_VOLTAGE` | 230 | AC grid voltage |
| `SAFETY_MARGIN_A` | 2 | Additional per-phase headroom buffer |
| `MIN_DISCHARGE_A` | 35 | Minimum meaningful discharge current (A) |
| `MIN_CHARGE_A` | 10 | Minimum meaningful charge current (A) — floor for negative price spread charging |

### Decision priority (highest to lowest)

```
1. Negative price AND canCharge          → spread charge across remaining negative hours;
                                           solar surplus offsets grid draw to keep netGrid ~0

2. isPriceLow AND canCharge              → charge at scaled amps
   isPriceLow is true when ANY of:
     a) SoC <= SOC_CRITICAL AND          → full amps, critical recovery
        price < avgPrice AND             (requires price <= chargeAbsMax since v3.8)
        price <= chargeAbsMax
     b) inChargingWindow = true          → scaled amps (planner window)
     c) isPriceVeryLow: ratio <= 0.80    → scaled amps outside planned window
        AND price <= avgPrice × 0.55

3. Solar surplus > threshold AND         → absorb surplus at surplus amps
   canChargeSolar AND                      (skipped during active discharge window
   NOT inDischargeWindow                   since v3.15 — discharge takes priority)

4. inPreDischargeWindow AND canDischarge → pre-discharge to free capacity before
                                           upcoming negative price window

5. isPriceHigh AND canDischarge          → discharge to cover import + export bias
     a) dischargeTarget > 0             → cover gross import + bias at full calc amps
     b) dischargeTarget = 0 AND         → export at MIN_DISCHARGE_A
        surplus < SOLAR_SUPPRESS_W        (suppressed if solar already exporting heavily)

6. None of the above                     → idle
```

### Charge reason strings

| Trigger | Reason string |
|---|---|
| `batterySoC <= SOC_CRITICAL` | `Critical SoC (19%) - charging at max 155 A (price 0.078 EUR below avg 0.192 EUR)` |
| `inChargingWindow = true` | `Planner: cheapest window (0.127 EUR) - charging at 94 A. Hours: 12h,13h,14h` |
| `isPriceVeryLow` override | `Price override: 41% of avg (0.078 EUR) below threshold - charging at 155 A outside planned window` |
| Negative price with solar offset | `Negative price (-0.084 EUR/kWh) - charging at 123 A (6 neg-price hour(s) remaining, spreading 30.0 kWh over window, solar covers ~42 A, grid ~81 A)` |
| Negative price no solar | `Negative price (-0.185 EUR/kWh) - charging at 10 A (4 neg-price hour(s) remaining, spreading 1.2 kWh over window)` |
| Pre-discharge | `Pre-discharge: neg price in 3h (4 neg hour(s)) - discharging 194 A to free 28.0 kWh over 3h (SoC 85% → 15%)` |
| Solar suppressed during discharge window | `High price (128% of avg, 0.261 EUR) - solar exporting 6329 W, battery idle (surplus > 500 W threshold)` |

### isPriceLow logic (v3.7 / v3.8)

```javascript
var isPriceVeryLow = priceRatio <= CFG.CHARGE_THRESHOLD        // price <= 80% of avg
                  && currentPrice <= chargeAbsMax;             // price <= avgPrice × 0.55

if (batterySoC <= CFG.SOC_CRITICAL && currentPrice < avgPrice && currentPrice <= chargeAbsMax) {
  isPriceLow = true;                     // critical SoC — cheap hour required (v3.8 guard)
} else if (inChargingWindow === null) {
  isPriceLow = isPriceVeryLow || isNegPrice;        // no planner — ratio fallback
} else {
  isPriceLow = inChargingWindow || isPriceVeryLow || isNegPrice; // planner + override
}
```

The `chargeAbsMax` guard on `SOC_CRITICAL` (added v3.8) prevents emergency charging at genuinely expensive hours when early-morning `avgPrice` is skewed by a partial day's data.

### Negative price charging — spread rate with solar offset (v3.10 / v3.11)

```
// Step 1 — spread rate across remaining negative price window
kwhHeadroom     = BATTERY_KWH × ((SOC_MAX − SOC_MIN) / 100) − currentKwh
spreadAmps      = (kwhHeadroom / negPriceHoursRemaining × 1000) / BATTERY_VOLTAGE / ROUND_TRIP_EFF

// Step 2 — offset grid draw with solar available for battery
homeLoadW        = solarTotalW + netGridW
solarForBatteryW = max(0, solarTotalW − homeLoadW)
solarOffsetAmps  = solarForBatteryW / BATTERY_VOLTAGE
gridAmps         = max(0, spreadAmps − solarOffsetAmps)

// Step 3 — clamp
targetAmps = clamp(spreadAmps, MIN_CHARGE_A, maxAllowedChargeA)
```

The spread rate automatically increases as the window shortens — the battery always fills before the negative window closes. `MIN_CHARGE_A: 10` prevents trivially small current commands at near-full SoC.

### Pre-discharge (v3.12)

```javascript
kwhToDischarge = BATTERY_KWH × ((SOC_DISCHARGE_MIN − SOC_MIN) / 100)  // capacity to free
targetKw       = kwhToDischarge / preDischargeHoursCount
preAmps        = clamp(targetKw × 1000 / BATTERY_VOLTAGE, MIN_DISCHARGE_A, MAX_AMPS)
```

Discharge rate is spread across all available pre-discharge hours to reach `SOC_DISCHARGE_MIN` just before the negative window starts.

### isPriceHigh guard (v3.6)

```javascript
if (msg.inDischargeWindow === true) {
  isPriceHigh = currentPrice >= avgPrice && currentPrice >= dischargeAbsMin;
} else {
  isPriceHigh = priceRatio >= dischargeRatioThreshold && currentPrice >= dischargeAbsMin;
}
```

`inDischargeWindow` alone is not sufficient — the price must still be above the daily average. Prevents discharge at below-average hours even when the planner flags them.

### Solar surplus suppression during discharge window (v3.15)

```javascript
} else if (actualSurplusW > solarSurplusThreshold && canChargeSolar && !msg.inDischargeWindow) {
```

When `inDischargeWindow = true`, the solar surplus charging branch is skipped entirely. Solar exports freely to grid; the battery discharges to cover any remaining home load. This prevents the battery from charging from solar surplus just before the planned discharge window.

### Minimum discharge suppression (v3.14)

```javascript
} else if (actualSurplusW < CFG.SOLAR_SUPPRESS_DISCHARGE_W) {
  // export at MIN_DISCHARGE_A
} else {
  // idle — solar already exporting heavily, preserve battery for discharge window
}
```

When `dischargeTarget = 0` (house load covered by solar) and solar surplus exceeds `SOLAR_SUPPRESS_DISCHARGE_W` (500 W), the minimum export discharge is suppressed. Battery preserves charge for the planned discharge window.

### canCharge / canDischarge guards

```javascript
canCharge    = batterySoC < CFG.SOC_MAX          // hard ceiling, no exceptions (v3.9)
canDischarge = batterySoC > CFG.SOC_MIN && batterySoC >= CFG.SOC_DISCHARGE_MIN  // v3.5
```

### Discharge amp calculation (v3.5)

```
dischargeTargetW = (gridImport_W + EXPORT_BIAS_W) × DISCHARGE_OVERSHOOT
dc_amps          = clamp(dischargeTargetW / BATTERY_VOLTAGE, MIN_DISCHARGE_A, MAX_AMPS)
```

`ROUND_TRIP_EFF` is NOT applied — efficiency only applies when charging.

### Charge amp calculation

```
worstPhaseHeadroomA = GRID_MAX_A_PHASE − maxPhaseLoad_A − SAFETY_MARGIN_A
maxAllowedChargeA   = min(MAX_AMPS, round((worstPhaseHeadroomA × GRID_VOLTAGE × 3) / BATTERY_VOLTAGE × ROUND_TRIP_EFF))

fraction   = clamp(0.50 + (1 − priceRatio) × 2.5, 0.10, 1.0)   // v3.13 clamp prevents negative
targetAmps = round(maxAllowedChargeA × fraction)
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
  version:     "EMS v3.15 / Planner v2.13",
  dc_amps:     130,
  charging:    false,
  discharging: true,
  reason:      "High price (128% of avg, 0.261 EUR) - discharging 130 A to cover 4178 W gross import + 800 W export bias",

  planner: {
    inWindow:               false,
    inDischargeWindow:      true,
    inPreDischargeWindow:   false,
    preDischargeHours:      [],
    hoursUntilNegWindow:    null,
    futureNegHoursCount:    0,
    negPriceHoursRemaining: 0,
    dischargeThreshold:     0.244,
    hoursNeeded:            3,
    kwhNeeded:              29.6,
    kwhFromGrid:            8.2,
    solarForecastKwh:       21.4,
    cheapestPrice:          0.060,
    plannedHours:           [12, 13, 14],
    plannedDischargeHours:  [19, 20, 21, 22, 23],
    reason:                 "Need 29.6 kWh total, solar covers ~21.4 kWh, 8.2 kWh from grid (3h). SoC 21% - cheapest hour: 13h (0.060 EUR)..."
  },

  inputs: {
    currentPrice:        0.261,
    avgPrice:            0.205,
    priceRatio:          127.6,
    chargeAbsMax:        0.113,
    dischargeAbsMin:     0.113,
    lastState:           "discharging",
    batterySoC:          22,
    solarTotal_W:        1530,
    netGrid_W:           4178,
    dischargeTarget_W:   4178,
    actualSurplus_W:     0,
    phaseL1_A:           4.05,
    phaseL2_A:           3.84,
    phaseL3_A:           10.28,
    maxPhaseLoad_A:      10.28,
    worstHeadroom_A:     5.72,
    gridHeadroom_W:      3948,
    maxAllowedCharge_A:  70,
    timestamp:           "2026-04-26T17:47:53.852Z"
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
| Charging at expensive hours at low SoC | Old pre-v3.8 script | `SOC_CRITICAL` branch lacks `chargeAbsMax` guard — update to v3.8+ |
| Battery charges above 95% at negative price | Old pre-v3.9 script | `canCharge` had `isNegPrice` override — update to v3.9+ |
| Discharge not stopping at low SoC | Old `canDischarge` bug | Ensure EMS v3.5+ is deployed |
| Solar forecast not deducting from kwhNeeded | `solarForecastRemaining` = 0 | Check `get-solar-forecast` node is in chain; verify `sensor.energy_production_remaining_today` |
| `[confirming 1/2]` in reason | State pending confirmation | Normal — resolves next cycle |
| `maxAllowedCharge_A: 0` | Home load near grid limit | Large appliance consuming headroom — EMS resumes when load drops |
| Charge amps fluctuate with appliances | Expected behaviour | Phase headroom recalculated every cycle — always within grid limit |
| Charging at max during negative price | Old pre-v3.10 script | Update to EMS v3.10+ for spread charging |
| Battery fills too fast during long negative window | `negPriceHoursRemaining` not received | Check Planner v2.10+ deployed; verify `msg.negPriceHoursRemaining` passed |
| Charging from grid when solar is exporting | Old pre-v3.11 script | Solar offset missing — update to EMS v3.11+ |
| `dc_amps: 10` at near-full battery during neg price | Expected — MIN_CHARGE_A floor | Normal; battery nearly full, current intentionally low |
| Charging planned at expensive evening hours | Old pre-v2.12 planner | Charge window validity gate missing — update to Planner v2.12+ |
| Negative amps / wrong Deye value | Old pre-v3.13 EMS | `fraction` not clamped — update to EMS v3.13+; also clamp value before writing to Deye entity |
| Battery discharging before evening window (solar hour) | Old pre-v3.14 script | Min-discharge suppression missing — update to EMS v3.14+ |
| Solar surplus charging during discharge window | Old pre-v3.15 script | Solar branch not gated on `inDischargeWindow` — update to EMS v3.15+ |
| Pre-discharge not firing before negative window | `inPreDischargeWindow` false | Check Planner v2.11+ deployed; verify hours before neg window are above dischargeAbsMin |
| Discharge window wrong size or wrong hours | Old pre-v2.13 planner | Discharge window used band percentage — update to Planner v2.13+ for centered expansion |
| Window biased too far right into solar hours | Old pre-v2.9 planner | Early-hour bias missing — update to Planner v2.9+ |

---

## Version History

| Version | Date | Change |
|---|---|---|
| **EMS v3.15** | 2026-04-28 | Solar surplus charging branch suppressed during active `inDischargeWindow`; discharge now takes priority over surplus absorption in the evening peak |
| **EMS v3.14** | 2026-04-28 | Minimum discharge (no home load) suppressed when solar surplus exceeds `SOLAR_SUPPRESS_DISCHARGE_W` (500 W); preserves battery charge before planned discharge window |
| **EMS v3.13** | 2026-04-26 | `fraction` in scaled charge clamped to [0.10, 1.0] — prevents negative dc_amps when price is above avg; charge window validity gate (`chargeWindowValid`) prevents charging in expensive hours |
| **EMS v3.12** | 2026-04-26 | Pre-discharge branch added — discharges to `SOC_DISCHARGE_MIN` when negative prices are forecast ahead, spread across available hours before the window |
| **EMS v3.11** | 2026-04-26 | Solar surplus offsets grid draw during negative price charging — `homeLoadW = solarTotalW + netGridW`; `solarOffsetAmps` reduces grid contribution to keep netGrid ~0 |
| **EMS v3.10** | 2026-04-26 | Negative price charging spread across full negative window: `spreadAmps = kwhHeadroom / negPriceHoursRemaining / 48V / eff`; ramps up automatically as window closes |
| **EMS v3.9** | 2026-04-26 | `canCharge` hard ceiling — removed `isNegPrice` override; battery never charges above 95% regardless of price |
| **EMS v3.8** | 2026-04-26 | `SOC_CRITICAL` branch requires `currentPrice <= chargeAbsMax`; prevents charging at expensive hours when early-morning `avgPrice` is skewed |
| **EMS v3.7** | 2026-04-25 | `isPriceVeryLow` ratio override added; distinct reason strings for planner window vs ratio override vs critical SoC |
| **EMS v3.6** | 2026-04-25 | `SOC_CRITICAL` check `<` → `<=`; `isPriceHigh` requires `currentPrice >= avgPrice` |
| EMS v3.5 | 2026-04-20 | `canDischarge` bug fixed; `ROUND_TRIP_EFF` removed from discharge calc |
| EMS v3.4 | 2026-04-19 | Fixed price thresholds replaced with `avgPrice` ratios |
| EMS v3.3 | 2026-04-15 | Solar surplus uses `canChargeSolar`; moved before discharge branch |
| EMS v3.2 | 2026-04-12 | `SOC_DISCHARGE_MIN` 15%; gross import as discharge base |
| EMS v3.1 | 2026-04-10 | `SOC_DISCHARGE_MIN` 15%; `SOC_CRITICAL` 20% |
| EMS v3.0 | 2026-04-08 | P1 phase readings used directly for per-phase headroom |
| **Planner v2.13** | 2026-04-28 | Discharge window centered on most expensive hour and expanded outward; sized from actual usable capacity (32 kWh ÷ 9.6 kW = 5h); `LATE_BIAS_EUR` keeps window in evening peak |
| **Planner v2.12** | 2026-04-26 | Charge window validity gate — window only built if `cheapestPrice <= chargePriceCap`; prevents scheduling charges at expensive evening hours |
| **Planner v2.11** | 2026-04-26 | Pre-discharge detection — scans for future negative price hours; flags pre-discharge window hours and passes `inPreDischargeWindow`, `hoursUntilNegWindow`, `futureNegHoursCount` to EMS |
| **Planner v2.10** | 2026-04-26 | Counts remaining negative price hours (`negPriceHoursRemaining`) for EMS spread charge calculation |
| **Planner v2.9** | 2026-04-26 | Early-hour bias (`EARLY_BIAS_EUR = 0.02`) in charge window expansion — prefers earlier hours when price diff is within margin |
| **Planner v2.8** | 2026-04-25 | Solar forecast deducted from `kwhNeeded`; `kwhFromGrid` and `solarForecastKwh` added to outputs |
| **Planner v2.7** | 2026-04-25 | Discharge band filter raised from `dischargeAbsMin` to `avgPrice` |
| **Planner v2.6** | 2026-04-25 | Charge window centered on cheapest hour, expanded outward |
| Planner v2.5 | 2026-04-20 | Price ratios aligned with EMS v3.5 |
| Planner v2.4 | 2026-04-19 | Frank Energie as price source; EnergyZero removed as price source |
| Planner v2.3 | 2026-04-17 | Version as single `PLANNER_VERSION` variable |
| Planner v2.2 | 2026-04-15 | `SOC_NORMAL_THRESHOLD` raised to 80% |
| Planner v2.1 | 2026-04-14 | Fixed `msg.plannerVersion` / `inWindow` undefined bugs |
| Planner v2.0 | 2026-04-13 | `dischargeThreshold` recalculated after conflict filter; discharge hours deduped |
