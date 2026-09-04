# Custom hud VCSS contract

Panorama CSS is **not** web CSS. There is no `display`, `flex`, `grid`, `float`,
`left/top/right/bottom`, `gap`, `filter`, `justify-content` or `align-items`. Layout is built
from flow (`flow-children`) plus alignment (`align`) plus sizing.

The property set is fixed at compile time in a single global registration table inside the
client. It cannot be extended by content, and the engine asserts if it ever overflows
(`Need to increase size of static g_StylePropertyRegistrations (MAX_PANORAMA_STYLE_SYMBOLS)
before registering more styles, failed on %s`).

## 1. Attaching a stylesheet

A custom hud has no inline styles. The only mechanism is:

```xml
<styles>
    <include src="s2r://panorama/styles/my_hud.vcss" />
</styles>
```

Only a `.vcss` resource may be referenced; the path is arbitrary. The `.vcss` itself is
**not inspected** by the validator — the full Panorama CSS language is available inside it.

Selectors: `#id`, `.class`, panel type (`Panel`, `Label`, `Image`, `Button`, and also
`CSGOCustomHudLayoutRoot`), pseudo-classes (`:hover`, `:active`, ...), and nesting.

## 2. All 140 properties

In registration order:

```
position, background-image, opacity, background-color, background-color-opacity,
border, overflow, color, padding, font, wash-color, box-shadow, letter-spacing,
paragraph-spacing, transform, text-shadow, img-shadow, pre-transform-scale2d,
text-align, z-index, white-space, opacity-mask, x, y, z, hue-rotation, saturation,
brightness, contrast, cursor, blur, background-blur, world-blur,
pre-transform-rotate2d, font-family, font-size, font-style, font-weight,
font-stretch, text-decoration, text-decoration-style, text-transform, text-overflow,
-s2-mix-blend-mode, texture-sampling,
border-top, border-right, border-bottom, border-left,
border-style, border-top-style, border-right-style, border-bottom-style, border-left-style,
border-width, border-top-width, border-right-width, border-bottom-width, border-left-width,
border-color, border-top-color, border-right-color, border-bottom-color, border-left-color,
border-radius, border-top-right-radius, border-bottom-right-radius,
border-bottom-left-radius, border-top-left-radius, border-brush,
clip, line-height, perspective, perspective-origin, transform-origin,
width, height, visibility, flow-children, ignore-parent-flow,
background-size, background-texture-size, background-position, background-repeat,
opacity-mask-position, opacity-mask-scale, opacity-mask-threshold,
padding-left, padding-top, padding-bottom, padding-right,
margin, margin-left, margin-top, margin-bottom, margin-right,
transition, transition-property, transition-duration, transition-timing-function,
transition-delay, transition-high-framerate, transition-frame-time,
animation, animation-name, animation-duration, animation-timing-function,
animation-iteration-count, animation-direction, animation-delay, animation-fill-mode,
animation-frame-time,
align, horizontal-align, vertical-align,
min-width, min-height, max-width, max-height,
tooltip-position, tooltip-body-position, tooltip-arrow-position,
context-menu-position, context-menu-body-position, context-menu-arrow-position,
sound, sound-out,
ui-scale, ui-scale-x, ui-scale-y, ui-scale-z,
layout-position, background-img-opacity, opacity-brush,
border-image, border-image-source, border-image-slice, border-image-width,
border-image-outset, border-image-repeat
```

`x`, `y`, `z` are Panorama's positional offset properties — the replacement for the absent
`left`/`top`.

## 3. Units

| Unit | Used for | Notes |
|------|----------|-------|
| `px` | lengths | the parser requires the `px` suffix; a bare number is **invalid** (except the literal `0`) |
| `%`  | lengths | `%` suffix |
| `s`  | time | `transition`, `animation` |
| `ms` | time | accepted by the parser (divides by 1000); unused in shipped content |
| `deg`| angles | mandatory — an angle without `deg` is rejected |

`vw`, `vh`, `em`, `rem`, `dp`, `turn` and `grad` **do not exist** — not in the parsers, and
not in any of the ~225 stylesheets shipped with the game.

## 4. Value sets for the layout properties

### `width` / `height`

| Value | Kind |
|-------|------|
| `<n>px` | length |
| `<n>%` | percentage |
| `fit-children` | size to content (the default) |
| `fill-parent-flow( <weight> )` | share of remaining space in the flow |
| `width-percentage( <pct> )` / `height-percentage( <pct> )` | derive from the other axis (aspect lock) |
| `safe-area-inset-left` / `-right` / `-top` / `-bottom` | safe-area inset |

Per-property rejections: `width` rejects `width-percentage`, `height` rejects
`height-percentage`, and `max-width`/`max-height` reject `fit-children`.

### `flow-children` — 10 values

`none`, `down`, `right`, `up`, `left`, `down-wrap`, `right-wrap`, `up-wrap`, `left-wrap`,
`down-wrap-left`

(`up-wrap`, `left-wrap` and `down-wrap-left` are accepted by the parser but are used in no
shipped stylesheet.)

### `horizontal-align` / `vertical-align`

- horizontal: `left`, `center`, `middle`, `right`, `center_nopixelsnap`
- vertical: `top`, `center`, `middle`, `bottom`, `center_nopixelsnap`
- `middle` is a **synonym** of `center` (same enum value), not a distinct alignment.

### `align`

A shorthand requiring exactly **two** tokens: horizontal then vertical
(`align: center center;`). A single-token `align: center;` is rejected outright.

### `overflow`

Values: `squish` (default), `clip`, `scroll`, `noclip`. One token applies to both axes;
an optional second token overrides the Y axis: `overflow: squish scroll;`.

### `ignore-parent-flow`

Boolean. `true` removes the panel from its parent flow, enabling `position` / `align`.

### `visibility`

`visible` (default) | `collapse` (invisible **and** removed from layout).

## 5. Other frequently needed declarations

```css
position:        3% 20px 0px;        /* x y z; only valid outside a flow */
transform:       translate3d(...), rotate3d(...), scale3d(...), rotatez(45deg);
transform-origin: 50% 50%;           /* default */
perspective:     1000;

background-color: #RRGGBBAA;
background-color: gradient( linear, 0% 0%, 0% 100%, from(#000000ff), to(#00000000) );
background-color: gradient( radial, 50% 50%, 0% 0%, 100% 100%, from(#fff), to(#000) );
background-image: url("file://{images}/pic.png");
background-size:  contains | auto | <px|%>;
background-repeat: repeat | space | round | no-repeat | repeat-x | repeat-y;

blur:            gaussian( 2 );  /* also gaussian(x, y, passes), mipmapgaussian(...) */
wash-color:      #ffffffff;

transition: color 0.2s ease-in-out 0.0s, opacity 0.15s linear 0.0s;
/* timing functions: ease, ease-in, ease-out, ease-in-out, linear */
```

## 6. Preprocessor

```css
@import url("s2r://panorama/styles/other.vcss_c");

@define my-accent: #eabe54;
.Label { color: my-accent; }        /* the name is used bare as a value */
```

`@define` resolves across files (a value declared in one `.vcss` is visible in another built
alongside it). Only one shipped file uses `@import`; the normal composition mechanism used by
Valve is `<styles><include>` in XML.

## 7. What you do not inherit

- `base.vcss` — the only stylesheet the host document includes — is the single rule
  `.WindowRoot{width:100%;height:100%;}`;
- the game HUD stylesheets (`panorama/styles/hud/*.vcss`) are attached to the `CSGOHud`
  tree, not to the custom hud tree; their ids (`#HudTopLeft`, ...) and classes
  (`.HudBlur`, ...) are unavailable to you;
- the engine attaches **no** classes of its own to custom hud panels. Every class that exists
  is one you declared and the server toggles via `SetHasClass`.

Hence the working pattern: **model states as CSS classes**, have the server switch them, and
let `transition` animate the change.
