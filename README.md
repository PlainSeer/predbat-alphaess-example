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
- [`home_assistant_alphaess_bridge.example.yaml`](./home_assistant_alphaess_bridge.example.yaml): the two rate helpers, charge bridge, load-aware discharge bridge, discharge-target script and target-sync automation
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
- the charge and load-aware discharge rate bridge automations;
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
    → bridge automation converts/adjusts the request
    → AlphaESS power number (kW)
```

For charging, the bridge is a straight W-to-kW conversion. During an active export, the discharge bridge also watches live house load and PV so it can prevent the fixed battery-discharge request from causing avoidable grid import when household demand rises.

This separation also makes the requested value visible in Home Assistant and allows safe range limits to be applied. The discharge helper retains its historical `export_rate` name because it is the entity configured as PredBat's `discharge_rate`.

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

The discharge bridge is deliberately load-aware. While `predbat.status` is `Exporting` and `switch.alphaess_inverter_force_discharging` is already on, it calculates:

```text
target discharge W = max(
    PredBat requested discharge W,
    house load W - PV W,
    0
)
```

The result is capped at `8000 W`, converted to kW and rounded to `0.1 kW` before writing:

```text
number.alphaess_inverter_force_discharging_power
```

The practical reason is important. If PredBat requests a 2.4 kW battery discharge and the house suddenly consumes 5–6 kW, a fixed 2.4 kW Force Discharging command can make the remaining house demand come from the grid even during an export period. The load-aware floor raises battery discharge only enough to cover the current house-load shortfall after PV, subject to the configured 8 kW cap. It does not try to preserve the same grid-export rate while a large house load is present.

The discharge bridge triggers when the requested rate, PredBat status, Force Discharging switch, live house load or live PV production changes. A `0.2 kW` deadband avoids repeated small writes. `mode: restart` means a newer trigger supersedes an older run.

There are also deliberate stale-write safeguards. A non-zero write is permitted only while both conditions are still true:

- `predbat.status` is `Exporting`;
- `switch.alphaess_inverter_force_discharging` is `on`.

The automation rechecks those guards immediately before the write and again inside the value template. If PredBat ends the export or AlphaESS Force Discharging turns off, the bridge clears any non-zero discharge-power setting to `0` and **never** turns Force Discharging back on. PredBat remains the owner of the Force Discharging switch, duration and stop-SOC target.

This prevents the race condition where a house-load/PV trigger could otherwise restart the automation just after PredBat stopped exporting and write the previous discharge power back into the inverter control.

Do not test the discharge bridge by changing the helper while Force Discharging is off and expecting a non-zero inverter write; the guard intentionally blocks that. Use a short, controlled PredBat export and watch the helper, house load, PV, Force Discharging switch and AlphaESS discharge-power number together.

### 3. Verify Force Discharging control

PredBat's `discharge_rate` is a requested battery discharge rate in watts. AlphaESS Force Export instead treats its power setting as a desired grid-export rate and continually compensates for house load and PV. This example therefore uses AlphaESS Force Discharging, with the Home Assistant bridge adding only a house-load floor during an active PredBat export.

The `{target_soc}` supplied to `discharge_start_service` is an internal control limit and can differ from the endpoint displayed in PredBat's status. The start service therefore calls `script.predbat_alphaess_export_stop_soc`. While PredBat is Exporting, the script extracts the final percentage displayed in `predbat.status` and writes it to:

```text
number.alphaess_inverter_force_discharging_stop_at_soc
```

If the live endpoint is not yet available during start-up, the script temporarily falls back to `{target_soc}`. The target-sync automation runs whenever `predbat.status` updates, replacing that fallback with the displayed endpoint and following later replans. It only writes the stop target; it never starts or restarts a forced mode.

The start service then sets a bounded duration and enables `switch.alphaess_inverter_force_discharging`. There is no percentage offset: if PredBat displays an export from 55% to 45%, AlphaESS is set to stop at 45%.

Test with a short, conservative export while watching battery power, grid power, house load and PV. With low house load, battery power should be close to PredBat's requested discharge rate. If house load rises above that request after allowing for PV, the bridge should increase battery discharge up to the configured cap so the house does not unnecessarily import from grid. Confirm the forced mode stops at the target SoC and that PredBat's stop service turns it off. After the export ends, confirm the discharge-power number returns to `0` and the bridge does not re-enable Force Discharging.

### DNO and grid-export limits

Force Discharging controls battery-side power; it does **not** dynamically hold grid export at a DNO limit. Approximate grid export is battery discharge plus PV generation, minus house load. If the permitted export limit is below the possible combined battery and solar output, Force Discharging can therefore exceed that limit.

PredBat's `export_limit` setting models an inverter's export ceiling when producing the plan, but it does not enforce that ceiling at the inverter. `inverter_limit_export` can cap the requested battery discharge rate, but it does not dynamically subtract changing PV generation or house load.

The load-aware bridge may raise battery discharge above PredBat's requested value when house demand exceeds that request after PV. That extra amount is intended to serve house load rather than increase export, but sensor latency and rapidly changing loads mean it is not an export-limit controller. The 8 kW cap in the example is also only an example installation limit, not a DNO export cap.

For an installation with a restrictive DNO limit, use a separately verified export-limiting control, choose a conservative discharge cap that remains safe under expected PV conditions, or retain AlphaESS Force Export if its grid-side regulation is required. Keep any inverter-native or installer-configured export limitation enabled. Do not rely on PredBat planning, voltage protection or this example alone to enforce a statutory grid-export limit.

### Migrating from the earlier Force Export example

If you installed an earlier version of this repository, update both `apps.yaml` and the Home Assistant discharge-rate bridge. The bridge keeps the existing `predbat_alphaess_export_rate_bridge` automation ID and `input_number.predbat_alphaess_export_rate_w` helper ID to avoid creating duplicates, but now targets the Force Discharging controls.

Update the existing stop-SOC script and sync automation rather than creating duplicates. Their entity IDs are retained, but both now target Force Discharging. The old delay and Force Export restart action must be removed: the revised automation only keeps the Force Discharging stop target aligned with PredBat.

If you already migrated to Force Discharging before the load-aware bridge was added, update the existing discharge bridge rather than creating a second automation. The revised bridge adds house-load and PV triggers, the `max(requested, load - PV)` calculation, the 8 kW cap, 0.2 kW deadband and session guards. No `apps.yaml` service change is required for this bridge update.

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
- `charge_freeze_service` maps Freeze Charge and standalone Hold for Car to AlphaESS Battery only Charges from PV (1), with Dispatch power explicitly set to `-8.0` kW. This retains the PV-permitting hold mapping.
- `discharge_freeze_service` maps PredBat Freeze Export to AlphaESS State of Charge Control (2), with Dispatch power explicitly set to `"0"` kW for the full-freeze command.

Every start sequence first releases conflicting AlphaESS modes. The AlphaESS Dispatch power register is retained and shared between Dispatch modes, so both freeze services explicitly write the value their selected mode requires. Every stop sequence releases Dispatch so a previous hold is not left behind.

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

The start services enable the required AlphaESS operating mode. They do not overwrite PredBat's requested rate with a fixed power value. During an active export, the Home Assistant discharge bridge may raise the instantaneous AlphaESS discharge-power setting above the PredBat helper value only when live house load minus PV is higher, so the battery does not remain artificially capped while the house imports the shortfall.

## Charge and discharge sequences

Before starting a new charge or discharge operation, the example turns off an active Dispatch hold and conflicting forced modes. This prevents an earlier freeze state from blocking the new request.

The discharge sequence calls the target script, sets a bounded Force Discharging duration, and enables Force Discharging. Subsequent PredBat status updates resynchronise the displayed endpoint. The separate discharge-rate bridge modulates power only while that PredBat-owned session remains active.

## Freeze and Hold for Car mappings

PredBat's plan names differ from the custom-inverter service names. This example uses PredBat's existing service hooks and requires no source-code changes.

| PredBat state | Control path | AlphaESS command |
|---|---|---|
| Freeze Export | `discharge_freeze_service` | State of Charge Control (2), 0 kW: full battery freeze command |
| Freeze Charge | `charge_freeze_service` | Battery only Charges from PV (1), −8 kW: PV-permitting hold |
| Standalone Hold for Car | `charge_freeze_service` | Same PV-permitting hold as Freeze Charge |
| Charging, Hold for Car | Normal charging path | Planned battery charging continues |
| Export | `discharge_start_service` | Force Discharging with the existing rate bridge and stop-SoC script |
| End Export or Freeze Export | `discharge_stop_service` | Release forced discharge and Dispatch |

**Full battery freeze is mapped to Freeze Export only.** Freeze Charge and standalone Hold for Car share the other hook in the inspected PredBat v9.5.3 control path. Hold for Car itself does not request grid charging; the normal charging plan determines that separately.

### Full battery freeze: Freeze Export

The relevant entries in `discharge_freeze_service` are:

```yaml
- service: select.select_option
  entity_id: select.alphaess_inverter_dispatch_mode
  option: "State of Charge Control (2)"

- service: number.set_value
  entity_id: number.alphaess_inverter_dispatch_power
  value: "0"
```

The full sequence stops conflicting forced modes and any existing Dispatch, selects Mode 2, explicitly writes zero power, sets a bounded Dispatch duration and enables Dispatch. The intended behaviour is no battery charging or discharging: PV can serve the house and surplus PV exports, subject to the system's export limits.

Keep the zero quoted. In the inspected [PredBat v9.5.3 service-template code](https://github.com/springfall2008/batpred/blob/v9.5.3/apps/predbat/inverter.py), a numeric `0` is omitted by a truth-value check; the string `"0"` is retained and passed to Home Assistant. Omitting the write could leave the shared Dispatch power setting at the previous hold's −8 kW.

A controlled test on the example system confirmed Mode 2 with zero active dispatch power and measured battery power reaching 0 W after activation. PV was below house demand during that recorded test, so it establishes stopped discharge, not an independent surplus-PV charging/export test. Verify both directions on your inverter and firmware during commissioning.

### PV-permitting hold: Freeze Charge / Hold for Car

The existing `charge_freeze_service` keeps:

```yaml
- service: select.select_option
  entity_id: select.alphaess_inverter_dispatch_mode
  option: "Battery only Charges from PV (1)"

- service: number.set_value
  entity_id: number.alphaess_inverter_dispatch_power
  value: -8.0
```

The negative Dispatch power retains the example's allowance for PV battery charging; it does not request grid charging. The 8 kW magnitude is installation-specific and must be checked against your equipment.

This is distinct from full freeze. Verify that the hold prevents unwanted battery support of the EV on your system: residual discharge was observed with Mode 1 at −8 kW in the recorded tests, so the mode name alone is not proof of zero discharge. During **Charging, Hold for Car**, PredBat retains the normal battery-charging path instead of substituting this standalone hold.

### Releasing a hold

Both stop services turn off Dispatch. Normal charge and export start sequences also release the earlier Dispatch hold before enabling their forced mode. Retain these release steps so Demand/ECO and later planned operations can resume.

Verify the mode names and numbers against the options exposed by your installed AlphaESS integration. The Dispatch duration is a time bound, not a substitute for the stop services.

## Freeze prediction and the former 240 W override

The sample leaves the earlier installation-specific override disabled:

```yaml
# inverter_freeze_export_discharge_rate: 240
```

That setting models a fixed battery discharge during Freeze Export; it does not command inverter power. The former 240 W assumption is not appropriate to the zero-power freeze command and should not be copied as an AlphaESS default.

There is a separate prediction limitation: in the inspected [PredBat v9.5.3 prediction source](https://github.com/springfall2008/batpred/blob/v9.5.3/apps/predbat/prediction.py), leaving this setting unset (default `0`) restores the normal battery-to-house discharge calculation during Freeze Export. It does **not** make the prediction model assume zero battery discharge. The configured hardware command and projected SoC/grid flows can therefore differ. Commenting out `240` removes that fixed-discharge assumption; it does not align the prediction with full hardware freeze. This example changes service configuration only.

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
5. Test the house-load sensor chain and charge helper conversion with conservative values.
6. Add the customised AlphaESS block with PredBat in Monitor/read-only mode.
7. Check all live power directions and daily totals.
8. Check battery capacity, minimum SoC and every power limit.
9. Verify Freeze Export sends Mode 2 with zero Dispatch power and stops battery discharge; with surplus PV available, also verify no battery charging and the expected grid export. Separately test the Mode 1 PV-permitting hold, standalone Hold for Car and Charging, Hold for Car.
10. Verify a new charge/discharge operation clears a previous Dispatch hold, grid/PV charging and deliberate discharge/export still work, and returning to Demand/ECO releases the hold. Compare actual SoC and grid flows with the plan in light of the prediction limitation above.
11. During a short controlled export, confirm low house load follows the PredBat requested discharge rate, then introduce a modest additional house load and confirm the bridge raises discharge toward `house load - PV` without exceeding the configured cap.
12. End the PredBat export and confirm Force Discharging turns off, discharge power clears to `0`, and later house-load/PV changes do not re-enable or restore the old discharge command.
13. Confirm AlphaESS receives the final percentage displayed by PredBat and stops at the target SoC.
14. Review PredBat and Home Assistant logs/traces.
15. Enable writes only after every check passes.

## Troubleshooting

### Rate helper changes do not reach AlphaESS

For charging, confirm the charge bridge automation is enabled, the helper is in watts, and the AlphaESS power entity accepts kilowatts.

For discharging, a non-zero helper value is intentionally ignored unless `predbat.status` is `Exporting` and `switch.alphaess_inverter_force_discharging` is `on`. Check those session guards first.

### Grid imports during an active PredBat export

Check that `sensor.alphaess_inverter_current_house_load` and `sensor.alphaess_inverter_current_pv_production` are updating with the expected units and directions, and that both are included as triggers in the discharge bridge. Compare the AlphaESS discharge-power number with `max(PredBat requested W, house load W - PV W)`, remembering the configured 8 kW cap and 0.1 kW rounding.

A brief import spike can still occur because load/PV sensors and inverter controls are not instantaneous. The bridge is intended to remove the persistent shortfall caused by a fixed discharge request, not to act as a sub-second grid-following controller.

### Discharge power comes back after PredBat stops exporting

Confirm the bridge is the revised guarded version. It must use `mode: restart`, require both `predbat.status == Exporting` and Force Discharging `on` before a non-zero write, recheck both conditions immediately before writing, and clear the power number to `0` when the session ends. The bridge must never turn Force Discharging on itself.

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