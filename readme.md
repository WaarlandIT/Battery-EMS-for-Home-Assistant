# Battery EMS – Installation & User Manual

**Version:** EMS v3.3 / Planner v2.4  
**Platform:** Home Assistant + Node-RED  
**License:** Free to use and modify

---

## What This Does

The Battery EMS (Energy Management System) is a Node-RED flow that automatically manages a home battery system by deciding when to charge and when to discharge. It uses dynamic electricity prices, solar production forecasts, and real-time grid measurements to maximise financial savings.

**Goals:**
- Charge the battery at the cheapest hours of the day
- Discharge the battery during expensive peak hours to avoid grid import
- Absorb excess solar production instead of exporting it
- Never exceed the grid connection capacity (phase protection)

**Outputs** — three Home Assistant entities your battery integration reads:
- `input_number.battery_dc_amps` — charge or discharge current (0–200 A DC)
- `input_boolean.battery_charging` — true when charging should be active
- `input_boolean.battery_discharging` — true when discharging should be active

---

## Requirements

### Hardware
- Home battery with 48 V DC bus, max 200 A charge/discharge
- 3-phase grid connection (20 A per phase)
- Solar panels with Growatt inverter (3-phase AC output)
- Dutch P1 smart meter (DSMR compatible)

### Software
- Home Assistant (any recent version)
- Node-RED (as HA add-on or standalone)
- Node-RED add-on: `node-red-contrib-home-assistant-websocket`

---

## Home Assistant Integrations

### 1. Frank Energie (HACS)
**Repository:** https://github.com/HiDiHo01/home-assistant-frank_energie  
**Purpose:** Provides hourly electricity prices for today and tomorrow (48-hour window)

**Installation:**
1. Open HACS → Integrations → Custom repositories
2. Add URL `https://github.com/HiDiHo01/home-assistant-frank_energie` as Integration
3. Install and restart HA
4. Go to Settings → Devices & Services → Add Integration → Frank Energie
5. No login required for price data

**Key entity used:**
```
sensor.frank_energie_prijzen_huidige_elektriciteitsprijs_all_in
```
This sensor's `prices` attribute contains the full 48-hour price schedule used by the planner.

---

### 2. DSMR Smart Meter
**Built-in HA integration** — Settings → Devices & Services → Add Integration → DSMR Smart Meter  
**Purpose:** Real-time grid import/export measurements per phase

**Requirements:**
- P1 cable connected from smart meter to HA (USB or network)
- Supported meters: DSMR v2.2, v4, v5 (Dutch), v5B (Belgian)

**Key entities used:**

| Entity | Description |
|---|---|
| `sensor.electricity_meter_energieverbruik` | Total grid import in kW |
| `sensor.electricity_meter_energieproductie` | Total grid export in kW |
| `sensor.electricity_meter_power_consumption_phase_l1` | L1 consumption kW |
| `sensor.electricity_meter_power_consumption_phase_l2` | L2 consumption kW |
| `sensor.electricity_meter_power_consumption_phase_l3` | L3 consumption kW |

> **Note:** Entity names depend on how you named your meter during setup. If your entities have different names, update them in the Node-RED flow nodes.

---

### 3. Growatt Solar Inverter (ESPHome)
**Repository:** https://github.com/WaarlandIT/ESPHOME-Growatt  
**Purpose:** Real-time AC power output per phase from the solar inverter

**Installation:**
1. Flash ESPHome firmware to an ESP32/ESP8266 connected to your Growatt inverter
2. The device automatically appears in HA via ESPHome integration

**Key entities used:**

| Entity | Description |
|---|---|
| `sensor.growatt_acpower_pac1_safe` | AC output phase 1 (W) |
| `sensor.growatt_acpower_pac2_safe` | AC output phase 2 (W) |
| `sensor.growatt_acpower_pac3_safe` | AC output phase 3 (W) |

---

### 4. Battery BMS (Your Own Integration)
The EMS does **not** include battery integration — you connect your own. The EMS reads:
```
sensor.battery_state_of_charge   ← replace with your actual entity
```
And writes commands to:
```
input_number.battery_dc_amps
input_boolean.battery_charging
input_boolean.battery_discharging
```

Your battery automation should listen to these entities and send the appropriate commands to your inverter or BMS.

---

## Installation Steps

### Step 1 — Install Node-RED

In HA: Settings → Add-ons → Add-on Store → Search "Node-RED" → Install  
After installation, open Node-RED and install the HA websocket palette:

Menu (☰) → Manage palette → Install → search `node-red-contrib-home-assistant-websocket` → Install

---

### Step 2 — Create HA Helpers

Go to **Settings → Devices & Services → Helpers → Create Helper**

| Type | Name | Entity ID | Min | Max | Step |
|---|---|---|---|---|---|
| Number | Battery DC Amps | `input_number.battery_dc_amps` | 0 | 200 | 1 |
| Toggle | Battery Charging | `input_boolean.battery_charging` | — | — | — |
| Toggle | Battery Discharging | `input_boolean.battery_discharging` | — | — | — |

---

### Step 3 — Create Template Sensors

Add to your `configuration.yaml`:

```yaml
template:
  - sensor:
      - name: "EMS Battery DC Amps"
        unique_id: ems_battery_dc_amps
        unit_of_measurement: "A"
        state_class: measurement
        state: >
          {% if is_state('input_boolean.battery_charging', 'on') %}
            {{ states('input_number.battery_dc_amps') | float }}
          {% else %}
            0
          {% endif %}

      - name: "EMS Battery Discharge Amps"
        unique_id: ems_battery_discharge_amps
        unit_of_measurement: "A"
        state_class: measurement
        state: >
          {% if is_state('input_boolean.battery_discharging', 'on') %}
            {{ states('input_number.battery_dc_amps') | float }}
          {% else %}
            0
          {% endif %}

      - name: "Frank Energie Prijs"
        unique_id: frank_energie_prijs
        unit_of_measurement: "€/kWh"
        state_class: measurement
        state: >
          {{ states('sensor.frank_energie_prijzen_huidige_elektriciteitsprijs_all_in') | float(0) }}
```

Restart HA or reload template entities via Developer Tools → YAML → Template entities.

---

### Step 4 — Import the Node-RED Flow

1. Open Node-RED
2. Menu (☰) → Import → paste the JSON from the flow artifact
3. Click Import
4. The flow appears on a new tab called **Battery EMS**

---

### Step 5 — Configure the HA Server Connection

1. Double-click any blue HA node (e.g. "Current price")
2. Click the pencil icon next to the Server field
3. Enter your HA details:
   - **Base URL:** `http://homeassistant:8123` (if Node-RED runs as HA add-on) or `http://YOUR-HA-IP:8123`
   - **Access Token:** generate one in HA → Profile → Long-Lived Access Tokens
4. Click Update → Done
5. All other HA nodes will now show the same server in their dropdown — select it for each

---

### Step 6 — Update Entity Names

Open each sensor node and verify the entity ID matches your installation. The most likely ones to need updating:

| Node | Default entity | Check if yours differs |
|---|---|---|
| DSMR power change | `sensor.electricity_meter_energieverbruik` | Search `energieverbruik` in Developer Tools |
| Grid import kW | `sensor.electricity_meter_energieverbruik` | Same as above |
| Grid export kW | `sensor.electricity_meter_energieproductie` | Search `energieproductie` |
| Phase L1/L2/L3 | `sensor.electricity_meter_power_consumption_phase_l1` | Search `power_consumption_phase` |
| Growatt PAC1/2/3 | `sensor.growatt_acpower_pac1_safe` | Search `growatt` in Developer Tools |
| Battery SoC | `sensor.battery_state_of_charge` | **Replace with your BMS entity** |

To find your actual entity names, run this in **Developer Tools → Template**:

```jinja2
DSMR:
{% for state in states %}
  {% if 'electricity_meter' in state.entity_id %}
    {{ state.entity_id }} = {{ state.state }}
  {% endif %}
{% endfor %}

GROWATT:
{% for state in states %}
  {% if 'growatt' in state.entity_id %}
    {{ state.entity_id }} = {{ state.state }}
  {% endif %}
{% endfor %}
```

---

### Step 7 — Deploy

Click the red **Deploy** button in Node-RED. The flow starts immediately. Check the debug panel — within 5 seconds you should see a diagnostics message from the EMS Diagnostics node.

---

### Step 8 — Connect Your Battery

Create an HA automation that reads the three output entities and sends commands to your battery:

```yaml
automation:
  - alias: "EMS Battery Charge"
    trigger:
      - platform: state
        entity_id: input_boolean.battery_charging
      - platform: state
        entity_id: input_number.battery_dc_amps
    action:
      - choose:
          - conditions:
              - condition: state
                entity_id: input_boolean.battery_charging
                state: "on"
            sequence:
              - service: YOUR_BATTERY_INTEGRATION.set_charge_current
                data:
                  amps: "{{ states('input_number.battery_dc_amps') | int }}"
          - conditions:
              - condition: state
                entity_id: input_boolean.battery_charging
                state: "off"
            sequence:
              - service: YOUR_BATTERY_INTEGRATION.stop_charging

  - alias: "EMS Battery Discharge"
    trigger:
      - platform: state
        entity_id: input_boolean.battery_discharging
      - platform: state
        entity_id: input_number.battery_dc_amps
    action:
      - choose:
          - conditions:
              - condition: state
                entity_id: input_boolean.battery_discharging
                state: "on"
            sequence:
              - service: YOUR_BATTERY_INTEGRATION.set_discharge_current
                data:
                  amps: "{{ states('input_number.battery_dc_amps') | int }}"
          - conditions:
              - condition: state
                entity_id: input_boolean.battery_discharging
                state: "off"
            sequence:
              - service: YOUR_BATTERY_INTEGRATION.stop_discharging
```

---

## How It Works

### Decision Priority

Each cycle the EMS evaluates conditions in this order — the first matching condition wins:

1. **Negative price** — charge at maximum grid headroom, being paid to consume
2. **Critical SoC** — SoC below 20% and price below average, charge at max regardless of planner
3. **Planner charge window** — current hour is one of today's cheapest, charge from grid
4. **Solar surplus** — exporting more than 500 W, absorb into battery (free energy, always wins)
5. **High price / discharge window** — peak price hour, discharge to cover import
6. **Idle** — no action

### Charge Window Planning

The Price Planner calculates how many hours are needed to fill the battery:

```
hoursNeeded = ceil(kwhNeeded / maxChargeRate) + 1 buffer hour
```

It then selects the cheapest N hours from today's remaining hours plus tomorrow. The price cap depends on SoC:

- **SoC ≥ 90%** — tight mode: only hours below 0.12 €/kWh qualify
- **SoC < 90%** — expanded mode: any hour below today's average price qualifies

This ensures a low battery will charge at more hours (accepting slightly higher prices) while a nearly full battery only charges at genuinely cheap moments.

### Discharge Window Planning

The planner identifies peak price hours using local maximum detection — an hour qualifies as a peak if its price is more than 0.008 €/kWh above both its neighbours. Shoulder hours (within 88% of the peak price) are also included. A morning fallback ensures at least one morning discharge hour is always selected.

### Grid Headroom Protection

The maximum charge rate is limited to prevent tripping the phase breaker:

```
headroom = GRID_MAX_A_PHASE - maxPhaseLoad - SAFETY_MARGIN
maxChargeAmps = headroom × 230V × 3 phases ÷ 48V × 0.85 efficiency
```

With `GRID_MAX_A_PHASE = 18` and `SAFETY_MARGIN_A = 2`, the effective limit per phase is 16 A before the charger kicks in.

### Discharge Targeting

During high-price discharge, the target is **gross grid import** (not net), ensuring all importing phases are covered even when other phases are exporting solar. An export bias of 800 W is added to slightly over-discharge and guarantee zero import.

---

## Diagnostics

The **EMS Diagnostics** debug node outputs a full payload each cycle. Key fields to watch:

| Field | What to check |
|---|---|
| `version` | Confirms which script versions are running |
| `reason` | Human-readable explanation of the current decision |
| `planner.inWindow` | true = currently in a planned cheap charge hour |
| `planner.inDischargeWindow` | true = currently in a planned peak discharge hour |
| `planner.plannedHours` | Array of today's charge hours |
| `planner.plannedDischargeHours` | Array of today's discharge hours |
| `inputs.maxPhaseLoad_A` | Highest phase load — charge rate limited by this |
| `inputs.worstHeadroom_A` | Available headroom before hitting phase limit |
| `inputs.dischargeTarget_W` | Gross import being targeted for zero |

### Node Status

Both function nodes show a live status line in the flow editor canvas:

**Price Planner:**
```
v2.4 CHG | SoC:84% cap:0.22 | dch>0.208EUR | hour 15h
```

**EMS Decision Engine:**
```
EMS v3.3 CHG 164A | SoC:84% | 0.126EUR | solar:8394W
```

---

## Configuration Reference

### Price Planner Settings

Open the **Price Planner** function node and edit the variables at the top.

| Variable | Default | Description |
|---|---|---|
| `PLANNER_VERSION` | `'v2.4'` | Version string — update when making changes |
| `BATTERY_KWH` | `40` | Total battery capacity in kWh |
| `BATTERY_VOLTAGE` | `48` | Nominal DC voltage |
| `MAX_AMPS` | `200` | Hardware maximum charge current (A) |
| `ROUND_TRIP_EFF` | `0.85` | Round-trip efficiency (85%) |
| `SOC_MIN` | `10` | Absolute minimum SoC % |
| `SOC_MAX` | `95` | Maximum SoC % |
| `SAFETY_BUFFER_H` | `1` | Extra hours added to charge plan as safety buffer |
| `DISCHARGE_ABS_MIN` | `0.19` | Never discharge below this price (€/kWh) |
| `CHARGE_ABS_MAX` | `0.12` | Tight mode price cap (€/kWh) |
| `SOC_NORMAL_THRESHOLD` | `90` | SoC % above which tight charge mode applies |

### EMS Decision Engine Settings

Open the **EMS Decision Engine** function node and edit the `CFG` block.

| Variable | Default | Description |
|---|---|---|
| `EMS_VERSION` | `'EMS v3.3'` | Version string — update when making changes |
| `MAX_AMPS` | `200` | Battery hardware limit (A DC) |
| `BATTERY_VOLTAGE` | `48` | Nominal DC voltage |
| `SOC_MIN` | `10` | Absolute minimum SoC — never discharge below |
| `SOC_MAX` | `95` | Maximum SoC — never charge above |
| `SOC_DISCHARGE_MIN` | `15` | Soft discharge floor in normal operation |
| `SOC_CRITICAL` | `20` | Below this SoC%, charge at any below-average price |
| `CHARGE_ABS_MAX` | `0.10` | Fallback charge threshold (no planner data) |
| `DISCHARGE_ABS_MIN` | `0.12` | Minimum price for ratio-based discharge |
| `DISCHARGE_OVERSHOOT` | `1.25` | Discharge 25% extra to guarantee zero import |
| `EXPORT_BIAS_W` | `800` | Extra watts added to discharge target |
| `HYSTERESIS` | `0.05` | Price ratio hysteresis to prevent oscillation |
| `SOLAR_SURPLUS_W` | `500` | Minimum surplus (W) to start solar absorption |
| `SOLAR_SURPLUS_EXIT_W` | `200` | Surplus (W) below which solar charging stops |
| `ROUND_TRIP_EFF` | `0.85` | Round-trip efficiency |
| `GRID_MAX_A_PHASE` | `18` | Safe maximum amps per phase |
| `GRID_VOLTAGE` | `230` | AC grid voltage |
| `SAFETY_MARGIN_A` | `2` | Safety buffer before headroom kicks in |
| `MIN_DISCHARGE_A` | `35` | Minimum discharge to avoid micro-cycles |

---

## Tuning Guide

### Battery charges too little / too late
- Lower `SOC_NORMAL_THRESHOLD` (e.g. from 90 to 80) so expanded mode stays active longer
- Raise `CHARGE_ABS_MAX` (e.g. from 0.12 to 0.15) so more hours qualify in tight mode
- Lower `SAFETY_BUFFER_H` from 1 to 0 if the battery consistently ends charge windows with capacity to spare

### Battery charges too aggressively at expensive prices
- Raise `SOC_NORMAL_THRESHOLD` (e.g. to 95) so tight mode kicks in earlier
- Lower `CHARGE_ABS_MAX` (e.g. to 0.08) to be more selective in tight mode

### Battery discharges too early / at low prices
- Raise `DISCHARGE_ABS_MIN` (e.g. from 0.19 to 0.22) to require a higher absolute price
- The discharge window is based on today's price peaks — if prices are generally high, many hours may qualify. The 88% shoulder threshold can be raised to reduce window size

### Battery discharges at wrong SoC level
- Raise `SOC_DISCHARGE_MIN` (e.g. from 15 to 25) to keep more reserve
- Raise `SOC_CRITICAL` (e.g. from 20 to 30) to trigger emergency charging earlier

### Grid import still visible during discharge
- Raise `EXPORT_BIAS_W` (e.g. from 800 to 1200)
- Raise `DISCHARGE_OVERSHOOT` (e.g. from 1.25 to 1.35)

### Phase breaker tripping during charge
- Lower `GRID_MAX_A_PHASE` (e.g. from 18 to 16)
- Raise `SAFETY_MARGIN_A` (e.g. from 2 to 4)

### Solar surplus not being absorbed
- Lower `SOLAR_SURPLUS_W` (e.g. from 500 to 300) to start absorbing at smaller surplus
- Check that `sensor.battery_state_of_charge` is returning a valid value — if it defaults to 50, the SoC protection may be blocking charging incorrectly

### Battery SoC jumping to wrong value
- Verify `sensor.battery_state_of_charge` is updating correctly in HA
- The planner caches the last SoC in Node-RED context — if the sensor is unavailable for a period, context will hold a stale value. Restart the flow to reset context.

---

## Updating Version Numbers

Both scripts define a single version variable at the top. When you change settings, increment the version so you can confirm the new code is running:

```javascript
// Price Planner — line 6
var PLANNER_VERSION = 'v2.5';

// EMS Decision Engine — line 6
var EMS_VERSION = 'EMS v3.4';
```

The version appears in the node status badge on the canvas, in the debug diagnostics output, and in the `version` field of every payload.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| All nodes show red (no connection) | HA server not configured | Open any node → pencil icon → configure server |
| Entity not found in cache | HA websocket connection reset | Restart the HA server connection in Node-RED config nodes |
| `Planner unknown` in version | Price planner not wired into the chain | Check wiring: `get-frank-prices → get-soc → price-planner → get-grid-import` |
| `inWindow: undefined` | Old planner code — `msg.plannerVersion` set before window assignments | Replace planner with latest script |
| Battery idle at cheap price | SoC above `SOC_NORMAL_THRESHOLD`, price above tight cap | Raise `SOC_NORMAL_THRESHOLD` or lower `CHARGE_ABS_MAX` |
| Battery idle with solar surplus | `actualSurplusW` below threshold, or solar sensor returning 0 | Check Growatt PAC entities are updating |
| Charging and discharging simultaneously | Template sensors both non-zero | Verify `battery_charging` and `battery_discharging` booleans are never both `on` |
| Phase breaker tripping | Phase load + charger exceeds limit | Lower `GRID_MAX_A_PHASE` |
| Grid import during discharge | Export bias too low | Raise `EXPORT_BIAS_W` |
| SoC showing 50% always | BMS entity unavailable or wrong name | Replace `sensor.battery_state_of_charge` with correct entity |
| Statistics graph shows no data | Entity missing `state_class` | Add `state_class: measurement` to template sensors |
