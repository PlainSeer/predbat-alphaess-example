# PredBat + AlphaESS Modbus Example Configuration

This repository contains a **worked example** of running PredBat against an AlphaESS battery/inverter through Home Assistant using the local AlphaESS Modbus integration.

It is intended as a reference for other AlphaESS users who want to understand the approach, the workarounds that were needed, and the reasoning behind the configuration.

It is **not** intended to be copied blindly into another installation.

> [!WARNING]
> This configuration can cause Home Assistant/PredBat to write directly to an AlphaESS inverter through Modbus. A wrong entity, power limit, SoC limit, dispatch mode, tariff rule or automation can cause unexpected charging, discharging, importing or exporting. It may increase electricity costs and, in the worst case, place equipment outside the operating conditions intended by the manufacturer.
>
> **Use this repository entirely at your own risk.**
>
> Test everything in PredBat Monitor/read-only mode first, verify every Home Assistant entity manually, and keep the AlphaESS/BMS native safety limits enabled. Nothing in this repository replaces the inverter or battery's own protection systems.

---

## What this example is trying to achieve

The setup represented by `apps.yaml` is deliberately a hybrid of several Home Assistant data sources:

- **AlphaESS Modbus** for live inverter/battery measurements and direct local control.
- **AlphaESS API** for the daily total house-load energy value used by `load_today`.
- **Solcast** for future PV generation forecasts.
- **PredBat LoadML** for future house-load forecasting.
- **BottlecapDave Octopus Energy integration** for live import/export tariff information and Intelligent Octopus Go data.
- **Hypervolt** for confirmation that the EV is genuinely charging rather than merely plugged in or scheduled to charge.

The intention is not to prove that these are the only correct data sources. They are the combination that proved most useful for this installation.

A major design principle is that **inverter control remains local through Modbus**. Cloud/API entities are used only where they provide information that is difficult or less reliable to reconstruct locally.

---

# Privacy / anonymisation

The public example must not contain account numbers, MPANs, meter serials, vehicle identifiers, Octopus device identifiers or other Home Assistant entity IDs that expose a specific installation.

In the example configuration, values such as the following should therefore be treated as placeholders:

```yaml
sensor.xxxxxx_total_load
sensor.octopus_energy_xxxxxx_intelligent_state_of_charge
number.octopus_energy_xxxxxx_intelligent_charge_target
binary_sensor.octopus_energy_xxxxxx_intelligent_dispatching
sensor.octopus_energy_electricity_xxxxxx_current_rate
```

Replace them with the corresponding entities from **your** Home Assistant installation.

Normal generic entities created by the AlphaESS Modbus integration, such as:

```yaml
sensor.alphaess_inverter_battery_state_of_charge
sensor.alphaess_inverter_battery_power
switch.alphaess_inverter_force_export
```

are shown because they describe the integration interface rather than a private account/device identifier. Your entity IDs may still differ if you renamed the device.

---

# Requirements

This example assumes you already have:

1. Home Assistant.
2. PredBat installed and running.
3. An AlphaESS inverter/battery supported by your AlphaESS Modbus integration.
4. AlphaESS Modbus TCP enabled and working locally.
5. Solcast installed in Home Assistant if you want to use the forecast section shown here.
6. The BottlecapDave Octopus Energy integration if you want the Intelligent Octopus Go portions.
7. A suitable EV charger entity if you want PredBat to know whether the vehicle is really charging.

Optional components can be removed if you do not use them.

---

# Installation approach

## 1. Install and test the AlphaESS Modbus integration first

Before involving PredBat, make sure Home Assistant can read the inverter correctly and that the required control entities exist.

At minimum, verify the following types of entities in **Developer Tools → States**:

```text
Battery SoC
Battery power
PV power
House load power
Grid power
Force charging controls
Force export controls
Dispatch switch
Dispatch mode select
Dispatch duration
```

Do **not** enable PredBat inverter writes until these have been checked independently.

## 2. Copy the example `apps.yaml`

Copy the relevant `pred_bat:` block into your PredBat/AppDaemon configuration and replace every installation-specific entity.

Important values that must be checked for your own system include:

- `soc_max`
- `battery_rate_max`
- `inverter_limit`
- `inverter_limit_charge`
- `inverter_limit_discharge`
- `inverter_limit_export`
- `battery_min_soc`
- DNO region
- tariff entities
- EV battery capacity
- export-protection windows

The numerical limits in this repository describe one AlphaESS installation. They are **not manufacturer recommendations for every AlphaESS system**.

## 3. Start in PredBat Monitor/read-only mode

Before allowing PredBat to control the inverter, confirm that:

- current battery SoC is correct;
- battery power direction is correct;
- grid import/export direction is correct;
- house load is plausible;
- PV generation is plausible;
- today's import/export/PV values increase correctly;
- the tariff is correct;
- the Solcast forecast is populated;
- LoadML starts successfully;
- EV/Intelligent Octopus information is sensible if enabled.

Only move on to live control after the data side is correct.

---

# Why this uses a custom AlphaESS Modbus inverter definition

PredBat supports many inverter families natively, but AlphaESS does not expose exactly the same control model as those inverters.

The example therefore declares:

```yaml
inverter_type: "ALPHAESS_MODBUS"
```

and tells PredBat which capabilities can be represented using Home Assistant services.

The important point is that PredBat does not write arbitrary Modbus registers itself. PredBat calls Home Assistant entities/services exposed by the AlphaESS Modbus integration.

This creates a useful separation:

```text
PredBat plan
    ↓
Home Assistant service calls
    ↓
AlphaESS Modbus integration
    ↓
AlphaESS inverter
```

This makes the control sequence visible and testable inside Home Assistant.

---

# Why both freeze capabilities are enabled

The current example contains:

```yaml
support_charge_freeze: true
support_discharge_freeze: true
```

This was not enabled simply because AlphaESS has buttons named "freeze". It does not.

The behaviours had to be mapped onto AlphaESS **Dispatch modes** that produce useful equivalents for PredBat.

The two freezes intentionally do different things.

---

# Charge Freeze — why it had to be changed

## What PredBat needs from a charge freeze

A PredBat charge-freeze period is not necessarily the same thing as "force the battery to charge".

For this installation, the desired behaviour during a charge freeze is:

- do **not** force-charge the battery from the grid;
- allow PV to supply the house;
- allow surplus PV to charge the battery;
- allow the battery to supply the house when PV is insufficient;
- avoid leaving an old forced export/charge operation active.

In other words, the requested behaviour is essentially **normal AlphaESS self-consumption**, but under an explicit PredBat-controlled dispatch state.

## The AlphaESS solution used here

The configuration stops any previous forced modes and selects:

```yaml
option: "Normal Mode (5)"
```

It then enables the AlphaESS Dispatch switch for a long duration.

The relevant sequence is conceptually:

```text
Stop Force Charging
Stop Force Charging Hold
Stop Force Export
Stop Force Discharging
Stop Force Import
Stop existing Dispatch
Select Dispatch Mode 5 — Normal Mode
Set Dispatch duration
Start Dispatch
```

This is why the current configuration's `charge_freeze_service` is more involved than the older example.

The earlier approach could treat "freeze" too much like a charging operation. Testing showed that what was actually wanted here was **normal self-consumption behaviour**, not a forced grid charge.

## Charge unfreeze / stop

When PredBat leaves that state, `charge_stop_service` turns off:

```yaml
switch.alphaess_inverter_force_charging
switch.alphaess_inverter_force_charging_hold
switch.alphaess_inverter_dispatch
```

Turning the Dispatch switch off is important. Otherwise AlphaESS could remain in the temporary dispatch state after PredBat believes the freeze has ended.

---

# Discharge Freeze — why a separate mode is required

## Desired behaviour

For this installation, a discharge freeze means:

- do not put additional energy into the battery;
- continue normal self-consumption;
- allow PV to supply the house;
- allow the battery to discharge for genuine house demand if required;
- export surplus PV rather than storing it.

That is not the same as simply setting battery power to zero.

## AlphaESS Dispatch Mode 19

The configuration therefore selects:

```yaml
option: "No Battery Charge (19)"
```

The sequence first clears every potentially conflicting force mode and then starts Dispatch Mode 19.

Conceptually:

```text
Stop Force Charging
Stop Force Charging Hold
Stop Force Export
Stop Force Discharging
Stop Force Import
Stop existing Dispatch
Select Dispatch Mode 19 — No Battery Charge
Set Dispatch duration
Start Dispatch
```

This prevents charging while retaining the useful parts of normal self-consumption.

## Discharge unfreeze / stop

`discharge_stop_service` explicitly removes both deliberate export/discharge and the temporary dispatch state:

```yaml
switch.alphaess_inverter_force_export
switch.alphaess_inverter_force_discharging
switch.alphaess_inverter_dispatch
```

Again, removing Dispatch matters because otherwise the inverter could remain in Mode 19 after PredBat has moved on to another plan state.

---

# Why every new charge/discharge operation first releases Dispatch

Both `charge_start_service` and `discharge_start_service` start with:

```yaml
- service: switch.turn_off
  entity_id: switch.alphaess_inverter_dispatch
```

This is deliberate.

A freeze uses AlphaESS Dispatch. A later PredBat plan may decide to charge or export instead. If the old Dispatch state is left active, two different control concepts may compete.

The rule is therefore:

> **Before starting a new deliberate charge/export operation, release any previous freeze/Dispatch state first.**

This makes transitions much more deterministic.

---

# Export/discharge target SoC — the +10 percentage-point correction

This is one of the most important AlphaESS-specific adaptations in the configuration.

PredBat supplies an export target as:

```text
{target_soc}
```

During testing on this setup, the physical/predicted endpoint corresponding to PredBat's export target was consistently about **10 percentage points higher**.

Examples observed during testing were of the form:

```text
PredBat target 10%  → AlphaESS physical stop approximately 20%
PredBat target 21%  → AlphaESS physical stop approximately 31%
PredBat target 64%  → AlphaESS physical stop approximately 74%
```

The important wording is **10 percentage points**, not "10 percent" mathematically.

For example:

```text
20% + 10 percentage points = 30%
```

It is **not**:

```text
20 × 1.10 = 22%
```

## Why not change PredBat's target itself?

Changing PredBat's own target would distort its planning and optimisation.

Instead, PredBat is allowed to keep its internal target and the AlphaESS-specific correction is applied only at the interface to the inverter.

The `apps.yaml` therefore calls:

```yaml
- service: script.predbat_alphaess_export_stop_soc
  target_soc: "{target_soc}"
```

That Home Assistant script receives PredBat's raw target, adds 10 percentage points, caps the result at 100%, and writes the corrected absolute SoC into:

```text
number.alphaess_inverter_force_export_stop_at_soc
```

This gives AlphaESS a **native inverter-side hard stop** rather than waiting for PredBat's next five-minute planning cycle to notice that SoC has crossed the intended boundary.

---

# Creating the +10 SoC Home Assistant script

The object PredBat calls is a **Home Assistant script**.

The easiest method is to add a script through Home Assistant's YAML/script editor.

An anonymised example is:

```yaml
predbat_alphaess_export_stop_soc:
  alias: PredBat - AlphaESS Export Stop SoC
  description: >-
    Receives PredBat's raw discharge/export target and writes the AlphaESS
    inverter stop target at +10 percentage points, capped at 100%.
  mode: restart

  fields:
    target_soc:
      name: PredBat target SoC
      description: Raw PredBat export target before AlphaESS correction
      required: true
      selector:
        number:
          min: 0
          max: 100
          step: 1

  sequence:
    - variables:
        raw_target: "{{ target_soc | float(0) }}"
        alphaess_target: "{{ [raw_target + 10, 100] | min }}"

    - service: number.set_value
      target:
        entity_id: number.alphaess_inverter_force_export_stop_at_soc
      data:
        value: "{{ alphaess_target }}"
```

After reloading scripts, Home Assistant should expose:

```text
script.predbat_alphaess_export_stop_soc
```

That is the entity referenced by `apps.yaml`.

## Test the script before PredBat uses it

In **Developer Tools → Actions/Services**, call the script manually with a harmless test value while Force Export itself is **off**.

For example:

```yaml
target_soc: 20
```

The AlphaESS number entity should become:

```text
30%
```

Test another value near the top:

```yaml
target_soc: 95
```

The result must cap at:

```text
100%
```

Do not test live export until you have confirmed the transformation is correct.

---

# Optional but recommended: independent +10 SoC watchdog

The wrapper script protects the initial write, but there is still another possible failure mode: a later Home Assistant/PredBat action could overwrite the AlphaESS stop target while Force Export is already running.

For extra protection, the reference setup uses an independent Home Assistant-side check.

To make that possible, Home Assistant needs to remember the **raw PredBat target** that the wrapper script last received.

## Step 1 — create an Input Number helper

In Home Assistant:

1. Open **Settings → Devices & services → Helpers**.
2. Select **Create helper**.
3. Choose **Number**.
4. Name it something such as `PredBat Raw Export Target SoC`.
5. Minimum: `0`.
6. Maximum: `100`.
7. Step: `1`.
8. Unit: `%`.

For example, the resulting entity might be:

```text
input_number.predbat_raw_export_target_soc
```

The exact entity ID is your choice.

## Step 2 — make the wrapper script remember the raw target

Add this action before the AlphaESS `number.set_value` action:

```yaml
- service: input_number.set_value
  target:
    entity_id: input_number.predbat_raw_export_target_soc
  data:
    value: "{{ raw_target }}"
```

The complete concept then becomes:

```text
PredBat target
     ↓
wrapper script
     ├── stores raw target in HA helper
     └── writes raw target +10 to AlphaESS
```

## Step 3 — create the watchdog automation

The watchdog can check the AlphaESS stop-at-SoC number whenever it changes and periodically while Force Export is active.

Example:

```yaml
alias: PredBat - AlphaESS Export Stop SoC Watchdog
description: >-
  Re-applies the PredBat +10 percentage-point AlphaESS export stop target if it
  is overwritten while force export is active.
mode: restart

triggers:
  - trigger: state
    entity_id: number.alphaess_inverter_force_export_stop_at_soc

  - trigger: state
    entity_id: switch.alphaess_inverter_force_export
    to: "on"

  - trigger: time_pattern
    minutes: "/5"

conditions:
  - condition: state
    entity_id: switch.alphaess_inverter_force_export
    state: "on"

  - condition: template
    value_template: >-
      {{ states('input_number.predbat_raw_export_target_soc') not in
         ['unknown', 'unavailable', 'none', ''] }}

actions:
  - variables:
      raw_target: >-
        {{ states('input_number.predbat_raw_export_target_soc') | float(0) }}
      required_target: >-
        {{ [raw_target + 10, 100] | min }}
      current_target: >-
        {{ states('number.alphaess_inverter_force_export_stop_at_soc') | float(-1) }}

  - condition: template
    value_template: >-
      {{ (current_target - required_target) | abs > 0.1 }}

  - action: number.set_value
    target:
      entity_id: number.alphaess_inverter_force_export_stop_at_soc
    data:
      value: "{{ required_target }}"
```

The automation only acts while Force Export is on and only writes when the value is actually wrong. That avoids unnecessary Modbus writes and prevents a feedback loop.

### Important limitation

This +10 correction was derived from testing this particular AlphaESS/PredBat combination. Do **not** assume every AlphaESS model, firmware version or future integration version needs exactly the same correction.

Confirm the relationship on your own installation before using it.

---

# Why `load_today` comes from the AlphaESS API

Although the main setup is described as "Modbus", this configuration deliberately uses an AlphaESS API-derived daily total-load sensor for:

```yaml
load_today:
  - sensor.xxxxxx_total_load
```

This is intentional.

PredBat needs `load_today` to represent **the cumulative energy consumed by the house today in kWh**.

The AlphaESS Modbus integration provides excellent live power data, including:

```yaml
load_power:
  - sensor.alphaess_inverter_current_house_load
```

However, at the time this configuration was developed, the available Modbus-derived daily load-energy reconstruction was not considered as trustworthy as the AlphaESS API's own daily total-load figure for PredBat forecasting.

Therefore the split is:

```text
Live house power       → AlphaESS Modbus
Battery/inverter control → AlphaESS Modbus
Today's grid import    → AlphaESS Modbus
Today's grid export    → AlphaESS Modbus
Today's PV generation  → AlphaESS Modbus
Today's total house load → AlphaESS API
```

This means the inverter does **not** depend on the AlphaESS cloud API for control. If the cloud/API sensor becomes unavailable, local Modbus control still exists, although PredBat may lose or degrade its daily load-history input until that sensor returns.

If you already have a trustworthy local monotonic kWh sensor for total house load today, you can use that instead.

---

# LoadML — why it is enabled

The example enables:

```yaml
load_ml_enable: true
load_ml_source: true
load_ml_max_days_history: 28
load_ml_database_days: 90
temperature_enable: true
```

LoadML is used so PredBat does not merely assume that future house consumption will look exactly like one historic day.

It learns from previous load behaviour and can include temperature information as an input.

The `days_previous` value is retained as part of the PredBat configuration/history context, while LoadML is the active forecasting source when enabled and healthy.

The important dependency is still accurate load data. A sophisticated forecast model cannot compensate for a broken or incorrectly-resetting `load_today` sensor.

---

# Solcast — why this configuration uses it

The example supplies PredBat with Home Assistant Solcast entities:

```yaml
pv_forecast_today: sensor.solcast_pv_forecast_forecast_today
pv_forecast_tomorrow: sensor.solcast_pv_forecast_forecast_tomorrow
pv_forecast_d3: sensor.solcast_pv_forecast_forecast_day_3
pv_forecast_d4: sensor.solcast_pv_forecast_forecast_day_4
```

PredBat needs a future PV forecast so it can answer questions such as:

- How much room should be left in the battery for tomorrow's solar?
- Is it worth buying cheap grid energy tonight?
- Is there enough expected PV tomorrow to avoid charging now?
- Is exporting battery energy now sensible if tomorrow's PV is strong?
- How much battery capacity should be retained for later?

The inverter itself knows what PV is producing **now**, but it cannot tell PredBat what cloud cover and PV generation are likely to be tomorrow afternoon.

That is why Solcast and Modbus serve different jobs:

```text
AlphaESS Modbus → what the system is doing now
Solcast         → what PV is likely to do in the future
LoadML          → what household demand is likely to do in the future
PredBat         → combines these with tariffs and battery constraints
```

## Solcast dampening / forecast quality

PredBat decisions can only be as good as the forecast data supplied to it.

If Solcast is consistently optimistic, PredBat may leave too little energy in the battery because it expects PV that never arrives. If it is consistently pessimistic, PredBat may buy/retain more grid energy than necessary.

For that reason, Solcast forecast performance and any dampening/learning should be monitored rather than changed casually after one unusual day.

The example references the normal Solcast forecast entities; it does not hard-code a private Solcast site identifier.

---

# EV charging — why actual Hypervolt power is used

The configuration uses:

```yaml
car_charging_now:
  - binary_sensor.hypervolt_ev_charging_above_1kw
```

This is a very deliberate choice.

PredBat needs to know whether the car is **actually consuming energy**, not simply whether:

- the cable is plugged in;
- Octopus has created a planned dispatch;
- Intelligent charging is enabled;
- the charger says it is available;
- the car is waiting to charge.

Those states are not equivalent.

A vehicle can be plugged in for many hours while drawing no electricity. An Intelligent Octopus dispatch can also exist before the car actually begins taking power or after charging has stopped.

Using a measured-power threshold gives PredBat a much stronger real-world signal:

```text
Hypervolt EV power > 1 kW = car genuinely charging
Hypervolt EV power <= 1 kW = do not treat it as active charging
```

The 1 kW threshold is high enough to ignore tiny standby/noise values while comfortably below normal EV charging power.

This signal supplements the Octopus dispatch information; it does not replace it.

---

# Creating `binary_sensor.hypervolt_ev_charging_above_1kw`

The cleanest method is a Home Assistant **template binary sensor** based on the Hypervolt power entity.

Replace `sensor.your_hypervolt_ev_power` below with the actual Hypervolt power entity on your system.

If that sensor reports **kW**:

```yaml
template:
  - binary_sensor:
      - name: "Hypervolt EV Charging Above 1kW"
        unique_id: hypervolt_ev_charging_above_1kw
        device_class: power
        state: >-
          {{ states('sensor.your_hypervolt_ev_power') | float(0) > 1.0 }}
        availability: >-
          {{ states('sensor.your_hypervolt_ev_power') not in
             ['unknown', 'unavailable', 'none', ''] }}
```

If the source reports **watts**, use `> 1000` instead:

```yaml
state: >-
  {{ states('sensor.your_hypervolt_ev_power') | float(0) > 1000 }}
```

After reloading template entities/restarting Home Assistant, the resulting entity can be renamed to:

```text
binary_sensor.hypervolt_ev_charging_above_1kw
```

## Alternative: helper + automation

If you prefer to create helpers in the UI rather than define a template binary sensor in YAML, create a **Toggle helper** named something like `Hypervolt EV Charging Above 1kW`, then use an automation to maintain it.

Example for a Hypervolt power sensor measured in kW:

```yaml
alias: Hypervolt - Maintain EV Charging Above 1kW Helper
mode: restart

triggers:
  - trigger: state
    entity_id: sensor.your_hypervolt_ev_power

  - trigger: homeassistant
    event: start

actions:
  - choose:
      - conditions:
          - condition: template
            value_template: >-
              {{ states('sensor.your_hypervolt_ev_power') | float(0) > 1.0 }}
        sequence:
          - action: input_boolean.turn_on
            target:
              entity_id: input_boolean.hypervolt_ev_charging_above_1kw
    default:
      - action: input_boolean.turn_off
        target:
          entity_id: input_boolean.hypervolt_ev_charging_above_1kw
```

If you use this alternative, point `car_charging_now` at the helper entity instead.

The direct template binary sensor is simpler and is the recommended option when possible.

---

# Why the EV charging sensor is also in `watch_list`

The configuration includes:

```yaml
watch_list:
  - "{octopus_intelligent_slot}"
  - "{octopus_ready_time}"
  - "{octopus_charge_limit}"
  - binary_sensor.hypervolt_ev_charging_above_1kw
  - sensor.octopus_energy_xxxxxx_intelligent_state_of_charge
```

This is important because EV charging is dynamic.

When the Hypervolt starts or stops drawing more than 1 kW, PredBat should not wait unnecessarily for a normal periodic re-plan if the change materially affects its understanding of the current cheap-rate/EV situation.

The watch list therefore gives PredBat a reason to recalculate when the relevant Home Assistant state changes.

---

# BottlecapDave Octopus Energy integration

For normal live operation, this example takes tariff and Intelligent Octopus data from Home Assistant's BottlecapDave Octopus Energy integration.

Examples include:

```yaml
metric_octopus_import: sensor.octopus_energy_electricity_xxxxxx_current_rate
metric_octopus_export: sensor.octopus_energy_electricity_xxxxxx_export_current_rate
metric_standing_charge: sensor.octopus_energy_electricity_xxxxxx_current_standing_charge
```

and Intelligent Octopus information such as:

```yaml
octopus_intelligent_slot: binary_sensor.octopus_energy_xxxxxx_intelligent_dispatching
octopus_ready_time: select.octopus_energy_xxxxxx_intelligent_target_time
octopus_charge_limit: number.octopus_energy_xxxxxx_intelligent_charge_target
```

This means PredBat's normal live tariff operation does not need to be built around direct Octopus REST calls.

---

# Why EV SoC and charge target are also supplied

The configuration supplies:

```yaml
car_charging_soc:
  - sensor.octopus_energy_xxxxxx_intelligent_state_of_charge

car_charging_limit:
  - number.octopus_energy_xxxxxx_intelligent_charge_target
```

along with the vehicle's battery capacity.

This gives PredBat more context about how much EV charging remains than a simple charging/not-charging binary state can provide.

For example, a vehicle at 78% with an 80% target presents a very different remaining demand from a vehicle at 20% with an 80% target.

These entities should only be configured if they reliably represent the vehicle being modelled.

---

# `octopus_slot_low_rate: false` — why this matters

The example deliberately uses:

```yaml
octopus_slot_low_rate: false
```

This is important for this particular operating strategy.

The intention is **not** to tell PredBat that every Intelligent Octopus planned dispatch automatically makes the whole household cheap for planning purposes regardless of what the EV is doing.

The setup instead combines Octopus dispatch information with a real charging signal from Hypervolt.

That is why the actual `car_charging_now` sensor and watch-list entry are important.

Users with a different tariff interpretation or different PredBat behaviour may choose differently; do not copy this setting without understanding what it does in your version of PredBat.

---

# Tariff comparison vs the tariff used for normal operation

The `compare_list` section is separate from the tariff PredBat uses to control the battery day-to-day.

It runs **what-if simulations** for the PredBat Compare page.

The example compares combinations such as:

- Intelligent Octopus Go import + Outgoing Fixed export;
- Intelligent Octopus Go import + Agile Outgoing export;
- Octopus Flux import + Flux export;
- manually defined comparison tariffs.

Some comparison entries use public Octopus API product URLs.

That does **not** mean the main live configuration has switched from BottlecapDave entities to direct Octopus API control.

Think of it as:

```text
Normal PredBat operation → Home Assistant tariff entities
Compare page simulations → comparison tariff definitions/API URLs
```

## DNO region

The example uses DNO region `H`, which corresponds to the region appropriate to that installation.

You must substitute your own DNO region before trusting comparison results.

Likewise, product codes and competitor tariff prices change over time. Treat comparison tariffs as dated configuration, not permanent facts.

---

# Overnight export protection

The configuration contains a `rates_export_override` that sets export value to zero during part of the normal cheap-rate period.

The intention is to prevent PredBat deliberately emptying the house battery into the grid during a period when retaining/charging energy is preferred.

This is a **planning rule**, not a hardware safety limit.

Your own export tariff, cheap-rate hours and objectives may be completely different. Review or remove this block rather than copying the times blindly.

---

# Daily energy sensors

The current layout intentionally uses different sources for different jobs:

```yaml
load_today:
  - sensor.xxxxxx_total_load

import_today:
  - sensor.alphaess_inverter_today_s_energy_from_grid

export_today:
  - sensor.alphaess_inverter_today_s_energy_feed_to_grid

pv_today:
  - sensor.alphaess_inverter_today_s_pv_generation
```

Before using these, verify that each sensor:

- reports kWh;
- represents the correct physical quantity;
- resets/rolls over in a way PredBat expects;
- remains sensible after a Home Assistant restart;
- does not jump backwards during the day.

Bad energy history can produce bad forecasts even when the live power sensors look perfect.

---

# Battery and inverter limits

The example contains values for battery/inverter power and minimum SoC.

These are **installation values**, not general AlphaESS defaults.

Before enabling live control, verify each one against:

- your inverter model;
- battery/BMS capability;
- AlphaESS settings;
- grid connection/export limitation;
- any DNO export agreement;
- cable/protection ratings where relevant.

Never increase a value merely because this repository shows a higher number.

---

# Recommended commissioning procedure

A safe commissioning sequence is:

1. Install AlphaESS Modbus and verify sensors only.
2. Confirm power directions in Home Assistant.
3. Confirm daily energy sensors.
4. Configure PredBat in Monitor/read-only mode.
5. Confirm Solcast forecasts.
6. Confirm LoadML starts and produces a plausible forecast.
7. Confirm tariff rates.
8. If using IOG, confirm BottlecapDave dispatch, target time, EV SoC and target entities.
9. Confirm `binary_sensor.hypervolt_ev_charging_above_1kw` changes only when the EV genuinely charges.
10. Test the +10 SoC wrapper script with Force Export **off**.
11. Test Charge Freeze briefly and confirm the inverter enters the intended normal/self-consumption behaviour.
12. Stop Charge Freeze and confirm Dispatch is released.
13. Test Discharge Freeze briefly and confirm battery charging is prevented while normal house supply behaviour remains.
14. Stop Discharge Freeze and confirm Dispatch is released.
15. Test a very short/low-risk charge operation.
16. Test a controlled export at a safe SoC with close supervision.
17. Confirm the AlphaESS hard-stop SoC is the corrected value you expect.
18. Only then allow normal automated PredBat control.

Do not perform first-time testing during an expensive tariff period or when the battery is near a critical SoC boundary.

---

# Troubleshooting checklist

## PredBat says freeze is unsupported

Check:

```yaml
support_charge_freeze: true
support_discharge_freeze: true
```

and verify that the corresponding service blocks exist and execute without Home Assistant service errors.

## Freeze starts but never clears

Check whether:

```text
switch.alphaess_inverter_dispatch
```

is still on after the stop service.

Both stop services in this example explicitly release Dispatch.

## Charge starts while a freeze is still active

Check the first action in `charge_start_service`. It should turn off the Dispatch switch before enabling Force Charging.

## Export goes below the intended PredBat SoC

Check:

1. Did `script.predbat_alphaess_export_stop_soc` run?
2. Did it receive `{target_soc}`?
3. Did it add **10 percentage points**?
4. Did the AlphaESS number entity update?
5. Is the Force Export stop-at-SoC entity functioning on your inverter firmware?
6. Did another automation overwrite it afterward?
7. If using the watchdog, is the raw target helper correct?

## PredBat thinks the EV is charging when it is not

Inspect the source used for `car_charging_now`.

A plugged-in sensor or Intelligent dispatch sensor is not enough for this setup. Confirm the >1 kW binary sensor is actually off when Hypervolt power is close to zero.

## PredBat remains in an EV-related plan after charging stops

Check that the Hypervolt binary sensor changes state promptly and that it is included in `watch_list` so PredBat gets a reason to recalculate.

## Load forecast looks erratic

Check the underlying `load_today` history before tuning LoadML. Missing data, resets, zero periods or an incorrectly reconstructed house-load sensor can all contaminate forecasting.

## PV plan seems consistently wrong

Compare recent Solcast forecasts with actual PV output over several days before changing dampening. One unusual weather day is not enough evidence to retune the forecast.

---

# What changed compared with the older example

The current configuration is significantly more complete than the original repository example.

## 1. Charge Freeze is now explicitly implemented through AlphaESS Dispatch

**Older approach:** freeze behaviour was comparatively simple and could be represented through force-charge controls.

**Current approach:** all forced modes are cleared, AlphaESS Dispatch Mode 5 (`Normal Mode`) is selected, and Dispatch is enabled for the freeze period.

**Reason:** the desired charge-freeze behaviour is normal self-consumption without forced grid charging.

## 2. Discharge Freeze is now supported

**Older approach:** `support_discharge_freeze` was disabled.

**Current approach:** it is enabled and implemented through AlphaESS Dispatch Mode 19 (`No Battery Charge`).

**Reason:** this produces the desired behaviour of preventing battery charging while still allowing normal house demand to be served and surplus PV to export.

## 3. Stop services now explicitly release Dispatch

Both charge and discharge stop paths turn off the Dispatch switch.

**Reason:** a freeze must not leave an AlphaESS dispatch mode behind after PredBat has moved to another state.

## 4. New charge/export operations clear a previous Dispatch freeze first

Both charge and export start paths release Dispatch before starting their new forced operation.

**Reason:** prevent stale or competing inverter control modes.

## 5. Export target SoC now goes through a Home Assistant wrapper script

**Older approach:** PredBat `{target_soc}` could be written straight into the AlphaESS export stop number.

**Current approach:** PredBat passes `{target_soc}` to `script.predbat_alphaess_export_stop_soc`; Home Assistant applies +10 percentage points (maximum 100%) and writes the corrected AlphaESS hard-stop value.

**Reason:** testing showed a repeatable ~10 percentage-point relationship between the PredBat export control target and the desired AlphaESS physical stop point on this setup.

## 6. Optional independent Home Assistant watchdog

A separate HA helper/automation can retain the raw PredBat target and reassert the corrected AlphaESS stop value if another write changes it during force export.

**Reason:** protection should not depend solely on the next PredBat planning cycle.

## 7. Modbus is now the main live/control data path

Battery SoC, live battery/PV/load/grid power, daily grid import/export/PV and inverter control come from the AlphaESS Modbus integration.

**Reason:** local communication is fast and removes cloud/API dependency from actual inverter control.

## 8. `load_today` remains AlphaESS API-derived

This is an intentional exception rather than an oversight.

**Reason:** the API daily total-load sensor has been the preferred reliable cumulative house-energy source for PredBat compared with reconstructing the same value from the available Modbus measurements.

## 9. LoadML is enabled

PredBat uses its ML load forecasting path with temperature data and historical data.

**Reason:** better modelling of future household demand than a simple single-day repetition.

## 10. Solcast forecast entities are explicitly supplied

Four days of Solcast forecast entities are mapped from Home Assistant.

**Reason:** PredBat needs future solar availability to optimise charging, retaining energy and export decisions.

## 11. EV charging uses a real power threshold

`car_charging_now` now uses:

```text
binary_sensor.hypervolt_ev_charging_above_1kw
```

**Reason:** "plugged in" and "Intelligent dispatch exists" do not prove that the EV is physically drawing energy.

## 12. EV SoC and target are supplied

PredBat receives the vehicle's current SoC, charge target and battery size.

**Reason:** this improves its estimate of remaining EV energy demand.

## 13. Intelligent Octopus entities come from BottlecapDave

Normal live tariff/dispatch information is read from Home Assistant entities.

**Reason:** one consistent HA data layer for PredBat, while public Octopus API URLs remain confined to Compare simulations.

## 14. EV/IOG entities are added to the watch list

PredBat can react when charging starts/stops or Octopus modifies the dispatch/target information.

## 15. Tariff comparison has been expanded

The Compare page can simulate several alternative tariff combinations without changing the tariff being used for normal operation.

## 16. Export-protection window is explicit

A zero-valued export override is used during the selected cheap-rate window to stop PredBat deliberately exporting battery energy during that period.

---

# What you must change before using this repository

At minimum, review all of the following:

- Home Assistant entity IDs containing `xxxxxx` or other placeholders.
- AlphaESS device/entity names if yours differ.
- `soc_max`.
- all battery/inverter/export power limits.
- minimum SoC.
- DNO region.
- tariff comparison product codes/prices.
- your tariff entities.
- your EV battery capacity.
- EV SoC/target entities.
- Hypervolt (or other charger) power entity.
- the unit used by the charger power sensor (W vs kW).
- Solcast entities.
- `load_today` source.
- export protection windows.
- +10 SoC correction suitability for your inverter/firmware.

If you do not use Octopus, Hypervolt, Solcast or the AlphaESS cloud/API integration, remove or replace those sections rather than creating fake entities simply to satisfy the example.

---

# Final safety notes

This repository documents one working strategy developed through testing. It should be viewed as an engineering example, not an official AlphaESS or PredBat configuration.

AlphaESS firmware, Home Assistant, PredBat and the custom Modbus integration may all change over time. Behaviour that is correct today may need retesting after an update.

In particular:

- never bypass BMS safety limits;
- never assume someone else's charge/export power is safe for your hardware;
- verify DNO/export restrictions;
- test Dispatch modes before relying on them unattended;
- test the +10 SoC mapping on your own system;
- keep a way to disable PredBat control quickly;
- inspect logs after configuration changes;
- re-test important control paths after major integration/firmware updates.

**No warranty is provided. Use entirely at your own risk.**

The author/contributors accept no responsibility for equipment damage, unexpected inverter behaviour, incorrect battery operation, electricity costs, data loss or any other consequence arising from the use or modification of this example.
