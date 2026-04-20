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
  -
