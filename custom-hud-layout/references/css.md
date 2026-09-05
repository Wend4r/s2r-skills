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
(see xml.md §6) — pseudo-classes and descendant nesting. Only `:hover` and `:active`
are confirmed; the full pseudo-class table was not extracted.

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
`down-wrap-left` are accepted by the parser but seldom needed.)

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

In practice it travels with `layout-position: fixed` as a single idiom — the pair is what turns a
panel into a free-floating layer inside its parent:

```css
/* A full-bleed layer: fills the parent, sits under or over its siblings, takes no part in flow. */
.layer
{
	width: 100%;
	height: 100%;
	x: 0px;
	y: 0px;
	layout-position: fixed;
	ignore-parent-flow: true;
}
```

Use it for glows, scrims, overlays and video beds — anything that must cover a parent without
pushing its siblings around. `z-index` then decides the order among them.

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

One token applies to both axes; an optional second overrides Y. So `overflow: clip` clips both
directions, and `overflow: squish scroll` is the everyday scrolling list — squish across, scroll
down. Only the axis you write `scroll` on gets a scrollbar.

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
background-color: gradient( linear, 0% 0%, 0% 100%, from( #fbfbfbff ), color-stop( 0.3, #ebebebff ), to( #c0c0c0c0 ) );
background-color: gradient( radial, 50% 50%, 0% 0%, 80% 80%, from( #00ff00ff ), to( #0000ffff ) );
background-color: #0d1c22ff, gradient( radial, 100% -0%, 100px -40px, 320% 270%, from( #3a464bff ), color-stop( 0.23, #0d1c22ff ), to( #0d1c22ff ) );
```

Three things about gradients that the examples above do not make obvious, and that you will want
the moment you build a coloured glow behind an item.

- **A gradient is a `background-color`, never a `background-image`.** This is the single most
  common thing to get wrong coming from the web. `background-image` takes `url()` and nothing
  else.
- **The centre point may sit outside the panel.** `gradient( radial, 50% -10%, … )` pushes the
  origin above the top edge, so the panel shows only the lower part of the falloff — which is
  how you get a glow that reads as coming from off-panel rather than from the middle.
- **The two radii are independent**, so `46% 46%` is a circle and `64% 55%` an ellipse.

And one about the colour you fade *to*. A gradient that ends transparent should end at the
surrounding colour with a zero alpha, not at `#00000000`:

```css
/* Fades into a #131517 background without greying out on the way. */
background-color: gradient( radial, 50% -10%, 0% 0%, 64% 55%, from( #31290b ), to( #13151700 ) );
```

Interpolation runs through the RGB channels as well as alpha, so fading to transparent *black*
drags every intermediate pixel toward black and leaves a visible dark fringe. Keep the hue and
zero only the alpha.

**Two stacked panels beat one clever gradient.** The usual shape for a soft two-tone glow is an
opaque radial on the panel itself and a transparent-tailed radial on a full-bleed sibling above
it. It costs one extra panel and buys each layer its own `transition-property` list, which a
single comma-stacked `background-color` cannot give you.

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

For artwork you reference **compiled textures over `s2r://`**:

```css
background-image: url( "s2r://panorama/images/my_hud/logo_small_png.vtex" );
```

The `_png.vtex` name is what the texture compiler produces from a PNG source, so the reference
carries the source format in the filename — a `.webp` source compiles to `…_webp.vtex` the same
way. The `.vcss`-only resource rule belongs to the XML validator and does not reach inside a
stylesheet, so CSS may point `url()` at `.vtex` freely. This is by far the most practical way to
get artwork on screen, and it scales to thousands of image references without a single `<Image>`
element.

One `url()` per declaration. The engine's help text advertises a comma-separated list of layers,
but the single-layer form is what working content uses everywhere; if you want two layers, use
two panels — which you will want anyway, because each then gets its own transition.

#### You can point `url()` at the game's own textures

An `s2r://` path is resolved against every mounted content path, not just your addon, so the base
game's entire compiled Panorama texture library is addressable from a custom hud stylesheet at
**zero packaging cost**:

```css
/* Ships with the game. Your addon contains no copy of this file. */
background-image: url( "s2r://panorama/images/econ/default_generated/weapon_ak47_cu_ak47_asiimov_light_png.vtex" );
```

This is what makes catalogue-shaped huds practical at all — a table of two thousand item images
that adds nothing to the download. The same applies to `<Image src>`, to localisation tokens (see
xml.md, *Localisation tokens*) and to sound events.

Two caveats. The paths are not a documented API and Valve can rename or remove them in an update,
so a missing texture is a blank panel with no diagnostic; and you are inheriting whatever art
direction ships, which may change under you. For anything load-bearing, ship your own copy.

#### Playing a video

The `background-image` layer really does accept a **movie**, and this is the only way to get
motion video into a custom hud — there is no `Movie` panel type to write (see
xml.md, *What no tag has*). A movie is referenced over `file://` with a path token,
not over `s2r://`:

```css
.hero-video
{
	width: 100%;
	height: 808px;
	x: 0px;
	y: 0px;

	/* Take it out of flow so it can sit behind its siblings and fill the card. */
	layout-position: fixed;
	ignore-parent-flow: true;
	z-index: 0;

	/* The video is clipped by the panel's radius like any other background layer. */
	border-radius: 32px;

	background-image: url( "file://{resources}/videos/my_hud/intro.webm" );
	background-size: 100% 100%;
	background-repeat: no-repeat;
	background-position: 50% 50%;
}
```

Things worth knowing before you reach for it.

- **`{resources}` resolves to the addon's `panorama/` root**, so the file above ships at
  `<addon>/panorama/videos/my_hud/intro.webm`.
- **The video is not compiled.** It ships as a plain `.webm`; there is no `_webm.vtex` step and
  nothing in the source tree renames it.
- **It composites like any other background layer** — `border-radius` clips it, `z-index` stacks
  siblings over it, `background-size`/`-position`/`-repeat` all apply.
- **It is expensive**, both in download size and in per-frame cost, and it plays whenever the
  panel is in layout. Gate it behind `visibility: collapse` on the states where it should not
  run, rather than merely `opacity: 0`.

### `background-size` *(engine doc, with a correction)*

> Sets the horizontal and vertical dimensions used to draw the background image. Can be set in
> pixels, percent, `"contains"` to size down to panel dimensions, or `"auto"` which preserves
> the image aspect ratio. By default `"auto"`, preserving the image's original size. Multiple
> background layers can be specified in a comma separated list.

> **The help text is wrong about `contains`.** The value parser accepts `contain`, not
> `contains`. Its full keyword set is `auto`, `none`, `contain`, `cover`, `stretch`, `stretchx`,
> `stretchy`, plus the long-form synonyms `stretch-to-fit-preserve-aspect` (= `contain`),
> `stretch-to-cover-preserve-aspect` (= `cover`), and
> `stretch-to-fit-x-preserve-aspect` / `stretch-to-fit-y-preserve-aspect` — alongside lengths in
> `px` or `%`. Use `contain` or `cover`; a value the parser does not know is
> dropped silently and the layer falls back to `auto`.

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
> `blur: gaussian( 2.5 );` `blur: gaussian( 6, 6, 1 );`

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
| `font-family` | The font face to use. `font-family: Arial;` / `font-family: "Comic Sans MS";` — see the note on fallback lists below |
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

### Shipping your own font, and addressing its weights

A custom hud can use a font the game does not have. Drop the `.ttf` into the addon's own
`panorama/fonts/` directory and it is registered — there is **no `@font-face` rule, no manifest
entry, and nothing in the stylesheet points at the file**. Putting it in that directory is the
whole installation step.

Which means the filename is irrelevant, and you must not read a family name out of it. The
matcher reads the font's internal *name table*; a file called `a.notosans.ttf` can perfectly well
be a cut of Manrope, and files named `a`…`g` can be seven cuts of one family. Open the font and
read the name, never guess it from the path.

**`font-weight` can only reach two of the cuts.** A font family in the usual convention has four
standard styles — regular, italic, bold, bold-italic — that share one family name and differ by
subfamily. Every *other* weight ships as its own family whose name has the weight appended, with
subfamily `Regular`:

| Cut | Family name in the font | Reachable as |
|-----|------------------------|--------------|
| Regular | `Manrope` | `font-family: Manrope; font-weight: normal;` |
| Bold | `Manrope` | `font-family: Manrope; font-weight: bold;` |
| Medium | `Manrope Medium` | `font-family: Manrope Medium;` — `font-weight` cannot reach it |
| SemiBold | `Manrope SemiBold` | `font-family: Manrope SemiBold;` |
| Light, ExtraLight, ExtraBold | `Manrope Light`, … | by family name only |

So a design with four weights is addressed by naming the cut, not by asking for a number. Which
is why `font-family` takes a **comma-separated fallback list**, and why a family name containing
spaces does **not** have to be quoted:

```css
/* Ask for the exact cut; fall back to the base family if it is missing. */
font-family: Manrope SemiBold, Manrope;
font-family: Manrope Medium, Manrope;
```

Keep `font-weight` for the regular/bold split of the base family, and reach everything else by
name. Getting this backwards — `font-family: Manrope; font-weight: medium;` — silently renders
regular, because no cut of that name has that weight.

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

The radius is clamped to what the box can take, so **a pill is just an oversized radius**: give a
42 px-tall panel `border-radius: 79px` and both ends round off completely, whatever its width
turns out to be. It beats computing half the height, and it survives the panel being resized.

Per-corner radii are how you round only the bottom of a card whose header is square:

```css
.card-body
{
	border-bottom-left-radius: 32px;
	border-bottom-right-radius: 32px;
}
```

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

The longhand lists pair up **by position**, so a two-property transition with two different
durations is written as two lists of the same length:

```css
/* background-color takes 0.2s, blur takes 0.15s. */
transition-property: background-color, blur;
transition-duration: 0.2s, 0.15s;
```

The first row above is the engine's own help text, and its example is written with the
`transition` shorthand — read it as belonging to `transition-property`. Give one duration and it
applies to every property in the list; give a list and it is consumed in order.

Two habits are worth copying. **Declare the transition on the base rule, never on the `:hover`
rule** — the base rule is where the property list and durations live, and the state rules supply
only the target value, so a hover effect and a server-driven class effect share one declaration.
And **you do not need `transition-timing-function`**: the default is usually right, and working
content omits it far more often than it sets it.

### `animation` family *(no shipped doc)*

Nine registered properties. The engine ships no help text for any of them, so the descriptions
below come from the parsers; the last column is practical guidance.

| Property | Value | In practice |
|----------|-------|-------------|
| `animation` | shorthand for the longhands below; its parser also references `none` and `infinite` | not in practical use — write the longhands |
| `animation-name` | the `@keyframes` name, **unquoted**; `none` disables | required on every animated rule |
| `animation-duration` | seconds, `s` suffix | around `1s` for a loop, `2–3s` for a fall |
| `animation-timing-function` | the `transition-timing-function` set: `ease`, `ease-in`, `ease-out`, `ease-in-out`, `linear`, `cubic-bezier( … )` | `ease-in-out` for loops, `ease-in` for entries |
| `animation-iteration-count` | `infinite`, or a repeat count | `infinite` for idle loops, `1` for one-shots |
| `animation-delay` | seconds, `s` suffix | the stagger axis for a burst |
| `animation-fill-mode` | which frame persists once the run ends | `forwards`, to hold the last frame |
| `animation-direction` | playback direction | value set unverified — see below |
| `animation-frame-time` | fixed time between frames, to simulate a lower framerate | value read across from its documented sibling `transition-frame-time` |

Two value sets are **not** confirmed: `animation-direction`, and everything in
`animation-fill-mode` beyond `forwards`. Neither has engine help text, and neither was read out of
the parser. The web keywords are the obvious candidates, but an unrecognised
keyword is dropped silently, so verify before depending on one.

`@keyframes` blocks are a first-class part of the language — the compiled stylesheet AST has
dedicated `KEYFRAMES` and `KEYFRAME_SELECTOR` node kinds.

**The name is quoted at the definition and unquoted at the reference.** Both forms are required,
and web CSS does neither:

```css
@keyframes 'pulse'
{
	0%
	{
		opacity: 0.15;
	}

	30%
	{
		opacity: 0.90;
	}

	100%
	{
		opacity: 0.15;
	}
}
```

```css
.beacon
{
	animation-name: pulse;
	animation-duration: 1.20s;
	animation-iteration-count: infinite;
	animation-timing-function: ease-in-out;
}
```

Selectors inside the block are percentages — write `0%` and `100%` rather than `from` and `to`.
The shorthand parser also references `none` and `infinite`, and shares its timing-function set
with `transition-timing-function`; the shorthand itself is not worth relying on, so write the
longhands.

#### What a keyframe may animate

Anything the engine can interpolate. Ordered by strength of evidence:

| Property | Evidence |
|----------|----------|
| `opacity` | animated inside shipped `@keyframes` |
| `transform` | animated inside shipped `@keyframes` |
| `wash-color` | transitioned in working content |
| `blur` | transitioned in working content |
| `background-color` | transitioned in working content |
| `color` | transitioned in working content |
| `brightness` | accepted by `transition-property` |
| `position` | accepted by `transition-property` |

A property that can appear in `transition-property` is interpolatable by definition, so it is
equally usable in a keyframe.

`blur` is the interesting one: it is **function-valued**, and it still interpolates —
`blur: gaussian( 1.5, 1.5, 1 )` fades in and out over a transition like any scalar. That makes
it the cheapest way to defocus a whole subtree behind a modal, and the usual way to mark a card
as not-the-one-you-are-pointing-at:

```css
/* Base rule owns the motion. */
.card
{
	transition-property: blur;
	transition-duration: 0.15s;
}

/* State rule owns only the target value. */
.list:hover .card
{
	blur: gaussian( 1.5, 1.5, 1 );
}
```

`wash-color` is the other one worth knowing: it tints a whole `Image` without a second texture,
so a monochrome icon plus a `wash-color` transition replaces an entire hover/active icon set.

The remaining colour-valued properties (`border-color`) and filter-like ones (`saturation`,
`hue-rotation`) are very likely animatable on the same grounds, but are untested.

#### A starting library

All of these are written in the forms the parsers accept: quoted block name, percentage
selectors, `deg` on every angle, `px` on every length. The attention loop `pulse` shown above
belongs to the same set.

```css
/* Reveal. Pair with animation-fill-mode: forwards so the panel keeps its last frame instead of
   snapping back to invisible when the animation ends. */
@keyframes 'fade-in'
{
	0%
	{
		opacity: 0.0;
	}

	100%
	{
		opacity: 1.0;
	}
}

/* Conceal. This only fades the pixels — the panel keeps its space in layout and stays
   hit-testable, so follow it with a visibility: collapse class if that matters. */
@keyframes 'fade-out'
{
	0%
	{
		opacity: 1.0;
	}

	100%
	{
		opacity: 0.0;
	}
}

/* Hard on/off with no interpolation. Two frames a hair apart do the work of a step() function,
   which Panorama does not have. */
@keyframes 'blink'
{
	0%
	{
		opacity: 1.0;
	}

	49.9%
	{
		opacity: 1.0;
	}

	50%
	{
		opacity: 0.0;
	}

	100%
	{
		opacity: 0.0;
	}
}

/* Spinner. The deg suffix is mandatory — an angle without it is rejected outright. */
@keyframes 'spin'
{
	0%
	{
		transform: rotateZ( 0deg );
	}

	100%
	{
		transform: rotateZ( 360deg );
	}
}

/* Enter from below. Moving and fading together reads as one motion rather than two. */
@keyframes 'rise'
{
	0%
	{
		opacity: 0.0;
		transform: translateY( 20px );
	}

	100%
	{
		opacity: 1.0;
		transform: translateY( 0px );
	}
}

/* Enter from the side. Flip the sign for the opposite edge. */
@keyframes 'slide-in-right'
{
	0%
	{
		opacity: 0.0;
		transform: translateX( 40px );
	}

	100%
	{
		opacity: 1.0;
		transform: translateX( 0px );
	}
}

/* Overshoot and settle. The middle frame past 1.0 is what makes it feel physical; without it the
   panel just grows. transform-origin defaults to the centre, so no extra property is needed. */
@keyframes 'pop'
{
	0%
	{
		opacity: 0.0;
		transform: scale3d( 0.80, 0.80, 1.0 );
	}

	60%
	{
		opacity: 1.0;
		transform: scale3d( 1.06, 1.06, 1.0 );
	}

	100%
	{
		opacity: 1.0;
		transform: scale3d( 1.00, 1.00, 1.0 );
	}
}

/* Rejection nudge. Keep the amplitude small and the duration short, or it reads as a bug. */
@keyframes 'shake'
{
	0%
	{
		transform: translateX( 0px );
	}

	25%
	{
		transform: translateX( -6px );
	}

	75%
	{
		transform: translateX( 6px );
	}

	100%
	{
		transform: translateX( 0px );
	}
}

/* Shimmer travelling across a highlight strip. Give the strip's parent overflow: clip clip so the
   sweep is masked at the edges. */
@keyframes 'sweep'
{
	0%
	{
		opacity: 0.0;
		transform: translateX( -120px );
	}

	20%
	{
		opacity: 0.6;
	}

	80%
	{
		opacity: 0.6;
	}

	100%
	{
		opacity: 0.0;
		transform: translateX( 120px );
	}
}

/* Colour flash on a state change, touching neither opacity nor layout — so it composes with
   whatever else the panel is doing. */
@keyframes 'flash'
{
	0%
	{
		background-color: #ffffff00;
	}

	40%
	{
		background-color: #ffffff59;
	}

	100%
	{
		background-color: #ffffff00;
	}
}

/* Fall and tumble — the usual shape for confetti: one-shot, ending
   invisible, with fill-mode forwards so it stays that way. */
@keyframes 'drop'
{
	0%
	{
		opacity: 1.0;
		transform: translateY( -80px ) rotateZ( 0deg );
	}

	100%
	{
		opacity: 0.0;
		transform: translateY( 900px ) rotateZ( 720deg );
	}
}
```

Multiple transform functions are written space-separated, as in `drop` above. The engine's own
help text for `transform` says comma-separated, yet its own second example uses spaces — and
spaces are what work.

#### Starting an animation from the server

There is no "play this animation" call. An animation runs because a rule carrying `animation-name`
began to match, so the server starts one by **adding a class**:

```css
/* Inert until the class arrives. */
.result
{
	opacity: 0;
}

/* Adding "celebrate" starts the run; fill-mode forwards keeps the final frame afterwards, so the
   panel does not snap back to opacity 0 when the animation ends. */
.result.celebrate
{
	animation-name: pop;
	animation-duration: 0.35s;
	animation-iteration-count: 1;
	animation-fill-mode: forwards;
	animation-timing-function: ease-out;
}
```

```js
hud.SetHasClass("ResultPanel", "celebrate", true);
```

Removing the class stops the run and the base rule takes over again. Whether re-adding it replays
a one-shot is untested — see the open questions in
[patterns.md](patterns.md).

#### Animating a panel back out

Removing the class does not play the animation backwards; it cuts. To animate *out* you need a
second keyframe block that is the mirror of the first, and a second class that swaps
`animation-name` on the same panel:

```css
/* Off-screen and out of layout until the server opens it. */
.toast
{
	visibility: collapse;
}

/* Open: put it in layout and drop it in. */
.toast.shown
{
	visibility: visible;
	animation-name: toast-drop;
	animation-duration: 0.3s;
	animation-timing-function: ease-out;
	animation-iteration-count: 1;
}

/* Close: the same panel, still shown, now running the mirror block. fill-mode: forwards is what
   holds it off-screen after the run instead of snapping back into view. */
.toast.shown.closing
{
	animation-name: toast-lift;
	animation-duration: 0.3s;
	animation-timing-function: ease-in;
	animation-iteration-count: 1;
	animation-fill-mode: forwards;
}

@keyframes 'toast-drop'
{
	0%
	{
		transform: translateY( -260px );
	}
	100%
	{
		transform: translateY( 0px );
	}
}

@keyframes 'toast-lift'
{
	0%
	{
		transform: translateY( 0px );
	}
	100%
	{
		transform: translateY( -260px );
	}
}
```

The server drives it in three steps: add `shown` to open, add `closing` when the player dismisses
it, then remove both after the exit has had time to finish.

Three details make this work and are easy to get wrong.

- **Only the exit needs `animation-fill-mode: forwards`.** The entry ends where the base rule
  already puts the panel, so holding its last frame changes nothing; the exit ends somewhere the
  base rule does not, so without `forwards` the panel flicks back on screen for the instant
  before the class is removed.
- **Use `ease-out` in and `ease-in` out.** Both decelerate toward the screen and accelerate away
  from it, which is what reads as physical.
- **The classes stack rather than replace.** `closing` is added *on top of* `shown`, so the
  selector is `.toast.shown.closing` — the panel must stay in layout while it animates out.

#### One name, many panels

A `@keyframes` block is a shared resource. Any number of panels may name it and stay
distinguishable through their own `animation-duration` and `animation-delay`. That is how a
hundred independently-moving particles come out of three keyframe blocks. Put everything shared in
a base class and write only the varying parts per index:

```css
/* Shared by every particle: which animation, how it runs, and where it ends up. */
.particle
{
	opacity: 0;
	animation-name: drop;
	animation-iteration-count: 1;
	animation-fill-mode: forwards;
	animation-timing-function: ease-in;
}

/* Per-index: only the two properties that make this copy different from its neighbours. */
.particle-0
{
	animation-duration: 2.20s;
	animation-delay: 0.00s;
}

.particle-1
{
	animation-duration: 2.40s;
	animation-delay: 0.09s;
}
```

Two ways to stop copies beating together:

- **A one-shot burst** staggers `animation-delay`.
- **An infinite loop** needs no delay at all — unequal `animation-duration` values decorrelate on
  their own, which is enough for a large set of idle animations.

Where several axes vary at once, choose counts that are coprime — delay by `i mod 17`, duration by
`i mod 7`, keyframe by `i mod 3` — and the visible repeat becomes their product rather than the
smallest of them.

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
| `px` | lengths | the parser requires the `px` suffix; a bare number is **invalid** (except the literal `0`). Fractions are fine and are kept — `18.67px`, `197.7778px` are all valid |
| `%` | lengths | `%` suffix |
| `s` | time | `transition`, `animation` |
| `ms` | time | accepted by the parser (divided by 1000); rarely used |
| `deg` | angles | mandatory — an angle without `deg` is rejected |

`vw`, `vh`, `em`, `rem`, `dp`, `turn` and `grad` **do not exist** — not in the parsers, and not
in any of the ~225 stylesheets shipped with the game.

## 14. Preprocessor

```css
@import url("s2r://panorama/styles/other.vcss_c");

@define my-accent: #eabe54;

.Label
{
	color: my-accent;
} /* the name is used bare as a value */
```

`@define` resolves across files (a value declared in one `.vcss` is visible in another built
alongside it). Only one shipped file uses `@import`; the normal composition mechanism used by
Valve is `<styles><include>` in XML.

## 15. What you do not inherit

- the host document includes only `base.vcss`, which grants you nothing — see
  xml.md §6 for what it actually contains;
- the game HUD stylesheets (`panorama/styles/hud/*.vcss`) are attached to the `CSGOHud` tree,
  not the custom hud tree; their ids (`#HudTopLeft`, …) and classes (`.HudBlur`, …) are
  unavailable to you. You can, however, `<include>` a game stylesheet yourself —
  `s2r://panorama/styles/gamestyles.vcss_c` is a `.vcss` like any other, so you can pull it in
  to reuse classes such as `CloseButton`;
- the engine attaches **no** classes of its own to custom hud panels. Every class that exists is
  one you declared and the server toggles via `SetHasClass`.

Hence the working pattern: **model states as CSS classes**, have the server switch them, and let
`transition` animate the change.

Two cautions on that pattern:

- `visibility: collapse` does not mix with a **transition**, but it does mix with an
  **animation**. A transition interpolates from the value the panel had a frame ago, and a
  collapsed panel was not in layout, so there is nothing to start from — the reveal snaps. A
  `@keyframes` block carries its own `0%` frame, so it does not care where the panel came from,
  and the two can go in the same rule:

  ```css
  /* Closed: out of layout entirely. */
  .popup
  {
  	visibility: collapse;
  }

  /* Open: back into layout AND slide in, in one rule. The keyframes supply the start frame. */
  .popup.shown
  {
  	visibility: visible;
  	animation-name: popup-slide-in;
  	animation-duration: 0.28s;
  	animation-timing-function: ease-out;
  	animation-iteration-count: 1;
  }
  ```

  If you would rather use a transition, keep the panel in layout and fade `opacity` instead —
  then `collapse` is only the hard off-switch for states that must not occupy space.
- The server can toggle a class and set a string, but it cannot send a number. A continuous
  value — a progress ring, a meter — has to be quantised into one class per step, with a rule
  per step in the stylesheet.

## Appendix — all 140 properties in registration order

**This is the whole language.** The engine builds its property table once, at startup, from a
fixed list; a name that is not below is not a property, however familiar it looks from the web.
Use this appendix as the yes/no answer when you are unsure something exists — everything in it is
documented above, and everything not in it does not exist.

The order is the engine's own registration order, not alphabetical. That is worth keeping,
because families sit contiguously: finding one property here shows you every longhand and
neighbour it ships with, which is the quickest way to discover that `border-image-slice` or
`opacity-mask-threshold` was available all along. Each entry's position is its internal index,
and those indices are packed into a byte — so the table can never exceed 256 entries, and 140 of
them are in use.

### What is missing, and what to reach for instead

The absences are more surprising than the contents, because Panorama looks like CSS until a
layout property silently does nothing. The common ones, with their equivalents:

| Not a property | Use instead |
|----------------|-------------|
| `display` | `visibility: collapse` — see §5 |
| `left`, `top`, `right`, `bottom`, `inset` | `x`, `y`, `z`, or `position` |
| `float`, `clear` | `flow-children` on the parent |
| the whole `flex-*` / `grid-*` family, `gap`, `order` | `flow-children`, plus `margin` on the children |
| `justify-content`, `align-items` | `horizontal-align` / `vertical-align` on the child, or `align` |
| `background` (the shorthand) | `background-color` and `background-image` separately |
| `overflow-x`, `overflow-y` | the two-token form: `overflow: squish scroll` |
| `filter`, `backdrop-filter` | `blur`, `background-blur`, `saturation`, `brightness`, `contrast`, `hue-rotation` |
| `mix-blend-mode` | `-s2-mix-blend-mode` |
| `clip-path` | `clip`, with `rect()` or `radial()` |
| `mask` | the `opacity-mask` family |
| `object-fit` | `background-size` |
| `box-sizing`, `outline`, `content`, `list-style`, `will-change`, `user-select`, `resize` | nothing; they have no analogue |
| `pointer-events` | not CSS at all — `hittest="false"` in the markup |

`aspect-ratio`, `translate` / `rotate` / `scale` as standalone properties, and logical properties
(`margin-inline`, …) are likewise absent; use `transform` for the second group.

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
