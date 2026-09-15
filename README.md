# Home Assistant automations

YAML automations for my Home Assistant instance. Each file under
`automations/` is a top-level automation list entry: paste it into
`automations.yaml`, or import it through
Settings -> Automations -> (new automation) -> Edit in YAML.

| File | What it does |
| --- | --- |
| [`automations/velux_roof_window_night_temperature.yaml`](automations/velux_roof_window_night_temperature.yaml) | Closes the Velux roof window at night below 19.7 deg C, reopens it above 20.3 deg C. For `automations.yaml`. |
| [`automations/velux_roof_window_night_temperature.ui.yaml`](automations/velux_roof_window_night_temperature.ui.yaml) | The same automation as a single mapping, for the UI's "Edit in YAML" box. |

## Which copy do I paste?

The two files hold the same automation in the two shapes Home Assistant
accepts, and they must be kept in step when either is edited.

- **`automations.yaml`** is a *list* of automations, so its copy keeps the
  leading `-`, the two-space indentation, and an explicit `id:`.
- **The UI's "Edit in YAML" box** validates one automation *mapping*, so its
  copy has no leading `-`, no indentation and no `id:` (the UI assigns one).

Pasting the list form into the UI box fails with
`Message malformed: not a valid option at ['0']` - `['0']` being the index of
the first list entry. The same error from `automations.yaml` means an
unrecognised key inside that entry instead; the usual cause is the modern
`triggers:`/`conditions:`/`actions:` schema on a core older than 2024.10.
Both files here use the legacy `trigger:`/`condition:`/`action:` keys with
`platform:` and `service:`, which every version accepts.

## Velux roof window - night temperature control

| Setting | Value | Where to change it |
| --- | --- | --- |
| Night window | 21:00 - 07:00 | The `time` trigger `at:` **and** the `time` condition `after:`/`before:` |
| Close below | 19.7 deg C | `numeric_state` trigger `below:` (in two places: the trigger and the night-start condition) |
| Open above | 20.3 deg C | `numeric_state` trigger `above:` |
| Debounce | 5 minutes | `for:` on both `numeric_state` triggers |

Edit the same row in **both** files.

Notes on the behaviour, so the edges aren't surprising:

- **The 0.6 deg gap is intentional.** Between 19.7 and 20.3 nothing happens;
  that hysteresis stops the window cycling on sensor noise around a single
  set point. The 5-minute `for:` on each trigger is a second layer of the
  same protection.
- **Outside 21:00-07:00 the automation does nothing at all.** It will not
  reopen the window in the morning just because it got warm - the window
  stays wherever it was left.
- **The 21:00 trigger exists because `numeric_state` only fires on a
  crossing.** If the temperature is already below 19.7 when the night window
  opens, no crossing ever happens, so the automation would otherwise sit
  there with the window open all night.
- **The 21:00 trigger only ever closes.** If it is warm at 21:00 the window
  is left alone, rather than being forced open - opening is reserved for an
  actual rise back through 20.3.
- **There is no rain or wind guard.** If the Velux integration does not
  already close on rain itself, consider adding a condition on a rain sensor
  before the open branch.
