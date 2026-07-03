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

### 1. Create the helpers

Settings → Devices & Services → Helpers → **+ Create Helper**.

Two Number helpers:

| Helper | Purpose | Suggested settings |
|---|---|---|
| `input_number.electricity_standing_charge_rate` | Today's daily standing charge, in your currency (e.g. `0.6123` for 61.23p) | Min `0`, Max `10`, Step `0.0001` |
| `input_number.electricity_standing_charge_total` | Running total the blueprint increments | Min `0`, Max `100000`, Step `0.0001` |

And one Date and/or time helper (type: **Date**, no time component):

| Helper | Purpose |
|---|---|
| `input_datetime.electricity_standing_charge_last_accrual` | Last date the blueprint successfully ran — used only for catch-up after Home Assistant has been offline. Leave it at its default value. |

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
- **Last Accrual Date Entity**: `input_datetime.electricity_standing_charge_last_accrual`
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

- As well as running at the configured time each day, the automation also
  runs once when Home Assistant starts up. On either trigger it checks how
  many days have passed since the last successful accrual and backfills
  all of them at once — so a restart, update, or outage spanning the
  accrual time no longer loses that day's charge. A backfilled gap uses
  the *current* rate for every missed day, which may be slightly off if
  your rate changed partway through the outage.
- The currency you set on the template sensor should match your Home
  Assistant instance's configured currency (Settings → General).
- If you're on a variable/tracker tariff where the standing charge changes
  daily, update the rate `input_number` (or point it at your supplier
  integration's sensor) before the accrual time each day.

## Advanced: using a supplier integration for the rate

The blueprint's **Standing Charge Rate Entity** input accepts any
`input_number`, `number`, or `sensor` entity — it just reads that entity's
state as a number, so it has no idea which supplier (if any) it came from.
If your supplier's integration already exposes the standing charge as its
own sensor state, just point the input at it directly and skip creating
the `_rate` helper.

Some integrations, however, only expose the standing charge as an
**attribute** on a cost sensor, not as its own entity — the blueprint can't
read attributes directly, only entity states. In that case, add a one-line
template sensor to pull the attribute out into its own state, and point
the blueprint at that instead. For example, the
[Octopus Energy integration](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy)
reports the standing charge as a `standing_charge` attribute on its
`..._current_accumulative_cost` / `..._previous_accumulative_cost` sensors
(find your exact entity id under Developer Tools → States, filtering for
`accumulative_cost` — it includes your meter's MPAN/serial number):

```yaml
template:
  - sensor:
      - name: "Electricity Standing Charge Rate"
        unique_id: electricity_standing_charge_rate
        unit_of_measurement: "GBP"
        device_class: monetary
        state: >-
          {{ state_attr('sensor.octopus_energy_electricity_YOUR_SERIAL_YOUR_MPAN_current_accumulative_cost', 'standing_charge') | float(0) }}
```

Use `previous_accumulative_cost` instead if you don't have a Home
Mini/Pro (that sensor only updates for the *previous* day, so the rate
will lag by a day — fine in practice since standing charges rarely change
daily, but worth knowing). Point the blueprint's rate entity at this
template sensor instead of an `input_number`, and you no longer need to
maintain the rate by hand.

This same attribute-extraction pattern works for any supplier integration
that buries the rate in an attribute rather than a dedicated entity — the
blueprint itself never needs to know or care.

## License

MIT — see [LICENSE](LICENSE).
