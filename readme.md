# Battery EMS – Node-RED Setup & Reference Guide

**EMS v3.71 / Planner v2.14 / EV v2.2** — Updated 14 May 2026

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
| **Zaptec** | [github.com/custom-components/zaptec](https://github.com/custom-components/zaptec) | EV charger control — smart price-based charging, solar surplus absorption, battery protection |

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
>
> **Important:** The DSMR P1 port reads the main grid meter, which includes **all** loads — house, EV charger, and everything else. The EMS uses these values directly as ground truth. Do not add or subtract EV amps from DSMR phase readings; they already reflect what is actually on the wire.

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

> Replace `sloeierd` with your charger's entity prefix. Find it by searching `zaptec` in Developer Tools → States. `sensor.sloeierd_laadvermogen` reports total 3-phase power in watts — used to subtract EV load from `netGridW` so the battery does not compensate for intentional EV consumption. The minimum charge current for Zaptec is 6A; setting 5A keeps the session alive but pauses actual charging. Setting 0A does not pause — use the switch instead.

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
                                                              ┌──────────────────────────────────────────────┘ └────────────────────┐
                                                    [Split outputs]                                                      [EV Output Router]
                                                ┌───────────────┬───────────┐                               ┌────────────────┬────────────────┐
                                        [Set dc_amps] [Set charging] [Set discharging]            [Zaptec switch]  [Set Zaptec A]  [EV Diagnostics]
                                                            [EMS Diagnostics]
```

**Important:** The EMS Decision Engine function node must be configured with **2 outputs** in Node-RED. Output 1 carries the battery message to `Split outputs`. Output 2 carries the EV message to the **EV Output Router** function node.

The EV Output Router (3 outputs) handles rate limiting and splits the EV message into:

- Output 1 → Switch node → `Turn ON` / `Turn OFF` (`switch.sloeierd_opladen`)
- Output 2 → `Set Zaptec current` — only fires when `evCharging = true`; uses `msg.zaptecValue` (float)
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
| Max charge power | ~4.08 kW (100 A × 48 V × 0.85) | derived |
| Max discharge power | 4.8 kW (100 A × 48 V / 1000) | derived |
| Round-trip efficiency | 85% | `ROUND_TRIP_EFF` |
| Usable discharge capacity | 32 kWh (SoC 95% → 15%) | derived |
| Grid connection | 3-phase, 20 A fuse per phase | `EV_L1_HARD_LIMIT` = 20 |

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
hoursNeeded    = ceil(kwhFromGrid / 4.08 kW) + 1 safety hour
```

Only solar arriving **after** the cheap window offsets grid charging — solar before the window fills the battery independently.

### Solar overfill pre-discharge (v2.14)

If current solar production rate × hours until cheap window exceeds available battery headroom, the planner flags pre-discharge hours to free capacity before the solar peak. This ensures the cheap grid window can actually be used rather than being skipped because the battery is already full from morning solar.

```
solarWillOverfill = solarBeforeCheapWindow >= currentHeadroom
kwhToFree        = max(kwhFromGrid, MIN_GRID_CHEAP_KWH=3.0 kWh)
targetSoC        = calculated to absorb kwhToFree before cheap window
```

### Charge window — centered on cheapest hour (v2.6 / v2.12)

The planner finds the single cheapest remaining hour and expands outward until `hoursNeeded` hours are filled, always preferring the cheaper neighbour. An **early-hour bias** (`EARLY_BIAS_EUR = 0.02`) keeps grid charging before the solar peak when neighbour prices are close.

The window is only built if `cheapestPrice <= avgPrice × 0.55`. If all remaining hours are expensive, `plannedChargeHours` is empty and the EMS falls back to ratio-based logic.

### Discharge window — centered on most expensive hour (v2.13)

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

## EMS Decision Engine (v3.71) — Configuration Reference

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
| `CHARGE_THRESHOLD` | 0.80 | Ratio for price override and fallback charging |
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
| `GRID_MAX_A_PHASE` | 18 | Usable amps per phase for battery headroom calculation (20 A fuse − 2 A margin) |
| `GRID_VOLTAGE` | 230 | AC grid voltage |
| `SAFETY_MARGIN_A` | 2 | Per-phase headroom buffer for battery charge calc |
| `MIN_DISCHARGE_A` | 35 | Minimum meaningful discharge current (A) |
| `MIN_CHARGE_A` | 10 | Minimum charge current — floor for negative price spread |
| `EV_PUBLIC_RATE` | 0.50 | Public charger reference rate (€/kWh) for cost comparison |

### EV constants (in EV Charge Controller section)

| Constant | Value | Description |
| --- | --- | --- |
| `EV_MIN_AMPS` | 6 | Absolute minimum EV charge current — never goes lower unless 6A itself still causes overload |
| `EV_NIGHT_MAX` | 10 | Maximum EV current during night window (23h–06h) |
| `EV_NIGHT_MIN` | 8 | Minimum EV current during night window |
| `EV_L1_HARD_LIMIT` | 20 | Per-phase hard limit (A) — actual fuse rating |
| `EV_L1_WARN_LIMIT` | 18 | Per-phase warn threshold (A) — diagnostic only in v2.2 |
| `EV_CUT_COOLDOWN_POLLS` | 10 | Polls to hold at 6A after a session cut before attempting ramp |
| `EV_RAMP_UP_POLLS` | 4 | Consecutive clear polls needed to add 1A during ramp-up |
| `EV_3PHASE_MIN_A` | 7 | Threshold above which Zaptec uses 3-phase (Zaptec behavior) |
| `EV_BLACKOUT_START` | 17 | EV charging blackout start hour (inclusive) |
| `EV_BLACKOUT_END` | 19 | EV charging blackout end hour (exclusive) |
| `EV_NIGHT_START` | 23 | Night window start hour (crosses midnight) |
| `EV_NIGHT_END` | 6 | Night window end hour |
| `EV_RATE_LIMIT_MS` | 900000 | 15 min between Zaptec writes (API recommendation) |

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
     b) dischargeTarget = 0 AND solar   → cover remaining import only (throttled)
     c) no surplus                      → export at MIN_DISCHARGE_A

7. Idle
```

### EV load subtraction

The DSMR P1 meter reports total grid load including the EV. The EMS subtracts measured EV charging power from `netGridW` and `dischargeTargetW` each cycle so the battery never discharges to compensate for intentional EV consumption:

```javascript
evChargingW      = EVChargeLoad sensor (W)   // actual 3-phase measured watts
netGridW         = max(0, (gridImport − gridExport) × 1000 − evChargingW)
dischargeTargetW = max(0, homeImportW)        // house load only, EV removed
```

---

## EV Charge Controller (v2.2) — How It Works

The EV controller runs at the end of the EMS Decision Engine function node. It shares all EMS calculated values and outputs a second message for the Zaptec charger.

### DSMR phase readings — ground truth

The DSMR P1 port reads the main grid meter. Phase readings (`phaseL1/2/3LoadA`) are the **actual amps on the wire** including EV, house, and everything else. The EMS uses these directly — no add-back of EV amps is needed or correct.

### Overload protection (v2.2)

When any raw DSMR phase exceeds 20A, the EV controller responds in two steps:

1. **Project what 6A L1 would do** — calculate what each phase would read after dropping the EV to 6A single-phase (L2/L3 lose all EV load, L1 loses `lastEvAmps − 6`)
2. **Decide based on projection:**
   - If 6A resolves the overload → **soft cut**: drop to 6A L1, keep charging, no session cut
   - If 6A still leaves any phase over 20A → **session cut**: drop to 5A (pauses charging, session stays alive)

The 5A session cut path is a last resort for genuine house load spikes (e.g. oven + kettle). Under normal EV charging conditions, 6A always resolves the overload because the EV itself caused it.

### Post-cut ramp recovery (v2.1+)

After any session cut (5A), the EV controller enters a cooldown period before ramping back up:

- **Cooldown** (`EV_CUT_COOLDOWN_POLLS = 10` polls): EV holds at 6A single-phase. No ramp, no 3-phase.
- **Ramp phase**: once cooldown expires, the controller counts consecutive polls with no overload (`evRampClearCount`). Every `EV_RAMP_UP_POLLS = 4` consecutive clear polls adds 1A to the target.
- Night ramp: `maxAllowed = min(10, max(6, lastEvAmps) + floor(rampClearCount / 4))`

This prevents the flap loop where a session cut causes Zaptec to reset the session to `connected_finished`, which the old logic interpreted as a fresh session and immediately jumped back to 8–10A, triggering another overload.

### EV charging rules

| Priority | Condition | EV amps | Phase mode |
| --- | --- | --- | --- |
| 1 | Car not connected | 0 | — |
| 2 | Car full (`connected_finished` + SoC ≥ 100%) | 0 | — Zaptec manages session |
| 3 | Session cut active (`l1SessionCut`) | 5 | L1 single-phase |
| 4 | Post-cut cooldown (`inCutCooldown`) | 6 | L1 single-phase |
| 5 | Soft cut (`l1SoftCut`) — 6A resolves overload | 6 | L1 single-phase |
| 6 | Blackout 17h–19h | 5 | L1 — session kept alive |
| 7 | Battery discharging | 6 | L1 single-phase |
| 8 | Night 23h–06h, 3-phase headroom | 8–10 (ramped) | 3-phase |
| 9 | Cheap price / solar surplus | up to 10 (ramped) | 3-phase if ≥ 7A |
| 10 | Baseline | 6 | L1 single-phase |

**Key rules:**

- **6A is the absolute minimum** — the EV never goes below 6A unless `l1SessionCut` fires (house itself over 20A with EV already at 6A), in which case 5A keeps the session alive
- **Blackout 17h–19h** — always 5A (session alive, no actual charging) regardless of price or solar
- **Night window 23h–06h** — charges at 8–10A on 3-phase; ramps up slowly after any overload event rather than jumping straight to max
- **3-phase mode** — Zaptec Go uses all 3 phases when commanded ≥ 7A; below 7A it uses L1 only
- **EV never charges from battery** — EV load is subtracted from `netGridW` so the battery only sees home load
- **Pausing**: 5A keeps the Zaptec session alive below the 6A charge threshold; 0A does not pause (use the switch for a true pause)

### EV session cost tracking (v3.44+)

```
evSessionKwh     — kWh charged this session (resets on disconnect)
evSessionCostAct — actual cost at dynamic price (€)
evSessionCostPub — cost at public rate 0.50 €/kWh (€)
evSessionSaving  — saving vs public charger (€)
evInstantCostAct — current cost rate (€/h)
evInstantCostPub — public rate cost rate (€/h)
```

The session resets automatically when the car disconnects. Long-term totals are best tracked using Home Assistant's Riemann sum integration on `sensor.sloeierd_laadvermogen`.

### EV rate limiting

Rate limiting is handled in the **EV Output Router** node. The router writes to Zaptec when:

1. The target amps value changes — immediate response
2. 15 minutes have elapsed since last write — periodic refresh

### EV diagnostic output (output 2)

```json
{
  "payload": {
    "version": "EV v2.2",
    "targetAmps": 8.0,
    "carConnected": true,
    "reason": "Night charging (0.082 EUR) - 8A [3-phase, max 10A] ramp=3/4 (headroom: L1=9 L2=11 L3=10)",
    "inputs": {
      "zaptecMode": "connected_charging",
      "currentPrice": 0.082,
      "avgPrice": 0.210,
      "lastEvAmps": 8,
      "currentHour": 1,
      "evInNight": true,
      "evPhaseMode": "3-phase",
      "phaseL1LoadA": 11.2,
      "phaseL2LoadA": 9.3,
      "phaseL3LoadA": 10.1,
      "projL1At6A": 9.2,
      "projL2At6A": 1.3,
      "projL3At6A": 2.1,
      "anyOver20": false,
      "sixAmpResolves": true,
      "l1SessionCut": false,
      "l1SoftCut": false,
      "inCutCooldown": false,
      "evCutCooldown": 0,
      "evRampClearCount": 3,
      "worst3PhaseA": 9.7,
      "l1OverloadCount": 0
    }
  },
  "zaptecValue": 8.0,
  "evTargetAmps": 8,
  "evShouldWrite": false
}
```

---

## Diagnostic Output Structure — Battery (output 1)

```json
{
  "version": "EMS v3.71 / Planner v2.14",
  "dc_amps": 0,
  "dc_power": 5616,
  "charging": false,
  "discharging": true,
  "reason": "High price (128% of avg, 0.261 EUR) - discharging 117 A to cover 4178 W gross import + 800 W export bias",
  "planner": {
    "inWindow": false,
    "inDischargeWindow": true,
    "hoursNeeded": 3,
    "kwhNeeded": 29.6,
    "cheapestPrice": 0.060,
    "cheapestHour": 13,
    "plannedHours": [12, 13, 14],
    "plannedDischargeHours": [19, 20, 21, 22, 23]
  },
  "inputs": {
    "currentPrice": 0.261,
    "avgPrice": 0.205,
    "priceRatio": 127.6,
    "batterySoC": 22,
    "solarTotal_W": 1530,
    "netGrid_W": 4178,
    "dischargeTarget_W": 4178,
    "actualSurplus_W": 0,
    "evCharging_W": 0,
    "evLastAmps": 0,
    "phaseL1_A": 4.05,
    "phaseL2_A": 3.84,
    "phaseL3_A": 10.28,
    "maxPhaseLoad_A": 10.28,
    "worstHeadroom_A": 5.72,
    "maxAllowedCharge_A": 70
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
| Discharge amps ~18% too high | Old pre-v3.5 script | Check version string in debug output |
| Discharging below avg price | Old pre-v3.6 script | `isPriceHigh` avgPrice guard missing — update to v3.6+ |
| Not charging at cheap hours outside window | Old pre-v3.7 script | `isPriceVeryLow` override missing — update to v3.7+ |
| Charging at expensive hours at low SoC | Old pre-v3.8 script | `SOC_CRITICAL` lacks `chargeAbsMax` guard — update to v3.8+ |
| Battery charges above 95% | Old pre-v3.9 script | `canCharge` had `isNegPrice` override — update to v3.9+ |
| Discharge not stopping at low SoC | Old `canDischarge` bug | Ensure EMS v3.5+ is deployed |
| Solar forecast not deducting | `solarForecastRemaining` = 0 | Check `get-solar-forecast` node; verify `sensor.energy_production_remaining_today` |
| `[confirming 1/2]` in reason | State pending confirmation | Normal — resolves next cycle |
| `maxAllowedCharge_A: 0` | Home load near grid limit | Large appliance consuming headroom — EMS resumes when load drops |
| Battery discharging to cover EV load | Old pre-v3.18 EMS | EV load subtraction missing — update to EMS v3.18+ |
| EV always 0A despite cheap price | EMS node set to 1 output | Change EMS function node outputs to 2; wire output 2 to EV Output Router |
| Set Zaptec current: Invalid JSON | Wrong data field template | Use `{"value": {{zaptecValue}}}` and uncheck Block input overrides |
| EV not pausing despite 0A command | Zaptec 6A minimum | Use `switch.sloeierd_opladen` off for a true pause — 5A keeps session alive |
| EV session cut flap loop | Old pre-v3.70 EMS | EV add-back doubled phase load causing phantom overloads — update to EMS v3.71+ |
| EV phases showing ~2× actual amps | Old pre-v3.71 EMS | Phase add-back bug — update to EMS v3.71+ which uses raw DSMR directly |
| EV drops to 5A when house load is fine | Old pre-v3.71 EMS | Wrong session cut rule — update to EMS v3.71+ |
| EV jumps to 8–10A right after a cut | Old pre-v3.70 EMS | Missing ramp recovery — update to EMS v3.70+ |
| Solar surplus not detected while EV charging | Old pre-v3.45 script | EV load clamp hid surplus — update to EMS v3.45+ |
| Battery charging slowly during cheap window | Old pre-v3.40 script | Solar adjustment was throttling — update to EMS v3.40+ |
| dc_amps set during discharge | Old pre-v3.43 script | dc_amps/dc_power not split — update to EMS v3.43+ |

---

## Version History

| Version | Date | Change |
| --- | --- | --- |
| **EMS v3.71 / EV v2.2** | 2026-05-14 | Phase loads now read directly from raw DSMR — no EV add-back; DSMR reads the main meter which already includes EV load; adding EV amps back was doubling the EV contribution and creating phantom overloads; overload logic simplified: if any DSMR phase >20A, project what 6A L1 would give; if 6A resolves it → soft cut (6A, keep charging); if not → session cut (5A); `houseOnlyLx`/`houseAloneOver20` removed; new diagnostics: `projL1/2/3At6A`, `anyOver20`, `sixAmpResolves` |
| **EMS v3.70 / EV v2.1** | 2026-05-14 | Session cut rule corrected: only cut session (5A) when house load alone exceeds 20A after removing EV contribution; if EV caused the overload → 6A soft cut (keep charging); post-cut ramp recovery: `EV_CUT_COOLDOWN_POLLS=10` polls at 6A after any cut, then `EV_RAMP_UP_POLLS=4` consecutive clear polls per 1A increase — prevents jumping back to 8–10A; new diagnostics: `houseOnlyL1/L2/L3`, `houseAloneOver20`, `inCutCooldown`, `evCutCooldown`, `evRampClearCount` |
| **EMS v3.69 / EV v2.0** | 2026-05-13 | `evCarFull` (connected_finished + evSoC ≥ 100%) sends 0A — Zaptec manages session when car is full |
| **EMS v3.68** | 2026-05-13 | Smarter overload response: try 6A L1 before dropping to 5A; 9 bug fixes including `EV_3PHASE_MIN_A` used before defined, phase add-back correction, night safety net overwrite |
| **EMS v3.67** | 2026-05-12 | Per-phase overload detection; EV phase add-back corrected to only add to phases EV actually uses |
| **EMS v3.65** | 2026-05-11 | Blackout window 17h–19h set to 5A (session alive, pauses charging) |
| **EV v2.0** | 2026-05-10 | Complete EV rewrite: baseline 6A 24/7; night 23h–06h up to 10A 3-phase; per-phase headroom; graduated overload response; solar surplus scaling; battery discharge cap; blackout 17h–19h; phase mode hysteresis; session accumulator |
| **EMS v3.45** | 2026-05-09 | Solar surplus detection uses `rawNetGridW` before EV clamp — fixes surplus not detected when EV charging pushes `netGridW` to 0 |
| **EMS v3.44** | 2026-05-09 | EV session cost tracking: `evSessionKwh`, costs, savings vs public charger |
| **EMS v3.43** | 2026-05-09 | `dc_amps` only when charging; `dc_power` only when discharging |
| **EMS v3.42** | 2026-05-09 | `MAX_AMPS` reduced to 100A safe limit |
| **EMS v3.40** | 2026-05-09 | Solar-aware charge rate removed from planned window — always charge at full amps when `inChargingWindow=true` |
| **EMS v3.39** | 2026-05-09 | `SOC_CRITICAL` lowered to 15%; critical charge requires `price <= avgPrice × 0.70` |
| **EMS v3.27** | 2026-05-05 | Mandatory night charging; `EV_NIGHT_AMPS` (8A) regardless of price |
| **EMS v3.26** | 2026-05-05 | Removed EV pause-on-battery-discharge guard — EV load subtraction already prevents double-dipping |
| **EMS v3.24** | 2026-05-04 | EV blackout window 17h–19h |
| **EMS v3.18** | 2026-05-03 | EV Charge Controller integrated as second output; EV load subtracted from `netGridW` and `dischargeTargetW` |
| **EMS v3.16** | 2026-05-01 | Solar overfill pre-discharge branch |
| **EMS v3.15** | 2026-04-28 | Solar surplus charging suppressed during discharge window |
| **EMS v3.13** | 2026-04-26 | `fraction` clamped to [0.10, 1.0] |
| **EMS v3.12** | 2026-04-26 | Pre-discharge before negative price windows |
| **EMS v3.10** | 2026-04-26 | Negative price charging spread across full window |
| **EMS v3.9** | 2026-04-26 | Hard `canCharge` ceiling — no negative price override |
| **EMS v3.7** | 2026-04-25 | `isPriceVeryLow` ratio override |
| **EMS v3.6** | 2026-04-25 | `SOC_CRITICAL` boundary fix; `isPriceHigh` avgPrice guard |
| **EMS v3.5** | 2026-04-20 | `canDischarge` bug fixed; `ROUND_TRIP_EFF` removed from discharge |
| **EMS v3.0** | 2026-04-08 | P1 phase readings for per-phase headroom |
| **Planner v2.14** | 2026-05-02 | Solar deduction split before/after cheap window; solar overfill pre-discharge; `MIN_GRID_CHEAP_KWH = 3.0`; `cheapestHour` output |
| **Planner v2.13** | 2026-04-28 | Discharge window centered on most expensive hour; `LATE_BIAS_EUR` |
| **Planner v2.12** | 2026-04-26 | Charge window validity gate |
| **Planner v2.11** | 2026-04-26 | Pre-discharge detection before negative windows |
| **Planner v2.6** | 2026-04-25 | Charge window centered on cheapest hour |
| **Planner v2.4** | 2026-04-19 | Frank Energie as price source |
| **Planner v2.0** | 2026-04-13 | `dischargeThreshold` after conflict filter |
