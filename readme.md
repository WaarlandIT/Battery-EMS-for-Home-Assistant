# Battery EMS – Node-RED Setup & Reference Guide

**EMS v3.74 / Planner v2.14 / EV v2.5** — Updated 23 May 2026

[![Node-red layout](https://github.com/WaarlandIT/Battery-EMS-for-Home-Assistant/raw/main/NodeRed.png)](NodeRed.png)

---

## My Setup

This flow was built around a **Growatt inverter** for the solar array and a **Deye inverter** for the battery bank. A **Zaptec Go EV charger** is also integrated — smart price-based EV charging runs as part of the same flow. The battery inverter connection is not included in this flow — how the DC amps, charging, and discharging booleans are wired to your inverter depends entirely on your hardware. The entity names used here reflect my installation; adjust them to match yours.

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
| --- | --- | --- |
| **Frank Energie** | [github.com/HiDiHo01/home-assistant-frank_energie](https://github.com/HiDiHo01/home-assistant-frank_energie) | Hourly dynamic electricity prices — primary price source for planner and EMS |
| **DSMR Smart Meter** | [home-assistant.io/integrations/dsmr](https://www.home-assistant.io/integrations/dsmr/) | Real-time grid import/export (kW) and per-phase load (kW) via P1 port |
| **Growatt ESPHome** | [github.com/WaarlandIT/ESPHOME-Growatt](https://github.com/WaarlandIT/ESPHOME-Growatt) | Live solar AC output per phase (PAC1/2/3 in W) |
| **Forecast.Solar** | [home-assistant.io/integrations/forecast_solar](https://www.home-assistant.io/integrations/forecast_solar/) | Solar production forecast — used by planner to reduce grid charge hours, detect pre-discharge opportunities, and prevent solar overfill |
| **ha-solarman** | [github.com/davidrapan/ha-solarman](https://github.com/davidrapan/ha-solarman) | Battery state of charge (%) from BMS via Solarman protocol |
| **EnergyZero** | [home-assistant.io/integrations/energyzero](https://www.home-assistant.io/integrations/energyzero/) | Used only as an hourly trigger — not the price source |
| **Zaptec** | [github.com/custom-components/zaptec](https://github.com/custom-components/zaptec) | EV charger control — smart price-based charging, solar surplus absorption, phase overload protection |

> Frank Energie is the authoritative price source since Planner v2.4. EnergyZero is kept only as a trigger to fire the flow at the start of each new price hour.

---

## Step 1 – Home Assistant Helpers

Create these in **Settings → Devices & Services → Helpers** before importing the flow:

| Type | Entity ID | Min | Max | Step |
| --- | --- | --- | --- | --- |
| `input_number` | `input_number.battery_dc_amps` | 0 | 200 | 1 |
| `input_boolean` | `input_boolean.battery_charging` | — | — | — |
| `input_boolean` | `input_boolean.battery_discharging` | — | — | — |

These are the three outputs the EMS writes every cycle. Your battery inverter integration reads from these.

---

## Step 2 – Sensor Entity Name Mapping

Verify these in the relevant `api-current-state` nodes after importing. Use **Developer Tools → States** to find your exact entity names.

### Frank Energie (price source) — [GitHub](https://github.com/HiDiHo01/home-assistant-frank_energie)

| Node | Entity ID | Output |
| --- | --- | --- |
| Frank Energie prijzen | `sensor.frank_energie_prijzen_huidige_elektriciteitsprijs_all_in` | `msg.frankPrices` (attributes.prices array) |

> The planner calculates `currentPrice` and `avgPrice` directly from this array, overriding anything from EnergyZero.

### DSMR Smart Meter — [HA Docs](https://www.home-assistant.io/integrations/dsmr/)

| Node | Entity ID | Output |
| --- | --- | --- |
| Grid import kW | `sensor.dsmr_reading_electricity_currently_delivered` | `msg.gridImport` (kW) |
| Grid export kW | `sensor.dsmr_reading_electricity_currently_returned` | `msg.gridExport` (kW) |
| Phase L1 kW | `sensor.dsmr_reading_phase_currently_delivered_l1` | `msg.phaseL1` |
| Phase L2 kW | `sensor.dsmr_reading_phase_currently_delivered_l2` | `msg.phaseL2` |
| Phase L3 kW | `sensor.dsmr_reading_phase_currently_delivered_l3` | `msg.phaseL3` |

> Entity names vary by DSMR version. Search `dsmr` in Developer Tools → States to confirm yours.

### Growatt Solar (AC output per phase) — [GitHub](https://github.com/WaarlandIT/ESPHOME-Growatt)

| Node | Entity ID | Output |
| --- | --- | --- |
| Solar PAC1 W | `sensor.growatt_pac1` | `msg.pac1` |
| Solar PAC2 W | `sensor.growatt_pac2` | `msg.pac2` |
| Solar PAC3 W | `sensor.growatt_pac3` | `msg.pac3` |

> AC watts per phase summed as `solarTotalW = pac1 + pac2 + pac3`.

### Forecast.Solar — [HA Docs](https://www.home-assistant.io/integrations/forecast_solar/)

| Node | Entity ID | Output |
| --- | --- | --- |
| Solar forecast remaining | `sensor.energy_production_remaining_today` | `msg.solarForecastRemaining` (kWh) |

> The planner deducts this from `kwhNeeded` to reduce grid charge hours when solar will cover part of the charge. Also used to detect when solar will fill the battery before the cheap window opens. Falls back safely to 0 if unavailable.

### Battery SoC — [ha-solarman GitHub](https://github.com/davidrapan/ha-solarman)

| Node | Entity ID | Output |
| --- | --- | --- |
| Battery SoC | `sensor.battery_state_of_charge` | `msg.batterySoC` (%) |

> Replace with your actual BMS/inverter SoC entity. For the StephanJoubert solarman integration use `sensor.deye_battery_soc`.

### Zaptec EV Charger — [GitHub](https://github.com/custom-components/zaptec)

| Node | Entity ID | Output / Action |
| --- | --- | --- |
| Get Zaptec mode | `sensor.sloeierd_charger_mode` | `msg.zaptecMode` (str) |
| Get EV charge load | `sensor.sloeierd_laadvermogen` | `msg.EVChargeLoad` (W, total 3-phase) |
| Set Zaptec current | `number.sloeierd_max_stroom` | Sets charge current per phase (6–10 A) |
| Set Zaptec switch on | `switch.sloeierd_opladen` | `switch.turn_on` — starts charging session |
| Set Zaptec switch off | `switch.sloeierd_opladen` | `switch.turn_off` — stops charging session |

> Replace `sloeierd` with your charger's entity prefix. Find it by searching `zaptec` in Developer Tools → States. `sensor.sloeierd_laadvermogen` reports total 3-phase power in watts — used to accurately subtract EV load from grid import. The minimum charge current for Zaptec is 6A; setting to 0 does not pause — use the switch instead.

### Trigger nodes — [EnergyZero HA Docs](https://www.home-assistant.io/integrations/energyzero/)

| Node | Entity ID | Purpose |
| --- | --- | --- |
| DSMR power change | `sensor.electricity_meter_energieverbruik` | Fire instantly on load change |
| Price hour change | `sensor.energyzero_today_energy_current_hour_price` | Fire on hourly price rollover |

---

## Step 3 – Node Chain Execution Order

**Battery SoC, Solar Forecast, and Zaptec mode must all be read before the EMS Decision Engine.**

```
[Every 5 min inject]  ──┐
[DSMR power change]   ──┼──► [Frank Energie prices] ──► [Battery SoC] ──► [Solar forecast] ──► [Set solarNow] ──► [Price Planner]
[Price hour change]   ──┘                                                                                               │
                                                                                                              [Grid import kW]
                                                                                                              [Grid export kW]
                                                                                                               [Phase L1 kW]
                                                                                                               [Phase L2 kW]
                                                                                                               [Phase L3 kW]
                                                                                                              [Solar PAC1 W]
                                                                                                              [Solar PAC2 W]
                                                                                                              [Solar PAC3 W]
                                                                                                           [Get Zaptec mode]
                                                                                                         [EMS Decision Engine]
                                                                                                          (outputs: 2)
                                                                              ┌──────────────────────────────────┘ └────────────────────┐
                                                                    [Split outputs]                                          [Set Zaptec current]
                                                        ┌───────────────┬───────────┐                                        [EV Diagnostics]
                                                [Set dc_amps] [Set charging] [Set discharging]
                                                                    [EMS Diagnostics]
```

**Important:** The EMS Decision Engine function node must be configured with **2 outputs** in Node-RED. Output 1 carries the battery message to `Split outputs`. Output 2 carries the EV message to the **EV Output Router** function node.

The EV Output Router (3 outputs) handles rate limiting and splits the EV message into:

- Output 1 → Switch node → `Turn ON` / `Turn OFF` (`switch.sloeierd_opladen`)
- Output 2 → `Set Zaptec current` (`number.sloeierd_max_stroom`, data `{"value": {{zaptecValue}}}`)
- Output 3 → `EV Diagnostics` debug node

### Set solarNow function node

Add a small function node between Solar PAC3 and Price Planner to pass current solar production to the planner for overfill detection:

```javascript
msg.solarNow = (parseFloat(msg.pac1) || 0)
             + (parseFloat(msg.pac2) || 0)
             + (parseFloat(msg.pac3) || 0);
return msg;
```

### Get EV charge load node

Add an `api-current-state` node between Solar PAC3 and `Get Zaptec mode` to read actual EV power consumption:

- Entity: `sensor.sloeierd_laadvermogen`
- Output property: `msg.EVChargeLoad` (num)

This gives the EMS the actual measured 3-phase EV load in watts, which is subtracted from `netGridW` and `dischargeTargetW` so the battery never compensates for intentional EV grid consumption.

---

## Step 4 – Trigger Configuration

| Trigger | Interval | Purpose |
| --- | --- | --- |
| Every 5 min inject | 300 s | Baseline heartbeat |
| DSMR power change | Every DSMR update (~2–10 s) | React instantly to home load changes |
| Price hour change | Every hour | React immediately when price rolls to next hour |

In both `server-state-changed` nodes, ensure:

- **Only send if state changes** — enabled
- **Ignore unavailable/unknown** — enabled on both incoming and outgoing state

---

## Step 5 – Battery Specs (verify in both scripts)

| Spec | Value | Config key |
| --- | --- | --- |
| Capacity | 40 kWh | `BATTERY_KWH` |
| Voltage | 48 V DC | `BATTERY_VOLTAGE` |
| Max charge/discharge | 100 A | `MAX_AMPS` |
| Max charge power | ~4.1 kW (100 A × 48 V × 0.85) | derived |
| Max discharge power | 4.8 kW (100 A × 48 V / 1000) | derived |
| Round-trip efficiency | 85% | `ROUND_TRIP_EFF` |
| Usable discharge capacity | 32 kWh (SoC 95% → 15%) | derived |
| Grid connection | 3-phase, 20 A fuse | `GRID_MAX_A_PHASE` = 18 (2 A margin) |

---

## Price Planner (v2.14) — How It Works

The planner runs first each cycle and calculates today's optimal charge and discharge schedule before any sensor data is read.

### Price source

Prices are read from the Frank Energie `attributes.prices` array. `currentPrice` and `avgPrice` are calculated from this array and written to `msg`, overriding anything from EnergyZero.

### kWh needed and solar deduction

```
usableCapacity = 40 × ((95% − 10%) / 100)  = 34 kWh
currentKwh     = 40 × ((SoC% − 10%) / 100)
kwhNeeded      = usableCapacity − currentKwh
solarUsable    = solarForecastRemaining × 0.90
```

Solar is split into before and after the cheap window using current production rate:

```
solarBeforeCheapWindow = solarNow_kW × hoursUntilCheapWindow
solarAfterCheapWindow  = max(0, solarUsable − solarBeforeCheapWindow)
kwhFromGrid    = max(0, kwhNeeded − solarAfterCheapWindow)
hoursNeeded    = ceil(kwhFromGrid / 8.16 kW) + 1 safety hour
```

Only solar arriving **after** the cheap window offsets grid charging — solar before the window fills the battery independently.

### Solar overfill pre-discharge (v2.14)

If current solar production rate × hours until cheap window exceeds available battery headroom, the planner flags pre-discharge hours to free capacity before the solar peak. This ensures the cheap grid window can actually be used rather than being skipped because the battery is already full from morning solar.

```
solarWillOverfill = solarBeforeCheapWindow >= currentHeadroom
kwhToFree        = max(kwhFromGrid, MIN_GRID_CHEAP_KWH=3.0 kWh)
targetSoC        = calculated to absorb kwhToFree before cheap window
```

### Charge window — centered on cheapest hour

The planner finds the single cheapest remaining hour and expands outward until `hoursNeeded` hours are filled, always preferring the cheaper neighbour. An **early-hour bias** (`EARLY_BIAS_EUR = 0.02`) keeps grid charging before the solar peak when neighbour prices are close.

The window is only built if `cheapestPrice <= avgPrice × 0.55`. If all remaining hours are expensive, `plannedChargeHours` is empty and the EMS falls back to ratio-based logic.

### Discharge window — centered on most expensive hour

```
kwhToDischarge       = 40 × ((95 − 15) / 100) = 32 kWh
dischargeHoursNeeded = ceil(32 / 9.6 kW) + 1  = 5h
```

The most expensive remaining hour above `avgPrice` becomes the center. The window expands outward with a **late-hour bias** (`LATE_BIAS_EUR = 0.02`) to keep it anchored in the evening peak. Hours overlapping the charge window are removed.

### Pre-discharge before negative price windows (v2.11)

When negative price hours are forecast later today, the planner flags hours before the negative window as pre-discharge candidates — so the battery is empty and ready to charge for free.

### Planner config constants

| Constant | Value | Description |
| --- | --- | --- |
| `BATTERY_KWH` | 40 | Battery capacity |
| `SOC_MIN` | 10% | Minimum SoC floor |
| `SOC_MAX` | 95% | Maximum SoC ceiling |
| `SOC_NORMAL_THRESHOLD` | 80% | Above this uses tight charge cap |
| `CHARGE_ABS_RATIO` | 0.55 | Tight charge cap = `avgPrice × 0.55` |
| `DISCHARGE_ABS_RATIO` | 0.55 | Discharge abs min = `avgPrice × 0.55` |
| `SOLAR_FORECAST_EFF` | 0.90 | 10% margin on solar forecast |
| `SAFETY_BUFFER_H` | 1 | Extra hour added to `hoursNeeded` |
| `ROUND_TRIP_EFF` | 0.85 | Max charge kW rate calculation |
| `EARLY_BIAS_EUR` | 0.02 | Prefer earlier hours in charge window expansion |
| `LATE_BIAS_EUR` | 0.02 | Prefer later hours in discharge window expansion |
| `MAX_DISCHARGE_KW` | 9.6 | Max discharge rate for window sizing |
| `SOC_DISCHARGE_MAX` | 95% | SoC from which discharge is measured |
| `SOC_DISCHARGE_MIN_PL` | 15% | SoC floor used in discharge window sizing |
| `SOLAR_OVERFILL_RATIO` | 1.0 | Solar/headroom ratio threshold for overfill detection |
| `MIN_GRID_CHEAP_KWH` | 3.0 | Minimum kWh to always absorb at cheapest price |

---

## EMS Decision Engine (v3.74) — Configuration Reference

### CFG parameters

| Parameter | Value | Description |
| --- | --- | --- |
| `MAX_AMPS` | 100 | Hard cap on DC output amps (safe limit) |
| `BATTERY_VOLTAGE` | 48 | Nominal DC bus voltage (V) |
| `BATTERY_KWH` | 40 | Usable capacity (kWh) |
| `SOC_MIN` | 10% | Discharge floor |
| `SOC_MAX` | 95% | Hard charge ceiling — no exceptions |
| `SOC_DISCHARGE_MIN` | 15% | Discharge guard — never discharges below this |
| `SOC_CRITICAL` | 15% | At or below this, charge at full amps if price is genuinely cheap |
| `CHARGE_THRESHOLD` | 0.80 | Ratio for price override and fallback charging (`isPriceVeryLow`) |
| `DISCHARGE_THRESHOLD` | 1.20 | Fallback discharge if price ≥ 120% of avg (no planner) |
| `CHARGE_ABS_RATIO` | 0.55 | `chargeAbsMax` = `avgPrice × 0.55` |
| `DISCHARGE_ABS_RATIO` | 0.55 | `dischargeAbsMin` = `avgPrice × 0.55` |
| `DISCHARGE_OVERSHOOT` | 1.25 | 25% overshoot on discharge target for inverter lag |
| `EXPORT_BIAS_W` | 800 | Watts added to discharge target to push toward export |
| `HYSTERESIS` | 0.05 | Price ratio band to prevent rapid toggling |
| `SOLAR_SURPLUS_W` | 500 | Min solar export surplus (W) to start solar charging |
| `SOLAR_SURPLUS_EXIT_W` | 200 | Min solar export surplus (W) to keep solar charging |
| `SOLAR_SUPPRESS_DISCHARGE_W` | 500 | Suppress min-discharge when solar surplus exceeds this |
| `ROUND_TRIP_EFF` | 0.85 | Charging efficiency — charge amps only, NOT discharge |
| `GRID_MAX_A_PHASE` | 18 | Usable amps per phase (20 A fuse − 2 A margin) |
| `GRID_VOLTAGE` | 230 | AC grid voltage |
| `SAFETY_MARGIN_A` | 2 | Per-phase headroom buffer |
| `MIN_DISCHARGE_A` | 35 | Minimum meaningful discharge current (A) |
| `MIN_CHARGE_A` | 10 | Minimum charge current — floor for negative price spread |
| `EV_RATE_LIMIT_MS` | 900000 | 15 min between Zaptec writes (API rate limit) |
| `EV_PUBLIC_RATE` | 0.50 | Public charger reference rate (€/kWh) for cost comparison |

### Decision priority — Battery (highest to lowest)

```
1. Negative price AND canCharge          → spread charge across negative window;
                                           solar offsets grid draw to keep netGrid ~0

2. isPriceLow AND canCharge              → charge at scaled amps
     a) SoC <= SOC_CRITICAL AND cheap    → full amps, critical recovery
     b) inChargingWindow = true          → scaled amps
     c) isPriceVeryLow (ratio ≤ 0.80     → scaled amps, outside planned window
        AND price ≤ avgPrice × 0.55)

3. Solar surplus AND NOT inDischargeWindow → absorb surplus (skipped during discharge window)

4. inSolarPreDischargeWindow             → pre-discharge before solar overfills battery

5. inPreDischargeWindow                  → pre-discharge before negative price window

6. isPriceHigh AND canDischarge
     a) dischargeTarget > 0             → cover gross import + export bias
     b) dischargeTarget = 0 AND         → export at MIN_DISCHARGE_A
        surplus < SOLAR_SUPPRESS_W        (suppressed if solar already exporting)

7. Idle
```

### EV load subtraction

When the EV is charging from grid, the DSMR meter reports that as home import. Without correction the battery would discharge to compensate for the intentional EV load. The EMS subtracts EV charging power from both `netGridW` and `dischargeTargetW` each cycle:

```
evChargingW      = lastEvAmps × GRID_VOLTAGE × 3   // e.g. 6A × 230V × 3 = 4140W
netGridW         = max(0, (gridImport − gridExport) × 1000 − evChargingW)
dischargeTargetW = max(0, gridImport × 1000 − evChargingW)
```

`lastEvAmps` is stored in context by the EV controller each cycle and read back by the EMS on the next cycle.

---

## EV Charge Controller (v2.5) — How It Works

The EV controller runs at the end of the EMS Decision Engine function node. It shares all EMS calculated values and outputs a second message for the Zaptec charger.

### Phase overload protection (v2.2)

The controller reads raw DSMR phase amps each cycle (including EV load — no add-back needed since DSMR already sees the full meter):

- If any phase exceeds **20A** and dropping to 6A L1 would resolve it → **soft cut**: cap to 6A single-phase, keep session alive
- If any phase exceeds **20A** and dropping to 6A still doesn't resolve it → **session cut**: drop to 5A and start a 10-poll cooldown before any ramp

### Ramp-up behaviour (v2.4)

After a cut, or when stepping up from 8A to 9A or 10A at night, the controller requires **10 consecutive clear polls (~5 minutes)** of stable headroom before adding 1A. The ramp counter only resets on an actual soft cut — minor phase fluctuations below 20A do not wipe accumulated stable time.

### EV charging rules

| Priority | Condition | EV amps | Phase mode |
| --- | --- | --- | --- |
| 1 | Car not connected / full | 0 | — |
| 2 | Phase overload, 6A resolves it | 6A | L1 single-phase (soft cut) |
| 3 | Phase overload, 6A doesn't resolve | 5A | L1 single-phase (session cut) |
| 4 | Post-cut cooldown | 6A | L1 single-phase |
| 5 | Blackout 17h–19h | 5A | L1 single-phase (session kept alive) |
| 6 | Night 23h–06h | 8–10A | 3-phase (slow ramp, 5 min per step) |
| 7 | Battery discharging | 6A | L1 single-phase (baseline only) |
| 8 | Price zero or negative (daytime) | 6–10A | 3-phase (slow ramp) |
| 9 | **Price very low — `isPriceVeryLow` (daytime)** | **6–8A** | **3-phase (slow ramp, max 8A)** |
| 10 | Solar surplus ≥ threshold | scaled | 3-phase |
| 11 | Default baseline | 6A | L1 single-phase |

**Key rules:**

- The EV **never charges from the battery** — EV load is subtracted from `netGridW` so the battery only sees home load
- **Blackout window 17h–19h** — EV drops to 5A (session kept alive) during cooking peak regardless of price or solar
- **Night 23h–06h** — EV charges at minimum 8A, up to 10A if headroom allows, using the slow 5-minute ramp per 1A step
- **Very low price daytime (v2.5)** — when `isPriceVeryLow` (price ≤ 80% of avg AND ≤ `avgPrice × 0.55`) the EV can use 3-phase up to **8A** (not 10A) if headroom allows; same slow ramp applies
- **3-phase load** — Zaptec Go charges on all 3 phases simultaneously. `evChargingW = evAmps × 230V × 3`
- Zaptec writes are rate-limited to once per 15 minutes or on state change — the Zaptec API dislikes frequent calls
- **Pausing** uses `switch.sloeierd_opladen` (off) not 0A — Zaptec minimum is 6A so 0A does not pause

### EV session cost tracking

```
evSessionKwh     — kWh charged this session (resets on disconnect)
evSessionCostAct — actual cost at dynamic price (€)
evSessionCostPub — cost at public rate 0.50 €/kWh (€)
evSessionSaving  — saving vs public charger (€)
evInstantCostAct — current cost rate (€/h)
evInstantCostPub — public rate cost rate (€/h)
```

---

## Diagnostic Output Structure — Battery (output 1)

```javascript
msg.payload = {
  version:     "EMS v3.74 / Planner v2.14",
  dc_amps:     0,        // amps when charging, 0 when discharging
  dc_power:    1680,     // watts when discharging (35A × 48V), 0 when charging
  charging:    false,
  discharging: true,
  reason:      "High price (131% of avg, 0.289 EUR) - discharging 35 A to cover 1143 W gross import (no bias, solar active) (raised to minimum 35 A)",
  planner: { ... },
  inputs: {
    currentPrice:        0.289,
    avgPrice:            0.221,
    batterySoC:          63,
    solarTotal_W:        0,
    netGrid_W:           0,
    evCharging_W:        1223,
    evLastAmps:          6,
    phaseL1_A:           0.65,
    phaseL2_A:           4.32,
    phaseL3_A:           0.00,
    worstHeadroom_A:     11.68,
    maxAllowedCharge_A:  100,
    ...
  }
}
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Always idle, dc_amps 0 | Wrong entity names | Use Developer Tools → States to verify all entity IDs |
| `priceRatio` always 100% | `avgPrice` = 0 | Check Frank Energie entity has `attributes.prices` populated |
| `inChargingWindow: null` | Frank Energie array empty | Check planner node status — yellow dot = no price data |
| SoC not updating | Wrong SoC entity | Replace `sensor.battery_state_of_charge` with your BMS entity |
| `[confirming 1/2]` in reason | State pending confirmation | Normal — resolves next cycle |
| `maxAllowedCharge_A: 0` | Home load near grid limit | Large appliance consuming headroom — EMS resumes when load drops |
| Solar forecast not deducting | `solarForecastRemaining` = 0 | Check `get-solar-forecast` node; verify `sensor.energy_production_remaining_today` |
| Battery discharging to cover EV load | Old pre-v3.18 EMS | EV load subtraction missing — update to EMS v3.18+ |
| EV always 0A despite cheap price | EMS node set to 1 output | Change EMS function node outputs to 2; wire output 2 to EV router |
| EV not pausing despite 0A command | Zaptec 6A minimum | Use `switch.sloeierd_opladen` off to actually pause |
| EV charges during cooking hour | Old pre-v3.24 EMS | Blackout window 17h-19h missing — update to v3.24+ |
| EV not charging overnight | Old pre-v3.27 EMS | Mandatory night charging missing — update to v3.27+ |
| EV oscillating 8A/10A every 30s | Old pre-v3.73 EMS | Ramp too fast — update to EMS v3.73+ (EV v2.4) |
| EV not charging at low daytime price | Old pre-v3.74 EMS | Very low price 3-phase missing — update to EMS v3.74+ (EV v2.5) |
| EV load not subtracted correctly | Single-phase calculation | EMS v3.19+ uses `× 3` for 3-phase; verify `sensor.sloeierd_laadvermogen` reports total W |
| Solar surplus not detected while EV charging | Old pre-v3.45 script | EV load clamp hid surplus — update to EMS v3.45+ |

---

## Version History

| Version | Date | Change |
| --- | --- | --- |
| **EMS v3.74 / EV v2.5** | 2026-05-23 | Daytime 3-phase EV charging up to 8A when `isPriceVeryLow` (price ≤ 80% of avg AND ≤ `avgPrice × 0.55`) and phase headroom allows; uses same 5-min slow ramp as night charging — won't jump straight to 8A |
| **EMS v3.73 / EV v2.4** | 2026-05-23 | `EV_RAMP_UP_POLLS` raised 4 → 10 (~5 min per 1A step); ramp counter now only resets on `l1SoftCut` (actual EV-caused overload) instead of any phase crossing 20A — eliminates 8A/10A oscillation when house load sits near headroom boundary |
| **EMS v3.72 / EV v2.3** | 2026-05-23 | Daytime 3-phase EV ramp restricted to price ≤ 0 only; previously fired at `isPriceVeryLow` which was too generous |
| **EMS v3.71 / EV v2.2** | 2026-05-22 | Raw DSMR phase amps used directly — no EV add-back; simplified overload: soft cut (6A L1) if EV caused it, session cut (5A) only if 6A doesn't resolve |
| **EMS v3.70 / EV v2.1** | 2026-05-21 | Session cut only when house load alone >20A; slow ramp after session cut (`EV_RAMP_UP_POLLS` consecutive clear polls per 1A); `evCutCooldown` and `evRampClearCount` diagnostics |
| **EMS v3.69 / EV v2.0** | 2026-05-20 | `evCarFull` sends 0A — Zaptec manages session when car is full |
| **EMS v3.45** | 2026-05-09 | Solar surplus detection uses `rawNetGridW` before EV clamp — fixes surplus not detected when EV charging pushes `netGridW` to 0 |
| **EMS v3.44** | 2026-05-09 | EV session cost tracking: `evSessionKwh`, `evSessionCostAct`, `evSessionCostPub`, `evSessionSaving`; `EV_PUBLIC_RATE = 0.50` €/kWh reference |
| **EMS v3.43** | 2026-05-09 | `dc_amps` only set when charging (0 when discharging); `dc_power` only set when discharging (0 when charging) |
| **EMS v3.42** | 2026-05-09 | `MAX_AMPS` reduced to 100A safe limit |
| **EMS v3.40** | 2026-05-09 | Solar-aware charge rate removed from planned cheap window — always charge at full amps when `inChargingWindow=true` |
| **EMS v3.39** | 2026-05-09 | `SOC_CRITICAL` lowered to 15%; critical charge requires `price <= avgPrice × 0.70` |
| **EMS v3.27** | 2026-05-05 | Mandatory night charging 23h–06h at minimum 8A regardless of price |
| **EMS v3.26** | 2026-05-05 | Removed EV pause-on-battery-discharge guard — EV load subtraction already prevents double-dipping |
| **EMS v3.24** | 2026-05-04 | EV blackout window 17h–19h — 5A session kept alive during cooking peak |
| **EMS v3.18** | 2026-05-03 | EV Charge Controller integrated as second output; EV load subtracted from `netGridW` and `dischargeTargetW` |
| **EMS v3.17** | 2026-05-02 | Solar-aware charge rate during cheap window |
| **EMS v3.16** | 2026-05-01 | Solar overfill pre-discharge branch |
| **EMS v3.15** | 2026-04-28 | Solar surplus charging suppressed during `inDischargeWindow` |
| **EMS v3.10** | 2026-04-26 | Negative price charging spread across full negative window |
| **Planner v2.14** | 2026-05-02 | Solar deduction split before/after cheap window; solar overfill pre-discharge; `cheapestHour` output |
| **Planner v2.13** | 2026-04-28 | Discharge window centered on most expensive hour; `LATE_BIAS_EUR` |
| **Planner v2.11** | 2026-04-26 | Pre-discharge detection before negative windows |
| **Planner v2.8** | 2026-04-25 | Solar forecast deduction; `kwhFromGrid` |
| **Planner v2.6** | 2026-04-25 | Charge window centered on cheapest hour |
| **Planner v2.4** | 2026-04-19 | Frank Energie as price source |
