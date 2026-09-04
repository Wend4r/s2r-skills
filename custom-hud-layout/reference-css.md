# Custom hud VCSS reference

Panorama CSS is **not** web CSS. There is no `display`, `flex`, `grid`, `float`,
`left/top/right/bottom`, `gap`, `filter`, `justify-content` or `align-items`. Layout is flow +
alignment + sizing.

The property set is fixed at compile time in a single global registration table inside the
client — 140 properties, listed exhaustively in the appendix. It cannot be extended by content,
and the engine asserts if it ever overflows (`Need to increase size of static
g_StylePropertyRegistrations (MAX_PANORAMA_STYLE_SYMBOLS) before registering more styles`).

Descriptions marked *(engine doc)* quote the engine's own help text for that property — the
same text the Panorama style debugger shows. Properties marked *(no shipped doc)* carry no
help text; their behaviour is described from the parsers instead. In the grouped tables the
Description column mixes both: quoted engine text where it exists, author-derived value sets
where it does not.

Two grammar rules govern every value literal below: a length needs its `px` or `%` suffix — a
bare number is invalid except the literal `0` — and an angle without `deg` is rejected. The
full unit table is in §13.

## 1. Attaching a stylesheet

A custom hud has no inline styles. The only mechanism is:

```xml
<styles>
	<include src="s2r://panorama/styles/my_hud.vcss" />
</styles>
```

Only a `.vcss` resource may be referenced; the path is arbitrary. The `.vcss` itself is
**not inspected** by the validator — the full Panorama CSS language is available inside it.

Selectors: `#id`, `.class`, panel type (`Panel`, `Label`, `Image`, `Button`), plus
`CSGOCustomHudLayoutRoot` — the engine-created wrapper, which you can select but cannot author
(see reference-xml.md §6) — pseudo-classes and descendant nesting. Only `:hover` and `:active`
were confirmed in shipped stylesheets; the full pseudo-class table was not extracted.

## 2. The layout model

- `flow-children` on the *parent* decides the direction children stack;
- a child opts out with `ignore-parent-flow: true`, after which `position` and `align` place
  it freely;
- `width` / `height` default to `fit-children`, so a container shrink-wraps unless told
  otherwise;
- `align` (and its `horizontal-align` / `vertical-align` halves) place a panel inside the
  space it was given.

Your mount point (`CSGOCustomHud`) is full-screen with `flow-children: none`, so **your root
panel must set its own `width`/`height`** and position itself.

## 3. Sizing

### `width` / `height` *(engine doc)*

> Sets the width for this panel. Possible values:
> `"fit-children"` — Panel size is set to the required size of all children (default)
> `<pixels>` — Any fixed pixel value (ex: `"100px"`)
> `<percentage>` — Percentage of parent width (ex: `"100%"`)
> `"fill-parent-flow( <weight> )"` — Fills to remaining parent width. If multiple children are
> set to this value, weight is used to determine final width. For example, if three children
> are set to fill-parent-flow of 1.0 and the parent is 300px wide, each child will be 100px
> wide.
> `"height-percentage( <percentage> )"` — Percentage of the panel's height, which allows you to
> enforce a particular aspect ratio. The height cannot also be width-percentage.

`height` is the same with the axes swapped (`width-percentage( … )` for aspect lock).

Additionally accepted by the parser: `safe-area-inset-left` / `-right` / `-top` / `-bottom`.

Per-property rejections: `width` rejects `width-percentage`, `height` rejects
`height-percentage`, and `max-width` / `max-height` reject `fit-children`.

### `min-width` / `min-height` / `max-width` / `max-height` *(no shipped doc)*

Clamp the computed size; same length grammar as `width`/`height`, minus `fit-children` on the
`max-*` pair.

## 4. Flow and alignment

### `flow-children` *(no shipped doc)*

Direction in which this panel stacks its children. The parser accepts **10** values:

`none`, `down`, `right`, `up`, `left`, `down-wrap`, `right-wrap`, `up-wrap`, `left-wrap`,
`down-wrap-left`

`none` means children do not flow at all — they must position/align themselves. The `*-wrap`
variants wrap onto a new line when the axis fills up. (`up-wrap`, `left-wrap` and
`down-wrap-left` are accepted but appear in no shipped stylesheet.)

### `horizontal-align` / `vertical-align` *(no shipped doc)*

- horizontal: `left`, `center`, `middle`, `right`, `center_nopixelsnap`
- vertical: `top`, `center`, `middle`, `bottom`, `center_nopixelsnap`

`middle` is a **synonym** of `center` (identical enum value), not a distinct alignment.
`center_nopixelsnap` centres without snapping to the pixel grid — useful for smoothly animated
elements that otherwise jitter.

### `align` *(no shipped doc)*

Shorthand requiring exactly **two** tokens, horizontal then vertical: `align: center center;`.
A single-token `align: center;` is **rejected**.

### `ignore-parent-flow` *(no shipped doc)*

Boolean. `true` removes the panel from its parent's flow so `position` / `align` apply.

### `margin` and `padding` *(no shipped doc)*

`margin` (outer) and `padding` (inner) spacing, in `px` or `%`. Shorthands take 1–4 values in
top/right/bottom/left order; longhands are `margin-left/-top/-right/-bottom` and
`padding-left/-top/-right/-bottom`.

## 5. Positioning and stacking

### `position` *(engine doc)*

> Sets the x, y, z position for a panel. Must not be in a flowing layout.
> Example: `position: 3% 20px 0px;`

### `x` / `y` / `z` *(no shipped doc)*

Individual positional offsets — Panorama's replacement for the absent `left`/`top`.

### `layout-position` *(engine doc)*

> Sets how the panel is positioned relative to its parent. `"static"` means position in the
> default way. `"fixed"` means position in the default way, but ignoring the parent's scroll
> offset. Example: `layout-position: fixed;`

### `z-index` *(engine doc)*

> Sets the z-index for a panel; panels will be sorted and painted in order within a parent
> panel. The sorting first sorts by the z-pos computed from position and transforms, then if
> panels have matching zpos, z-index is used. z-index is different from z-pos in that it
> doesn't affect rendering perspective, just paint/hit-test ordering. The default z-index value
> is 0, and any floating point value is accepted.

### `visibility` *(engine doc)*

> Controls if the panel is visible and is included in panel layout.
> `"visible"` — panel is visible and included in layout (default)
> `"collapse"` — panel is invisible and **not** included in layout

### `overflow` *(engine doc)*

> Specifies what to do with contents that overflow the available space for the panel.
> `"squish"` — Children are squished to fit within the panel's bounds if needed (default)
> `"clip"` — Children maintain their desired size but their contents are clipped
> `"scroll"` — Children maintain their desired size and a scrollbar is added to this panel
> `"noclip"` — Children maintain their desired size and content is allowed to overflow
>
> `overflow: squish squish;` squishes in both directions;
> `overflow: squish scroll;` scrolls contents in the Y direction.

One token applies to both axes; an optional second overrides Y.

### `clip` *(engine doc)*

> Specifies a clip region within the panel, where contents will be clipped at render time. This
> clipping has no impact on layout, and is fast and supported for transitions/animations.
> Radial clip mode takes a center point, start angle and angular width of the revealed sector.
> `clip: rect( 10%, 90%, 90%, 10% );`
> `clip: radial( 50% 50%, 0deg, 90deg );`

`clip: radial(...)` is the standard way to build a radial cooldown/progress sweep.

## 6. Color, background, opacity

### `background-color` *(paraphrased; examples from the engine doc)*

Solid color, gradient, or a comma-separated stack of layers.

```css
background-color: #FFFFFFFF;
background-color: gradient( linear, 0% 0%, 0% 100%, from( #fbfbfbff ), to( #c0c0c0c0 ) );
background-color: gradient( linear, 0% 0%, 0% 100%,
							from( #fbfbfbff ), color-stop( 0.3, #ebebebff ), to( #c0c0c0c0 ) );
background-color: gradient( radial, 50% 50%, 0% 0%, 80% 80%,
							from( #00ff00ff ), to( #0000ffff ) );
background-color: #0d1c22ff, gradient( radial, 100% -0%, 100px -40px, 320% 270%,
							from( #3a464bff ), color-stop( 0.23, #0d1c22ff ), to( #0d1c22ff ) );
```

### `background-color-opacity` *(engine doc)*

> Sets the background color opacity for a panel (does nothing on its own, but when merged with
> a full background-color it overrides the opacity). `background-color-opacity: 0.5;`

### `color` *(engine doc)*

> Sets the foreground fill color/gradient/combination for a panel. This color is the color used
> to render **text** within the panel. Accepts the same gradient syntax as `background-color`.

### `background-image` *(engine doc)*

> Comma separated list of images or movies to draw in the background. Can specify `"none"` to
> not draw a background layer. Combined with background-position, background-size,
> background-texture-size and background-repeat values.
> `background-image: url("file://{images}/default.tga"), url("file://{movies}/Background1080p.webm");`

### `background-size` *(engine doc)*

> Sets the horizontal and vertical dimensions used to draw the background image. Can be set in
> pixels, percent, `"contains"` to size down to panel dimensions, or `"auto"` which preserves
> the image aspect ratio. By default `"auto"`, preserving the image's original size. Multiple
> background layers can be specified in a comma separated list.

### `background-position` *(engine doc)*

> Controls the horizontal and vertical placement of the background image, with the format
> `<left|center|right> <horizontal length> <top|center|bottom> <vertical length>`. If length is
> a percent, the specified location within the image is positioned over that same position in
> the background. If pixels, the top-left corner is placed relative to the alignment keywords.
> If 1 value is specified, the other is assumed to be center.

### `background-repeat` *(engine doc)*

> `"repeat"` (default) — repeated until it fills the panel
> `"space"` — repeated as many times as fit without clipping; space added between images
> `"round"` — repeated as many times as fit without clipping; images resized to align to edges
> `"no-repeat"` — not repeated
> Single values `"repeat-x"` / `"repeat-y"` expand to `repeat no-repeat` / `no-repeat repeat`.

### `background-texture-size` *(engine doc)*

> Sets the size used for generating textures from vector graphics (`.svg` files); default `-1`
> takes the size from the svg file. `background-texture-size: 100px 50px;`

### `background-img-opacity` *(engine doc)*

> Sets the Opacity of background-image. `background-img-opacity: 0.5;`

### `opacity` *(engine doc)*

> Sets the opacity or amount of transparency applied to the panel and all its children during
> composition. Default of 1.0 means fully opaque, 0.0 fully transparent.

### `wash-color` *(engine doc)*

> Specifies a 'wash' color — a color blended over the panel and all its children at composition
> time, tinting them. The alpha value determines the intensity of the tinting.
> `wash-color: #39b0d325;`

### `opacity-mask` + `-position` / `-scale` / `-threshold` *(engine doc)*

> Applies an image as an opacity mask that stretches to the panel bounds and fades out its
> content based on the alpha channel. The second float value is an optional opacity for the
> mask itself; the image won't interpolate/cross-fade, but you can animate the opacity to fade
> the mask in/out. `opacity-mask-threshold` lets you specify a threshold and softness
> percentage: below the threshold pixels are fully transparent, above it fully opaque; the
> softness applies a range during which opacity is scaled by the mask alpha.

### `opacity-brush` *(engine doc)*

> Sets an opacity brush to apply to the panel and all its children during composition.
> `opacity-brush: gradient( linear, 0% 0%, 0% 100%, from( #ffffffff ), to( #ffffff00 ) );`

## 7. Filters and compositing

These apply to the panel **and all its children** at composition time.

| Property | Description |
|----------|-----------|
| `saturation` | Default 1.0 = no adjustment, 0.0 = fully desaturated to gray scale, > 1.0 = over-saturation. `saturation: 0.4;` |
| `brightness` | A multiplier on the HSB brightness value. `brightness: 1.5;` |
| `contrast` | `contrast: 1.5;` |
| `hue-rotation` | Default 0.0 = no adjustment, domain in degrees. `hue-rotation: 180deg;` |

### `blur` / `background-blur` / `world-blur` *(engine doc)*

> Sets the amount of blur to apply … Default is no blur; for now Gaussian is the only blur type
> and takes a horizontal standard deviation, vertical standard deviation, and number of passes.
> Good std deviation values are around 0–10; if 10 is still not intense enough consider more
> passes, but **more than one pass is bad for perf**. As shorthand you can specify just one
> value, used for both directions with 1 pass.
> `blur: gaussian( 2.5 );`  `blur: gaussian( 6, 6, 1 );`

- `blur` — blurs this panel and its children;
- `background-blur` — blurs **what is behind** the panel (frosted-glass effect);
- `world-blur` — blurs the world/backbuffer before drawing; also accepts
  `mipmapgaussian( 6, 6, 4 )`, where each pass is preceded by a quarter-area downsample.

### `-s2-mix-blend-mode` *(engine doc)*

> Controls blending mode for the panel. See CSS mix-blend-mode docs on web, except `normal` for
> us is with alpha blending.
> Values: `normal`, `multiply`, `screen`, `additive`, `opaque`, `overlay`, `hardlight`,
> `linearburn`, `darken`, `lighten`, `colordodge`, `colorburn`, `hue`.

### `texture-sampling` *(engine doc)*

> Controls texture sampling mode for the panel. Set to `alpha-only` to use the texture's alpha
> data across all 3 color channels, or `point` for point sampling.
> Values: `normal`, `alpha-only`, `point`.

## 8. Text

Applies to `Label`, and to text rendered inside any panel.

| Property | Description |
|----------|-----------|
| `font-family` | The font face to use. `font-family: Arial;` / `font-family: "Comic Sans MS";` |
| `font-size` | Target font face height in pixels. The engine's own example omits the unit (`font-size: 12;`), but the parser requires `font-size: 12px;` — see §13 |
| `font-style` | `normal`, `italic` |
| `font-weight` | `light`, `thin`, `normal`, `medium`, `bold`, `black` |
| `font-stretch` | `normal`, `condensed`, `expanded` |
| `font` | Shorthand over the family/size/style/weight group |
| `text-align` | `left` (default), `right`, `center`, `justify`, `justify-letter-spacing` |
| `text-transform` | `none` (default), `uppercase`, `lowercase` |
| `text-decoration` | `none` (default), `underline`, `line-through` |
| `text-decoration-style` | (values not extracted) |
| `white-space` | `normal` wraps on whitespace; `nowrap` does no wrapping at all |
| `letter-spacing` | `normal` (no manual spacing) or `<pixels>` |
| `paragraph-spacing` | Only affects multiple line breaks in a row. `normal` defaults to line height, or `<pixels>` |

### `line-height` *(engine doc)*

> Specifies the line height (distance between top edge of line above and line below) to use for
> text. By default `normal`, and a value matching the font-size reasonably is used
> automatically. Unlike the web, we don't have a weird CSS inheritance problem with the 120% vs
> 1.2 styles.
> `line-height: normal | 20px | 1.2 | 120%;`

### `text-overflow` *(engine doc)*

> Controls truncation of text that doesn't fit in a panel. `"clip"` means to simply truncate
> (on char boundaries), `"ellipsis"` means to end with '…', and `"shrink"` means lower the font
> size to fit. `"noclip"` allows the text to overflow based on the `overflow` style.
> `"shrink min( 10px )"` won't shrink beyond a minimum font size, clipping the overflow.
> `"shrink min( 10px ) ellipsis"` is similar but will ellipsis the overflow.
> **We default to ellipsis, which is contrary to the normal CSS spec.**

### `text-shadow` *(engine doc)*

> Specifies text shadows. The shadow shape will match the text the panel can generate, and this
> is only meaningful for labels. Syntax takes horizontal offset pixels, vertical offset pixels,
> blur radius pixels, strength, and then shadow color.
> `text-shadow: 2px 2px 8px 3.0 #333333b0;`

## 9. Borders, shadows, border-image

### `border` and longhands *(engine doc)*

> `border-right: 2px solid #111111FF;` — shorthand for one edge: width, style, and color.
> Supported styles are `solid`, `dashed`, `none`.
>
> `border-color` — if a single color value is set it applies to all sides; if 2 are set the
> first is top/bottom and the second left/right; if all four are set they are top, right,
> bottom, left in order.

Longhands: `border-top/-right/-bottom/-left`, `border-width` + four per-edge widths,
`border-style` + four per-edge styles, `border-color` + four per-edge colors.

### `border-radius` and per-corner radii *(engine doc)*

> Rounds off corners of the panel, adjusting the border to smoothly round and also clipping
> background image/color and contents to the specified elliptical or circular values. In the
> shorthand you may specify a single value for all radii, or horizontal / vertical separated by
> `/`. Per-corner properties take 1 or 2 values in px or %: the first is the horizontal radius
> for an elliptical corner, the second the vertical; if only one is specified the corner is
> circular.

Corners: `border-top-left-radius`, `border-top-right-radius`, `border-bottom-right-radius`,
`border-bottom-left-radius`.

### `border-brush` *(engine doc)*

> **EXPERIMENTAL:** Sets a more-complex brush for the entire border paint area (ignores normal
> border color). Accepts the gradient syntax.

### `box-shadow` *(engine doc)*

> Specifies shadows for boxes, or inset shadows/glows. The shadow shape will match the border
> box for the panel, so use `border-radius` to affect rounding. Syntax takes optional shape
> `inset`, `fill`, or `hollow`, then color, horizontal offset pixels, vertical offset pixels,
> blur radius pixels, and spread distance in pixels. `inset` means an inner shadow/glow, `fill`
> is a shadow behind the entire box, `hollow` means clipping it to outside the border area only
> (before offset). **A negative blur radius** gives a hard-edged look to the shadow, effectively
> a rounded outline of the same size as the blur.

### `img-shadow` *(engine doc)*

> Specifies image shadows. The shadow shape will match the image the panel can generate, and
> this is only meaningful for images. Syntax takes horizontal offset pixels, vertical offset
> pixels, blur radius pixels, strength, shadow color and then an optional texture sample mode
> (`alpha-only`, `legacy`, or `point`).
> `img-shadow: 2px 2px 8px 3.0 #333333b0 alpha-only;`

### `border-image` family *(engine doc)*

> Shorthand syntax: `<border-image-source> || <border-image-slice> [ / <border-image-width>?
> [ / <border-image-outset> ]? ]? || <border-image-repeat>`.
> `border-image: url( "file://message_border.png" ) 25% repeat;`
>
> `border-image-slice` — insets for top, right, bottom, left slice offsets used to cut the
> source image into 9 regions. The `fill` keyword may optionally appear before or after the
> length values and specifies to draw the middle region as a fill for the body background of
> the panel; without it the middle region will not be drawn.
>
> `border-image-outset` — the amount by which the border image should draw outside of the
> normal content/border box, allowing it to extend into the margin area and outside the panel's
> bounds. May still be clipped by a parent whose bounds are too close.
>
> `border-image-repeat` — how the top/right/bottom/left/middle images of the 9-slice regions are
> stretched to fit: `stretch`, `repeat`, `round` (tile, but scale so a whole number of tiles is
> used with no partial tile at the edge), or `space` (tile, but add padding).

Also: `border-image-source`, `border-image-width`.

## 10. Transforms

### `transform` *(engine doc)*

> Sets the transforms to apply to the panel in 2d or 3d space. You can combine various
> transforms (comma separated) and they will be applied in order to create a 4x4 3d transform
> matrix. The possible operations are: `translate3d( x, y, z )`, `translatex( x )`,
> `translatey( y )`, `translatez( z )`, `scale3d( x, y, z )`, `rotate3d( x, y, z )`,
> `rotatex( x )`, `rotatey( y )`, `rotatez( z )`.
> `transform: translate3d( -100px, -100px, 0px );`
> `transform: rotateZ( -32deg ) rotateX( 30deg ) translate3d( 125px, 520px, 230px );`

### `transform-origin` *(engine doc)*

> Sets the transform origin about which transforms will be applied. Default is `50% 50%` on the
> panel so a rotation/scale is centered.

### `pre-transform-rotate2d` / `pre-transform-scale2d` *(engine doc)*

> Sets 2-dimensional rotation degrees (respectively scale) that apply to the quad for this
> panel **prior to** 3-dimensional transforms. This applies without perspective and leaves the
> panel centered at the same spot as it started.
> `pre-transform-rotate2d: 45deg;`

Use these for a spinner or a pulse that must not interact with 3D perspective.

### `perspective` / `perspective-origin` *(engine doc)*

> `perspective` sets the perspective depth space available for children of the panel. Default of
> 1000 would mean that children at 1000px zpos are right at the viewer's eye; -1000px are just
> out of view distance faded to nothing.
> `perspective-origin` sets the perspective origin used when transforming children of this
> panel — think of it as the camera x/y position relative to the panel.

### `ui-scale`, `ui-scale-x/-y/-z` *(engine doc)*

> Specifies a scale to apply to this panel's layout and all descendants. This scale happens at
> the **layout level rather than the bitmap level**, so things like text will increase their
> font size rather than just bitmap scaling.
> `ui-scale: 150%;` — 150% for X, Y and Z
> `ui-scale: 50% 100% 150%;`

## 11. Transitions and animations

### `transition` *(engine doc)*

> Specifies which properties should transition smoothly to new values if a class/pseudo class
> changes the styles. Also specifies duration, timing function, and delay. Valid timing
> functions are: ease, ease-in, ease-out, ease-in-out, linear.
> `transition: position 2.0s ease-in-out 0.0s, perspective-origin 1.2s ease-in-out 0.8s;`

Longhands each accept a comma-separated list applied to the properties named in
`transition-property` in order; a single value applies to all of them.

| Property | Description |
|----------|-----------|
| `transition-property` | Which properties transition. `transition: position, transform, background-color;` |
| `transition-duration` | Duration in seconds. `transition-duration: 2.0s, 1.2s;` |
| `transition-timing-function` | `ease`, `ease-in`, `ease-out`, `ease-in-out`, `linear`, and `cubic-bezier( 0.785, 0.385, 0.555, 1.505 )` |
| `transition-delay` | Delay in seconds. `transition-delay: 0.0s, 1.2s;` |
| `transition-high-framerate` | Request higher framerate during this transition, if we have control. `true` / `false` |
| `transition-frame-time` | Fixed time between frames, to simulate a lower framerate for stylistic reasons. Default `0s`. `transition-frame-time: 0.2s;` |

### `animation` family *(no shipped doc)*

`animation`, `animation-name`, `animation-duration`, `animation-timing-function`,
`animation-iteration-count`, `animation-direction`, `animation-delay`, `animation-fill-mode`,
`animation-frame-time`.

`@keyframes` blocks are a first-class part of the language — the compiled stylesheet AST has
dedicated `KEYFRAMES` and `KEYFRAME_SELECTOR` node kinds. The shorthand parser references
`none` and `infinite`, and shares its timing-function set with `transition-timing-function`.
The exact selector grammar inside a `@keyframes` block was not extracted.

## 12. Interaction, tooltips, sound

### `cursor` *(engine doc)*

> Sets the mouse cursor used on hover for this panel. Default is to inherit.
> `cursor: text;` `cursor: crosshair;`

Only meaningful while input capture is enabled for that player.

### `tooltip-position` / `context-menu-position` families *(engine doc)*

> Specifies where to position a tooltip relative to this panel. Valid options include `left`,
> `top`, `right`, and `bottom`. List up to 4 positions to determine the order that positions are
> tried if the tooltip doesn't fully fit on screen. Default is `right left bottom top`.

Also `tooltip-body-position`, `tooltip-arrow-position`, and the three `context-menu-*`
equivalents. Both families need a tooltip or context menu to exist, which a custom hud cannot
create — listed for completeness.

### `sound` / `sound-out` *(engine doc)*

> `sound` — Specifies a sound name to play when this selector is **applied**. `sound: "whoosh_in";`
> `sound-out` — Specifies a sound name to play when this selector is **removed**. `sound-out: "whoosh_out";`

These fire on selector application, so a class toggled by the server plays a sound with no
scripting at all.

## 13. Units

| Unit | Used for | Notes |
|------|----------|-------|
| `px` | lengths | the parser requires the `px` suffix; a bare number is **invalid** (except the literal `0`) |
| `%`  | lengths | `%` suffix |
| `s`  | time | `transition`, `animation` |
| `ms` | time | accepted by the parser (divided by 1000); unused in shipped content |
| `deg`| angles | mandatory — an angle without `deg` is rejected |

`vw`, `vh`, `em`, `rem`, `dp`, `turn` and `grad` **do not exist** — not in the parsers, and not
in any of the ~225 stylesheets shipped with the game.

## 14. Preprocessor

```css
@import url("s2r://panorama/styles/other.vcss_c");

@define my-accent: #eabe54;
.Label { color: my-accent; }        /* the name is used bare as a value */
```

`@define` resolves across files (a value declared in one `.vcss` is visible in another built
alongside it). Only one shipped file uses `@import`; the normal composition mechanism used by
Valve is `<styles><include>` in XML.

## 15. What you do not inherit

- the host document includes only `base.vcss`, which grants you nothing — see
  reference-xml.md §6 for what it actually contains;
- the game HUD stylesheets (`panorama/styles/hud/*.vcss`) are attached to the `CSGOHud` tree,
  not the custom hud tree; their ids (`#HudTopLeft`, …) and classes (`.HudBlur`, …) are
  unavailable to you;
- the engine attaches **no** classes of its own to custom hud panels. Every class that exists is
  one you declared and the server toggles via `SetHasClass`.

Hence the working pattern: **model states as CSS classes**, have the server switch them, and let
`transition` animate the change.

## Appendix — all 140 properties in registration order

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
