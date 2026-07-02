# home-assistant-blueprints

A small collection of Home Assistant automation blueprints.

| Blueprint | File | Description |
| --- | --- | --- |
| IKEA BILRESA scroll wheel (shutter) | [`ikea-bilresa-scroll-wheel-shutter.yaml`](ikea-bilresa-scroll-wheel-shutter.yaml) | Control shutters with the IKEA BILRESA Matter scroll wheel. |
| Sun Azimuth Cover Control | [`sun-azimuth-cover-control.yaml`](sun-azimuth-cover-control.yaml) | Position covers automatically based on where the sun is. |

---

## Sun Azimuth Cover Control

Automatically positions four groups of covers (north / south / east / west
facades) based on the sun's **azimuth**, so each facade reacts to whether the
sun is not on it, arriving, shining directly, or leaving.

### How it works

Every facade *faces* a cardinal direction, shifted by the building offset:

| Facade | Faces |
| --- | --- |
| North | `0° + offset` |
| East  | `90° + offset` |
| South | `180° + offset` |
| West  | `270° + offset` |

For each facade the blueprint computes the signed angular difference between
the sun azimuth and the facade direction, normalised to `[-180°, +180°]`, and
picks a zone:

| Condition (`delta = azimuth − facing`) | Zone | Meaning |
| --- | --- | --- |
| `-45° ≤ delta < 45°` | **direct** | Sun shines straight onto the facade |
| `-90° ≤ delta < -45°` | **indirect-early** | Sun just started grazing (rising side) |
| `45° ≤ delta < 90°` | **indirect-late** | Sun about to leave (setting side) |
| otherwise | **none** | Sun is not on the facade |

Each zone maps to a configurable cover position. The full 360° daily sweep is
handled with modular arithmetic, so wrap-around at 0°/360° is correct.

**Worked example** — South facade (`facing = 180°`, offset 0):

- azimuth `90°–135°` → indirect-early
- azimuth `135°–225°` → direct
- azimuth `225°–270°` → indirect-late
- everything else → none

At azimuth `135°` (south-east) the West facade (`delta = -135°`) and North
facade (`delta = 135°`) are both **none** — matching the intuition that the sun
is nowhere near them.

### Position convention

Positions follow the Home Assistant standard: **`100` = fully open, `0` = fully
closed**. Covers must support `cover.set_cover_position`. The example defaults
are `none = 100`, `indirect-early = 70`, `direct = 30`, `indirect-late = 70`
(more sun → more closed). If you prefer to think in "how much closed", just
invert your configured values.

### Schedule, twilight & night fallback

Sun-azimuth positioning is applied while the sun is **above the horizon** (read
from the elevation entity) **and** inside the active window. The active window
is one of two modes:

- **Time window** (default) — `Allowed from` … `Allowed until` (may cross
  midnight).
- **Sunrise/sunset** — enable *"Use sunrise/sunset instead of the time window"*
  and the fixed times are ignored; the active window becomes exactly
  sunrise-to-sunset (elevation > 0). Morning/evening positions are unused in
  this mode.

In **time-window mode**, while inside the window but the sun is still below the
horizon, all covers take a transitional position:

- between the **start time and sunrise** → **Morning position**
- between **sunset and the end time** → **Evening position**

(Morning vs evening is decided by the window's midpoint, which also works for
windows that cross midnight.)

**Outside the active window** (and outside sunrise-sunset in sun mode):

- **Close fully at night = on** → every configured cover is driven to the
  **Night position** (default `0`, fully closed).
- **Close fully at night = off** → covers are left untouched and keep whatever
  position they had.

If the elevation entity is unavailable/non-numeric, the automation does nothing
that run. A cover is only ever commanded when its current position differs from
the target, avoiding needless motor movement.

### Disable / guard

An optional guard can stop the automation **completely** (no sun positioning
and no night close). Two independent inputs are provided; if either one
triggers, the run is halted:

- **Disable entity** — pick a single boolean-like entity (`input_boolean`,
  `binary_sensor`, `switch`). While its state is `on`, the automation does
  nothing. Ideal for a simple "out of home" helper.
- **Disable when (advanced)** — a full condition builder. If **any** condition
  you add here is true, the automation stops. To express *"out of home OR
  vacation"*, add two state conditions (one per helper) — each is checked
  independently, so either being `on` halts the run. For more complex logic,
  nest an `and`/`or`/`not` block inside.

Both are optional; leaving them empty disables the guard.

### Re-evaluation

The automation re-runs on:

- any change of the azimuth entity,
- every 5 minutes (`time_pattern`),
- the window start and end times,
- Home Assistant startup.

It runs in `mode: queued` so bursts of azimuth updates are handled in order.

### Inputs

| Input | Notes |
| --- | --- |
| **Sun azimuth entity** | A `sensor`/`input_number` whose **state** is the azimuth in degrees (0–360). For the built-in Sun integration use `sensor.sun_solar_azimuth` — the `sun.sun` entity exposes azimuth only as an *attribute* and cannot be used directly. |
| **Sun elevation entity** | A `sensor`/`input_number` whose **state** is the elevation in degrees (positive = above horizon). Use `sensor.sun_solar_elevation`. |
| **Building offset** | `-45°`…`+45°`, rotation of the building away from true N-S / E-W. |
| **North / East / South / West covers** | Cover groups (each optional). |
| **None / Indirect-early / Direct / Indirect-late** | Target position (0–100%) per zone. |
| **Use sunrise/sunset instead of the time window** | Ignore the fixed times; active window becomes sunrise→sunset. |
| **Allowed from / until** | Fixed schedule window (may cross midnight); ignored in sunrise/sunset mode. |
| **Morning position** | All-cover position between start time and sunrise (time-window mode). |
| **Evening position** | All-cover position between sunset and end time (time-window mode). |
| **Close fully at night** | Toggle for the night fallback. |
| **Night position** | Position used outside the window when the fallback is on. |
| **Disable entity** | Optional boolean-like entity; while `on` the automation is stopped. |
| **Disable when (advanced)** | Optional condition(s); if any is true the automation is stopped. |

### Requirements

- Numeric azimuth **and** elevation sources. With the built-in **Sun**
  integration, enable the `sensor.sun_solar_azimuth` and
  `sensor.sun_solar_elevation` entities (Settings → Devices & Services →
  Entities).
- Covers that support `set_cover_position`.

### Installation

1. In Home Assistant: **Settings → Automations & Scenes → Blueprints →
   Import Blueprint**.
2. Paste the raw URL of `sun-azimuth-cover-control.yaml` (or copy the file into
   `config/blueprints/automation/<your-folder>/`).
3. Create an automation from the blueprint and fill in the inputs.
