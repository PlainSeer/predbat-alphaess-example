# PredBat + AlphaESS Modbus example

This repository is a reusable starting point for connecting PredBat to an AlphaESS inverter through Home Assistant.

It is an example, not a drop-in configuration. Replace every placeholder, verify every entity and limit, and commission in read-only mode before allowing inverter writes.

## What is included

- [`apps.yaml`](./apps.yaml): the reusable PredBat/AlphaESS inverter block
- [`home_assistant_alphaess_bridge.example.yaml`](./home_assistant_alphaess_bridge.example.yaml): the two rate helpers, charge/export rate bridges, export-target script and export-target resync automation

The repository deliberately excludes installation-specific EV, tariff, household-control and notification automations.

## Requirements

- Home Assistant with a working AlphaESS Modbus integration
- PredBat
- AlphaESS entities for battery SoC, battery/PV/load/grid power and daily energy
- AlphaESS controls for force charging, force export and Dispatch
- the Home Assistant helpers and bridge automations described below

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
- force-charge, force-export and Dispatch controls behave as expected.

Do not enable PredBat inverter writes until these checks pass.

## Install the Home Assistant bridge

The complete example is in [`home_assistant_alphaess_bridge.example.yaml`](./home_assistant_alphaess_bridge.example.yaml). It contains definitions intended for different Home Assistant configuration areas; do not include the entire file blindly as one package unless you have converted it to your package structure.

### 1. Create the two rate helpers

PredBat expresses charge and discharge rates in watts. The AlphaESS power controls used by this example accept kilowatts, so two helpers hold PredBat's requested values:

- `input_number.predbat_alphaess_charge_rate_w`
- `input_number.predbat_alphaess_export_rate_w`

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
- **PredBat AlphaESS Export Rate Bridge**

The charge bridge converts watts to kilowatts and writes:

```text
input_number.predbat_alphaess_charge_rate_w
    → number.alphaess_inverter_force_charging_power
```

The export bridge converts watts to kilowatts and writes:

```text
input_number.predbat_alphaess_export_rate_w
    → number.alphaess_inverter_force_export_power
```

Each automation runs when its helper changes and again when the associated forced mode starts. This ensures the AlphaESS control receives the latest PredBat rate.

After creating them, manually change each helper to a conservative test value while the associated forced mode is off. Confirm the matching AlphaESS power number receives the expected kW value—for example, `1000 W → 1.0 kW`.

### 3. Create the export-target script

Create `script.predbat_alphaess_export_stop_soc` from the bridge example.

While PredBat is exporting, the script reads the percentages shown in the `detail` attribute of `predbat.status` and uses the last percentage as the current export end target. If no live Exporting target can be parsed, it falls back to the `target_soc` passed by PredBat.

It writes the result to:

```text
number.alphaess_inverter_force_export_stop_at_soc
```

### 4. Create the export-target resync automation

Create **PredBat AlphaESS Export SOC Sync** from the bridge example.

When `predbat.status` changes while its state is `Exporting`, the automation reruns the export-target script. This keeps the AlphaESS native stop target aligned after a PredBat replan changes the displayed export endpoint.


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
- `discharge_start_service` and `discharge_stop_service` start and stop force export.
- `charge_freeze_service` maps PredBat charge freeze to AlphaESS Normal Mode (5).
- `discharge_freeze_service` maps PredBat discharge freeze to AlphaESS No Battery Charge (19).

Every start sequence first releases conflicting AlphaESS modes. Every stop sequence releases Dispatch so a previous hold is not left behind.

### Live sensors and direction flags

`soc_percent`, `battery_power`, `pv_power`, `load_power`, and `grid_power` provide the current battery and household state. The `*_invert` options correct sign conventions only where proven necessary from live states.

### Daily energy

`load_today`, `import_today`, `export_today`, and `pv_today` provide cumulative daily energy. Replace `sensor.xxxxxx_total_load` with a reliable local entity.

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

### Scaling

`import_export_scaling` applies a final scaling factor to imported/exported energy calculations. The example leaves it at `1.0`.

## Variable charge and export rates

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

## Charge and export sequences

Before starting a new charge or export operation, the example turns off an active Dispatch hold and conflicting forced modes. This prevents an earlier freeze state from blocking the new request.

The export sequence calls `script.predbat_alphaess_export_stop_soc`, sets a bounded force-export duration, and enables force export.

## Freeze mappings

PredBat freeze capabilities are mapped to AlphaESS Dispatch modes.

### Charge freeze

Charge freeze uses:

```yaml
option: "Normal Mode (5)"
```

The sequence stops conflicting forced modes, selects Normal Mode, sets a bounded Dispatch duration and enables Dispatch. This retains normal self-consumption rather than forcing grid charging.

### Discharge freeze

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

Verify this value against the inverter's supported range and the behaviour you want.

## Power and energy sensors

The example expects:

- `soc_percent`: battery percentage
- `battery_power`: live battery power
- `pv_power`: live PV production
- `load_power`: live house load
- `grid_power`: live grid import/export
- `load_today`, `import_today`, `export_today`, `pv_today`: cumulative daily energy

Set the `*_invert` flags only after observing the live direction conventions. Replace the `sensor.xxxxxx_total_load` placeholder with a reliable cumulative daily load sensor.

## Commissioning

1. Back up the existing PredBat and Home Assistant configuration.
2. Create the two helpers, two rate bridges, export-target script and resync automation.
3. Reload or restart Home Assistant and confirm all five entities load.
4. Test the helper-to-AlphaESS conversions with conservative values.
5. Add the customised AlphaESS block with PredBat in Monitor/read-only mode.
6. Check all live power directions and daily totals.
7. Check battery capacity, minimum SoC and every power limit.
8. Verify charge freeze and discharge freeze start and clear correctly.
9. Verify a new charge/export operation clears a previous Dispatch hold.
10. During a controlled export, confirm the AlphaESS stop SoC tracks the final percentage in `predbat.status` detail, including after replans.
11. Review PredBat and Home Assistant logs.
12. Enable writes only after every check passes.

## Troubleshooting

### Rate helper changes do not reach AlphaESS

Confirm the relevant bridge automation is enabled, the helper is in watts, and the AlphaESS power entity accepts kilowatts.

### Freeze starts but does not clear

Check that the relevant stop service turns off `switch.alphaess_inverter_dispatch`.

### Charge or export does not start

Check for a remaining Dispatch or conflicting forced mode. The start sequence should turn those modes off first.

### Export stop SoC does not follow a replan

Check:

- `predbat.status` is `Exporting`;
- its `detail` attribute contains one or more percentages;
- the resync automation is enabled;
- the script can write `number.alphaess_inverter_force_export_stop_at_soc`.

### Power direction is wrong

Check `battery_power_invert`, `grid_power_invert`, and `load_power_invert` against live Home Assistant states.

## Privacy

Before sharing a derived configuration, remove API keys, meter/account identifiers, vehicle UUIDs, webhook URLs, notification targets, private hostnames and installation-specific automations.

## Safety

Battery and inverter control can move significant energy and may affect warranties, export limits and electrical safety.

Keep the system in read-only mode until the complete control path has been tested. Use conservative values, retain backups, and make only one controlled change at a time.
