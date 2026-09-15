# Home Assistant automations

YAML automations for my Home Assistant instance. Each file under
`automations/` is a top-level automation list entry: paste it into
`automations.yaml`, or import it through
Settings -> Automations -> (new automation) -> Edit in YAML.

| File | What it does |
| --- | --- |
| [`automations/velux_roof_window_night_temperature.yaml`](automations/velux_roof_window_night_temperature.yaml) | Closes the Velux roof window at night below 19.7 deg C, reopens it above 20.3 deg C. For `automations.yaml`. |
| [`automations/velux_roof_window_night_temperature.ui.yaml`](automations/velux_roof_window_night_temperature.ui.yaml) | The same automation as a single mapping, for the UI's "Edit in YAML" box. |
| [`automations/velux_roof_window_co2_airing.yaml`](automations/velux_roof_window_co2_airing.yaml) | Airs the room during the day when CO2 goes over 1000 ppm. For `automations.yaml`. |
| [`automations/velux_roof_window_co2_airing.ui.yaml`](automations/velux_roof_window_co2_airing.ui.yaml) | The same automation as a single mapping, for the UI's "Edit in YAML" box. |

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
| Restart settle | 2 minutes | The `delay:` in the `if:` guarded by `id: ha_start` |

Edit the same row in **both** files.

Notes on the behaviour, so the edges aren't surprising:

- **The 0.6 deg gap is intentional.** Between 19.7 and 20.3 nothing happens;
  that hysteresis stops the window cycling on sensor noise around a single
  set point. The 5-minute `for:` on each trigger is a second layer of the
  same protection.
- **Outside 21:00-07:00 the automation does nothing at all.** It will not
  reopen the window in the morning just because it got warm - the window
  stays wherever it was left.
- **The 21:00 and restart triggers exist because `numeric_state` only fires
  on a crossing.** If the temperature is already below 19.7 when the night window
  opens, no crossing ever happens, so the automation would otherwise sit
  there with the window open all night.
- **The catch-up triggers only ever close.** If it is warm at 21:00, or warm
  when Home Assistant restarts, the window is left alone rather than being
  forced open - opening is reserved for an actual rise back through 20.3. A
  window that opens itself seconds after a reboot is the worse surprise.
- **The restart path waits two minutes before reading the sensor.** After a
  restart the temperature is often still `unknown` while its integration
  reconnects, and a `numeric_state` condition against `unknown` is false, so
  the catch-up would quietly do nothing. The two threshold crossings are not
  delayed.
- **There is no rain or wind guard.** If the Velux integration does not
  already close on rain itself, consider adding a condition on a rain sensor
  before the open branch.

## Velux roof window - CO2 airing

| Setting | Value | Where to change it |
| --- | --- | --- |
| Day window | 07:00 - 21:00 | The `time` condition `after:`/`before:` |
| Open above | 1000 ppm | The `numeric_state` trigger `above:` **and** the `numeric_state` condition |
| Close below | 800 ppm | The `below:` inside `wait_for_trigger` |
| Max airing | 15 minutes | `timeout:` on the `wait_for_trigger` |
| Pause after closing | 10 minutes | The trailing `delay:` |
| Re-check interval | 15 minutes | The `time_pattern` trigger |

Edit the same row in **both** files.

Both automations drive the same window, so they are separated in time rather
than by priority: 07:00-21:00 belongs to CO2, 21:00-07:00 to temperature.
`after:` is inclusive and `before:` exclusive in Home Assistant, so the two
windows tile the day exactly - no overlap, no gap.

- **Closing is "whichever comes first".** `wait_for_trigger` waits for CO2 to
  fall below 800; `timeout` with `continue_on_timeout: true` closes anyway
  after 15 minutes. No helper entity is needed for this.
- **Only a closed window is ever touched.** A window you opened yourself is
  left alone, rather than being shut in your face 15 minutes later.
- **`mode: single` is load-bearing.** The whole airing cycle - open, wait,
  close, pause - is one run, and further triggers are dropped while it is in
  progress. That is what stops a window that timed out at 1100 ppm from
  reopening immediately.
- **The `time_pattern` trigger is the catch-up**, covering the three cases a
  threshold crossing misses: already over 1000 at 07:00 (after a night in a
  bedroom, the normal case), 15 minutes of airing were not enough, or Home
  Assistant restarted.
- **Known limitation: a restart mid-airing leaves the window open.**
  `wait_for_trigger` does not survive a restart, and the catch-up only acts on
  a *closed* window, so nothing closes it again. Closing it automatically
  would mean closing manually opened windows too. If this turns out to matter,
  the fix is an `input_boolean` helper marking "airing in progress", which
  does survive a restart.
