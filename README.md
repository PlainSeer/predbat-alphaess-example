PredBat AlphaESS Modbus Example Configuration

Disclaimer: This repository contains an example PredBat configuration for an AlphaESS system using the Home Assistant Modbus integration. It is intended as a reference only and is not a drop-in configuration.

Purpose

This repository is provided to help other AlphaESS users understand how PredBat can be configured when using the Home Assistant Modbus integration rather than the AlphaESS cloud integration.

Every Home Assistant installation is different, so entity names, hardware capabilities and integrations will vary. You should expect to modify this configuration to suit your own installation.

Important

This configuration has been anonymised before being published. Any entity names containing:

* your_inverter
* your_ev_charger
* your_octopus
* your_serial

are placeholders and must be replaced with the correct entities from your own Home Assistant installation.

Why load_today uses a custom sensor

The sample configuration contains:

load_today:
  - sensor.alphaess_your_serial_total_load

This is not a standard AlphaESS entity.

PredBat requires an accurate daily house consumption (load_today) sensor. At the time this example was created, the AlphaESS Modbus integration did not provide a native daily house load energy sensor, so a template/integration sensor was created within Home Assistant to calculate the home’s daily energy consumption.

The published configuration therefore references that custom sensor simply to demonstrate where load_today should point.

If your installation already has a suitable daily house load sensor, you should use that instead. Otherwise, you will need to create an equivalent sensor appropriate for your own system.

What you’ll need to change

At a minimum you should review and update:

* All Home Assistant entity IDs.
* Battery capacity (soc_max).
* Charge and discharge power limits.
* Minimum SoC.
* Inverter capabilities.
* EV charger entities (if applicable).
* Octopus Energy entities (if applicable).
* Any custom sensors.

Notes

This configuration reflects one particular installation and one user’s preferred operating strategy. It is not intended to be the “correct” or “best” way to configure PredBat.

There are many ways to achieve the same result, and your hardware, tariff, battery size and personal preferences may require a different approach.

Suggestions and improvements are always welcome.

Disclaimer

This configuration is provided as-is, without any warranty or guarantee.

If you choose to use any part of this example, you do so entirely at your own risk.

I cannot accept responsibility for:

* incorrect operation
* battery behaviour
* inverter behaviour
* unexpected charging or exporting
* increased electricity costs
* equipment damage
* data loss
* or any other consequences arising from the use or modification of this example.

Always test any changes carefully before relying on them in a live environment.
