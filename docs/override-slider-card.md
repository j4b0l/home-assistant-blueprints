# Override slider card

A compact Lovelace card for an `input_number` helper that uses **-1 as "off"**
and **0–100 as a value**, e.g. the per-cover override helpers of
[Sun Azimuth Cover Control (Seasonal)](../sun-azimuth-cover-control-seasonal.yaml)
(`input_number.<cover>_override`: -1 = no pin, 0 = closed, 100 = open).

```
┌────────────────────────────────────────────────────────────────┐
│  Kitchen                                                       │
│  [  ✕  ] [  ⏪  ] [━━━━━━━━━━━━━━━●─────────────────] [  ⏩  ]  │
└────────────────────────────────────────────────────────────────┘
    1/8      1/8                  5/8                     1/8
```

| Control | Icon | Action |
| --- | --- | --- |
| Name | – | configurable title (`name:`); defaults to the helper's friendly name |
| Disable | `mdi:close` | set the helper to **-1** |
| Set 0 | `mdi:rewind` | set the helper to **0** |
| Slider | – | any value over the helper's whole range (min..max, step) |
| Set 100 | `mdi:fast-forward` | set the helper to **100** |

The button matching the current value (-1 / 0 / 100) is highlighted in the
theme's primary colour; the slider follows the value live.

Requirements:
- HACS frontend cards **`custom:button-card`** and **Mushroom**
  (`custom:mushroom-number-card`). No card-mod needed.
- The helper's range must include -1 and 100 (e.g. min -1, max 100, step 1).
  Outside that range `input_number.set_value` is rejected and the button does
  nothing.

---

## Card

Set **`entity:`** and **`name:`** - nothing else refers to the helper
directly. Every inner card reads it through `[[[ return entity.entity_id ]]]`;
button-card evaluates `[[[ … ]]]` templates inside `custom_fields` cards in the
context of the OUTER card, so `entity` there is always this card's helper. (A
template meant for the inner card itself would need `[[[[ … ]]]]`; none is
used here.)

```yaml
type: custom:button-card
entity: input_number.kitchen_blind_override
name: Kitchen
show_icon: false
show_name: true
show_state: false
tap_action:
  action: none
hold_action:
  action: none
# Sections view: full width, height follows the content.
grid_options:
  columns: full
  rows: auto
styles:
  card:
    - padding: 12px
  grid:
    - grid-template-areas: '"n n n n" "disable zero slider full"'
    - grid-template-columns: 1fr 1fr 5fr 1fr
    - row-gap: 8px
    - column-gap: 8px
    - align-items: center
  name:
    - justify-self: start
    - padding: 0 4px
    - font-size: 14px
    - font-weight: 500
  custom_fields:
    slider:
      # Strip the inner Mushroom card's own background/border/padding so the
      # slider sits flush in the row (CSS variables inherit into it).
      - --ha-card-background: transparent
      - --ha-card-box-shadow: none
      - --ha-card-border-width: "0"
      - --mush-spacing: 0px
      - --mush-control-height: 42px
      - --mush-control-border-radius: 12px
# button-card's host is max-width: fit-content; lift it so the row (and each
# button) fills its width.
extra_styles: |
  :host { max-width: none !important; }
  #disable, #zero, #slider, #full { min-width: 0; }
custom_fields:
  disable:
    card:
      type: custom:button-card
      entity: "[[[ return entity.entity_id ]]]"
      icon: mdi:close
      show_name: false
      show_state: false
      size: 24px
      tap_action:
        action: perform-action
        perform_action: input_number.set_value
        target:
          entity_id: "[[[ return entity.entity_id ]]]"
        data:
          value: -1
      hold_action:
        action: none
      extra_styles: |
        :host { max-width: none !important; }
      styles:
        card:
          - height: 42px
          - padding: 0
          - border: none
          - box-shadow: none
          - border-radius: 12px
          - background-color: |
              [[[
                return Number(entity.state) === -1
                  ? 'rgba(var(--rgb-primary-color), 0.2)'
                  : 'rgba(var(--rgb-primary-text-color), 0.05)';
              ]]]
        icon:
          - color: |
              [[[
                return Number(entity.state) === -1
                  ? 'var(--primary-color)'
                  : 'var(--primary-text-color)';
              ]]]
  zero:
    card:
      type: custom:button-card
      entity: "[[[ return entity.entity_id ]]]"
      icon: mdi:rewind
      show_name: false
      show_state: false
      size: 24px
      tap_action:
        action: perform-action
        perform_action: input_number.set_value
        target:
          entity_id: "[[[ return entity.entity_id ]]]"
        data:
          value: 0
      hold_action:
        action: none
      extra_styles: |
        :host { max-width: none !important; }
      styles:
        card:
          - height: 42px
          - padding: 0
          - border: none
          - box-shadow: none
          - border-radius: 12px
          - background-color: |
              [[[
                return Number(entity.state) === 0
                  ? 'rgba(var(--rgb-primary-color), 0.2)'
                  : 'rgba(var(--rgb-primary-text-color), 0.05)';
              ]]]
        icon:
          - color: |
              [[[
                return Number(entity.state) === 0
                  ? 'var(--primary-color)'
                  : 'var(--primary-text-color)';
              ]]]
  slider:
    card:
      type: custom:mushroom-number-card
      entity: "[[[ return entity.entity_id ]]]"
      display_mode: slider
      icon_type: none
      primary_info: none
      secondary_info: none
  full:
    card:
      type: custom:button-card
      entity: "[[[ return entity.entity_id ]]]"
      icon: mdi:fast-forward
      show_name: false
      show_state: false
      size: 24px
      tap_action:
        action: perform-action
        perform_action: input_number.set_value
        target:
          entity_id: "[[[ return entity.entity_id ]]]"
        data:
          value: 100
      hold_action:
        action: none
      extra_styles: |
        :host { max-width: none !important; }
      styles:
        card:
          - height: 42px
          - padding: 0
          - border: none
          - box-shadow: none
          - border-radius: 12px
          - background-color: |
              [[[
                return Number(entity.state) === 100
                  ? 'rgba(var(--rgb-primary-color), 0.2)'
                  : 'rgba(var(--rgb-primary-text-color), 0.05)';
              ]]]
        icon:
          - color: |
              [[[
                return Number(entity.state) === 100
                  ? 'var(--primary-color)'
                  : 'var(--primary-text-color)';
              ]]]
```

---

## Several helpers

Because only `entity` and `name` vary, define the card once with a YAML anchor
and merge it for the rest (Home Assistant expands this when it saves):

```yaml
cards:
  - &override_slider
    type: custom:button-card
    entity: input_number.kitchen_blind_override
    name: Kitchen
    # ... rest of the card above ...
  - <<: *override_slider
    entity: input_number.bedroom_blind_override
    name: Bedroom
```

---

## Tweaks

- **Proportions:** `grid-template-columns: 1fr 1fr 5fr 1fr` is the 1/8 · 1/8 ·
  5/8 · 1/8 split (the 8 px gaps come out of the total).
- **Spacing / rounding:** outer padding `12px`, gaps `8px`, controls `42px`
  high with `12px` corners - the same metrics as Mushroom cards, so the row
  lines up with them. Change all `42px` / `12px` together to keep the buttons
  and the slider matched.
- **No title:** set `show_name: false` and drop the `"n n n n"` row from
  `grid-template-areas`.
