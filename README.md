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

### Schedule & night fallback

Sun-based positioning is only applied while the current time is inside the
**allowed window** (`Allowed from` … `Allowed until`, which may cross midnight).

Outside the window:

- **Close fully at night = on** → every configured cover is driven to the
  **Night position** (default `0`, fully closed).
- **Close fully at night = off** → covers are left untouched and keep whatever
  position they had.

A cover is only ever commanded when its current position differs from the
target, avoiding needless motor movement.

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
| **Building offset** | `-45°`…`+45°`, rotation of the building away from true N-S / E-W. |
| **North / East / South / West covers** | Cover groups (each optional). |
| **None / Indirect-early / Direct / Indirect-late** | Target position (0–100%) per zone. |
| **Allowed from / until** | Schedule window (may cross midnight). |
| **Close fully at night** | Toggle for the night fallback. |
| **Night position** | Position used outside the window when the fallback is on. |
| **Disable entity** | Optional boolean-like entity; while `on` the automation is stopped. |
| **Disable when (advanced)** | Optional condition(s); if any is true the automation is stopped. |

### Requirements

- A numeric azimuth source. With the built-in **Sun** integration, enable the
  `sensor.sun_solar_azimuth` entity (Settings → Devices & Services → Entities).
- Covers that support `set_cover_position`.

### Installation

1. In Home Assistant: **Settings → Automations & Scenes → Blueprints →
   Import Blueprint**.
2. Paste the raw URL of `sun-azimuth-cover-control.yaml` (or copy the file into
   `config/blueprints/automation/<your-folder>/`).
3. Create an automation from the blueprint and fill in the inputs.
