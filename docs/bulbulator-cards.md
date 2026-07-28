# Bulbulator — dashboard cards

Ready-to-adapt Lovelace cards for the three Bulbulator blueprints. The dynamic
ones use **`custom:button-card`** (HACS). Replace the entity IDs (and the
`automation.bulbulator_*` targets) with your own; everything else works as-is.

Each button-card reads the light/switch states directly (no helper), so the icon
and label update the moment the fixture changes — including manual switch flips —
thanks to `triggers_update`.

---

## 1. Single-room light cycler

Tap to cycle `0 → 1 → 2 → 3 → 0` bulbs. Shows the live count `N / 3`, greys out
when off, glows amber when lit.

```yaml
type: custom:button-card
name: Kitchen light
icon: mdi:ceiling-light-multiple
show_label: true
show_state: false
tap_action:
  action: perform-action
  perform_action: automation.trigger
  target:
    entity_id: automation.bulbulator_kitchen
  data:
    skip_condition: true
variables:
  sw1: switch.kitchen_one_bulb      # controls 1 bulb
  sw2: switch.kitchen_two_bulbs     # controls 2 bulbs
triggers_update:
  - switch.kitchen_one_bulb
  - switch.kitchen_two_bulbs
label: |
  [[[
    var n = (states[variables.sw1].state === 'on' ? 1 : 0)
          + (states[variables.sw2].state === 'on' ? 2 : 0);
    return n + ' / 3';
  ]]]
styles:
  card:
    - height: 92px
  label:
    - font-size: 13px
    - opacity: 0.8
  icon:
    - color: |
        [[[
          var n = (states[variables.sw1].state === 'on' ? 1 : 0)
                + (states[variables.sw2].state === 'on' ? 2 : 0);
          return n === 0 ? 'var(--disabled-text-color)' : 'var(--state-light-active-color, orange)';
        ]]]
```

Optional — swap the icon by count instead of just recolouring it (replace the
`icon:` line with a template):

```yaml
icon: |
  [[[
    var n = (states[variables.sw1].state === 'on' ? 1 : 0)
          + (states[variables.sw2].state === 'on' ? 2 : 0);
    return ['mdi:lightbulb-outline','mdi:lightbulb','mdi:lightbulb-on','mdi:lightbulb-group'][n];
  ]]]
```

---

## 2. Input-number bulb control

This one is driven by a slider, so the natural card is a number slider rather
than a tap button. Using **`custom:mushroom-number-card`** (HACS), 0–3 maps to
0–3 bulbs:

```yaml
type: custom:mushroom-number-card
entity: input_number.kitchen_bulbs
name: Kitchen bulbs
icon: mdi:ceiling-light-multiple
display_mode: slider
```

Plain-core alternative (no HACS):

```yaml
type: tile
entity: input_number.kitchen_bulbs
name: Kitchen bulbs
features:
  - type: numeric-input
    style: slider
```

If you prefer a **tap-to-step button** over a slider, reuse the single-room
button above but point `tap_action` at this automation and have the label read
`states['input_number.kitchen_bulbs'].state`.

---

## 3. Multi-room route light cycler

Tap to move the single lit light along the route. Shows the current step
`krok N / M` (or `off`), and lists which entry is lit.

```yaml
type: custom:button-card
name: Hallway route
icon: mdi:transit-connection-variant
show_label: true
show_state: false
tap_action:
  action: perform-action
  perform_action: automation.trigger
  target:
    entity_id: automation.bulbulator_hallway
  data:
    skip_condition: true
variables:
  route:
    - light.hall_1
    - light.hall_2
    - light.stairs
    - light.landing
triggers_update:
  - light.hall_1
  - light.hall_2
  - light.stairs
  - light.landing
label: |
  [[[
    var r = variables.route, cur = 0;
    for (var i = 0; i < r.length; i++) {
      if (states[r[i]] && states[r[i]].state === 'on') cur = i + 1;
    }
    if (cur === 0) return 'off';
    var name = states[r[cur - 1]].attributes.friendly_name || r[cur - 1];
    return 'krok ' + cur + ' / ' + r.length + ' · ' + name;
  ]]]
styles:
  card:
    - height: 92px
  label:
    - font-size: 12px
    - opacity: 0.8
  icon:
    - color: |
        [[[
          var r = variables.route, on = false;
          for (var i = 0; i < r.length; i++) {
            if (states[r[i]] && states[r[i]].state === 'on') on = true;
          }
          return on ? 'var(--state-light-active-color, orange)' : 'var(--disabled-text-color)';
        ]]]
```

> Note: keep the `triggers_update` list in sync with `variables.route` so the
> card refreshes when any route member changes (including manual flips). You can
> also use `triggers_update: all` to update on any state change if you'd rather
> not maintain the list.
