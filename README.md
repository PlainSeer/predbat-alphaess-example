# PredBat + AlphaESS Modbus example

This repository is a reusable starting point for connecting PredBat to an AlphaESS inverter through Home Assistant.

It is an example, not a drop-in configuration. Replace every placeholder, verify every entity and limit, and commission in read-only mode before allowing inverter writes.

## Disclaimer — use at your own risk

This repository is provided as an example without warranty or guarantee. You are responsible for deciding whether it is suitable for your equipment, firmware, tariff, electrical installation and local rules.

Incorrect configuration can cause unexpected battery charging or discharging, grid import or export, increased energy costs, breach of export limits, loss of backup reserve, equipment faults, or warranty implications. Neither the repository owner nor its contributors accept responsibility for loss, damage, costs or other consequences arising from its use.

Do not assume the entity names, mode numbers, power limits, battery capacity, minimum SoC, sign conventions or control behaviour are correct for another AlphaESS system. Verify them against live Home Assistant states, the installed inverter and battery, manufacturer information, and any installer/DNO restrictions.

Keep PredBat in Monitor/read-only mode until every sensor and service has been tested with conservative values. Maintain backups and ensure the inverter/BMS native protections remain enabled.

## What is included

- [`apps.yaml`](./apps.yaml): the reusable PredBat/AlphaESS inverter block
- [`home_assistant_alphaess_bridge.example.yaml`](./home_assistant_alphaess_bridge.example.yaml): the two rate helpers, charge/discharge rate bridges, discharge-target script and target-sync automation
- [`home_assistant_house_load.example.yaml`](./home_assistant_house_load.example.yaml): the Integral and daily Utility Meter sensors used to derive `load_today` from live Modbus house-load power
- [`home_assistant_predbat_alphaess_package.example.yaml`](./home_assistant_predbat_alphaess_package.example.yaml): an optional single-file Home Assistant package combining the reusable helpers, sensors, script and automations

The repository deliberately excludes installation-specific EV, tariff, household-control and notification automations.

## Requirements

- Home Assistant with a working AlphaESS Modbus integration
- PredBat
- AlphaESS entities for battery SoC, battery/PV/load/grid power and daily energy
- AlphaESS controls for Force Charging, Force Discharging and Dispatch
- the Home Assistant helpers and bridge automations described below
- an incrementing daily house-load energy sensor, either supplied by your integration or created from the included example

The entity names are examples. They must match the entities exposed by your own AlphaESS integration.

## Architecture

```text
PredBat plan
    ↓
Home Assistant rate helpers, scripts and service calls
    ↓
AlphaESS Modbus integration
    ↓
AlphaESS inverter
```

PredBat does not write arbitrary Modbus registers in this example. It uses Home Assistant entities and services exposed by the AlphaESS integration.

## Verify AlphaESS first

Before adding PredBat control, confirm independently in Home Assistant that:

- battery SoC is correct;
- charge and discharge power have the expected sign;
- grid import and export have the expected sign;
- PV and house-load power are plausible;
- daily import, export, PV and load sensors reset and accumulate correctly;
- Force Charging, Force Discharging and Dispatch controls behave as expected.

Do not enable PredBat inverter writes until these checks pass.

## Optional single-file Home Assistant package

If you prefer to keep the reusable Home Assistant-side configuration together, use [`home_assistant_predbat_alphaess_package.example.yaml`](./home_assistant_predbat_alphaess_package.example.yaml). It combines:

- the two PredBat rate helpers;
- the Integral and daily Utility Meter house-load sensors;
- the charge and discharge rate bridge automations;
- the Force Discharging stop-SOC script and target-sync automation.

This package is an alternative to the UI helpers and the separate example fragments. Do not install both forms with the same entity IDs. Before enabling it, remove or rename any existing duplicates and confirm that no other automation or script depends on the definitions being replaced.

The package deliberately does not include `apps.yaml`, tariff configuration, EV/Hypervolt logic, secrets or installation-specific inverter limits beyond the example helper maximums. Review every AlphaESS entity and change the 8 kW helper maximums when they do not match the safe limits of your inverter, battery or grid connection.

To load one explicit package file, add this under your existing `homeassistant:` section in `configuration.yaml`:

```yaml
homeassistant:
  packages:
    predbat_alphaess: !include home_assistant_predbat_alphaess_package.yaml
```

Copy the example file into the Home Assistant configuration directory as `home_assistant_predbat_alphaess_package.yaml`. If you already use a packages directory, you may instead place the file there and load it with your existing package include arrangement. Package names must be unique.

Before restarting:

1. Back up the current Home Assistant configuration.
2. Replace any entity names that differ on your AlphaESS integration.
3. Decide whether to keep the package or the existing UI/separate-YAML definitions, and remove duplicates.
4. Run Home Assistant's configuration check.
5. Restart Home Assistant and confirm every helper, sensor, script and automation loads.
6. Keep PredBat in Monitor/read-only mode while testing the complete chain.

Home Assistant packages merge several integration configurations into one file, but keyed entities such as helpers must still have unique keys across the main configuration and every package. See the [Home Assistant packages documentation](https://www.home-assistant.io/docs/configuration/packages/).

## Install the Home Assistant bridge

The complete example is in [`home_assistant_alphaess_bridge.example.yaml`](./home_assistant_alphaess_bridge.example.yaml). It contains definitions intended for different Home Assistant configuration areas; do not include the entire file blindly as one package unless you have converted it to your package structure.

### 1. Create the two rate helpers

PredBat expresses charge and discharge rates in watts. The AlphaESS power controls used by this example accept kilowatts, so two helpers hold PredBat's requested values:

- `input_number.predbat_alphaess_charge_rate_w`
- `input_number.predbat_alphaess_export_rate_w`

The helpers are needed because PredBat's inverter interface expects writable rate entities in its own units, while the AlphaESS integration exposes separate power controls with different units and operating switches. PredBat cannot safely use those AlphaESS number entities as direct drop-in replacements.

The helpers provide a stable boundary between the two systems:

```text
PredBat requested rate (W)
    → Home Assistant helper (W)
    → bridge automation converts W to kW
    → AlphaESS power number (kW)
```

This separation also makes the requested value visible in Home Assistant, allows safe range limits to be applied, and lets the bridge resend the value when Force Charging or Force Discharging starts. Resending matters because a mode change or another AlphaESS action may have changed the inverter power control since PredBat last updated its requested rate.

Recommended UI method:

1. Open **Settings → Devices & services → Helpers**.
2. Select **Create helper → Number**.
3. Create **PredBat AlphaESS charge rate** with entity ID `input_number.predbat_alphaess_charge_rate_w`.
4. Set minimum `0`, maximum `8000`, step `100`, unit `W`, and input mode.
5. Repeat for **PredBat AlphaESS export rate** with entity ID `input_number.predbat_alphaess_export_rate_w`.

Change the maximum values if 8 kW is not safe or supported by your inverter/battery.

If you manage helpers in YAML, copy only the `input_number:` section from the bridge example into the appropriate configuration file and reload/restart Home Assistant as required.

### 2. Create the rate bridge automations

Create the two automations from the bridge example:

- **PredBat AlphaESS Charge Rate Bridge**
- **PredBat AlphaESS Discharge Rate Bridge**

The charge bridge converts watts to kilowatts and writes:

```text
input_number.predbat_alphaess_charge_rate_w
    → number.alphaess_inverter_force_charging_power
```

The discharge bridge converts watts to kilowatts and writes:

```text
input_number.predbat_alphaess_export_rate_w
    → number.alphaess_inverter_force_discharging_power
```

Each automation runs when its helper changes and again when the associated forced mode starts. This ensures the AlphaESS control receives the latest PredBat rate. The helper retains its historical `export_rate` name because it is the entity configured as PredBat's `discharge_rate`.

After creating them, manually change each helper to a conservative test value while the associated forced mode is off. Confirm the matching AlphaESS power number receives the expected kW value—for example, `1000 W → 1.0 kW`.

### 3. Verify Force Discharging control

PredBat's `discharge_rate` is a requested battery discharge rate in watts. AlphaESS Force Export instead treats its power setting as a desired grid-export rate and continually compensates for house load and PV. This example therefore uses AlphaESS Force Discharging so the battery-side power request matches PredBat's planning model.

The `{target_soc}` supplied to `discharge_start_service` is an internal control limit and can differ from the endpoint displayed in PredBat's status. The start service therefore calls `script.predbat_alphaess_export_stop_soc`. While PredBat is Exporting, the script extracts the final percentage displayed in `predbat.status` and writes it to:

```text
number.alphaess_inverter_force_discharging_stop_at_soc
```

If the live endpoint is not yet available during start-up, the script temporarily falls back to `{target_soc}`. The target-sync automation runs whenever `predbat.status` updates, replacing that fallback with the displayed endpoint and following later replans. It only writes the stop target; it never starts or restarts a forced mode.

The start service then sets a bounded duration and enables `switch.alphaess_inverter_force_discharging`. There is no percentage offset: if PredBat displays an export from 55% to 45%, AlphaESS is set to stop at 45%.

Test with a short, conservative export while watching battery power, grid power, house load and PV. Battery power should follow PredBat's requested discharge rate; grid export will be the remaining power after the house load is supplied and PV is included. Confirm the forced mode stops at the target SoC and that PredBat's stop service turns it off.

### Migrating from the earlier Force Export example

If you installed an earlier version of this repository, update both `apps.yaml` and the Home Assistant discharge-rate bridge. The bridge keeps the existing `predbat_alphaess_export_rate_bridge` automation ID and `input_number.predbat_alphaess_export_rate_w` helper ID to avoid creating duplicates, but now targets the Force Discharging controls.

Update the existing stop-SOC script and sync automation rather than creating duplicates. Their entity IDs are retained, but both now target Force Discharging. The old delay and Force Export restart action must be removed: the revised automation only keeps the Force Discharging stop target aligned with PredBat.

## Create the Modbus-derived daily house-load sensor

The `load_today` entry in `apps.yaml` points to:

```text
sensor.alphaess_inverter_predbat_experimental_modbus_house_load_today
```

This is a generic example name, not a sensor supplied automatically by PredBat or the AlphaESS integration. We created it because PredBat needs a reliable daily house-load total that increases in kWh through the day, but the native AlphaESS daily house-load candidates on this installation were unavailable. The Modbus integration did provide a working live whole-house power sensor, so the least invasive solution was to derive the missing daily energy value from that existing signal rather than alter the inverter integration or rely on unavailable cloud entities.

It is built in two stages:

```text
sensor.alphaess_inverter_current_house_load (W)
    → Integral helper, using the left method
    → sensor.alphaess_inverter_predbat_experimental_modbus_house_load_energy (kWh)
    → daily Utility Meter
    → sensor.alphaess_inverter_predbat_experimental_modbus_house_load_today (kWh)
```

The live example uses the `left` Integral method because the Modbus power sensor is a sampled value that remains at its previous reading until the next update. This treats the previous reading as applying over the elapsed interval and avoids the artificial sloping introduced by trapezoidal interpolation when the load changes abruptly. It is still an approximation: more frequent and reliable source updates produce a more accurate energy total.

The daily Utility Meter resets the accumulated value at local midnight and gives PredBat the incrementing daily kWh sensor expected by `load_today`. The live example has `periodically_resetting` and `always_available` enabled. Test those choices against your source sensor's restart and availability behaviour rather than assuming they suit every installation.

To reproduce the live setup in the Home Assistant UI:

1. Create an **Integral** helper named **PredBat Experimental Modbus House Load Energy**.
2. Select `sensor.alphaess_inverter_current_house_load` as its input.
3. Select the **Left** method, `k` metric prefix, hours as the integration time and precision `4`.
4. Create a **Utility Meter** helper named **PredBat Experimental Modbus House Load Today**.
5. Use the Integral sensor as its source, choose a daily cycle, enable periodically resetting and enable always available.

Alternatively, copy the definitions from [`home_assistant_house_load.example.yaml`](./home_assistant_house_load.example.yaml) into the appropriate Home Assistant YAML configuration. Do not create both the UI helpers and YAML versions with the same names.

After creation, confirm that the source power is in watts, the Integral sensor is in kWh, the daily sensor resets at local midnight, and its value only increases during the day. The live sensor currently exposes `device_class: energy` and `state_class: total_increasing` and has been observed incrementing normally.

## Merge the AlphaESS block into PredBat

Copy the `pred_bat:` settings from [`apps.yaml`](./apps.yaml) into your normal PredBat configuration. The file contains the reusable, anonymised settings used by the current example, including tariff comparison, LoadML, Solcast and optional standard EV/tariff inputs. It excludes the installation-specific automations and charger-threshold logic.

Review every entity and numerical value, especially:

- `soc_max`
- `battery_rate_max`
- `inverter_limit`
- `inverter_limit_charge`
- `inverter_limit_discharge`
- `inverter_limit_export`
- `battery_min_soc`
- `inverter_freeze_export_discharge_rate`
- daily energy sensors

The limits in this repository describe one example installation. They are not universal AlphaESS specifications.

## What each `apps.yaml` section does

The example is intentionally a full, anonymised PredBat configuration rather than only an inverter fragment. Remove optional sections only when you understand what consumes them.

### General PredBat settings

- `prefix` controls the Home Assistant entity prefix created by PredBat.
- `timezone` must match Home Assistant.
- `currency_symbols` controls display units.
- `threads` and `forecast_hours` control calculation concurrency and planning horizon.
- `days_previous` supplies the conventional historic-load comparison input.

### Tariff comparison

`dno_region` and `compare_list` populate PredBat's Compare page with alternative import/export tariffs. They do not replace the live tariff entities used for the active plan.

Product codes and fixed prices become stale. Check every tariff before relying on a comparison.

### LoadML and temperature

`load_ml_enable`, `load_ml_source`, `load_ml_max_days_history`, `load_ml_database_days`, and `temperature_enable` enable the machine-learning load forecast used by the current setup.

These settings depend on good load history. Fix an unreliable `load_today` sensor before tuning LoadML.

### AlphaESS capability declaration

`num_inverters`, `inverter_type`, and the nested `inverter:` mapping tell PredBat which AlphaESS capabilities are available and that control happens through Home Assistant services.

Do not change a capability flag simply because another inverter example uses a different value. The service sequences and freeze behaviour depend on these declarations.

### Control services

- `charge_start_service` and `charge_stop_service` start and stop force charging.
- `discharge_start_service` and `discharge_stop_service` start and stop Force Discharging.
- `charge_freeze_service` maps PredBat charge freeze to AlphaESS Normal Mode (5).
- `discharge_freeze_service` maps PredBat discharge freeze to AlphaESS No Battery Charge (19).

Every start sequence first releases conflicting AlphaESS modes. Every stop sequence releases Dispatch so a previous hold is not left behind.

### Live sensors and direction flags

`soc_percent`, `battery_power`, `pv_power`, `load_power`, and `grid_power` provide the current battery and household state. The `*_invert` options correct sign conventions only where proven necessary from live states.

### Daily energy

`load_today`, `import_today`, `export_today`, and `pv_today` provide cumulative daily energy. The included `load_today` entity is created by [`home_assistant_house_load.example.yaml`](./home_assistant_house_load.example.yaml); replace its source if your AlphaESS live house-load entity has a different name.

### Battery and inverter limits

The capacity, minimum SoC, maximum rates, inverter limits, scaling and clock-skew settings describe the example installation. Verify every numerical value against the installed inverter, battery, grid connection and export permission.

### Solcast forecast

The four `pv_forecast_*` entries connect PredBat to Home Assistant Solcast forecast entities for today through day four. Replace or remove them if you use a different forecast source.

### Optional EV information

`num_cars`, `car_charging_battery_size`, `car_charging_soc`, and `car_charging_limit` help PredBat estimate the energy required by one EV. The example uses anonymised BottlecapDave Octopus Energy entity placeholders.

No charger-power threshold or custom `car_charging_now` entity is included. Add one only if it is reliable and appropriate for your setup.

### Live tariff entities

`metric_octopus_import`, `metric_octopus_export`, and `metric_standing_charge` read the active rates from Home Assistant. Replace every `xxxxxx` placeholder with the correct entity from your own tariff integration.

`octopus_slot_low_rate`, `octopus_slot_max`, `combine_charge`, and `calculate_export_during_charge` affect how the example treats Intelligent Octopus slots and overlapping charge/export planning. Check these against your tariff and desired behaviour.

### Watch list

The watch list uses PredBat's `+[car_charging_soc]` list-reference syntax. PredBat reruns when the configured optional EV state-of-charge entity changes; it does not add or depend on a custom `car_charging_now` helper.

### Axle Energy VPP

The Axle example reads its API key from `!secret axle_api_key`. Add the real value to Home Assistant's `secrets.yaml`; never commit it. `axle_control: false` deliberately leaves inverter control with the existing AlphaESS Modbus service sequences during an event.

### Scaling

`import_export_scaling` applies a final scaling factor to imported/exported energy calculations. The example leaves it at `1.0`.

## Variable charge and discharge rates

The PredBat interface is:

```yaml
output_charge_control: "power"
charge_discharge_with_rate: true

charge_rate:
  - input_number.predbat_alphaess_charge_rate_w

discharge_rate:
  - input_number.predbat_alphaess_export_rate_w
```

The start services enable the required AlphaESS operating mode. They do not overwrite PredBat's requested rate with a fixed power value.

## Charge and discharge sequences

Before starting a new charge or discharge operation, the example turns off an active Dispatch hold and conflicting forced modes. This prevents an earlier freeze state from blocking the new request.

The discharge sequence calls the target script, sets a bounded Force Discharging duration, and enables Force Discharging. Subsequent PredBat status updates resynchronise the displayed endpoint.

## Freeze mappings

PredBat freeze capabilities are mapped to AlphaESS Dispatch modes. PredBat's plan uses the name **Freeze Export**, while the custom-inverter configuration uses the service name `discharge_freeze_service`.

| PredBat plan state | Service used from `apps.yaml` |
|---|---|
| Freeze Charge | `charge_freeze_service` |
| Freeze Export | `discharge_freeze_service` |
| Export | `discharge_start_service` |
| End Export or Freeze Export | `discharge_stop_service` |

### Charge freeze

Charge freeze uses:

```yaml
option: "Normal Mode (5)"
```

The sequence stops conflicting forced modes, selects Normal Mode, sets a bounded Dispatch duration and enables Dispatch. This retains normal self-consumption rather than forcing grid charging.

### Freeze Export / discharge freeze

Discharge freeze uses:

```yaml
option: "No Battery Charge (19)"
```

The sequence stops conflicting forced modes, selects No Battery Charge, sets a bounded Dispatch duration and enables Dispatch.

Verify the mode names and numbers against the options exposed by your AlphaESS integration.

### Stop services

The charge and discharge stop services also turn off Dispatch so a temporary hold does not remain active.

## Low-rate export freeze

The example contains:

```yaml
inverter_freeze_export_discharge_rate: 240
```

This is a PredBat prediction-model setting; it does not command the inverter to discharge at 240 W. During Freeze Export, this AlphaESS system was observed to leak a small, fairly steady amount of battery power instead of discharging enough to cover the full house load. The value `240` is a rounded representative average of that observed battery-side discharge on this particular installation.

It is not a universal AlphaESS value, an inverter minimum or a safety limit. Another installation should measure its sustained battery power over several Freeze Export periods, exclude short switching transients, and use a representative average only if the same fixed-leak behaviour is present. Leave the setting at PredBat's default of `0` if the inverter normally covers house load during Freeze Export or if the behaviour has not been established. Recheck the value after inverter firmware, operating-mode or control changes.

## Power and energy sensors

The example expects:

- `soc_percent`: battery percentage
- `battery_power`: live battery power
- `pv_power`: live PV production
- `load_power`: live house load
- `grid_power`: live grid import/export
- `load_today`, `import_today`, `export_today`, `pv_today`: cumulative daily energy

Set the `*_invert` flags only after observing the live direction conventions. Use the included two-stage house-load example only when its source entity and behaviour match your system.

## Commissioning

1. Back up the existing PredBat and Home Assistant configuration.
2. Create the house-load Integral and daily Utility Meter sensors if your integration does not already provide a reliable `load_today` entity.
3. Create the two rate helpers, two rate bridges, discharge-target script and target-sync automation.
4. Reload or restart Home Assistant and confirm the new entities load.
5. Test the house-load sensor chain and helper-to-AlphaESS conversions with conservative values.
6. Add the customised AlphaESS block with PredBat in Monitor/read-only mode.
7. Check all live power directions and daily totals.
8. Check battery capacity, minimum SoC and every power limit.
9. Verify charge freeze and discharge freeze start and clear correctly.
10. Verify a new charge/discharge operation clears a previous Dispatch hold.
11. During a short controlled export, confirm battery power follows PredBat's requested discharge rate and AlphaESS receives the final percentage displayed by PredBat.
12. Confirm the discharge stop service turns Force Discharging off.
13. Review PredBat and Home Assistant logs.
14. Enable writes only after every check passes.

## Troubleshooting

### Rate helper changes do not reach AlphaESS

Confirm the relevant bridge automation is enabled, the helper is in watts, and the AlphaESS power entity accepts kilowatts.

### Freeze starts but does not clear

Check that the relevant stop service turns off `switch.alphaess_inverter_dispatch`.

### Charge or export does not start

Check for a remaining Dispatch or conflicting forced mode. The start sequence should turn those modes off first.

### Export stops at the wrong SoC

Check that the target-sync automation is enabled, `predbat.status` is `Exporting`, and its `detail` attribute contains a percentage. Confirm the script writes the final displayed percentage to `number.alphaess_inverter_force_discharging_stop_at_soc` and that the inverter is using Force Discharging rather than Force Export.

### Power direction is wrong

Check `battery_power_invert`, `grid_power_invert`, and `load_power_invert` against live Home Assistant states.

## Credits

This example brings together work from the following open-source projects:

- [PredBat](https://github.com/springfall2008/batpred) by springfall2008 — battery planning, optimisation and inverter-control framework.
- [ha-alphaess-modbus](https://github.com/senalse/ha-alphaess-modbus) by senalse — the Home Assistant AlphaESS Modbus integration that provides the inverter sensors and controls used by this example.
- [Home Assistant Octopus Energy](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy) by BottlecapDave — the Home Assistant tariff and Intelligent Octopus entities referenced by the optional Octopus configuration.

Thank you to their maintainers and contributors. This repository is an independent example configuration and is not affiliated with or endorsed by those projects.

## Privacy

Before sharing a derived configuration, remove API keys, meter/account identifiers, vehicle UUIDs, webhook URLs, notification targets, private hostnames and installation-specific automations.

## Safety

Battery and inverter control can move significant energy and may affect warranties, export limits and electrical safety.

Keep the system in read-only mode until the complete control path has been tested. Use conservative values, retain backups, and make only one controlled change at a time.
