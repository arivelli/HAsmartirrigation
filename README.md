[![hacs_badge](https://img.shields.io/badge/HACS-Default-orange.svg)](https://github.com/custom-components/hacs)

[![Support the author on Patreon][patreon-shield]][patreon]

[![Buy me a coffee][buymeacoffee-shield]][buymeacoffee]

[patreon-shield]: https://frenck.dev/wp-content/uploads/2019/12/patreon.png
[patreon]: https://www.patreon.com/dutchdatadude

[buymeacoffee]: https://www.buymeacoffee.com/dutchdatadude
[buymeacoffee-shield]: https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png


# Smart Irrigation
![](logo.png?raw=true)

Smart Irrigation custom component for Home Assistant. Partly based on the excellent work at https://github.com/hhaim/hass/.
This component calculates the time to run your irrigation system to compensate for moisture lost by evaporation / evapotranspiration. Using this component you water your garden, lawn or crops precisely enough to compensate what has evaporated. It takes into account precipitation (rain,snow) and adjusts accordingly, so if it rains or snows less or no irrigation is required.

The component keeps track of hourly precipitation and at 23:00 (11:00 PM) local time stores it in a daily value.
It then calculates the exact runtime in seconds to compensate for the net evaporation.
Note that this is the default behavior and this can be disabled if you want more control. Also, the time auto refresh happens (if not disabled) is configurable.
This is all the component does, and this is on purpose to provide maximum flexibility. Users are expected to use the value of `sensor.smart_irrigation.daily_adjusted_run_time` to interact with their irrigation system and afterwards call the `smart_irrigation.reset_bucket` service. [See the example automations below](#step-3-creating-automation).

This component uses reference evapotranspiration values and calculates base schedule indexes and water budgets from that. This is an industry-standard approach. Information can be found at https://www.rainbird.com/professionals/irrigation-scheduling-use-et-save-water, amongst others.
The component uses the [PyETo module to calculate the evapotranspiration value (fao56)](https://pyeto.readthedocs.io/en/latest/fao56_penman_monteith.html). Also, please see the [How this works](https://github.com/jeroenterheerdt/HAsmartirrigation/wiki/How-this-component-works) Wiki page.

## Operation modes
You can use this component in various modes:
1. **Full Open Weather Map**. In this mode all data comes from the Open Weather Map service. You will need to create and provide an API key. See [Getting Open Weater Map API Key](#getting-open-weather-map-api-key) below for instructions.
2. **Full Sensors**. Using sensors. In this mode all data comes from sensors such as a weather station. Open Weather Map is not used and you do not need an API key.
3. **Mixed**. A combination of 1) and 2). In this mode part of the data is supplied by sensors and part by Open Weather Map. In this mode you will need to create and provide an API key. See [Getting Open Weater Map API Key](#getting-open-weather-map-api-key) below for instructions.

When planning to set up mode 2) (Full Sensors) or 3) (Mixed), see [Measurements and Units](https://github.com/jeroenterheerdt/HAsmartirrigation/wiki/Measurements-and-Units) for more information on the measurements and units expected by this component.

## Configuration

In this section:
- [One-time set up](#step-1-configuration-of-component)
- [List of entities and attributes created](#step-2-checking-entities)
- [Example automation](#step-3-creating-automation)
- [Optional settings](#step-4-configuring-optional-settings)
### Step 1: configuration of component
Install the custom component (preferably using HACS) and then use the Configuration --> Integrations pane to search for 'Smart Irrigation'.
You will need to specify the following:
- Names of sensors that supply required measurements (optional). Only required in mode 2) and 3). See [Measurements and Units](https://github.com/jeroenterheerdt/HAsmartirrigation/wiki/Measurements-and-Units) for more information on the measurements and units expected by this component.
- API Key for Open Weather Map (optional). Only required in mode 1) and 3). See [Getting Open Weater Map API Key](#getting-open-weather-map-api-key) below for instructions.
- Reference Evapotranspiration for all months of the year. See [Getting Monthly ET values](#getting-monthly-et-values) below for instructions. Note that you can specify these in inches or mm, depending on your Home Assistant settings.
- Number of sprinklers in your irrigation system
- Flow per spinkler in gallons per minute or liters per minute. Refer to your sprinkler's manual for this information.
- Area that the sprinklers cover in square feet or m<sup>2</sup>
 
 > **When entering any values in the configuration of this component, keep in mind that the component will expect inches, sq ft, gallons, gallons per minute, or mm, m<sup>2</sup>, liters, liters per minute respectively depending on the settings in Home Assistant (imperial vs metric system).
For sensor configuration take care to make sure the unit the component expects is the same as your sensor provides.**

### Step 2: checking entities
After successful configuration, you should end up with three entities and their attributes, listed below as well as [three services](#available-services).
#### `sensor.smart_irrigation_base_schedule_index`
The number of seconds the irrigation system needs to run assuming maximum evapotranspiration and no rain / snow. This value and the attributes are static for your configuration.
Attributes:
| Attribute | Description |
| --- | --- |
|`number of sprinklers`|number of sprinklers in the system|
|`flow`|amount of water that flows through a single sprinkler in liters or gallon per minute|
|`throughput`|total amount of water that flows through the irrigation system in liters or gallon per minute.|
|`reference evapotranspiration`|the reference evapotranspiration values provided by the user - one for each month.|`peak evapotranspiration`|the highest value in `reference evapotranspiration`|
|`area`|the total area the irrigation system reaches in m<sup>2</sup> or sq ft.|
|`precipitation rate`|the output of the irrigation system across the whole area in mm or inch per hour|
|`base schedule index minutes`|the value of the entity in minutes instead of seconds|
|`auto refresh`|indicates if automatic refresh is enabled.|
|`auto refresh time`|time that automatic refresh will happen, if enabled.|
|`force mode duration`|duration of irrigation in force mode (seconds).|

Sample screenshot:

![](images/bsi_entity.png?raw=true)

#### `sensor.smart_irrigation_hourly_adjusted_run_time`
The adjusted run time in seconds to compensate for any net moisture lost. Updated approx. every 60 minutes.
Attributes:
| Attribute | Description |
| --- | --- |
|`rain`|the predicted rainfall in mm or inch|
|`snow`|the predicted snowfall in mm or inch|
|`precipitation`|the total predicted precipitation in mm or inch|
|`evapotranspiration`|the expected evapotranspiration|
|`netto precipitation`|the net evapotranspiration in mm or inch, negative values mean more moisture is lost than gets added bu rain/snow, while positive values mean more value is added by rain/snow than evaporates|
|`water budget`|percentage of expected `evapotranspiration` vs `peak evapotranspiration`|
|`adjusted run time minutes`|adjusted run time in minutes instead of seconds.|

Sample screenshot:

![](images/hart.png?raw=true)

#### `sensor.smart_irrigation_daily_adjusted_run_time`

The adjusted run time in seconds to compensate for any net moisture lost. Updated every day at 11:00 PM / 23:00 hours local time. Use this value for your automation (see step 3, below).
Attributes:
| Attribute | Description |
| --- | --- |
|`water budget`|percentage of net precipitation / base schedule index|
|`bucket`|running total of net precipitation. Negative values mean that irrigation is required. Positive values mean that more moisture was added than has evaporated yet, so irrigation is not required. Should be reset to `0` after each irrigation, using the `smart_irrigation.reset_bucket` service|
|`lead_time`|time in seconds to add to any irrigation. Very useful if your system needs to handle another task first, such as building up pressure.|
|`maximum_duration`|maximum duration in seconds for any irrigation, including any `lead_time`.|
|`adjusted run time minutes`|adjusted run time in minutes instead of seconds.|

Sample screenshot:

![](images/dart.png?raw=true)

You will use `sensor.smart_irrigation_daily_adjusted_run_time` to create an automation (see step 3, below).

The [How this works Wiki page](https://github.com/jeroenterheerdt/HAsmartirrigation/wiki/How-this-component-works) describes the entities, the attributes and the calculations

#### Showing other attributes as entities (sensors)
[See the Wiki for more information on how to expose other values this components calculates as sensors](https://github.com/jeroenterheerdt/HAsmartirrigation/wiki/Showing-other-sensors).


### Step 3: creating automation
Since this component does not interface with your irrigation system directly, you will need to use the data it outputs to create an automation that will start and stop your irrigation system for you. This way you can use this custom component with any irrigation system you might have, regardless of how that interfaces with Home Assistant. In order for this to work correctly, you should base your automation on the value of `sensor.smart_irrigation_daily_adjusted_run_time` as long as you run your automation after it was updated (11:00 PM / 23:00 hours local time). If that value is above 0 it is time to irrigate. Note that the value is the run time in seconds. Also, after irrigation, you need to call the `smart_irrigation.reset_bucket` service to reset the net irrigation tracking to 0.

> **The last step in any automation is very important, since you will need to let the component know you have finished irrigating and the evaporation counter can be reset by calling the `smart_irrigation.reset_bucket` service**

#### Example automation 1: one valve, potentially daily irrigation
Here is an example automation that run everyday at 6 AM local time. It checks if `sensor.smart_irrigation_daily_adjusted_run_time` is above 0 and if it is it turns on `switch.irrigation_tap1`, waits the number of seconds as indicated by `sensor.smart_irrigation_daily_adjusted_run_time` and then turns off `switch.irrigation_tap1`. Finally, it resets the bucket by calling the `smart_irrigation.reset_bucket` service:
```
- alias: Smart Irrigation
  description: 'Start Smart Irrigation at 06:00 and run it only if the adjusted_run_time is >0 and run it for precisely that many seconds'
  trigger:
  - at: 06:00
    platform: time
  condition:
  - above: '0'
    condition: numeric_state
    entity_id: sensor.smart_irrigation_daily_adjusted_run_time
  action:
  - data: {}
    entity_id: switch.irrigation_tap1
    service: switch.turn_on
  - delay:
      seconds: '{{states("sensor.smart_irrigation_daily_adjusted_run_time")}}'
  - data: {}
    entity_id: switch.irrigation_tap1
    service: switch.turn_off
  - data: {}
    service: smart_irrigation.reset_bucket
```

[See more advanced examples in the Wiki](https://github.com/jeroenterheerdt/HAsmartirrigation/wiki/Automation-examples).

### Step 4: configuring optional settings
After setting up the component, you can use the options flow to configure the following:
| Option | Description |
| --- | --- |
|Lead time|Time in seconds to add to any irrigation. Very useful if your system needs to handle another task first, such as building up pressure.|
|Maximum duration|maximum duration in seconds for any irrigation, including any `lead_time`. -1 means no maximum.|
|Show units|If enabled, attributes values will show units. By default units will be hidden for attribute values.|
|Automatic refresh|By default, automatic refresh is enabled. Disabling it will require the user to call `smart_irrigation.calculate_daily_adjusted_run_time` manually.|
|Automatic refresh time|Specifies when to do the automatic refresh if enabled.|


## Available services
The component provides the following services:
| Service | Description |
| --- | --- |
|`smart_irrigation.reset_bucket`|this service needs to be called after any irrigation so the bucket is reset to 0.|
|`smart_irrigation.calculate_daily_adjusted_run_time`|calling this service results in the `smart_irrigation.daily_adjusted_run_time` entity and attributes to be updated right away.|
|`smart_irrigation.calculate_hourly_adjusted_run_time`|calling this service results in the `smart_irrigation.hourly_adjusted_run_time` entity and attributes to be updated right away.|
|`smart_irrigation.enable_force_mode`|Enables force mode. In this mode, `smart_irrigation.daily_adjusted_run_time` will also be set to the configured force mode duration.|
|`smart_irrigation.disable_force_mode`|Disables force mode. Normal operation resumes.|

## How this works
[See the Wiki](https://github.com/jeroenterheerdt/HAsmartirrigation/wiki/How-this-component-works).

## Getting Open Weather Map API key
Go to https://openweathermap.org and create an account. You can enter any company and purpose while creating an account. After creating your account, go to API Keys and get your key. If the key does not work right away, no worries. The email you should have received from OpenWeaterMap says it will be activated 'within the next couple of hours'. So if it does not work right away, be patient a bit.

## Getting Monthly ET values
To get the monthly ET values use Rainmaster (US only), World Water & Climate Institute (worldwide) or another source that has this information for your area. 
> **When entering the ET values in the configuration of this component, keep in mind that the component will expect inches or mm depending on the settings in Home Assistant (imperial vs metric system).**

### Using Rainmaster (US only)
Go to http://www.rainmaster.com/historicET.aspx and enter your 5-digit zipcode and click 'Find'. The values you are looking for are listed for each month in inch/day:
![](images/rainmaster.png?raw=true)


### World Water & Climate Institute (International)
Go to http://wcatlas.iwmi.org/ and create an account. Once logged in, enter the locations you are interested in and click 'Submit'. Be sure to select N/S and E/W according to your coordinates:
![](images/iwmi1.PNG?raw=true)

The values you are looking for are in the last column (Penman ETo (mm/day)):
![](images/iwmi2.png?raw=true)

