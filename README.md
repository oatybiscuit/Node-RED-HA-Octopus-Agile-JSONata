# Node-RED-HA-Octopus-Agile-JSONata

[![GitHub Release](https://img.shields.io/github/v/release/oatybiscuit/Node-RED-HA-Octopus-Agile-JSONata)](https://github.com/oatybiscuit/Node-RED-HA-Octopus-Agile-JSONata/releases)
![GitHub Downloads (all assets, latest release)](https://img.shields.io/github/downloads/oatybiscuit/node-red-ha-octopus-agile-jsonata/latest/total)
[![License: MIT](https://img.shields.io/github/license/oatybiscuit/node-red-ha-octopus-agile-jsonata)](/LICENSE)

[![Node RED](https://img.shields.io/badge/Node--RED-8f0000)](https://nodered.org/)
[![JSONata](https://img.shields.io/badge/JSONata-285154)](https://jsonata.org/)
[![Octopus Agile](https://img.shields.io/badge/Octopus_Agile-f050f8)](https://octopus.energy/help-and-faqs/articles/what-is-agile-octopus-and-how-do-i-join/)
[![Home Assistant](https://img.shields.io/badge/Home_Assistant-038fc7)](https://www.home-assistant.io/)

A **Node-RED** flow to read, process, and store into context variables, **Octopus Agile Tariff Prices** using **JSONata**. Ability to create sensors in **Home Assistant** for agile prices, best periods, and a binary switch, with potential to use and display these in graph and table format.

![node-red-flow](/images/NodeRedFlow20240201.png)

>[!NOTE]
> I don't use Octopus Agile myself, so this was formed out of personal interest rather than necessity. It does work, but it is a *learning example* rather than active home automation forged in the white heat of real need.

## What this Node-RED flow does

<details>
<summary> Flow action </summary>

This flow will trigger once every day to:

- Read Octopus agile *import* and also agile *export* tariffs for the most recent 48 hours (96 records)
  - merge both import and export tariffs into one array
  - add local time (DST aware) to the provided UTC timestamps
  - save this tariff array to flow context
  - provide an HA sensor with the current and next prices, updated every half hour
  - provide the full tariff array in a sensor attribute

- Process the tariff array to obtain the *best price periods* for tomorrow
  - save 15 of the best price records into flow context
  - combine consecutive periods to obtain *blocks* of best prices
  - save the best contiguous blocks into context
  - provide an HA sensor with the best price blocks for the day

- Setup and run a binary sensor (switch) for the *import* tariff
  - using the best 10 price periods, regardless of when they occur
  - turn on and off by schedule, with persistent memory and restart recovery

</details>

## Prerequisites

> [!WARNING]
> If you are not comfortable running Node-RED then please do not attempt to use this code! This code uses JSONata throughout rather than function nodes and JavaScript. JSONata is a declarative-functional language. This may require more time and effort to understand, modify and debug the code *should you wish to make changes*.

<details>
<summary> Installation requirements </summary>

This is a **Node-RED** flow, written to run in Node-RED alongside Home Assistant. You will need:

- **Node-RED**, ideally running as a [Home Assistant add-on](https://github.com/hassio-addons/addon-node-red#readme)
- The API call *references* for your own or chosen Agile tariff and pricing area
- The [WebSocket nodes](https://github.com/zachowj/node-red-contrib-home-assistant-websocket) installed and their HA server configuration working correctly
- Additional palette nodes for the scheduler - [cron-plus](https://flows.nodered.org/node/node-red-contrib-cron-plus)

</details>

## Octopus Agile Tariffs

<details>
<summary> Octopus Agile and API calls </summary>

**Octopus Energy UK** provide API calls to access tariff information. These product calls are open and do not need any authorization. The calls do require a valid product reference, and the correct area code.

You can find out more about [Octopus Energy API Documentation - Agile](https://developer.octopus.energy/docs/api/#agile-octopus) in their documentation.

The API call I am currently using is:

 `https://api.octopus.energy/v1/products/AGILE-22-08-31/electricity-tariffs/E-1R-AGILE-22-08-31-L/standard-unit-rates/?page_size=96`

The key elements in this are the tariff name **AGILE-22-08-31** which is the product I am using. Other Agile tariff products are available. The **L** is for *my* region (South-West) here in the UK. The [regions are explained in detail here](https://www.energy-stats.uk/dno-region-codes-explained/)!

A great place to find all the Agile import products and regions, and to check that your settings are generating the correct price, is the [agileprices.co.uk website](https://agileprices.co.uk/?tariff=AGILE-22-08-31&region=L)

Every day, reliably around 16:00 local time, Octopus publish the next set of records by adding to the end of the tariff file. The records are for each 30 minute period, starting at the hour and the half-hour. Thus the file grows by 48 records every day. Tariffs are published using UTC time only. They are priced on the European market, so issued from CET midnight, and therefore run from 23:00 today to 23:00 tomorrow UK time. For the sake of my sanity, each daily set will be called 'today' and 'tomorrow', with the 'tomorrow' set including the two records 23:00-23:30 and 23:30-00:00 from today.

> The flow maintains only the most recently published 96 records - hence this should always be the full set of 'yesterday' and 'today' up to around 16:00, and the full set for 'today' and 'tomorrow' after 16:00.

### Daily and half-hourly updates

Every half hour (on the hour and at 30 minutes past) the current tariff price will be fetched from the context tariff array and presented in the Home Assistant sensor. The flow will count the *remaining* records in the array at each update, and when less than 10 will initiate new API calls. Where tariff updates are delayed, this means that the flow will continue to poll the API regularly until a successful read occurs.

At each daily update, the set of best price periods, contiguous best periods, and the next binary sensor schedules are re-calculated. Although these new calculations will only be based on the figures for the *next* day, any pending existing periods and binary schedules from *today* will be retained.

</details>

## Installing the flow

As is standard, the Node-RED flow is contained within a JSON file. The file contents can be copied, and imported using the usual Node-RED import from clipboard facility, or loaded directly from file.

> [!TIP]
> If you are updating from an earlier version, may I suggest that you disable all the flow entity sensor nodes and their corresponding sensor configuration nodes first! This 'shuts down' the old flow and reduces the risk of potential conflict! You may well see warning messages that you are importing nodes that already exist!

<details>
<summary> Installing the Node-RED code </summary>

> To avoid potential issues with an update, it may be worth taking a full backup of your existing flow, disabling the sensor and sensor-configuration nodes, redeploying and restarting Home Assistant and Node-RED, so as to remove the existing entity registrations in Home Assistant first. Then deleting the existing flow entirely before importing the update, so as to prevent duplication of the sensor or configuration nodes and the problems this can sometimes generate.

- Save the Node-RED flow (see the release file). In Node-RED go to the hamburger menu, select ‘import’, and import the JSON flow file.

- Add any missing nodes to your palette. You will need [node-red-contrib-cron-plus](https://flows.nodered.org/node/node-red-contrib-cron-plus) to run the binary sensor schedules.

- Deploy and test that the API calls work by manually triggering the Inject node and checking that the context is updated with saved variables.

</details>

### Setting your parameters

**The API calls in the flow need to be hard-coded for your particular product and region.**

<details>
<summary> Product and region setting </summary>

If you are using Octopus Agile, then you can obtain your product code and region from your current account. If, like me, you are using this for interest only, then you need to search the Octopus Agile products for something that suits, and check your region using the link above. Make the necessary changes and test that the flow works by using the Inject node, and looking at the flow context variables. A working flow will produce an *OctAgileTariff* variable in flow context containing a 96-record array.

</details>

### Connection to Home Assistant

As good practice, the Home Assistant WebSocket nodes have been [scrubbed](https://zachowj.github.io/node-red-contrib-home-assistant-websocket/scrubber/) of their Home Assistant service configuration details, and *additionally disabled*. **You will need to edit these nodes, *and* the node configuration node behind each one, *and possibly the Home Assistant server configuration node itself.***

> [!IMPORTANT]  
> As a *minimum*, you will need to edit the sensor configuration nodes, at least to update and redeploy, in order to reconnect these configuration nodes with *your* default homeassistant server.

<details>
<summary> Connecting to Home Assistant </summary>

- If you are not already running a working connection with Home Assistant, make sure that you have installed the WebSocket nodes in the Node-RED palette, the [Node-RED companion integration](https://github.com/zachowj/hass-node-red) in Home Assistant, and you have correctly configured the *Home Assistant server configuration node*.
  - You should already be able to see a global context variable in Node-RED called `homeassistant` which contains a copy of the Home Assistant state.

In full detail:

- double click each Home Assistant Sensor node in turn to edit
  - enable the node (option bottom left)
  - click the edit pencil next to the *Entity config* entry to edit the ha-entity-config node
  - enable the ha-entity-config node (option bottom left)
  - check the *Server* entry - this should be HomeAssistant or similar
    - if you *do not* have a working HomeAssistant server, use the *Add new server...* option and set up the server
    - if you do have a working HomeAssistant server, ensure this is correctly selected in the *Server* entry field
  - update the *server* node, the *ha-entity-config* node, and finally save the *sensor* node and redeploy the flow once you have updated all the sensor nodes

![configuration-setup](/images/configsetup.png)

</details>

### Other settings

> [!CAUTION]
> The tariff array is passed to Home Assistant as a sensor attribute. The Home Assistant Recorder has a capacity limit and will generate an error message in the Home Assistant logs, to the effect that this entity state and attributes are too large and are not being saved by the Recorder.

There is no way to overcome this without reducing the size of the array. The error messages *are advisory* but can be prevented by adding the following to your Home Assistant configuration file. This prevents the Recorder from attempting to save this particular sensor.

```yaml
recorder:
  exclude:
    entities:
      - sensor.octopus_agile_prices
```

## Using this Node-RED flow

The flow should run without issue. Octopus Agile updates have been reliable, with only a short period of late updates, and a brief issue with corrupt (duplicate) data records. The flow here has no error checking and recovery - if the API calls fail then they will be repeated hourly until they succeed. If the Octopus Agile Tariff data is corrupt, then there is not much we can do about it (and you can always check on one of the many other websites providing Agile Tariff details to see if they are having issues too...)

### Daylight Saving Time (DST)

<details>
<summary> How DST and local time is managed </summary>

Octopus provide all Agile tariffs with times based on UTC. For the UK this is the same as GMT in the winter, but local time becomes BST or UTC +1 in the summer. Ideally device switching should operate using UTC throughout so as to avoid DST change issues, however to facilitate 'local time' for display and control, the flow creates a DST change-over record in context, and uses this to add local time to each record.

Daylight Saving Time (DST) is calculated using the European / UK rule of 01:00 UTC on the last Sunday in March and in October. This information is held in context, refreshed for each new year, and updated on every API update.

> DST changes and using 'local time' rather than UTC can give rise to timing issues. Whilst this flow provides UTC, local time, and DST change information, and has been successfully tested over BST to GMT transition, care should be taken when using local time rather than UTC, and when dealing with periods that span over DST time changes.

</details>

### Context variables

<details>
<summary> Details of all the flow context variables </summary>

#### OctDST

Holds sufficient information to be able to determine the current timezone setting and the next change, with fields for:

- the current year
- last update timestamp and Unix ms
- DST start and stop, as timestamps and Unix ms
- current DST offset as ms
- midpoint between DST on and off
- current timezone as "GMT" or "BST"
- DST on, DST changing shortly, and direction of change Boolean flags
- timezone changing from and to ("GMT"/"BST")

This is created on first update, recreated each new year, and updated on every API call.

#### OctAgileTariff

Holds an *array* of **96 periods**, each period being an *object* with fields for:

- period from and to timestamps in UTC
- import price (inc VAT)
- export price (inc VAT)
- separate fields for date and times
- local date and times
- DST changing flag
- ISO format from and to timestamps

This array is updated on successful API update, and 'tomorrow' will *in effect* replace 'yesterday'.

#### OctAgileBest

Holds an *object* for **import** and **export**, each being an array of the best 15 periods. Each period is an *object* with fields for:

- period from and upto timestamps in UTC
- separate date and time strings
- date, from and to times in local time
- DST changing flag
- ISO from and to timestamps
- tariff (inc VAT)
- year-month-day reference
- period index, concurrency links, and position

This array will hold *both* the best periods for 'yesterday' and 'today', and is updated to hold those for 'today' and 'tomorrow' on successful API update. The year-month-day field is a reference to the effective date of each tariff day, and can be used to differentiate between the periods belonging to either day.

The period position will be 'only' for single isolated periods, and 'start', then 'middle' for more than two in a row, and finally 'end' for any period groups that are concurrent within the 15 saved.

#### OctAgilePeriod

Holds an object with arrays for import, export, and both with the best concurrent periods, and fields for 'today' and 'tomorrow'.

The best 15 sampled periods are extracted to provide just *consecutive* time-blocks of two or more periods, thereby reducing the instances of on-off switching required. Each block contains:

- mode (import / export)
- original best period sample size (15)
- year-month-day reference to distinguish day-sets
- sequence of the contiguous block in each day
- range, showing the original sample from-to indexes
- from (start) and upto (end) timestamps in UTC
- separate date, time from and time to
- local time from and to
- ISO format local time from and to
- count of periods in the contiguous block
- duration in minutes
- average price value for the block
- DST changing flag (if start and end times are in different timezones due to DST)

#### OctAgileInBin

At each API update, a further reduced selection of 10 of the 15 best periods are taken from the import best periods array and used 'as is' without blocking.

This reduced array is checked for concurrent periods, and a minimal set of on and off time schedules are created. For a selection of 10 half-hour periods, this will provide the lowest import cost for five hours during a set tariff 'day'.

The scheduling is carried out using a cron-plus node, which is programmed with the necessary dynamic schedules, created at each API update. Dynamic schedules are held until used and subsequently deleted, so schedules from 'today' can still remain when the new schedules for 'tomorrow' are added.

At API update, a status command request is used to read out the retained dynamic schedule table, and this output is stored into the OctAgileInBin context variable. It is read back from context when each schedule event is triggered, so as to be able to add the full schedule array back into the Binary Sensor attribute 'array' variable.

The status array objects contain cron-plus scheduling configuration and status data, which is of little practical use. However the important full schedule array is extracted from the array config.payload field.

</details>

### Home Assistant Sensors

#### Agile Prices

<details>
<summary> Octopus Agile Prices </summary>

The *state value* is the UTC timestamp string for the current agile period start time.

Attribute fields hold:

- a reduced (fields removed) copy of the 96 entry tariff array
- values for import and for export prices for the current period
- values for import and for export prices for the next period
- a count of the number of forward periods remaining in the tariff array

This sensor is updated every half-hour.

Some fields in the array object have been deleted before passing to the sensor. This is to reduce the overall size of the attribute array. The deleted fields can be modified by adding or removing field names from the transformation-delete operation in the 'Value Now' Change node if required.

</details>

#### Agile Sequences (Periods)

<details>
<summary> Octopus Agile Sequence Table </summary>

The *state value* is the count of both import and export blocks for 'tomorrow'.

Attribute fields hold:

- an array of the *export* periods (for both 'today' and 'tomorrow')
- an array of the *import* periods (as above)
- an array containing *both* import and export periods (as above)
- the bid-offer spread value, as the difference between the lowest import period average price 'tomorrow' and the highest export period average price 'tomorrow'
- the year-month-day reference for 'today' and for 'tomorrow'

This sensor is updated only when the API calls update the tariff array, which should be once per day. Note that, since the sequence array can continue to hold any remaining future periods for 'today' as well as new ones for 'tomorrow' after update, the state value count applies *only* to the most recent day-set.

</details>

#### Agile Import Binary Switch

<details>
<summary> Octopus Agile Import Binary </summary>

The *state value* is a binary `true` or `false` and is switched according to a schedule of 'on' and 'off' times from the 10 best import periods for each day.

![binary switch](/images/binary-switch.png)

The code permits an adjustment period (here 25 seconds) which is added to the start time and subtracted from the stop time to avoid switching on the exact hour. This can be adjusted or removed as required.

This sensor is updated at each schedule 'on' or 'off' trigger time, and *also* at each API update, *as well as* at Node-RED or flow restarts and deployments.

Attribute fields will hold:

- the event reference "20240203-1/2" combining the year-month-day with the event out-of sequence total
- time on and time off as ISO timestamps
- switching period duration in minutes
- average price over the period
- array of stored binary schedules

The *event* attribute fields are only set for each *schedule* trigger. These values will be the same for both the "ON" and "OFF" event triggers.

The *array* attribute field is set for any Node-RED restart, flow update, or schedule trigger, and will contain all of the current dynamic schedules held in the cron-plus node.

> The cron-plus node is set for *persistent* dynamic schedules, so these are saved to file and recovered after a Node-RED restart. The dynamic schedules are normally only updated *once* each day when the API call updates. At this point, used and expired schedules are first deleted from the node, then the newly created schedules for the *next* day are added, then the schedule array is read and saved to context. At any *schedule trigger*, the saved schedule array is read back from context (not from the node) for the sensor attribute. The sensor array may therefore contain a mix of now expired, current, and future schedules.

</details>

> [!WARNING]
> The code will attempt to recover an 'in-flight' schedule when *restarting* the flow for any reason. If the most recently used and expired schedule was for an "ON" event, **then this will be re-issued and the sensor turned on**. If the expired schedules have been removed, but the *next pending* schedule is for "OFF" with a *currently active time period*, then an "ON" schedule will be regenerated and **used to turn the sensor on for the remainder of the current schedule period**.

### Flow parameters

Where possible flow parameters are exposed in easy to change settings, in the Change node just before the appropriate JSONata code. The 15 best sample size, and the 10 binary sensor sub-sample size can both be easily changed.

The context store read and writes are all performed in Change nodes. This facilitates more easily selecting a different store than 'default' if you have multiple context stores and wish to make any of the context variables persistent.

## Display

### Sensor entities

The provided variables are mostly held in the sensor *attributes*, and these can be extracted within Home Assistant using appropriate template functions and configuration settings. Attributes are used for passing the arrays, out of necessity, however the flow could easily be modified to add further entity sensors, should you prefer to pass attributes as single entities.

As an example, the main Agile tariff **sensor.octopus_agile_prices** can provide a display of the current and next prices. The card shown here is a standard entities card, with the YAML configuration as given below.

![agile entities](/images/agile-entities.png)

<details>
<summary> Entity card YAML configuration </summary>

```yaml
type: entities
entities:
  - entity: sensor.octopus_agile_prices
    name: Time period
    secondary_info: last-updated
    icon: mdi:av-timer
  - type: attribute
    entity: sensor.octopus_agile_prices
    attribute: periods_left
    name: Periods Left in array
  - type: attribute
    entity: sensor.octopus_agile_prices
    attribute: import
    name: Import Price - now
    suffix: p
    icon: mdi:currency-gbp
  - type: attribute
    entity: sensor.octopus_agile_prices
    attribute: import_next
    name: Import Price - next
    suffix: p
    icon: mdi:currency-gbp
  - type: attribute
    entity: sensor.octopus_agile_prices
    attribute: export
    name: Export Price - now
    suffix: p
    icon: mdi:currency-gbp
  - type: attribute
    entity: sensor.octopus_agile_prices
    attribute: export_next
    name: Export Price - next
    suffix: p
    icon: mdi:currency-gbp
  - type: divider
  - type: attribute
    entity: sensor.octopus_agile_sequence_table
    attribute: bid_offer_spread
    name: Best bid-offer
    suffix: p
    icon: mdi:basket
title: Octopus Agile
show_header_toggle: false
state_color: false
```

</details>

### Tables

Every sensor provides an *array* of values, and one of the better ways to display array data is using tables. I use the custom component [flex-table-card](https://github.com/custom-cards/flex-table-card), which can display a sensor attribute array as a table.

#### Agile Tariff

<details>
<summary> Agile Tariff in a table display </summary>

The Agile Tariff array is a good example of what can be shown. This table has been coded to show the timestamp in *local time* (so BST in summer) and to colour-highlight import prices based on a set value, as well as only showing the remaining table going forward from the current active period.

![agile tariff as a table](/images/agile-import-table.png)

The custom flex-table-card requires a configuration to achieve this, provided as shown below. Note the use of a hidden column and the 'strict' setting in order to select only the future periods.

```yaml
type: custom:flex-table-card
strict: true
title: Octopus Agile Rates
entities:
  include: sensor.octopus_agile_prices
columns:
  - data: array
    modify: >-
      (new Date(x.from)).toLocaleString("en-GB", {day:"2-digit", month:"short",
      hour:"2-digit", minute:"2-digit", timeZoneName:"short"})
    name: Start Time (local)
  - data: array
    modify: |-
      let disp=(new Date(x.from) > new Date());
      if (disp) 'yes';
    name: TEST
    hidden: true
  - data: array
    modify: x.export
    name: Export
  - data: array
    modify: |-
      let val=x.import;
      let cr="f0a0a0";
      if   (val<15) cr="a0f0a0";
      else if (val<25) cr="f0f0a0";
      '<div style="background-color: #' + cr + ';">' + val + '</div>';
    name: Import

```

</details>

#### Agile Periods

<details>
<summary> Best price periods in a table display </summary>

Here I display the best-price period sequences from **sensor.octopus_agile_sequence_table**. This table includes both the import and export periods, and now includes the 'today' set as well as 'tomorrow'. The card shows periods that relate to 'today' using `*` and the set from 'yesterday' or 'tomorrow' with `-` and `+` respectively. The 'D' column is driven by the current timestamp, so a set showing `* +` will change to show `- *` over midnight, and then revert to `* +` at the next API call update.

![best price periods table](/images/best-periods-table.png)

The necessary configuration settings are given below.

```yaml
type: custom:flex-table-card
title: Octopus Agile Period
entities:
  include: sensor.octopus_agile_sequence_table
columns:
  - data: both_array
    modify: x.mode.substr(0,3)
    name: Md
  - data: both_array
    modify: |-
      let blk=x.ymd;
      const dt=new Date();
      const yyyy=dt.getFullYear();
      let dd = dt.getDate();
      let mm = dt.getMonth() +1;
      if (mm < 10) mm = '0' + mm;
      if (dd < 10) dd = '0' + dd;
      let test = yyyy + mm + dd;
      if (test==blk) {"*"} else if (test<blk) {"+"} else {"-"};
    name: D
  - data: both_array
    modify: >-
      (new Date(x.from)).toLocaleString("en-GB", {day:"2-digit", month:"short",
      hour:"2-digit", minute:"2-digit", timeZoneName:"short"})
    name: Start
  - data: both_array
    modify: >-
      (new Date(x.upto)).toLocaleString("en-GB", {day:"2-digit", month:"short",
      hour:"2-digit", minute:"2-digit", timeZoneName:"short"})
    name: Stop
  - data: both_array
    modify: x.duration
    name: Mins
  - data: both_array
    modify: x.average
    name: Price (p)

```

</details>

#### Binary Schedules

<details>
<summary> Binary Sensor - schedule table </summary>

![binary schedule table](/images/binary-schedule-table.png)

This table is formed from the cron-plus dynamic schedules, written as an array to context store following the daily API call update, which is then read back into the sensor attribute at each schedule or update trigger. It is therefore an *historic* record of the actual schedules held in the cron-plus node.

This table can hold schedules that have expired (used and not active in the cron-plus node) as well as a record of schedules that have been used (now in the past). To mark this, the table shows blank for cron-plus expired (not active) schedules, a `-` for time-expired schedules, a `>` for a currently active schedule, and `+` for future schedules.

![binary sensor two days](/images/binary-schedule-two-days.png)

Each schedule in the array record is marked with a full event ID, including the year-month-day and the position within the list for the day. In the example shown, just the `3/3` part is shown for the last unused schedule for 'today', and `1/6` through to `6/6` for the schedules for 'tomorrow', following the daily API call update.

The configuration code required for this table is given below.

```yaml
type: custom:flex-table-card
title: Octopus Agile Binary Switch Schedule
entities:
  include: binary_sensor.octopus_agile_import_binary
columns:
  - data: array
    modify: x.event.substring(9,12)
    name: Id
  - data: array
    modify: >-
      const don = new Date(x.timeon); const dof = new Date(x.timeoff); const dat
      = new Date(); if (dat>=don && dat<dof) {">"} else if (dat<=don) {"+"} else
      if (x.active) {"-"} else {""};
    name: X
  - data: array
    modify: >-
      (new Date(x.timeon)).toLocaleString("en-GB", {day:"2-digit",
      month:"short", hour:"2-digit", minute:"2-digit", timeZoneName:"short"})
    name: 'On'
  - data: array
    modify: >-
      (new Date(x.timeoff)).toLocaleString("en-GB", {day:"2-digit",
      month:"short", hour:"2-digit", minute:"2-digit", timeZoneName:"short"})
    name: 'Off'
  - data: array
    modify: x.duration
    name: Mins
  - data: array
    modify: if(x.average) {x.average} else {""}
    name: Price (p)

```

</details>

### Graphs

I use the custom [apexcharts-card](https://github.com/RomRider/apexcharts-card) to display the tariff prices. This requires several graph configuration settings as well as data generator code to get the display I want. The graph shows the Agile import price and export price, over a complete 48 hour period of 'today' and 'tomorrow', with the best-price periods also given, as well as annotations for peak and off-peak periods (based on Octopus Flux) and the current offered standard variable-tariff prices for my region.

![agile tariff graph](/images/agile-graph.png)

<details>
<summary> Agile tariff graph configuration </summary>
Card settings:

Graph_span is set to 48h, and span to start at the beginning of the day offset by -1h so the graph runs from 23:00 yesterday to 23:00 tomorrow.

Tariff series as step-line, using a data generator. This takes the *attribute.array* and pulls the *from* (timestamp) and *price* (import/export) mapping from the full array to the new time/price array required for the chart. The step-line will plot horizontally from the start (from) of the period data-point to the next data point. At the end of the line, one extra data point is needed to make the last entry run horizontally on to the end of the day. Hence the last `final.push` adding the upto timestamp (of the last period) and the last period price.

On top of the tariff prices are plotted the best-price sequences. This is a little more complex as the data comes from the other sensor, *octopus_agile_sequence_table* and requires a data generator to build an array of all the start-end data points. As this is again a step-line plot, the first data point is at the start of the graph period with value zero, then the next data point is the first sequence-start time and sequence-start average (to give a horizontal from the day start to sequence start, then a vertical at the sequence start). The data plot array therefore begins with [start, 0], then builds [sequence-start, value] and [sequence-end, 0] to give the step-hump for each sequence. The end of the plot finishes at the end of the graph with [end, 0]. For visual clarity, the sequence lower value is plotted at -12, with the graph axis to displayed from -10, thus hiding the connecting horizontal lines.

Additional annotations are added to cover the Octopus Flux periods of 02:00 - 05:00 for 'off peak' and 16:00 - 19:00 for 'peak'. Horizontal annotations add the fixed-tariff prices current for comparison. The graph 'Now' option does not work with annotations (it stops them showing) so extra code is added to self-generate a 'now' line for completeness.

Configuration code is given below:

```yaml
type: custom:apexcharts-card
header:
  show: true
  title: Octopus Agile Prices
show:
  last_updated: true
now:
  show: false
  label: now
graph_span: 48h
span:
  start: day
  offset: '-1h'
series:
  - entity: sensor.octopus_agile_prices
    data_generator: |
      let prices = entity.attributes.array;
      let ends = prices.length-1;
      let final = prices.map((item, index) => {
        return [item.from, item.import];
      });
      final.push([prices[ends].upto, prices[ends].import]);
      return final;
    curve: stepline
    name: Import
    show:
      legend_value: false
      extremas: true
    stroke_width: 2
    color: red
  - entity: sensor.octopus_agile_prices
    data_generator: |
      let prices = entity.attributes.array;
      let ends = prices.length-1;
      let final = prices.map((item, index) => {
        return [item.from, item.export];
      });
      final.push([prices[ends].upto, prices[ends].export]);
      return final;
    curve: stepline
    name: Export
    show:
      legend_value: false
      extremas: true
    stroke_width: 2
    color: green
  - entity: sensor.octopus_agile_sequence_table
    data_generator: |
      let prices = entity.attributes.import_array;
      let final = [];
      let i=0;
      let j=0;
      final[j]=[start, -12];
      j++;
      for (i=0; i<prices.length; i++){
        final[j] = [prices[i].from, prices[i].average];
        final[j+1] = [prices[i].upto, -12];
        j=j+2;
      };
      final[j]=[end, -12];
      return final;
    curve: stepline
    name: Low Import
    stroke_width: 1
    color: purple
    show:
      legend_value: false
  - entity: sensor.octopus_agile_sequence_table
    data_generator: |
      let prices = entity.attributes.export_array;
      let final = [];
      let i=0;
      let j=0;
      final[j]=[start, -12];
      j++;
      for (i=0; i<prices.length; i++){
        final[j] = [prices[i].from, prices[i].average];
        final[j+1] = [prices[i].upto, -12];
        j=j+2;
      };
      final[j]=[end, -12];
      return final;
    curve: stepline
    name: High Export
    stroke_width: 1
    color: blue
    show:
      legend_value: false
apex_config:
  annotations:
    yaxis:
      - 'y': 15
        borderColor: green
        label:
          text: EX
          borderWidth: 0
          style:
            background: '#0000'
      - 'y': 28.12
        borderColor: red
        label:
          text: IN
          borderWidth: 0
          style:
            background: '#0000'
    xaxis:
      - x: EVAL:new Date().setHours(16,0,0)
        x2: EVAL:new Date().setHours(19,0,0)
        fillColor: '#FEB019'
        label:
          text: Peak
          borderWidth: 0
          style:
            background: '#0000'
      - x: EVAL:new Date().setHours(2,0,0)
        x2: EVAL:new Date().setHours(5,0,0)
        fillColor: '#B3F7CA'
        label:
          text: Cheap
          borderWidth: 0
          style:
            background: '#0000'
      - x: EVAL:new Date(Date.now()+86400000).setHours(16,0,0)
        x2: EVAL:new Date(Date.now()+86400000).setHours(19,0,0)
        fillColor: '#FEB019'
        label:
          text: Peak
          borderWidth: 0
          style:
            background: '#0000'
      - x: EVAL:new Date(Date.now()+86400000).setHours(2,0,0)
        x2: EVAL:new Date(Date.now()+86400000).setHours(5,0,0)
        fillColor: '#B3F7CA'
        label:
          text: Cheap
          borderWidth: 0
          style:
            background: '#0000'
      - x: EVAL:Math.floor(Date.now()/1800000)*1800000
        label:
          text: Now
          position: bottom
          borderWidth: 1
yaxis:
  - min: -10

```

</details>

## JSONata

Is a very different language, so if you are looking to either understand the code used here, or to use and modify, then you are probably going to have to learn a bit about JSONata.

The JSONata documentation can be found [here](https://docs.jsonata.org/overview.html)

There is a great [sandbox](https://try.jsonata.org/) which I use for all my development work.

## Buy me a coff*ee*?

I hope that you like this. I wrote it just for fun (seriously). I am retired, and they said 'learn a language, it will keep your brain active'. I ended up learning JSONata. Brain overload.

I am no longer working, so my time is my own. If you have a problem, want to ask a question, need some help, then please ask, but *please* don't expect me to jump just for you!

People seem to go for 'buy me a coff*ee*'. I don't need more coffee - and this is just a hobby. I guess I could start 'buy me a coff*in*' as it would be of more practical use (joke-*noir*) but hey let's look on the bright side of life while we still can! Buy *yourself* a coffee. Buy *someone you know* a coffee. Buy a *random stranger* a coffee. Buy [Jason](https://github.com/zachowj) a coffee for all the work he does on the WebSocket nodes (without which you would *not* be reading this). Put the money in a jar and save it - one coffee a day for forty years adds up to a lot, I know, I saved that money so that I might have a comfortable retirement and be able to buy *me* a coffee *myself*!

Good luck and best wishes with your Octopus Agile project!
