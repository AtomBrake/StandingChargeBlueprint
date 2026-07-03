# Standing Charge Blueprint for Home Assistant

UK (and other) energy tariffs charge a fixed **daily standing charge** on top
of per-kWh usage. The Home Assistant Energy Dashboard tracks usage costs
brilliantly, but has no built-in way to add a daily standing charge, so the
totals it shows are always short of your real bill.

This project adds the missing piece: a small blueprint automation that
accrues the standing charge once a day into a running total, which you then
show on the Energy Dashboard as a separate cost-only source (electricity and
gas are set up independently, and each of you can even have different
standing charges e.g. if you're on a variable tariff).

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FAtomBrake%2FStandingChargeBlueprint%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fstanding_charge%2Fstanding_charge_accrual.yaml)

## How it works

Home Assistant's Energy Dashboard lets you add a source of consumption and
tell it to work out cost either from a fixed price, a price-per-kWh entity,
or **an entity that already tracks the total cost**. That last option is the
trick this project relies on: we create a source with **zero usage** whose
cost entity is a running total that increases by the standing charge every
day. The dashboard then adds that cost into your totals, cost breakdown
chart, and any statistics, right alongside your normal usage costs.

Because it's a real source in the dashboard, it appears as its own line item
(e.g. "Electricity Standing Charge") so you can still see usage cost and
standing charge cost separately if you want to.

## Requirements

- An existing Energy Dashboard already configured with electricity and/or
  gas usage and cost.
- Willingness to create a handful of Helpers (a few clicks each, no YAML
  required unless you prefer it).

## Setup

Repeat this whole process once per fuel (electricity, gas).

### 1. Create two Number helpers

Settings → Devices & Services → Helpers → **+ Create Helper** → Number.

| Helper | Purpose | Suggested settings |
|---|---|---|
| `input_number.electricity_standing_charge_rate` | Today's daily standing charge, in your currency (e.g. `0.6123` for 61.23p) | Min `0`, Max `10`, Step `0.0001` |
| `input_number.electricity_standing_charge_total` | Running total the blueprint increments | Min `0`, Max `100000`, Step `0.0001` |

Set the rate helper to your current standing charge now. You'll need to
update it by hand whenever your tariff changes (or point the blueprint at a
sensor from your supplier's integration instead — see [Advanced](#advanced)).

Repeat with `gas_` names for gas.

### 2. Create two Template sensors

Settings → Devices & Services → Helpers → **+ Create Helper** → Template →
Template a sensor. Create both of these (values shown are for electricity —
repeat for gas):

**Standing charge cost sensor** — mirrors the running total, tagged so the
Energy Dashboard treats it as money:
- Name: `Electricity Standing Charge Total`
- State template: `{{ states('input_number.electricity_standing_charge_total') | float(0) }}`
- Unit of measurement: `GBP` (or your currency)
- Device class: `Monetary`
- State class: `Total`

**Dummy zero-usage sensor** — the Energy Dashboard requires a usage entity
to attach a cost to, so this always reports 0:
- Name: `Electricity Standing Charge Usage`
- State template: `0`
- Unit of measurement: `kWh`
- Device class: `Energy`
- State class: `Total`

(For gas, use `m³` and device class `Gas` for the dummy sensor, or `kWh` /
`Energy` if your real gas source is already in kWh.)

Prefer YAML? See [`examples/template_helpers.yaml`](examples/template_helpers.yaml)
for both sensors, both fuels, ready to drop into `configuration.yaml`.

### 3. Import and configure the blueprint

Click the import badge above, or in Home Assistant go to Settings →
Automations & Scenes → Blueprints → Import Blueprint, and paste in:

```
https://github.com/AtomBrake/StandingChargeBlueprint/blob/main/blueprints/automation/standing_charge/standing_charge_accrual.yaml
```

Then create an automation from the blueprint (once for electricity, once
for gas):

- **Standing Charge Rate Entity**: `input_number.electricity_standing_charge_rate`
- **Cumulative Total Entity**: `input_number.electricity_standing_charge_total`
- **Accrual Time**: `00:00:00` (default — see the blueprint description for
  why you might change this)

### 4. Add the source to the Energy Dashboard

Settings → Dashboards → Energy → Electricity grid → **Add Consumption**:

1. For "consumed energy", pick `sensor.electricity_standing_charge_usage`
   (the dummy 0 kWh sensor).
2. Untick "Use a static price" / "Use an entity with the current price".
3. Enable **"Use an entity tracking the total costs"** and select
   `sensor.electricity_standing_charge_total`.
4. Save.

Repeat under "Gas consumption" for the gas sensors.

Your Energy Dashboard totals, cost breakdown, and statistics will now
include both usage and standing charge.

## Notes and caveats

- The accrual automation only runs at the configured time — if Home
  Assistant is offline at that exact moment, that day's charge is skipped
  (there's no catch-up logic). Pick a time you're confident HA will be up,
  or check the running total periodically.
- The currency you set on the template sensor should match your Home
  Assistant instance's configured currency (Settings → General).
- If you're on a variable/tracker tariff where the standing charge changes
  daily, update the rate `input_number` (or point it at your supplier
  integration's sensor) before the accrual time each day.

## Advanced

If your energy supplier's Home Assistant integration (e.g. Octopus Energy)
already exposes a sensor for today's standing charge, you can point the
blueprint's **Standing Charge Rate Entity** directly at that sensor instead
of a manually maintained `input_number` — skip creating the `_rate` helper
and use the supplier's sensor entity ID instead.

## License

MIT — see [LICENSE](LICENSE).
