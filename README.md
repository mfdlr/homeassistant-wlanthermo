# WLANThermo for Home Assistant

The WLANThermo bbq thermometer (https://wlanthermo.de/) uses a local wifi and optional MQTT server to push the temperature changes to.

## Setup MQTT in WLANThermo
Link your WLANThermo to the same MQTT server which Home Assistant is using. You can do this easily on the provided web interface.


## Home Assistant Configuration

### Sensor
First we add a sensor containing all information, battery status as master value. We force it to expire after 1 minute of no new MQTT message, which leads Home Assistant to set the status to `Unknown`.

Adjust your MQTT topic appropriately, so change the topic prefix `WLanThermo/NANO-2a21b5` to your modelname.

```yaml
sensor:
# Basic sensor, read everything from MQTT
- platform: mqtt
  name: "WLANThermo"
  # change topic to your modelname!
  state_topic: "WLanThermo/NANO-2a21b5/status/data"
  value_template: "{{ value_json.system.soc }}"
  # change topic to your modelname!
  json_attributes_topic: "WLanThermo/NANO-2a21b5/status/data"
  json_attributes_template: "{{ value_json | tojson }}"
  device_class: battery
  unit_of_measurement: "%"
  expire_after: 60

# Template sensor per channel
- platform: template
  sensors:
    # Channel 1
    wlanthermo_channel_1_all:
      value_template: "{{ state_attr('sensor.wlanthermo', 'channel')[0] }}"
      entity_id: sensor.wlanthermo
    wlanthermo_channel_1:
      friendly_name: "Kanal 1"
      # set to zero if wlanthermo is turned off or channel not plugged in
      value_template: >
          {% if not is_state('sensor.wlanthermo', 'unknown') %}
          {% set temp = state_attr('sensor.wlanthermo', 'channel')[0]['temp'] %}
          {% if temp >= -30 and temp <= 300 %}
          {{ temp }}
          {% else %}
          0
          {% endif %}
          {% else %}
          0
          {% endif %}
      unit_of_measurement: "°C"
      device_class: temperature
      entity_id: sensor.wlanthermo

    # Channel 2
    wlanthermo_channel_2_all:
      value_template: "{{ state_attr('sensor.wlanthermo', 'channel')[1] }}"
      entity_id: sensor.wlanthermo
    wlanthermo_channel_2:
      friendly_name: "Kanal 2"
      # set to zero if wlanthermo is turned off or channel not plugged in
      value_template: >
          {% if not is_state('sensor.wlanthermo', 'unknown') %}
          {% set temp = state_attr('sensor.wlanthermo', 'channel')[1]['temp'] %}
          {% if temp >= -30 and temp <= 300 %}
          {{ temp }}
          {% else %}
          0
          {% endif %}
          {% else %}
          0
          {% endif %}
      unit_of_measurement: "°C"
      device_class: temperature
      entity_id: sensor.wlanthermo

      # add further channels here...
```



### Input Number
Now we need to define input_number elements, so that we are able to control a slider in the UI.

```yaml
input_number:
  wlanthermo_channel_1_min:
      name: Kanal 1 min
      min: 0
      max: 100
      step: 1
      unit_of_measurement: °C
      icon: mdi:thermometer
  wlanthermo_channel_1_max:
      name: Kanal 1 max
      min: 0
      max: 150
      step: 1
      unit_of_measurement: °C
      icon: mdi:thermometer
  wlanthermo_channel_2_min:
      name: Kanal 2 min
      min: 0
      max: 100
      step: 1
      unit_of_measurement: °C
      icon: mdi:thermometer
  wlanthermo_channel_2_max:
      name: Kanal 2 max
      min: 0
      max: 150
      step: 1
      unit_of_measurement: °C
      icon: mdi:thermometer
```

### Automations (updating sensor values)
In the automations you have to define how to set the defined input_number elements (we want to use the values which are coming from MQTT) and what to do if someone changes the slider in the UI (we want to update the WLANThermo).

```yaml
# First automation: set Home Assistant sensor value from MQTT messages, so we
# update if someone changes the values on web interface
- id: '1565774399160'
  alias: WLANThermo update channels min/max from MQTT
  trigger:
  - entity_id: sensor.wlanthermo
    platform: state
  condition: []
  action:
  - data_template:
      entity_id: input_number.wlanthermo_channel_1_min
      value: '{{ state_attr(''sensor.wlanthermo'', ''channel'').0.min }}'
    service: input_number.set_value
  - data_template:
      entity_id: input_number.wlanthermo_channel_1_max
      value: '{{ state_attr(''sensor.wlanthermo'', ''channel'').0.max }}'
    service: input_number.set_value
  - data_template:
      entity_id: input_number.wlanthermo_channel_2_min
      value: '{{ state_attr(''sensor.wlanthermo'', ''channel'').1.min }}'
    service: input_number.set_value
  - data_template:
      entity_id: input_number.wlanthermo_channel_2_max
      value: '{{ state_attr(''sensor.wlanthermo'', ''channel'').1.max }}'
    service: input_number.set_value

# automation per input_number: update value set in Lovelace UI on WLANThermo
- id: '1565867288673'
  alias: WLANThermo set channel 1 min
  trigger:
  - entity_id: input_number.wlanthermo_channel_1_min
    platform: state
  condition: []
  action:
  - service: mqtt.publish
    data_template:
      topic: WLanThermo/NANO-2a21b5/set/channels
      retain: true
      payload: '{{ ''{"number":1,"min":''+states(''input_number.wlanthermo_channel_1_min'')+''}''
        }}'

# automation per input_number: update value set in Lovelace UI on WLANThermo
- id: '1565867288674'
  alias: WLANThermo set channel 1 max
  trigger:
  - entity_id: input_number.wlanthermo_channel_1_max
    platform: state
  condition: []
  action:
  - service: mqtt.publish
    data_template:
      topic: WLanThermo/NANO-2a21b5/set/channels
      retain: true
      payload: '{{ ''{"number":1,"max":''+states(''input_number.wlanthermo_channel_1_max'')+''}''
        }}'

# automation per input_number: update value set in Lovelace UI on WLANThermo
- id: '1565867288675'
  alias: WLANThermo set channel 2 min
  trigger:
  - entity_id: input_number.wlanthermo_channel_2_min
    platform: state
  condition: []
  action:
  - service: mqtt.publish
    data_template:
      topic: WLanThermo/NANO-2a21b5/set/channels
      retain: true
      payload: '{{ ''{"number":2,"min":''+states(''input_number.wlanthermo_channel_2_min'')+''}''
        }}'

# automation per input_number: update value set in Lovelace UI on WLANThermo
- id: '1565867288676'
  alias: WLANThermo set channel 2 max
  trigger:
  - entity_id: input_number.wlanthermo_channel_2_max
    platform: state
  condition: []
  action:
  - service: mqtt.publish
    data_template:
      topic: WLanThermo/NANO-2a21b5/set/channels
      retain: true
      payload: '{{ ''{"number":2,"max":''+states(''input_number.wlanthermo_channel_2_max'')+''}''
        }}'

# add additional channels here...
```

### Lovelace configuration
![Screenshot Lovelace UI WLANThermo](/screenshot.png?raw=true "WLANThermo Lovelace")

```yaml
  - title: Thermometer
    icon: 'mdi:pig'
    cards:
      - type: entities
        entities:
          - entity: sensor.wlanthermo
      - type: history-graph
        entities:
          - entity: sensor.wlanthermo_channel_1
          - entity: sensor.wlanthermo_channel_2
        title: Temperaturverlauf
      - type: conditional
        conditions:
          - entity: sensor.wlanthermo_channel_1
            state_not: '0'
          - entity: sensor.wlanthermo_channel_1
            state_not: unavailable
        card:
          type: vertical-stack
          cards:
            - type: gauge
              entity: sensor.wlanthermo_channel_1
              min: 0
              max: 250
              severity:
                green: 90
                yellow: 115
                red: 200
            - type: entities
              entities:
                - entity: input_number.wlanthermo_channel_1_min
                - entity: input_number.wlanthermo_channel_1_max
      - type: conditional
        conditions:
          - entity: sensor.wlanthermo_channel_2
            state_not: '0'
          - entity: sensor.wlanthermo_channel_2
            state_not: unavailable
        card:
          type: vertical-stack
          cards:
            - type: gauge
              entity: sensor.wlanthermo_channel_2
              min: 0
              max: 100
              severity:
                green: 0
                yellow: 0
                red: 0
            - type: entities
              entities:
                - entity: input_number.wlanthermo_channel_2_min
                - entity: input_number.wlanthermo_channel_2_max
```
