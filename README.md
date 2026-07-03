# Standing Charge Blueprint for Home Assistant

UK (and other) energy tariffs charge a fixed **daily standing charge** on top
of per-kWh usage. The Home Assistant Energy Dashboard tracks usage costs
brilliantly, but has no built-in way to add a daily standing charge, so the
totals it shows are always short of your real bill.

This project adds the missing piece: a small blueprint automation that
accrues the standing charge once a day into a running total, which you then
show on the Energy Dashboard as a separate cost-only source. One instance
of the blueprint covers a dual-fuel household — electricity and gas each
get their own rate and running total, so they can differ (e.g. different
tariffs or renewal dates) even though a single automation handles both. If
you're electricity-only, just leave the gas fields blank.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FAtomBrake%2FHA-Standing-Charge-Blueprint%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fstanding_charge%2Fstanding_charge_accrual.yaml)

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

- Home Assistant 2024.6 or later (needed for the blueprint's collapsible
  Electricity/Gas input sections).
- An existing Energy Dashboard already configured with electricity and/or
  gas usage and cost.
- Willingness to create a handful of Helpers per fuel (a few clicks each,
  no YAML required unless you prefer it).

## Setup

Steps 1, 2 and 4 below are per fuel — do them once for electricity, then
again for gas if you have a gas supply. Step 3 (importing the blueprint)
only happens **once**: a single automation instance handles electricity
and gas together.

### 1. Create the helpers

Prefer YAML over clicking through the Helpers UI? Steps 1 and 2 (all the
helpers below, both fuels) are together in
[`examples/helpers.yaml`](examples/helpers.yaml), split into a REQUIRED
section (needed regardless) and an OPTIONAL section (only needed if you
don't already have a supplier-exposed rate entity — see below). Ready to
merge into `configuration.yaml` — the `input_number:`/`input_datetime:`
keys need a restart to take effect, the `template:` sensors just need
Developer Tools → YAML → Reload Template Entities.

Settings → Devices & Services → Helpers → **+ Create Helper**.

You need two of these regardless of where your rate comes from — a Number
helper for the running total, and a Date helper for catch-up bookkeeping.
**The Number helper here is not the same entity as the template sensor
you'll create in step 2** — you need both; the blueprint writes to this
one, and the template sensor in step 2 just mirrors it for the Energy
Dashboard:

| Helper | Purpose | Suggested settings |
|---|---|---|
| `input_number.electricity_standing_charge_accrued` | Running total the blueprint writes to directly | Min `0`, Max `100000`, Step `0.0001` |
| `input_datetime.electricity_standing_charge_last_accrual` (type: **Date**, no time component) | Last date the blueprint successfully ran — used only for catch-up after Home Assistant has been offline. Leave it at its default value. |

**For the rate itself**, most UK energy integrations (Octopus Energy and
others) already expose it somewhere, so you likely don't need to create
anything: see [Using a supplier integration for the rate](#using-a-supplier-integration-for-the-rate)
below, find the right entity for your supplier, and use that directly as
the **Standing Charge Rate Entity** in step 3.

Only if you don't have a suitable supplier-exposed entity, fall back to a
manually maintained Number helper instead:

| Helper | Purpose | Suggested settings |
|---|---|---|
| `input_number.electricity_standing_charge_rate` | Today's daily standing charge, in your currency (e.g. `0.6123` for 61.23p) | Min `0`, Max `10`, Step `0.0001` |

You'll need to update this by hand whenever your tariff changes.

If you also have a gas supply, repeat the above with `gas_` names. If
you're electricity-only, skip ahead — there's nothing to create for gas.

### 2. Create two Template sensors

Settings → Devices & Services → Helpers → **+ Create Helper** → Template →
Template a sensor. Create both of these (values shown are for electricity —
repeat for gas):

**Standing charge cost sensor** — mirrors the `input_number` from step 1,
tagged so the Energy Dashboard treats it as money. This is a *different*
entity from the `input_number` it reads from — note the different domain
(`sensor.` vs `input_number.`) and name (`_total` vs `_accrued`):
- Name: `Electricity Standing Charge Total`
- State template: `{{ states('input_number.electricity_standing_charge_accrued') | float(0) }}`
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

### 3. Import and configure the blueprint

Click the import badge above, or in Home Assistant go to Settings →
Automations & Scenes → Blueprints → Import Blueprint, and paste in:

```
https://github.com/AtomBrake/HA-Standing-Charge-Blueprint/blob/main/blueprints/automation/standing_charge/standing_charge_accrual.yaml
```

Create **one** automation from the blueprint — it has an Electricity
section and a Gas section.

Fill in the Electricity section (required):

- **Standing Charge Rate Entity**: your supplier's entity (see step 1), or
  `input_number.electricity_standing_charge_rate` if you created the
  fallback helper instead
- **Cumulative Total Entity**: `input_number.electricity_standing_charge_accrued`
  — this must be the `input_number` from step 1, **not** the
  `sensor.electricity_standing_charge_total` template sensor from step 2
  (that one won't appear in this field's picker at all, since it isn't an
  `input_number`)
- **Last Accrual Date Entity**: `input_datetime.electricity_standing_charge_last_accrual`

If you have a gas supply, expand the Gas section and fill in the
equivalent three fields with your `gas_` entities. **Electricity-only?
Leave the whole Gas section collapsed and blank** — it's skipped entirely
when empty.

Finally, set the **Accrual Time** (default `00:00:00`) — this applies to
both fuels, since standing charges typically both start from midnight.

### 4. Add the source to the Energy Dashboard

Settings → Dashboards → Energy → Electricity grid → **Add Consumption**:

1. For "consumed energy", pick `sensor.electricity_standing_charge_usage`
   (the dummy 0 kWh sensor).
2. Untick "Use a static price" / "Use an entity with the current price".
3. Enable **"Use an entity tracking the total costs"** and select
   `sensor.electricity_standing_charge_total`.
4. On Home Assistant 2026.6 and later, this step also lets you set a
   custom display name for the source — independent of the entity's own
   name — so it's worth labelling it something clear like "Standing
   Charge" here rather than relying on whatever the dummy sensor happens
   to be called.
5. Save.

If you have a gas supply, repeat under "Gas consumption" for the gas
sensors. The gas dialog also shows a **"Gas flow rate"** field — that's
an unrelated, separate feature for real-time flow monitoring, not
billing. It's optional (the Save button doesn't require it); since the
dummy usage sensor has no real flow rate to report, just leave it blank.

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
  daily, make sure whatever the rate entity points at is updated before the
  accrual time each day — automatic if it's a supplier-integration sensor,
  manual if you're using the fallback `input_number`.

## Using a supplier integration for the rate

The blueprint's **Standing Charge Rate Entity** input accepts any
`input_number`, `number`, or `sensor` entity — it just reads that entity's
state as a number, so it has no idea which supplier (if any) it came from.
If your supplier's integration already exposes the standing charge as its
own sensor state, just point the input at it directly and skip creating
the `_rate` helper entirely.

For example, the
[Octopus Energy integration](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy)
exposes the standing charge directly as its own sensor:

```
sensor.octopus_energy_electricity_<METER_SERIAL>_<MPAN>_current_standing_charge
sensor.octopus_energy_gas_<METER_SERIAL>_<MPRN>_current_standing_charge
```

Find your exact entity id under Developer Tools → States (filter for
`standing_charge`) and use it directly as the rate entity for that fuel —
no template sensor or manual helper needed.

If your supplier's integration doesn't expose a dedicated entity like
this and only buries the rate inside an attribute on some other sensor,
you can still pull it out with a one-line template sensor, e.g.:

```yaml
template:
  - sensor:
      - name: "Electricity Standing Charge Rate"
        unique_id: electricity_standing_charge_rate
        unit_of_measurement: "GBP"
        device_class: monetary
        state: >-
          {{ state_attr('sensor.your_suppliers_cost_sensor', 'standing_charge') | float(0) }}
```

Point the blueprint's rate entity at that template sensor instead. This
attribute-extraction pattern works for any integration that buries the
rate in an attribute rather than a dedicated entity — the blueprint
itself never needs to know or care either way.

## License

MIT — see [LICENSE](LICENSE).
