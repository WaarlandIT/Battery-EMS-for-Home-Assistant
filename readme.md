# Battery EMS – Node-RED Setup & Reference Guide

**EMS v3.90 / Planner v2.15 / EV v2.19 / Logger v1.4** — Updated 9 October 2026

[![Node-red layout](https://github.com/WaarlandIT/Battery-EMS-for-Home-Assistant/raw/main/NodeRed.png)](NodeRed.png)

---

## My Setup

This flow was built around a **Growatt inverter** for the solar array and a **Deye inverter** for the battery bank. A **Zaptec Go EV charger** is also integrated — smart price-based EV charging runs as part of the same flow. The solar is on the Growatt only; the Deye is battery-only and charges from the AC side (grid plus any Growatt surplus). The EMS script is the only thing that commands the Deye.

The battery inverter connection is not included in this flow — how the DC amps, charging, and discharging booleans are wired to your inverter depends entirely on your hardware. The entity names used here reflect my installation; adjust them to match yours.

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
| **Frank Energie** | [github.com/HiDiHo01/home-assistant-frank\_energie](https://github.com/HiDiHo01/home-assistant-frank_energie) | Hourly dynamic electricity prices — primary price source for planner and EMS |
| **DSMR Smart Meter** | [home-assistant.io/integrations/dsmr](https://www.home-assistant.io/integrations/dsmr/) | Real-time grid import/export (kW) and per-phase load (kW) via P1 port |
| **Growatt ESPHome** | [github.com/WaarlandIT/ESPHOME-Growatt](https://github.com/WaarlandIT/ESPHOME-Growatt) | Live solar AC output per phase (PAC1/2/3 in W) |
| **Forecast.Solar** | [home-assistant.io/integrations/forecast\_solar](https://www.home-assistant.io/integrations/forecast_solar/) | Solar production forecast — used by the planner to reduce grid charge hours and detect solar overfill |
| **ha-solarman** | [github.com/davidrapan/ha-solarman](https://github.com/davidrapan/ha-solarman) | Battery state of charge (%) from BMS via Solarman protocol |
| **EnergyZero** | [home-assistant.io/integrations/energyzero](https://www.home-assistant.io/integrations/energyzero/) | Used only as an hourly trigger — not the price source |
| **Zaptec** | [github.com/custom-components/zaptec](https://github.com/custom-components/zaptec) | EV charger control — price-based charging, solar surplus absorption, phase overload protection |

> Frank Energie is the authoritative price source since Planner v2.4. EnergyZero is kept only as a trigger to fire the flow at the start of each new price hour.

---

## Step 1 – Home Assistant Helpers

Create these in **Settings → Devices & Services → Helpers** before importing the flow:

| Type | Entity ID | Min | Max | Step |
| --- | --- | --- | --- | --- |
| `input_number` | `input_number.battery_dc_amps` | 0 | 200 | 1 |
| `input_boolean` | `input_boolean.battery_charging` | — | — | — |
| `input_boolean` | `input_boolean.battery_discharging` | — | — | — |

These are the three outputs the EMS writes every cycle. In my setup they are the Deye commands — nothing else manages the Deye.

---

## Step 2 – Sensor Entity Name Mapping

Verify these in the relevant `api-current-state` nodes after importing. Use **Developer Tools → States** to find your exact entity names.

### Frank Energie (price source) — [GitHub](https://github.com/HiDiHo01/home-assistant-frank_energie)

| Node | Entity ID | Output |
| --- | --- | --- |
| Frank Energie prijzen | `sensor.frank_energie_prijzen_huidige_elektriciteitsprijs_all_in` | `msg.frankPrices` (attributes.prices array) |

> The planner calculates `currentPrice` and `avgPrice` directly from this array, overriding anything from EnergyZero. Since Planner v2.15 the hour of each price is taken from the parsed timestamp, so both local-offset and UTC (`Z`) timestamps work.

### DSMR Smart Meter — [HA Docs](https://www.home-assistant.io/integrations/dsmr/)

| Node | Entity ID | Output |
| --- | --- | --- |
| Grid import kW | `sensor.dsmr_reading_electricity_currently_delivered` | `msg.gridImport` (kW) |
| Grid export kW | `sensor.dsmr_reading_electricity_currently_returned` | `msg.gridExport` (kW) |
| Phase L1 kW | `sensor.dsmr_reading_phase_currently_delivered_l1` | `msg.phaseL1` |
| Phase L2 kW | `sensor.dsmr_reading_phase_currently_delivered_l2` | `msg.phaseL2` |
| Phase L3 kW | `sensor.dsmr_reading_phase_currently_delivered_l3` | `msg.phaseL3` |

> Entity names vary by DSMR version. Search `dsmr` in Developer Tools → States to confirm yours. The phase readings are the main meter, so they already include the EV load.

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

> Falls back safely to 0 if unavailable.

### Battery SoC — [ha-solarman GitHub](https://github.com/davidrapan/ha-solarman)

| Node | Entity ID | Output |
| --- | --- | --- |
| Battery SoC | `sensor.battery_state_of_charge` | `msg.batterySoC` (%) |

> Replace with your actual BMS/inverter SoC entity. For the StephanJoubert solarman integration use `sensor.deye_battery_soc`.

### Zaptec EV Charger — [GitHub](https://github.com/custom-components/zaptec)

| Node | Entity ID | Output / Action |
| --- | --- | --- |
| Get Zaptec mode | `sensor.sloeierd_charger_mode` | `msg.zaptecMode` (str) |
| Get EV charge load | `sensor.sloeierd_laadvermogen` | `msg.EVChargeLoad` (W, total of all phases) |
| Get EV SoC | your car's SoC sensor | `msg.ev_soc` (%) |
| Set Zaptec current | `number.sloeierd_max_stroom` | Sets charge current per phase (5–16 A) |
| Set Zaptec switch on/off | `switch.sloeierd_opladen` | Starts / stops a charging session |

> Replace `sloeierd` with your charger's entity prefix. Find it by searching `zaptec` in Developer Tools → States. The Zaptec ignores very low current values, so the EMS never relies on 0A to pause a session — use the switch for a real pause. If `ev_soc` is unavailable, the EMS trusts Zaptec's `connected_finished` state as "car full".

### Trigger nodes

| Node | Entity ID | Purpose |
| --- | --- | --- |
| DSMR power change | `sensor.electricity_meter_energieverbruik` | Fire instantly on load change |
| Price hour change | `sensor.energyzero_today_energy_current_hour_price` | Fire on hourly price rollover |
| Battery SoC change | `sensor.battery_state_of_charge` | Fire within seconds of a SoC change, so the 97% hard stop reacts immediately |

---

## Step 3 – Node Chain Execution Order

**Battery SoC, Solar Forecast and Zaptec mode must all be read before the Price Planner and EMS Decision Engine.**

```
Triggers (5 min inject / DSMR change / price hour / SoC change)
  → Frank Energie prices → Battery SoC → Solar forecast → Set solarNow
  → Price Planner
  → Grid import kW → Grid export kW → Phase L1/L2/L3 kW → Solar PAC1/2/3 W
  → Get EV charge load → Get Zaptec mode → Get EV SoC
  → EMS Decision Engine (2 outputs)

Output 1 (battery) → Split outputs → Set dc_amps / Set charging / Set discharging
                   → EMS Logger → write file
                   → EMS Debug

Output 2 (EV)      → switch "msg.evShouldWrite is true" → Set Zaptec current
                   → EMS Logger → write file
                   → EV Debug
```

**Important:**

- The EMS Decision Engine function node must be configured with **2 outputs**.
- Output 2 must go through a **switch node** before `Set Zaptec current`. Configure it as property `msg.evShouldWrite`, rule **`is true`**. Do *not* use `== "true"` with the string type (`a/z`) — the value is a boolean and a string comparison never matches, so Zaptec would never receive a command.
- `Set Zaptec current` uses data `{"value": {{zaptecValue}}}`.
- The debug and logger nodes are wired *before* the switch node, so they always see every EV message.

### Set solarNow function node

Add a small function node before the Price Planner to pass current solar production for overfill detection:

```javascript
msg.solarNow = (parseFloat(msg.pac1) || 0)
             + (parseFloat(msg.pac2) || 0)
             + (parseFloat(msg.pac3) || 0);
return msg;
```

### Get EV charge load node

Add an `api-current-state` node before `Get Zaptec mode`:

- Entity: `sensor.sloeierd_laadvermogen`
- Output property: `msg.EVChargeLoad` (num)

This gives the EMS the measured EV load in watts. It is used to keep the battery from discharging into the EV and to track session kWh and cost.

---

## Step 4 – Trigger Configuration

| Trigger | Interval | Purpose |
| --- | --- | --- |
| Every 5 min inject | 300 s | Baseline heartbeat |
| DSMR power change | Every DSMR update (~2–10 s) | React instantly to home load changes |
| Price hour change | Every hour | React immediately when price rolls to next hour |
| Battery SoC change | Every SoC update | Hard stop fires within seconds of reaching 97% |

In all `server-state-changed` nodes, ensure:

- **Only send if state changes** — enabled
- **Ignore unavailable/unknown** — enabled on both incoming and outgoing state

> **Timing note:** the EV ramp and cooldowns count *polls*, not minutes. "15 minutes" assumes about one EMS run every 30 seconds, which matches my logs. If your triggers fire much more often, the ramp and cooldowns become shorter.

---

## Step 5 – Battery Specs (verify in both scripts)

| Spec | Value | Config key |
| --- | --- | --- |
| Capacity | 40 kWh | `BATTERY_KWH` |
| Voltage | 48 V DC | `BATTERY_VOLTAGE` |
| Max charge/discharge | 100 A | `MAX_AMPS` (EMS **and** planner) |
| Max charge power | ~4.1 kW (100 A × 48 V × 0.85) | derived |
| Max discharge power | 4.8 kW (100 A × 48 V / 1000) | derived (`MAX_DISCHARGE_KW`) |
| Round-trip efficiency | 85% | `ROUND_TRIP_EFF` |
| Usable discharge capacity | 32 kWh (SoC 95% → 15%) | derived |
| Grid connection | 3-phase, 20 A fuse | `GRID_MAX_A_PHASE` = 18 (2 A margin) |

> `MAX_AMPS` must be the same in the EMS and the planner. Before Planner v2.15 the planner used 200 A, which made it plan only half the charge hours the battery actually needs.

---

## Price Planner (v2.15) — How It Works

The planner runs first each cycle and calculates today's charge and discharge schedule. It plans **only the remaining hours of today** — tomorrow's prices are not used for the window, so overnight charging after midnight is handled by the EMS price-ratio override.

### Price source and fallback

Prices are read from the Frank Energie `attributes.prices` array. `currentPrice` and `avgPrice` (today's average) are calculated from this array and written to `msg`.

If the array is empty, or contains no remaining hours for today (for example only tomorrow's prices), the planner switches to **fallback mode**: no windows are set, the node status turns yellow, and the EMS uses its ratio-based logic. Before v2.15 the "no hours left today" case crashed the planner and stopped the whole flow.

### kWh needed and solar deduction

```
usableCapacity = 40 × ((95% − 10%) / 100)  = 34 kWh
currentKwh     = 40 × ((SoC% − 10%) / 100)
kwhNeeded      = usableCapacity − currentKwh
solarUsable    = solarForecastRemaining × 0.90
```

Solar is split into before and after the cheap window:

```
solarBeforeCheapWindow = min(solarNow_kW × hoursUntilCheapWindow, solarUsable)
solarAfterCheapWindow  = max(0, solarUsable − solarBeforeCheapWindow)
kwhFromGrid            = max(0, kwhNeeded − solarAfterCheapWindow)
hoursNeeded            = ceil(kwhFromGrid / 4.08 kW) + 1 safety hour
```

Only solar arriving **after** the cheap window offsets grid charging. The before-window estimate is capped by the remaining forecast, because the current production rate does not last past sunset.

### Charge window — centered on cheapest hour

The planner finds the cheapest remaining hour and expands outward until `hoursNeeded` hours are filled, preferring the cheaper neighbour. An **early-hour bias** (`EARLY_BIAS_EUR = 0.02`) keeps grid charging before the solar peak when neighbour prices are close.

- The window is only built if the cheapest hour is below the **charge price cap**: `avgPrice × 0.55` when SoC ≥ 80%, otherwise `avgPrice × 0.99`.
- The window **never expands into above-average hours** — it stops at the edge of the price valley even if fewer than `hoursNeeded` hours fit.

### Discharge window — centered on most expensive hour

```
kwhToDischarge       = 40 × ((95 − 15) / 100) = 32 kWh
dischargeHoursNeeded = ceil(32 / 4.8 kW) + 1  = 8h
```

The most expensive remaining hour at or above `avgPrice` becomes the center. The window expands only into neighbouring hours that are also at or above average, with a **late-hour bias** (`LATE_BIAS_EUR = 0.02`) to keep it in the evening peak. In practice the window is often shorter than 8h because it stops where prices drop below average. Hours overlapping the charge window are removed.

### Solar overfill pre-discharge

If solar arriving before the cheap window would fill the battery, the planner discharges first so the cheap grid window can still be used:

```
solarWillOverfill = cheap window ahead AND not yet open
                    AND solarBeforeCheapWindow > 0
                    AND solarBeforeCheapWindow >= currentHeadroom
kwhToFree         = min(max(kwhFromGrid, 3.0 kWh), energy above 15% SoC)
targetSoC         = SoC after freeing kwhToFree (never below 15%)
```

Pre-discharge only runs in hours before the cheap window with a price of at least `avgPrice × 0.55`.

### Pre-discharge before negative price windows

When negative price hours are forecast later today, the hours before them (priced at least `avgPrice × 0.55`) become pre-discharge hours, so the battery has room to charge for free.

### Planner config constants

| Constant | Value | Description |
| --- | --- | --- |
| `BATTERY_KWH` | 40 | Battery capacity |
| `MAX_AMPS` | 100 | Must match EMS `MAX_AMPS` |
| `SOC_MIN` | 10% | Minimum SoC floor |
| `SOC_MAX` | 95% | Maximum SoC ceiling |
| `SOC_NORMAL_THRESHOLD` | 80% | At or above this, the tight charge cap is used |
| `CHARGE_ABS_RATIO` | 0.55 | Tight charge cap = `avgPrice × 0.55` |
| `DISCHARGE_ABS_RATIO` | 0.55 | Minimum price for pre-discharge hours = `avgPrice × 0.55` |
| `SOLAR_FORECAST_EFF` | 0.90 | 10% margin on solar forecast |
| `SAFETY_BUFFER_H` | 1 | Extra hour added to `hoursNeeded` |
| `ROUND_TRIP_EFF` | 0.85 | Used for the charge rate (4.08 kW) |
| `EARLY_BIAS_EUR` | 0.02 | Prefer earlier hours in charge window expansion |
| `LATE_BIAS_EUR` | 0.02 | Prefer later hours in discharge window expansion |
| `MAX_DISCHARGE_KW` | 4.8 | Derived from `MAX_AMPS × BATTERY_VOLTAGE` |
| `SOC_DISCHARGE_MAX` | 95% | SoC from which discharge window sizing starts |
| `SOC_DISCHARGE_MIN_PL` | 15% | SoC floor for discharge and pre-discharge |
| `SOLAR_OVERFILL_RATIO` | 1.0 | Solar/headroom ratio for overfill detection |
| `MIN_GRID_CHEAP_KWH` | 3.0 | Minimum kWh to free for the cheapest hours |

---

## EMS Decision Engine (v3.90) — Configuration Reference

### CFG parameters

| Parameter | Value | Description |
| --- | --- | --- |
| `MAX_AMPS` | 100 | Hard cap on DC output amps |
| `BATTERY_VOLTAGE` | 48 | Nominal DC bus voltage (V) |
| `BATTERY_KWH` | 40 | Usable capacity (kWh) |
| `SOC_MIN` | 10% | Discharge floor |
| `SOC_MAX` | 95% | Charge ceiling used by the decision branches |
| `SOC_DISCHARGE_MIN` | 15% | Never discharges below this |
| `SOC_CRITICAL` | 15% | At or below this, charge at full amps if price ≤ `avgPrice × 0.70` |
| `CHARGE_THRESHOLD` | 0.80 | Ratio for price override charging (`isPriceVeryLow`) |
| `DISCHARGE_THRESHOLD` | 1.20 | Discharge outside the planned window if price ≥ 120% of avg |
| `CHARGE_ABS_RATIO` | 0.55 | `chargeAbsMax` = `avgPrice × 0.55` |
| `DISCHARGE_ABS_RATIO` | 0.55 | `dischargeAbsMin` = `avgPrice × 0.55` |
| `DISCHARGE_OVERSHOOT` | 1.25 | 25% overshoot on discharge target for inverter lag |
| `EXPORT_BIAS_W` | 800 | Watts added to discharge target when no solar and no EV |
| `HYSTERESIS` | 0.05 | Price ratio band that keeps discharging from toggling |
| `SOLAR_SURPLUS_W` | 500 | Solar export (W) needed to start solar charging |
| `SOLAR_SURPLUS_EXIT_W` | 200 | Solar export (W) needed to keep solar charging |
| `ROUND_TRIP_EFF` | 0.85 | Charging efficiency (charge amps only) |
| `GRID_MAX_A_PHASE` | 18 | Usable amps per phase (20 A fuse − 2 A margin) |
| `GRID_VOLTAGE` | 230 | AC grid voltage |
| `SAFETY_MARGIN_A` | 2 | Per-phase headroom buffer |
| `MIN_DISCHARGE_A` | 35 | Minimum meaningful discharge current |
| `MIN_CHARGE_A` | 10 | Minimum charge current during negative prices (grid headroom still wins) |
| `EV_RATE_LIMIT_MS` | 900000 | 15 min keep-alive interval for Zaptec writes |
| `EV_PUBLIC_RATE` | 0.50 | Public charger reference rate (€/kWh) for cost comparison |

### Hard stop — SoC ≥ 97%

Just before the output is sent, every cycle: if `batterySoC >= 97`, `dc_amps` is forced to 0 and `charging` to false, overriding every decision branch. The node status turns yellow and shows `HARD STOP SoC:X%`.

The hard stop can only react when the EMS runs. With only the 5-minute heartbeat, the battery could go from 95% to 100% between two runs, which is how the battery fault in June happened. Use the **Battery SoC change** trigger from Step 4 so the hard stop fires within seconds.

### Decision priority — Battery (highest to lowest)

```
1. Negative price AND canCharge          → spread charge across the negative window
                                           (min 10A, but never above grid headroom)

2. isPriceLow AND canCharge              → charge at scaled amps
     a) SoC <= SOC_CRITICAL AND cheap    → full amps, critical recovery
     b) inChargingWindow = true          → scaled amps
     c) isPriceVeryLow (ratio ≤ 0.80     → scaled amps, outside planned window
        AND price ≤ avgPrice × 0.55)

3. Solar surplus AND NOT inDischargeWindow → absorb surplus

4. inSolarPreDischargeWindow             → pre-discharge before solar overfills battery

5. inPreDischargeWindow                  → pre-discharge before negative price window

6. isPriceHigh AND canDischarge
     a) dischargeTarget > 0             → cover house import (+ export bias)
     b) solar exporting                 → throttled discharge or idle
     c) otherwise                       → export at MIN_DISCHARGE_A

7. Idle

Then: hard stop (SoC ≥ 97% → dc_amps = 0)
```

### State confirmation and direction changes

A new battery state is confirmed before it is applied (1 cycle for price/solar driven decisions, 2 cycles otherwise). While a change is being confirmed the battery **holds idle** — amps computed for one direction are never applied in the other. A switch between charging and discharging always passes through idle first. The stored state is the one actually sent to the inverter, including the hard stop.

### EV load handling

The DSMR meter sees the EV as house load. To keep the battery from discharging into the EV, the discharge target uses house load only when the EV is drawing:

```
evChargingW      = EVChargeLoad sensor (W), or lastEvAmps × 230 V × 3 as fallback
homeLx           = phase load − EV share on that phase
                   (L1: always lastEvAmps; L2/L3: only when the EV is on 3 phases)
dischargeTargetW = EV drawing ? sum(homeL1..L3) : gridImport
```

---

## EV Charge Controller (v2.19) — How It Works

The EV controller runs at the end of the EMS Decision Engine function node and produces output 2 for the Zaptec charger.

### Time windows

| Window | Hours | Behaviour |
| --- | --- | --- |
| Night | 22:30–07:00 | 8A minimum, ramps up to 12A as headroom allows |
| Blackout | 17:00–19:00 | 5A on L1 (cooking peak), session kept alive |
| Day | rest | 6A L1 baseline, higher only at very low/negative price or solar surplus |

At 22:30 the EV switches from single-phase to 3-phase. The Zaptec briefly stops and restarts the session to change phases — this "stopped / started charging" notification is normal.

### Phase headroom

```
headroom per phase = 20 A − measured phase load + EV current on that phase
```

The EV's own current is only added back while the EV is actually drawing (L1 always, L2/L3 only on 3 phases). If the car is paused or finished, nothing is added back, so the controller cannot allow a current that would overload the phase when the car resumes.

### Phase overload protection

Only active when a car is connected:

| Situation | Response |
| --- | --- |
| Any phase > 20A and EV above 6A | **Soft cut**: drop to 6A L1, session kept alive |
| Any phase > 20A and EV at 6A or less | **Session cut**: drop to 5A L1, hold for 10 polls (cooldown) |

At night the 8A minimum applies again once the phase is back under 20A.

### Ramp-up (15 minutes per step)

The EV may only go up **1A at a time**, and each step needs **30 consecutive clear polls (~15 minutes)** of stable headroom. The counter resets:

- after every step up (so each step must be earned again),
- after any soft cut, session cut or step down,
- while no car is connected.

From a cold start at night the EV goes 8A → 9A → 10A → 11A → 12A over about one hour. Before v2.19 the counter never reset after a step, so after the first 15 minutes the EV climbed 1A every poll.

### EV charging rules (highest priority first)

| Priority | Condition | EV amps | Phase mode |
| --- | --- | --- | --- |
| 1 | Car not connected | 0 | — |
| 2 | Car full (`connected_finished` and SoC 100%, or SoC unknown) | 0 | — |
| 3 | Session cut (phase > 20A, EV ≤ 6A) | 5A | L1 |
| 4 | Post-cut cooldown | 5A | L1 |
| 5 | Soft cut (phase > 20A, EV > 6A) | 6A | L1 |
| 6 | Blackout 17:00–19:00 | 5A | L1 |
| 7 | Daytime, battery discharging | 6A | L1 |
| 8 | Night, battery charging | up to 8A | 3-phase |
| 9 | Night | 8–12A (ramp) | 3-phase |
| 10 | Daytime, battery charging (or just stopped) | 6A | L1 |
| 11 | Daytime, price ≤ 0 | 6–12A (ramp) | 3-phase when ≥ 7A |
| 12 | Daytime, `isPriceVeryLow` | 6–8A (ramp) | 3-phase when ≥ 7A |
| 13 | Solar surplus ≥ 7A per phase | up to 16A | 3-phase |
| 14 | Default baseline | 6A | L1 |

Night battery discharging does not limit the EV: discharging adds grid headroom, so the full ramp applies.

### Zaptec API rate limiting

The Zaptec API does not like frequent calls, so the controller only asks for a write when needed:

```
evShouldWrite       = amps changed OR 15 min since last write OR finished-early nudge
msg.evShouldWrite   = evShouldWrite AND car connected   (this gates the switch node)
```

**Finished-early nudge:** if Zaptec reports `connected_finished` while the car's SoC is below 100%, the controller re-sends the current to nudge the session back into charging — at most **3 times**, about 2.5 minutes apart. It stops after that, because the car may simply have reached its own charge limit. No nudges are sent when `ev_soc` is unavailable.

On disconnect the controller resets the session counters, cooldowns, ramp and last commanded amps, so the first command of the next session is always sent.

### EV session cost tracking

```
evSessionKwh     — kWh charged this session (resets on disconnect)
evSessionCostAct — actual cost at dynamic price (€)
evSessionCostPub — cost at public rate 0.50 €/kWh (€)
evSessionSaving  — saving vs public charger (€)
evInstantCostAct — current cost rate (€/h)
evInstantCostPub — public rate cost rate (€/h)
```

These depend on `sensor.sloeierd_laadvermogen`. Check that it reports the total power of all phases in watts — if it reads low while the EV draws 12A on 3 phases (~8.3 kW), the session kWh will be wrong.

---

## EMS Logger (v1.4)

A function node that writes both outputs to a daily text file (`/media/logs/ems_log_YYYY-MM-DD.txt`) via a `write file` node. Timestamps are in local time (CET/CEST).

- **Battery entries** are only written when the state, amps or SoC change.
- **EV entries** are only written when the state, amps, phase mode or reason change.
- Each EV entry shows `shouldWrite`, so you can see exactly when a command went to Zaptec, and the `L1 safety` line shows real overload values (`hard=` / `warn=`).

---

## Diagnostic Output Structure — Battery (output 1)

```javascript
msg.payload = {
  version:     "EMS v3.90 / Planner v2.15",
  dc_amps:     0,        // amps when charging, 0 otherwise
  dc_power:    1680,     // watts when discharging (35A × 48V), 0 otherwise
  charging:    false,
  discharging: true,
  reason:      "High price (131% of avg, 0.289 EUR) - discharging 35 A ...",
  planner: { ... },
  inputs: {
    currentPrice:        0.289,
    avgPrice:            0.221,
    batterySoC:          63,
    solarTotal_W:        0,
    netGrid_W:           0,
    dischargeTarget_W:   1380,
    evCharging_W:        1380,
    evLastAmps:          6,
    phaseL1_A:           8.00,
    phaseL2_A:           2.00,
    phaseL3_A:           2.00,
    worstHeadroom_A:     8.00,
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
| Planner status yellow / `inChargingWindow: null` | No price data, or no hours left for today | Normal fallback mode — EMS uses ratio logic. Check the Frank Energie entity if it lasts |
| Whole flow stops, no new log entries | Planner crash on price array without today's hours | Update to Planner v2.15+ |
| SoC not updating | Wrong SoC entity | Replace `sensor.battery_state_of_charge` with your BMS entity |
| `[confirming 1/2]` or `[pending change …]` in reason | State change being confirmed | Normal — battery holds idle for 1–2 cycles |
| `maxAllowedCharge_A: 0` | Home load near grid limit | Large appliance using the headroom — EMS resumes when load drops |
| Battery reaches 99–100% | Hard stop reacts too late | Add the Battery SoC change trigger (Step 4); EMS v3.76+ |
| Battery never fills in the cheap window | Planner `MAX_AMPS` 200 (window half as long as needed) | Update to Planner v2.15+ |
| Battery charges in expensive hours | Charge window expanded into above-average hours | Update to Planner v2.15+ |
| Battery pre-discharges at midday or at night for "solar overfill" | Overfill estimate ran past sunset / triggered without sun | Update to Planner v2.15+ |
| Battery discharges briefly at charge amps when switching | Old state machine | Update to EMS v3.90+ |
| Battery discharging to cover EV load | EV load not subtracted (6A single-phase case before v3.90) | Update to EMS v3.90+; check `sensor.sloeierd_laadvermogen` |
| EV always 0A / Zaptec never changes | Switch node uses string compare | Switch node must use `msg.evShouldWrite` **is true** (boolean) |
| EV never gets a command | EMS node has 1 output | Set EMS function node to 2 outputs |
| Zaptec relay clicking all night | Every cycle written to Zaptec | Add the `evShouldWrite` switch node (Step 3) |
| EV jumps to 12A soon after 22:30 | Ramp counter not reset after a step | Update to EMS v3.90+ (EV v2.19) |
| EV stuck in `connected_finished` below 100% | Zaptec ended the session early | EV v2.13+ nudges up to 3 times (v2.19); after that, check the car's own charge limit |
| EV stops/starts at 22:30 | Switch from 1 to 3 phases | Normal Zaptec behaviour |
| EV not pausing at 0A | Zaptec ignores low current | Use `switch.sloeierd_opladen` off |
| Session kWh too low | EV load sensor reads low | Verify `sensor.sloeierd_laadvermogen` reports total W |

---

## Version History

| Version | Date | Change |
| --- | --- | --- |
| **EMS v3.90 / EV v2.19** | 2026-10-09 | Battery state machine: holds idle while confirming, never applies amps in the wrong direction, always passes through idle between charge and discharge; EV ramp max +1A per 15-min window with counter reset after every step-up and while no car is connected; single-phase EV load subtracted from discharge target and added back to L1 headroom; headroom add-back only while EV draws current; overload cut only with car connected; negative-price minimum charge no longer overrides grid headroom; finished-early nudge limited to 3 and skipped when `ev_soc` unavailable; `l1HardOverload` / `l1WarnOverload` exposed for the logger |
| **Planner v2.15** | 2026-10-09 | Fallback instead of crash when no hours remain for today; `MAX_AMPS` 200 → 100 (charge window was half the needed length); `MAX_DISCHARGE_KW` derived (4.8 kW, was 9.6); charge window never expands into above-average hours; solar-before-window capped by forecast; no overfill flag without solar; solar pre-discharge frees at least 3 kWh; hour/date from parsed timestamp |
| **EMS v3.89 / EV v2.18** | 2026-10-09 | `evPhaseMode` / `evCanCharge` always set; discharge target without EV simplified |
| **EMS v3.88 / EV v2.17** | 2026-10-09 | `evLastAmps` reset on disconnect so the first command of a new session is always sent; dead variables removed |
| **EMS v3.87 / EV v2.16** | 2026-10-09 | Dead code removed; finished-early nudge cooldown (~2.5 min); session cut sends 5A instead of 0A |
| **EMS v3.86 / EV v2.15** | 2026-09-21 | Overload response based on EV amps: EV above 6A → soft cut to 6A L1; EV at 6A → session cut |
| **EMS v3.85 / EV v2.14** | 2026-09-21 | Graduated cut 20–22.5A → 8A (replaced by v3.86) |
| **EMS v3.84 / EV v2.13** | 2026-08-19 | Force Zaptec write when `connected_finished` but car not full |
| **EMS v3.83 / EV v2.12** | 2026-07-09 | Zaptec writes only when the car is connected (`msg.evShouldWrite`) |
| **Logger v1.4** | 2026-07-09 | `shouldWrite` added to EV log lines |
| **EMS v3.82 / EV v2.11** | 2026-07-08 | `EV_NIGHT_MAX` 10 → 12A |
| **EMS v3.81 / EV v2.10** | 2026-07-08 | Night window 23:00–06:00 → 22:30–07:00 |
| **EMS v3.80 / EV v2.9** | 2026-07-07 | `EV_RAMP_UP_POLLS` 10 → 30 (~15 min per step) |
| **EMS v3.79 / EV v2.8** | 2026-07-05 | Phase headroom adds back EV's own current; ramp-down lock |
| **EMS v3.78 / EV v2.7** | 2026-06-29 | Night EV not limited by battery discharge; capped at 8A while battery charges |
| **EMS v3.77 / EV v2.6** | 2026-06-28 | Night EV up to 8A 3-phase while battery discharges |
| **EMS v3.76** | 2026-06-22 | Hard stop: SoC ≥ 97% forces `dc_amps = 0` just before output |
| **EMS v3.75** | 2026-06-20 | (superseded by v3.76) |
| **EMS v3.74 / EV v2.5** | 2026-05-23 | Daytime 3-phase EV up to 8A when `isPriceVeryLow` |
| **EMS v3.73 / EV v2.4** | 2026-05-23 | `EV_RAMP_UP_POLLS` 4 → 10; ramp counter only resets on soft cut |
| **EMS v3.72 / EV v2.3** | 2026-05-23 | Daytime 3-phase EV ramp only at price ≤ 0 |
| **EMS v3.71 / EV v2.2** | 2026-05-22 | Raw DSMR phase amps used directly; soft cut / session cut |
| **EMS v3.70 / EV v2.1** | 2026-05-21 | Slow ramp after session cut; cooldown diagnostics |
| **EMS v3.69 / EV v2.0** | 2026-05-20 | `evCarFull` sends 0A |
| **EMS v3.45** | 2026-05-09 | Solar surplus detection uses `rawNetGridW` before EV clamp |
| **EMS v3.44** | 2026-05-09 | EV session cost tracking |
| **EMS v3.43** | 2026-05-09 | `dc_amps` only when charging; `dc_power` only when discharging |
| **EMS v3.42** | 2026-05-09 | `MAX_AMPS` reduced to 100A |
| **EMS v3.40** | 2026-05-09 | Full amps when `inChargingWindow=true` |
| **EMS v3.39** | 2026-05-09 | `SOC_CRITICAL` 15%; critical charge requires price ≤ `avgPrice × 0.70` |
| **EMS v3.27** | 2026-05-05 | Mandatory night EV charging at minimum 8A |
| **EMS v3.26** | 2026-05-05 | Removed EV pause-on-battery-discharge guard |
| **EMS v3.24** | 2026-05-04 | EV blackout window 17:00–19:00 |
| **EMS v3.18** | 2026-05-03 | EV Charge Controller integrated as second output |
| **EMS v3.17** | 2026-05-02 | Solar-aware charge rate during cheap window |
| **EMS v3.16** | 2026-05-01 | Solar overfill pre-discharge branch |
| **EMS v3.15** | 2026-04-28 | Solar surplus charging suppressed during `inDischargeWindow` |
| **EMS v3.10** | 2026-04-26 | Negative price charging spread across full negative window |
| **Planner v2.14** | 2026-05-02 | Solar deduction split before/after cheap window; solar overfill pre-discharge; `cheapestHour` output |
| **Planner v2.13** | 2026-04-28 | Discharge window centered on most expensive hour; `LATE_BIAS_EUR` |
| **Planner v2.11** | 2026-04-26 | Pre-discharge before negative windows |
| **Planner v2.8** | 2026-04-25 | Solar forecast deduction |
| **Planner v2.6** | 2026-04-25 | Charge window centered on cheapest hour |
| **Planner v2.4** | 2026-04-19 | Frank Energie as price source |
