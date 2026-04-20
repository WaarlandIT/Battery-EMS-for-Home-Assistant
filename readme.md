# Battery EMS – Node-RED Setup & Reference Guide
**EMS v3.5 / Planner v2.5** — Updated 20 April 2026

The EMS decision script and Price Planner are included in the JSON but when these scripts have updates (bug fixes) the JSON will not get updated. 

---

## My Setup

I use a Growatt inverter for my solar array and a Deye inverter for the batteries. 
The battries are not connected when you set this up, that has to be done depending on your setup.
The value of the state of charge is named in this setup but you have to adjust the enity to your needs. 

I used Frank Energie because they have an open API and give you access to the actual sinus of the market pricing of energy in the Netherlands. Even if you use another energy provider these graphs are acurate for the price variations, they all use the same genral market value but adjust the sales price to their needs.  

#### Some nodes are not set correctly because typical values are needed from Home Assistent, they are marked red after import.

---

## Prerequisites

Install in Node-RED via **Manage palette**:
- `node-red-contrib-home-assistant-websocket` — all Home Assistant nodes

---

## Home Assistant Integrations

All integrations listed below must be installed and working in Home Assistant before the EMS flow can function. The entity names in the flow are derived from these integrations.

| Integration | Source | Purpose in EMS |
|---|---|---|
| **Frank Energie** | [github.com/HiDiHo01/home-assistant-frank_energie](https://github.com/HiDiHo01/home-assistant-frank_energie) | Hourly dynamic electricity prices — primary price source for planner and EMS |
| **DSMR Smart Meter** | [home-assistant.io/integrations/dsmr](https://www.home-assistant.io/integrations/dsmr/) | Real-time grid import/export (kW) and per-phase load (kW) via P1 port |
| **Growatt ESPHome** | [github.com/WaarlandIT/ESPHOME-Growatt](https://github.com/WaarlandIT/ESPHOME-Growatt) | Live solar AC output per phase (PAC1/2/3 in W) |
| **Forecast.Solar** | [home-assistant.io/integrations/forecast_solar](https://www.home-assistant.io/integrations/forecast_solar/) | Solar production forecast — available for future enhancements |
| **ha-solarman** | [github.com/davidrapan/ha-solarman](https://github.com/davidrapan/ha-solarman) | Battery state of charge (%) from BMS via Solarman protocol |
| **EnergyZero** | [home-assistant.io/integrations/energyzero](https://www.home-assistant.io/integrations/energyzero/) | Used only as an hourly trigger (price hour change node) — not the price source |

> **Note:** Frank Energie is the authoritative price source since Planner v2.4. EnergyZero is kept only as a trigger entity to fire the flow at the start of each new price hour.

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

> The planner calculates `currentPrice` and `avgPrice` directly from this array and writes them to `msg`, overriding anything from EnergyZero. EnergyZero is only used as an hourly trigger, not as a price source.

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

> AC watts per phase from the Growatt ESPHome integration. Summed as `solarTotalW = pac1 + pac2 + pac3`.

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

After importing, verify the wiring follows this exact order. **Battery SoC must be read before the Price Planner** so it can calculate the correct charge window.

```
[Every 5 min inject]  ──┐
[DSMR power change]   ──┼──► [Frank Energie prices] ──► [Battery SoC] ──► [Price Planner]
[Price hour change]   ──┘                                                        │
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
| Max charge power | ~8.16 kW (200A × 48V × 0.85) | derived |
| Round-trip efficiency | 85% | `ROUND_TRIP_EFF` |
| Full charge time (from SOC_MIN) | ~4.2 h at max rate | derived |
| Grid connection | 3-phase, 20 A fuse | `GRID_MAX_A_PHASE` = 18 (2 A margin) |

---

## Price Planner (v2.5) — How It Works

The planner runs first each cycle and calculates today's optimal charge and discharge schedule.

### Price source
Prices are read directly from the Frank Energie `attributes.prices` array. The planner calculates `currentPrice` and `avgPrice` from this array and overwrites `msg.currentPrice` / `msg.avgPrice` so the EMS always uses Frank Energie pricing regardless of EnergyZero sensor state.

### Charge window calculation

```
usableCapacity = 40 × ((95% − 10%) / 100) = 34 kWh
currentKwh     = 40 × ((SoC% − 10%) / 100)
kwhNeeded      = usableCapacity − currentKwh
hoursNeeded    = ceil(kwhNeeded / 8.16 kW) + 1 safety hour
```

### SoC-aware charge price cap

| SoC | Cap formula | Behaviour |
|---|---|---|
| ≥ 80% | `avgPrice × 0.55` | Tight — only genuinely cheap hours qualify |
| < 80% | `avgPrice × 0.99` | Expanded — any below-average hour qualifies |

This ensures a low battery always finds charge hours even when prices are not dramatically cheap.

### Discharge hour selection
- Morning band: 06h–11h; evening band: 16h–23h
- Top 50% by price from each band, filtered to prices above `avgPrice × 0.55`
- Any discharge hour overlapping a charge hour is removed
- Discharge threshold recalculated from the filtered set

### Planner outputs written to msg

| Property | Type | Description |
|---|---|---|
| `msg.inChargingWindow` | bool / null | true = current hour is a planned charge hour; null = no price data |
| `msg.inDischargeWindow` | bool | true = current hour is a planned discharge hour |
| `msg.dischargeThreshold` | float | Lowest price in selected discharge hours (EUR/kWh) |
| `msg.plannedChargeHours` | int[] | Sorted charge hours, e.g. [12, 13, 14, 15, 16] |
| `msg.plannedDischargeHours` | int[] | Sorted discharge hours, e.g. [7, 8, 19, 20] |
| `msg.hoursNeeded` | int | Hours needed to charge to 95% SoC |
| `msg.kwhNeeded` | float | kWh needed to charge to 95% SoC |
| `msg.cheapestPrice` | float | Cheapest remaining hour today |
| `msg.currentPrice` | string | Current hour price from Frank Energie |
| `msg.avgPrice` | string | Today's average price from Frank Energie |
| `msg.plannerVersion` | string | `'v2.5'` |
| `msg.plannerReason` | string | Human-readable summary of decisions |

---

## EMS Decision Engine (v3.5) — Configuration Reference

### CFG parameters

| Parameter | Value | Description |
|---|---|---|
| `MAX_AMPS` | 200 | Hard cap on DC output amps |
| `BATTERY_VOLTAGE` | 48 | Nominal DC bus voltage (V) |
| `BATTERY_KWH` | 40 | Usable capacity (kWh) |
| `SOC_MIN` | 10% | Discharge floor — never goes below this |
| `SOC_MAX` | 95% | Charge ceiling — stops charging above this (except negative price) |
| `SOC_DISCHARGE_MIN` | 15% | Discharge guard — will not discharge below this even in peak window |
| `SOC_CRITICAL` | 20% | Below this, charge at full amps regardless of price window |
| `CHARGE_THRESHOLD` | 0.80 | Fallback: charge if price ≤ 80% of avg (no planner data) |
| `DISCHARGE_THRESHOLD` | 1.20 | Fallback: discharge if price ≥ 120% of avg (no planner data) |
| `CHARGE_ABS_RATIO` | 0.55 | Charge abs max = `avgPrice × 0.55` |
| `DISCHARGE_ABS_RATIO` | 0.55 | Discharge abs min = `avgPrice × 0.55` |
| `DISCHARGE_OVERSHOOT` | 1.25 | 25% overshoot on discharge target to compensate for inverter lag |
| `EXPORT_BIAS_W` | 800 | Watts added to discharge target to push net flow toward export |
| `HYSTERESIS` | 0.05 | Price ratio band to prevent rapid toggling at thresholds |
| `SOLAR_SURPLUS_W` | 500 | Min solar export surplus (W) to start solar charging |
| `SOLAR_SURPLUS_EXIT_W` | 200 | Min solar export surplus (W) to keep solar charging going |
| `ROUND_TRIP_EFF` | 0.85 | Charging efficiency — applied to **charge amps only**, NOT to discharge |
| `GRID_MAX_A_PHASE` | 18 | Usable amps per phase (20 A fuse − 2 A margin) |
| `GRID_VOLTAGE` | 230 | AC grid voltage |
| `SAFETY_MARGIN_A` | 2 | Additional per-phase headroom buffer |
| `MIN_DISCHARGE_A` | 35 | Minimum meaningful discharge current (A) |

### Decision priority (highest to lowest)

```
1. Negative price AND canCharge         → charge at maxAllowedChargeA
2. Low price / planner window AND       → charge at scaled amps
   canCharge                              (scales 50%–100% of max by price discount;
                                           full max amps at critical SoC < 20%)
3. Solar surplus > threshold AND        → absorb solar surplus into battery
   canChargeSolar                         (always before discharge — free energy first)
4. High price / discharge window AND    → discharge to cover import + export bias
   canDischarge
5. None of the above                    → idle
```

### Discharge amp calculation (corrected in v3.5)

```
grossTargetW      = gridImport_W + EXPORT_BIAS_W
dischargeTargetW  = grossTargetW × DISCHARGE_OVERSHOOT    ← applied once only
dc_amps           = clamp(dischargeTargetW / BATTERY_VOLTAGE, MIN_DISCHARGE_A, MAX_AMPS)
```

Example (import 3698 W, bias 800 W, overshoot 1.25, voltage 48 V):
```
(3698 + 800) × 1.25 ÷ 48 = 117 A
```

> `ROUND_TRIP_EFF` is **not** applied here. It is only used in the charge headroom calculation. Applying it to discharge inflated amps by ~18% in v3.4 and earlier (bug fix in v3.5).

### Charge amp calculation

```
worstPhaseHeadroomA = GRID_MAX_A_PHASE − maxPhaseLoad_A − SAFETY_MARGIN_A
gridHeadroomW       = worstPhaseHeadroomA × GRID_VOLTAGE × 3
maxAllowedChargeA   = min(MAX_AMPS, round(gridHeadroomW / BATTERY_VOLTAGE × ROUND_TRIP_EFF))
```

For scaled charging (cheap window, SoC ≥ SOC_CRITICAL):
```
priceDiscount = 1 − priceRatio
fraction      = 0.50 + (priceDiscount × 2.5)
targetAmps    = round(maxAllowedChargeA × fraction)
```

### canDischarge guard (v3.5 fix)

```javascript
// v3.5 — correct:
canDischarge = batterySoC > SOC_MIN && batterySoC >= SOC_DISCHARGE_MIN
// Effectively: SoC >= 15%

// v3.4 and earlier — bug:
canDischarge = (batterySoC > SOC_MIN || batterySoC > SOC_MAX) && batterySoC >= SOC_DISCHARGE_MIN
// || SOC_MAX was almost never true but the || made the first condition unreliable
```

### State confirmation / debounce

| Trigger type | Confirm cycles | Effect |
|---|---|---|
| Price signal (cheap/high/negative/solar surplus) | 1 | Immediate state change |
| All other signals | 2 | One confirmation cycle required |

Reason string shows `[confirming N/2]` while pending. This is normal behaviour.

---

## Diagnostic Output Structure

```
msg.payload = {
  version:     "EMS v3.5 / Planner v2.5",
  dc_amps:     117,
  charging:    false,
  discharging: true,
  reason:      "High price (126% of avg, 0.318 EUR) - discharging 117 A to cover 3698 W gross import + 800 W export bias",

  planner: {
    inWindow:              false,
    inDischargeWindow:     true,
    dischargeThreshold:    0.286,
    hoursNeeded:           5,
    kwhNeeded:             32.0,
    cheapestPrice:         0.174,
    plannedHours:          [12, 13, 14, 15, 16],
    plannedDischargeHours: [7, 8, 19, 20],
    reason:                "Need 32.0 kWh (5h). SoC 15% - charge cap: 0.250 EUR (expanded). Avg: 0.252 EUR. ..."
  },

  inputs: {
    currentPrice:        0.318,
    avgPrice:            0.252,
    priceRatio:          126,        // integer %
    chargeAbsMax:        0.139,      // avgPrice × CHARGE_ABS_RATIO
    dischargeAbsMin:     0.139,      // avgPrice × DISCHARGE_ABS_RATIO
    lastState:           "discharging",
    batterySoC:          15,
    solarTotal_W:        676,
    netGrid_W:           3698,       // positive = importing
    dischargeTarget_W:   3698,       // gross import kW × 1000 (pre-bias reference)
    actualSurplus_W:     0,
    phaseL1_A:           6.56,
    phaseL2_A:           4.20,
    phaseL3_A:           5.31,
    maxPhaseLoad_A:      6.56,
    worstHeadroom_A:     9.44,
    gridHeadroom_W:      6513,
    maxAllowedCharge_A:  115,
    timestamp:           "2026-04-20T05:38:03.555Z"
  }
}
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Always idle, dc_amps 0 | Wrong entity names | Use Developer Tools → States to verify entity IDs |
| `priceRatio` always 100% | `avgPrice` = 0 | Check Frank Energie entity has `attributes.prices` populated |
| `inChargingWindow: null` | Frank Energie array empty | Check planner node status — yellow = no price data |
| SoC not updating | Wrong SoC entity | Replace `sensor.battery_state_of_charge` with your BMS entity |
| Discharge amps too high | Old v3.4 script still running | Check version string — must show `EMS v3.5` |
| Discharge not stopping at low SoC | Old `canDischarge` bug | Ensure EMS v3.5 is deployed |
| `[confirming 1/2]` in reason | State pending confirmation | Normal — resolves on next cycle |
| `maxAllowedCharge_A: 0` | Home load near grid limit | High appliance load consuming full phase headroom — EMS resumes automatically when load drops |
| Battery discharging during solar export | Solar branch not before discharge in code | Ensure EMS v3.3+ is deployed — check solar surplus check appears before `isPriceHigh` block |

---

## Version History

| Version | Date | Change |
|---|---|---|
| **EMS v3.5** | 2026-04-20 | Fixed `canDischarge` (`\|\|` → `&&`); removed `ROUND_TRIP_EFF` from discharge amp calc (was inflating amps ~18%) |
| EMS v3.4 | 2026-04-19 | All fixed price thresholds replaced with ratios of daily avg (`CHARGE_ABS_RATIO`, `DISCHARGE_ABS_RATIO`) |
| EMS v3.3 | 2026-04-15 | Solar surplus uses `canChargeSolar`; moved before discharge branch in priority |
| EMS v3.2 | 2026-04-12 | `SOC_DISCHARGE_MIN` 15%; gross grid import as discharge base |
| EMS v3.1 | 2026-04-10 | `SOC_DISCHARGE_MIN` 15%; `SOC_CRITICAL` 20% |
| EMS v3.0 | 2026-04-08 | P1 phase readings used directly for per-phase headroom |
| **Planner v2.5** | 2026-04-20 | Price ratios aligned with EMS v3.5 |
| Planner v2.4 | 2026-04-19 | `currentPrice`/`avgPrice` from Frank Energie array; EnergyZero no longer price source |
| Planner v2.3 | 2026-04-17 | Version as single `PLANNER_VERSION` variable |
| Planner v2.2 | 2026-04-15 | `SOC_NORMAL_THRESHOLD` raised to 80% |
| Planner v2.1 | 2026-04-14 | Fixed `msg.plannerVersion` / `inWindow` undefined bugs |
| Planner v2.0 | 2026-04-13 | `dischargeThreshold` recalculated after conflict filter; discharge hours deduped |
