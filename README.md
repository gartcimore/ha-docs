# ha-docs
Home Assistant notes

# install HACS
https://hacs.xyz/

install mushroom cards from HACS
https://github.com/piitaya/lovelace-mushroom

install custom brand icons
https://github.com/elax46/custom-brand-icons

install Weather Chart Card
https://github.com/mlamberts78/weather-chart-card

for the mobile dashboard using mushrooms cards, in the entity card for the room add :
```yaml
{% if is_state('light.a_light_for_this_room', 'on') %}
  orange
{% endif %}
```

and add:
```yaml
#Showing the state of a temperature in a template card:
{{ states('sensor.a_temp_sensor') }}°C / {{ states('sensor.xxx_humidity_sensor')}} 💧
```

in the mushroom template card *Primary information*

add a unique name in the /Navigation path/

add Navigate action either as /Tap action/ or /Hold action/


add an helper called /lights_on/ using a template (to count ligts on)

State template :
```yaml
{{ states.light
  |selectattr('state','equalto','on')
  |list
  |length
}}
```

Check if today is Sunday
```yaml
{{ now().isoweekday() == 0 }}
```

and state class : Total
in the dashboard, add a chip card based on entity and select the helper created.

# custom sensor
there is a sensor displaying if someone is home :
binary_sensor.someone_home

```yaml
binary_sensor:
  #See if anyone is home - https://community.home-assistant.io/t/how-to-see-if-anyone-is-home/106340/4
  - platform: template
    sensors:
      someone_home:
        friendly_name: Is Someone Home
        icon_template: >-
          {% if is_state('binary_sensor.someone_home','on') %}
            mdi:thumb-up
          {% else %}
            mdi:thumb-down
          {% endif %}
        value_template: "{{ is_state('person.XX','home') or is_state('person.YY','home') }}"
 ```


# Automations

## leaves an area (work)
this will send a simple push notification with title /HA/ and message /Leaving work zone/ to a mobile phone that uses the companion app
```yaml
alias: X leaves work
description: ""
trigger:
  - platform: zone
    entity_id: person.x
    zone: zone.name_of_a_zone
    event: leave
condition:
  - condition: time
    after: "17:10:00"
action:
  - service: notify.mobile_app_a_phone
    data:
      message: Leaving work zone
      title: HA
mode: single
```
## turn light on in the morning when motion is detected
Turn on a light if motion is detected and the hours is between 11 PM and 9 AM and if the sun didn't rise and if there are no lights on already
```yaml
alias: Living room motion lights
description: Living room motion lights
trigger:
  - platform: state
    entity_id:
      - binary_sensor.netatmo_welcome_g9ddc62_occupancy
    to: "on"
    for:
      hours: 0
      minutes: 0
      seconds: 0
    from: "off"
condition:
  - condition: time
    before: "09:00:00"
    after: "23:00:00"
    weekday:
      - mon
      - tue
      - wed
      - thu
      - fri
      - sat
      - sun
  - condition: state
    entity_id: light.bibliotheque
    state: "off"
  - condition: state
    entity_id: light.salon
    state: "off"
  - condition: state
    entity_id: light.lampe_sur_pied
    state: "off"
  - condition: sun
    before: sunrise
action:
  - service: light.turn_on
    data:
      transition: 15
      brightness_pct: 60
      rgb_color:
        - 255
        - 255
        - 255
    target:
      device_id: 367f768486599ea7bba2a264bc4ce011
mode: single
```

## turn on light if occupancy is detected

```yaml
alias: placard occupancy
description: ""
trigger:
  - platform: state
    entity_id:
      - binary_sensor.philips_sml001_occupancy
    from: "off"
    to: "on"
    for:
      hours: 0
      minutes: 0
      seconds: 0
condition: []
action:
  - service: light.turn_on
    metadata: {}
    data:
      rgb_color:
        - 255
        - 255
        - 255
    target:
      area_id: placard
mode: single
```

## turn off light if occupancy is off for 1 minute

```yaml
alias: placard empty
description: ""
trigger:
  - platform: state
    entity_id:
      - binary_sensor.philips_sml001_occupancy
    from: null
    to: "off"
    for:
      hours: 0
      minutes: 1
      seconds: 0
condition: []
action:
  - service: light.turn_off
    target:
      device_id:
        - 7c846d55ae5d7550bc216cc4c84b1ab8
    data: {}
mode: single
```

## turn off light if my phone is charging during bed time

```yaml
alias: pocophone charging
description: ""
trigger:
  - platform: state
    entity_id:
      - sensor.pocophone_f1_battery_state
    to: charging
condition:
  - condition: time
    before: "06:00:00"
    after: "22:30:00"
    weekday:
      - mon
      - tue
      - wed
      - thu
      - fri
      - sat
      - sun
  - condition: state
    entity_id: person.gart
    state: home
action:
  - service: light.turn_off
    metadata: {}
    data: {}
    target:
      entity_id: light.1er
      area_id: toilettes
mode: single
```

## start a device (smart plug) if a sensor is at a certain value

```yaml
alias: start dehumidifyer
description: ""
trigger:
  - platform: numeric_state
    entity_id:
      - sensor.unknown_70_ee_50_a4_32_de_sdb_humidity
    above: 74
condition: []
action:
  - type: turn_on
    device_id: e52c865cc7178278fb75edb2cc06cc25
    entity_id: e823d05b23cfccf086f682bc9d51524c
    domain: switch
mode: single
```

## stop a device (smart plug) if a sensor is below a certain value

```yaml
alias: stop dehumidifyer
description: ""
trigger:
  - platform: numeric_state
    entity_id:
      - sensor.unknown_70_ee_50_a4_32_de_sdb_humidity
    below: 66
condition: []
action:
  - type: turn_off
    device_id: e52c865cc7178278fb75edb2cc06cc25
    entity_id: e823d05b23cfccf086f682bc9d51524c
    domain: switch
mode: single

```

## wake up on lan the NAS

```yaml
alias: Wake NAS
description: ""
trigger:
  - platform: state
    entity_id:
      - input_button.wake_nas
condition: []
action:
  - service: wake_on_lan.send_magic_packet
    metadata: {}
    data:
      mac: MAC_ADD_HERE
mode: single
```

## toggle a helper binary sensor based on calendar event
When an event will occur (offset is -20 h) check if the text of the event contains /vacance/. If yes, set the helper on, if not, set it off. Helper will be used in another automation
```yaml
alias: Calendar Toggle
description: ""
trigger:
  - platform: calendar
    event: start
    offset: "-20:0:0"
    entity_id: calendar.gart_algar_gmail_com
    id: vacances start
    alias: Vacances Start
  - platform: calendar
    event: end
    offset: "-20:0:0"
    entity_id: calendar.gart_algar_gmail_com
    id: Vacances End
    alias: Vacances End
condition:
  - condition: template
    value_template: "{{ trigger.calendar_event.summary is search('(vacance)') }}"
action:
  - choose:
      - conditions:
          - condition: trigger
            id:
              - Vacances start
        sequence:
          - service: input_boolean.turn_on
            data: {}
            target:
              entity_id: input_boolean.vacances_helper
      - conditions:
          - condition: trigger
            id:
              - Vacances End
        sequence:
          - service: input_boolean.turn_off
            data: {}
            target:
              entity_id: input_boolean.vacances_helper
mode: single
```

## When NFC tag is scanned for plants
Store in an input date time when plants are watered
```yaml
alias: Tag Plants is scanned
description: ""
trigger:
  - platform: tag
    tag_id: e0ce9ca3-fe75-4a48-b2aa-72296939befa
condition: []
action:
  - service: input_datetime.set_datetime
    metadata: {}
    data:
      datetime: "{{ now() }}"
    target:
      entity_id: input_datetime.plants_last_time_watered
mode: single
```

## Check if plants have been watered less than 14 days ago
check on Sunday (isoweekday() ==0) if plants have been watered less than 14 days ago. Compare current time (now()) to stored time from NFC tag
```yaml
alias: push plants need water
description: ""
trigger:
  - platform: template
    value_template: "{{ now().isoweekday() == 0 }}"
condition:
  - condition: template
    value_template: >-
      {{ now()|as_timestamp -
      as_timestamp(state_attr("input_datetime.plants_last_time_watered",
      "timestamp") | timestamp_local ) > 14 * 24 * 60 * 60  }}
action:
  - service: notify.mobile_app_pocophone_f1
    metadata: {}
    data:
      message: 🪴 Plants needs water ! 🚰
      title: "HA: plants"
  - if:
      - condition: state
        entity_id: person.chastity
        state: home
    then:
      - service: notify.mobile_app_chas_phone
        data:
          message: 🪴 Plants needs water ! 🚰
          title: "HA: plants"
mode: single
```

## Start timer for laundry
Start the timer for laundry. A helper timer named *laundry* needs to exists. The duration is set there

```yaml
alias: Tag Laundry is scanned
description: ""
trigger:
  - platform: tag
    tag_id: 7124c1e1-03ff-47b4-baf3-7be96c0c1bb2
condition: []
action:
  - service: timer.start
    metadata: {}
    data: {}
    target:
      entity_id: timer.laundry
mode: single
```

## Notify when timer for laundry is done
when the timer for laundry, started by scanning NFC tag, is done then send a notifications to the phones

```yaml
alias: laundry done
description: ""
trigger:
  - platform: state
    entity_id:
      - timer.laundry
    to: idle
    from: active
condition: []
action:
  - service: notify.mobile_app_pocophone_f1
    metadata: {}
    data:
      message: linge propre 🧼 👕
      title: "HA : lave-linge"
  - service: notify.mobile_app_chas_phone
    metadata: {}
    data:
      title: "HA : washing machine"
      message: linge propre 🧼 👕
mode: single
```
