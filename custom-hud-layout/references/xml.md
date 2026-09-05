# Custom hud XML contract

## Contents

- [1. The four permitted tags](#1-the-four-permitted-tags)
- [2. AST node types](#2-ast-node-types)
- [3. Attribute values are not validated](#3-attribute-values-are-not-validated)
- [4. Resource references — .vcss only](#4-resource-references-vcss-only)
- [5. Shape of a valid document](#5-shape-of-a-valid-document)
- [6. The host document you are mounted into](#6-the-host-document-you-are-mounted-into)
- [7. When validation runs](#7-when-validation-runs)

## 1. The four permitted tags

The client builds a whitelist once, on first use: a hash table mapping panel-type name to a
list of allowed attribute names. It has exactly four entries — this is the **entire** markup
vocabulary available to a custom hud.

The engine calls a tag name a *panel type*; the two words mean the same thing here.

| Tag | Panorama class | Allowed attributes |
|-----|----------------|--------------------|
| `Panel` | `panorama::CPanel2D` | `id`, `class`, `hittest` |
| `Label` | `panorama::CLabel` | `id`, `class`, `hittest`, `text` |
| `Image` | `panorama::CImagePanel` | `id`, `class`, `hittest`, `src`, `texturewidth`, `textureheight` |
| `Button` | `panorama::CButton` | `id`, `class` |

Unknown tag → `Layout contains disallowed panel type '%s'.`
Unknown attribute → `Layout contains disallowed attribute %s for panel type '%s'.`

### Attributes shared by all four

#### `id`

The panel's identifier, unique within the document. It is the **addressing key for everything
the server does**: `SetHasClass(panelId, …)` and `SetDialogVariableString(panelId, …)` resolve
the panel by calling `FindChildTraverse` with this string. A `<Button>`'s `id` is also what
comes back as `buttonId` in the `OnCustomHudClicked` event.

> A panel with no `id` cannot be touched by the server and, if it is a `Button`, cannot be
> identified when clicked. Give an `id` to every panel you intend to drive or read.

#### `class`

Space-separated list of CSS class names. Since there is no `style` attribute, **`class` and
`id` are your only styling hooks**. Classes are also the mechanism for server-driven state:
declare a variant as a class, then have the server toggle it.

#### `hittest`

Whether the panel participates in mouse hit-testing. `hittest="false"` makes the panel
transparent to the mouse, so clicks fall through to whatever is behind it. It only has an
effect while input capture is enabled for the player — see entity.md, *Input capture*.

**The default is `true`**, so hit-testing is what you get by leaving the attribute off; `false` is
the value you write. And `Button` does not accept the attribute at all — it is the only tag that
does not, and a button is always hit-testable.

**Write `hittest="false"` on everything, then take it off the few panels that must catch the
mouse.** That is the discipline working content follows almost without exception — every `Label`,
every `Image`, and all but a handful of `Panel`s. It is worth doing even in a hud that never
enables input capture, because a hud that starts as an overlay tends to grow a menu later, and
retrofitting hit-testing onto markup written without it means auditing every panel.

The panels that keep hit-testing are containers with a job:

```xml
<!-- The pass-through wrapper: never catches anything. -->
<Panel class="screen" hittest="false">

	<!-- A scrim. It is hit-testable so the mouse cannot reach the game behind the modal,
	     and its first child is a childless Button covering the same area, so a click
	     anywhere outside the dialog reports one buttonId and the server closes it. -->
	<Panel id="Scrim" class="scrim">
		<Button id="ScrimDismiss" class="scrim-hit" />

		<!-- The dialog itself goes back to pass-through; its Buttons do the catching. -->
		<Panel id="Dialog" class="dialog" hittest="false">
			...
		</Panel>
	</Panel>
</Panel>
```

A childless `<Button>` is a click region and nothing else — no caption, no artwork, sized
entirely by CSS. It is the standard way to build click-outside-to-dismiss, and the only reason
the scrim `Panel` above it needs `hittest` at all is to stop the mouse falling through to the
world.

### `Label` — `text`

The string the label renders. Text is styled with the `color`, `font-*`, `text-*`,
`line-height`, `letter-spacing`, `paragraph-spacing` and `white-space` properties.

Dialog variables are substituted here. Panorama spells them `{s:name}`:

```xml
<Label id="Timer" class="Timer" text="Time left: {s:value}" />
```

```js
hud.SetDialogVariableString("Timer", "value", "01:23");
```

Until the server sets the variable it is unset (`m_bIsSet = false`) and nothing is
substituted. Rich/inline markup inside a label normally requires the `html` attribute, which
is **not** whitelisted for custom huds — treat `text` as plain text plus dialog variables.

#### Localisation tokens — the other half of `text`

A `text` value beginning with `#` is a **localisation token**, resolved on the client against the
string catalogue for that player's language:

```xml
<Label class="weapon-name" hittest="false" text="#SFUI_WPNHUD_AK47" />
```

This is not a minor alternative to dialog variables — in text-heavy huds it carries the large
majority of all labels, and it does something `{s:…}` cannot: it renders in each player's own
language with no server call at all.

Two catalogues are available and both are free to use.

- **The base game's**, which is enormous — weapon names, item and finish names, equipment
  categories. A custom hud can pull the whole thing without shipping a localisation file of its
  own.
- **Your own**, shipped as `resource/<name>_<language>.txt` in the addon, one file per language
  with identical key sets. Prefix your keys with something specific to the addon so they cannot
  collide with the game's.

> **A dialog variable's value is not rescanned for tokens.** `SetDialogVariableString(id, "v",
> "#SFUI_WPNHUD_AK47")` renders the literal text `#SFUI_WPNHUD_AK47`. Substitution happens once,
> and the result is not resolved again.

That restriction is the whole reason for the idiom below. When the string must be **both
localised and chosen at runtime**, the server cannot send it — so you enumerate every possible
token in the markup and reveal one:

```xml
<!-- Every candidate ships as its own Label, all collapsed by the shared class. -->
<Panel id="WeaponName" class="name-slot" hittest="false">
	<Label class="name opt-ak47" hittest="false" text="#SFUI_WPNHUD_AK47" />
	<Label class="name opt-awp" hittest="false" text="#SFUI_WPNHUD_AWP" />
	<Label class="name opt-deagle" hittest="false" text="#SFUI_WPNHUD_DesertEagle" />
</Panel>
```

```css
/* Collapsed by default; the server adds one selector class to the slot to reveal one. */
.name
{
	visibility: collapse;
}

.pick-ak47 .opt-ak47
{
	visibility: visible;
}
.pick-awp .opt-awp
{
	visibility: visible;
}
.pick-deagle .opt-deagle
{
	visibility: visible;
}
```

Give each candidate its own `id` instead of a class if you would rather spend panel ids than
class names — both work, and the pool budget decides which. See
[patterns.md](patterns.md), *Enumerate what you cannot compute*.

### `Image` — `src`, `texturewidth`, `textureheight`

`src` is the image to display. Attribute *values* are never inspected by the validator (§3), so
the resource whitelist does not reach here — `src` may name any resource type the image loader
can open, and `s2r://` compiled references are the normal spelling:

```xml
<!-- A vector icon. -->
<Image class="icon" hittest="false" src="s2r://panorama/images/ui/icon_check.vsvg" />

<!-- A raster, compiled from a source image. -->
<Image class="banner" hittest="false" src="s2r://panorama/images/ui/banner_webp.vtex" texturewidth="308" textureheight="56" />
```

Three source types are worth knowing.

| Extension | What it is | Notes |
|-----------|-----------|-------|
| `.vsvg` | a compiled SVG | scales cleanly; the natural choice for icons and glyphs |
| `.vtex` | a compiled raster | the name carries the source format mid-name — `logo.png` compiles to `logo_png.vtex`, `logo.webp` to `logo_webp.vtex` |
| the game's own textures | anything under the base game's `panorama/images/…` | reachable by `s2r://` from a custom hud; you do not have to ship a copy |

**`.vsvg` is usually the right default.** A hud is scaled to the player's resolution and ui-scale,
so vector icons stay sharp where a raster does not, and a one-colour glyph plus a `wash-color`
rule replaces a whole set of pre-tinted images.

`texturewidth` / `textureheight` are documented in the engine as:

> texturewidth and textureheight can be used to override the size of vector graphics.
> Default value of -1 indicates texture width/height to be determined from source file

So they set the size of the **texture the vector is rasterised into**, in texels. On-screen size
is CSS (`width` / `height`), and the two are independent — which is the point. Set the texture
larger than the CSS box and the icon supersamples; leave them equal and it does not.

```xml
<!-- Drawn at 18.67px by CSS, rasterised at 64x64 so it stays crisp when the hud scales up. -->
<Image class="icon" hittest="false" src="s2r://panorama/images/ui/icon_check.vsvg" texturewidth="64" textureheight="64" />
```

In practice:

- **set them together or not at all** — one without the other is not a shape that occurs;
- **keep the aspect ratio**, or the raster is generated distorted and CSS cannot undo it;
- **omit them and the source file's own `width`/`height` decide.** That is the tidier route: bake
  the oversize into the SVG (`width="64" height="64" viewBox="0 0 32 32"`) and leave the markup
  alone. Most `<Image>` elements in practice carry no size attributes at all;
- writing `-1` explicitly is legal but pointless — omitting the attribute means the same thing.

They are not tied to the file type: the same `.vsvg` is routinely written both with and without
them in one hud, depending on whether that particular use needs the supersample.

> **Keep arcs out of your SVGs.** Flatten every `A`/`a` command to cubic béziers before shipping —
> most vector editors export that way already. It costs nothing, and it removes a whole class of
> "renders differently here than in the browser" surprises.

> An `<Image>` is not the only way to show a picture: any panel can carry a
> `background-image` in CSS, which additionally gives you `background-size`,
> `background-position` and `background-repeat`. Reach for `<Image>` when the picture *is* the
> element, and for `background-image` when it decorates a panel that exists anyway — in
> particular when a generated stylesheet supplies the URL for hundreds of variants.

### `Button`

A clickable panel accepting only `id` and `class`. It has **no `text`** — put a nested
`<Label>` inside it for a caption — and **no `hittest`**.

There are no event-handler attributes anywhere in Panorama's custom-hud subset. A click is
delivered to the *server*, not to client-side script:

```xml
<Button id="BtnReady" class="Btn">
	<Label class="BtnCaption" text="Ready" />
</Button>
```

```js
Instance.OnCustomHudClicked = (ev) => {
	if (ev.buttonId === "BtnReady") { /* ... */ }
};
```

Clicks only reach the server while input capture is enabled for that player
(`SetInputCaptureEnabled(playerSlot, true)`).

**A `Button` may contain another `Button`**, and the inner one reports its own `id`. That gives
two concentric click regions out of one visual object — a large target that selects the row, and
a small glyph inside it that does something else:

```xml
<Button id="Slot0" class="slot">
	<Panel class="slot-bg" hittest="false" />
	<Button id="Slot0Remove" class="slot-remove">
		<Image class="glyph" hittest="false" src="s2r://panorama/images/ui/icon_x.vsvg" />
	</Button>
</Button>
```

The inner button wins the click where the two overlap, so size it deliberately. Both ids are
interned, so a grid of 60 rows with an inner button costs 120 panel-id entries, not 60.

### What no tag has

No `style` (inline CSS is impossible), no event handlers (`onactivate`, `onload`,
`onmouseover`, `onmouseout`, `oncancel`), no `dialogvariable`, `visible`, `enabled`,
`tabindex`, `acceptsinput`, `hittestchildren`, `defaultsrc`, `scaling`, `tooltip`, `args`,
`snippet`, `html`. And no other panel type exists — no `TextEntry`, `ProgressBar`, `Movie`,
`ToggleButton`, `RadioButton`, `DropDown`, `Carousel`, `Slider`, `NumberEntry`.

### Panels the engine creates for you

The whitelist governs what you may *write*, not what may exist. A panel styled
`overflow: ... scroll` gets a scrollbar built from panel types no layout may name, and those are
still addressable from your stylesheet — `VerticalScrollBar` as a type selector, `ScrollThumb` as
a class:

```css
.MyGrid VerticalScrollBar .ScrollThumb
{
	min-height: 32px;
	border-radius: 2px;
}
```

This is the only non-whitelisted panel type a custom hud can reach, and it arrives from the engine
rather than from the layout. See [patterns.md](patterns.md).

## 2. AST node types

Panorama compiles XML into an AST whose node kinds are the `EPanelNodeType` enum. The
validator accepts only a subset:

| Value | Name | In a custom hud |
|-------|------|-----------------|
| 0 | `ROOT` | ✅ |
| 1 | `STYLES` | ✅ |
| 2 | `SCRIPT_BODY` | ❌ |
| 3 | `SCRIPTS` | ❌ |
| 4 | `SNIPPETS` | ❌ |
| 5 | `INCLUDE` | ✅ |
| 6 | `SNIPPET` | ❌ |
| 7 | `PANEL` | ✅ |
| 8 | `PANEL_ATTRIBUTE` | ✅ |
| 9 | `PANEL_ATTRIBUTE_VALUE` | ✅ |
| 10 | `REFERENCE_CONTENT` | ✅ |
| 11 | `REFERENCE_COMPILED` | ✅ |
| 12 | `REFERENCE_PASSTHROUGH` | ❌ |

Practical consequence: `<scripts>`, `<snippets>`, `<snippet>` and inline script bodies are
rejected **structurally** — by node kind, not by tag name. No amount of renaming gets around it.

Disallowed kind → `Layout contains disallowed node '%s' (type: %d).`

## 3. Attribute values are not validated

When the walker reaches an attribute node it checks the attribute *name* and returns success
immediately, **without descending into the child node that holds the value**. Consequently the
strings in `class`, `text`, `src` and `id` are unconstrained.

## 4. Resource references — `.vcss` only

A second whitelist holds exactly one allowed resource type: `vcss`. Any node that carries a
compiled-resource reference has its name run through the resource-name normaliser, its
extension lowercased and truncated at the first `_` (so `vcss_c` collapses to `vcss`), and the
result compared against that list.

- ✅ `<styles><include src="s2r://<path>.vcss_c" /></styles>` — any path; no prefix check,
  so an addon may ship its stylesheet anywhere in its content tree. The compiled name
  (`.vcss_c`) is what the compiler emits in `src`; both forms survive the check, since the extension is
  truncated at the first `_`;
- ❌ every other resource: `.vjs`, a nested `.vxml`, `.vsnd`, or `.vtex` via `s2r://`.

**A `<styles>` block may hold several `<include>`s, and the order is the cascade order** — later
sheets win ties. The useful split is a small hand-written sheet that owns behaviour (layout,
transitions, `:hover`) and a large generated sheet that owns data (which image, which colour,
which coordinates):

```xml
<styles>
	<include src="s2r://panorama/styles/custom_game/my_hud.vcss_c" />
	<include src="s2r://panorama/styles/custom_game/my_hud_table.vcss_c" />
</styles>
```

Keep the two sets of properties disjoint and the order stops mattering, which is what you want —
a generated file should never have to know what the authored one already said.

Rejection → `Layout contains reference to disallowed resource type '%s'.`

> This rule belongs to the XML AST and does not reach inside a stylesheet: `url()` in a `.vcss`
> may point at `.vtex` freely, and that is how textures are referenced.
>
> **It does not reach `<Image src>` either.** Attribute values are not walked (§3), so the
> whitelist never sees them, and `<Image src="s2r://….vtex">` and `<Image src="s2r://….vsvg">`
> both validate and load. That follows from the walker's control flow and is confirmed by working
> huds that reference both types from `src` at scale.

## 5. Shape of a valid document

```xml
<root>
	<styles>
		<include src="s2r://panorama/styles/my_hud.vcss" />
	</styles>

	<Panel id="Root" class="Root" hittest="false">
		<Label id="Timer" class="Timer" text="{s:value}" />
		<Image id="Logo" class="Logo" src="file://{images}/logo.png" texturewidth="-1" textureheight="-1" />
		<Button id="BtnReady" class="Btn">
			<Label class="BtnCaption" text="Ready" />
		</Button>
	</Panel>
</root>
```

There are **no** limits on depth, node count, attribute count or file size — the walk is
directly recursive and unbounded. A document whose root resolves to nothing validates as a pass.

> **The outer panel must not carry an `id`.** This is the resource compiler's rule, not the
> validator's, and it fails the build rather than the load:
> `Found root panel with 'id' attribute, which is not permitted` — the loader assigns it.
> Keep the outer panel an anonymous wrapper and give the panel one level in the id the server
> addresses. Valve's own `custom_game` layouts have the same shape.

## 6. The host document you are mounted into

`panorama/layout/hud/customhuds.vxml`:

```xml
<root>
	<styles>
		<include src="s2r://panorama/styles/base.vcss" />
	</styles>
	<Panel class="WindowRoot" hittest="false">
		<CSGOCustomHud id="CustomHud" style="width: 100%; height: 100%; flow-children: none;" />
	</Panel>
</root>
```

- the top-level window is named `CSGOCustomHuds` and loads
  `file://{resources}/layout/hud/customhuds.xml`;
- `base.vcss` is the only stylesheet included, and it consists of exactly one rule:
  `.WindowRoot{width: 100%;height: 100%;}`. You inherit **no** class or token vocabulary;
- the `CSGOCustomHud` mount point is full-screen with `flow-children: none`, so your root
  panel is positioned/aligned rather than flowed, and needs an explicit `width`/`height`
  (the default for both is `fit-children`);
- beneath it the engine creates a `CSGOCustomHudLayoutRoot`, one per entity, **with no `id`
  and no `class`**. It is still usable as a type selector in your stylesheet
  (`CSGOCustomHudLayoutRoot { ... }`).

## 7. When validation runs

Validation is invoked from three places:

1. loading the layout named by the replicated `m_strLayout` field — the main path;
2. the panel factory, re-validating before allocating the panel object;
3. the resource hot-reload handler: valid and no panel yet → create;
   invalid and a panel exists → **destroy it**.

The panel is created and the `CustomHudLayout` JS global registered only after validation
passes. There is no fallback layout — a rejected document yields a blank screen, never a
partial render.
