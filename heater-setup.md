# Heater Setup

## Overview

The heating system uses French "fil pilote" (pilot wire) technology, controlled via Zigbee devices through Zigbee2MQTT. Each heater has a pilot wire controller that accepts mode commands (off, comfort, eco, etc.).

## Architecture

```
┌─────────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Generic Thermostat │────▶│  Template Switch │────▶│   Automation    │
│  (climate entity)   │     │  (switch entity) │     │                 │
└─────────────────────┘     └──────────────────┘     └────────┬────────┘
                                     │                        │
                            ┌────────▼────────┐               │
                            │  Input Boolean  │               │
                            │  (state store)  │               │
                            └─────────────────┘               │
                                                              ▼
                                                    ┌─────────────────┐
                                                    │  Pilot Wire     │
                                                    │  (Zigbee select)│
                                                    └─────────────────┘
```

## Required Helpers (per heater)

To add a new heater to the system, create the following helpers:

| Helper Type | Purpose | Naming Convention |
|-------------|---------|-------------------|
| `input_boolean` | Stores the on/off state | `input_boolean.radiateur_<room>` |
| Template Switch | Exposes a switch entity for the thermostat | `switch.radiateur_<room>` |
| Generic Thermostat | Climate control with temperature sensor | `climate.<room>` |

## Setup Guide

### Step 1: Create Input Boolean

Settings → Devices & Services → Helpers → Create Helper → Toggle

- Name: `radiateur_<room>`
- Icon: `mdi:radiator`

### Step 2: Create Template Switch

Settings → Devices & Services → Helpers → Create Helper → Template → Template Switch

- Name: `radiateur_<room>`
- State template: `{{ is_state('input_boolean.radiateur_<room>', 'on') }}`
- Turn on action:
  ```yaml
  action: input_boolean.turn_on
  target:
    entity_id: input_boolean.radiateur_<room>
  ```
- Turn off action:
  ```yaml
  action: input_boolean.turn_off
  target:
    entity_id: input_boolean.radiateur_<room>
  ```

**Important**: The state template must include `{{ }}` curly braces.

### Step 3: Create Generic Thermostat

Settings → Devices & Services → Add Integration → Generic Thermostat

- Name: `<room>`
- Heater: `switch.radiateur_<room>`
- Temperature sensor: your room's temperature sensor
- Set temperature tolerances as needed

### Step 4: Create Automation

Create an automation that watches the input boolean and controls the pilot wire:

```yaml
alias: radiateur <room>
triggers:
  - trigger: state
    entity_id: input_boolean.radiateur_<room>
    id: radiateur on
    to: 'on'
  - trigger: state
    entity_id: input_boolean.radiateur_<room>
    id: radiateur off
    to: 'off'
actions:
  - choose:
      - conditions:
          - condition: trigger
            id: radiateur on
        sequence:
          - action: select.select_option
            target:
              entity_id: select.<pilote_device>_pilot_wire_mode
            data:
              option: comfort
      - conditions:
          - condition: trigger
            id: radiateur off
        sequence:
          - action: select.select_option
            target:
              entity_id: select.<pilote_device>_pilot_wire_mode
            data:
              option: 'off'
mode: parallel
max: 10
```

## Example: Chambre Rue

### Entities

| Entity | Value |
|--------|-------|
| Input Boolean | `input_boolean.radiateur_chambre_rue` |
| Template Switch | `switch.radiateur_chambre_rue` |
| Pilot Wire Select | `select.pilote_chambre_rue_pilot_wire_mode` |
| Temperature Sensor | `sensor.snzb_02d_bureau_temperature` |
| Climate | `climate.chambre_rue` |

### Automation

```yaml
- id: '1729776836690'
  alias: radiateur chambre rue
  triggers:
  - trigger: state
    entity_id:
    - input_boolean.radiateur_chambre_rue
    id: radiateur on
    to: 'on'
  - trigger: state
    entity_id:
    - input_boolean.radiateur_chambre_rue
    id: radiateur off
    to: 'off'
  actions:
  - choose:
    - conditions:
      - condition: trigger
        id:
        - radiateur on
      sequence:
      - device_id: e7ad9099aca509dbed29499ee25c13ce
        domain: select
        entity_id: b9dd6899508e3a9eb1588094d8304740
        type: select_option
        option: comfort
    - conditions:
      - condition: trigger
        id:
        - radiateur off
      sequence:
      - device_id: e7ad9099aca509dbed29499ee25c13ce
        domain: select
        entity_id: b9dd6899508e3a9eb1588094d8304740
        type: select_option
        option: 'off'
  mode: parallel
  max: 10
```

## Pilot Wire Modes

| Mode | Description |
|------|-------------|
| `off` | Heater completely off |
| `comfort` | Full heating |
| `eco` | Reduced temperature (-3.5°C from comfort) |
| `frost_protection` | Minimum temperature to prevent freezing |

## Current Implementations

| Room | Pilot Wire Device | Temperature Sensor |
|------|-------------------|-------------------|
| Salon | `pilote salon2` | `sensor.capteur_salon_temperature` |
| Chambre Rue | `pilote chambre rue` | `sensor.snzb_02d_bureau_temperature` |
