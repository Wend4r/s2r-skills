---
name: custom-hud-layout
license: MIT
description: Build a custom hud in CS2 with the custom_hud_layout entity (CCSCustomHudLayout) — Panorama XML markup, VCSS styling, server-side JS API, and click handling. Use when authoring a custom hud for a CS2 map or plugin, when you need to know which tags and attributes a custom hud allows, why a layout fails validation, how to toggle panel classes and text from the server, how to receive button clicks, how to give a player a mouse cursor, or how to use animations, custom fonts, SVG icons, localised text or video in a hud
---

# Source 2 | Custom Hud Layout

The `custom_hud_layout` entity (C++ `CCSCustomHudLayout`) lets a map or server-side plugin
present players with a HUD authored in Panorama XML + VCSS.

**Read this first:** a custom hud is *not* an ordinary Panorama document. The loaded XML is
run through a dedicated whitelist validator that permits **only four panel types and a fixed
attribute set**. There is no `style=`, no `<script>`, and no event handlers in markup.
All dynamic behaviour is driven from the server.

## The full pipeline

```
custom_hud_layout (server, keyvalue layout="...")
   │  m_strLayout is replicated to the client
   ▼
CSGOCustomHuds (Panorama window) → layout/hud/customhuds.xml
   └─ <Panel class="WindowRoot" hittest="false">
        └─ <CSGOCustomHud id="CustomHud" style="width:100%; height:100%; flow-children:none;">
             └─ CSGOCustomHudLayoutRoot   ← created from C++, one per entity,
                  └─ YOUR XML                with no id and no class
```

Return path (a click): user message `CS_UM_CustomHudClicked` (390) → server-side JS event
`OnCustomHudClicked` carrying `{ player, layout, buttonId }`.

## Two kinds of hud

Every custom hud is one of two things, and the difference is a single per-player call:

| | **Overlay** — the default | **Cursor** — `SetInputCaptureEnabled(slot, true)` |
|---|---|---|
| What it is | pixels on the screen | a screen the player can use |
| Mouse | none; the crosshair keeps aiming | a cursor appears over the hud |
| Clicks | nothing is clickable | `Button` clicks reach the server |
| `:hover` | never matches | works, and costs the server nothing |
| Build | timers, scoreboards, kill feeds, banners | shops, vote dialogs, pickers, menus |

Capture is **off every time the layout is built**, so it has to be turned on again after the
entity spawns — see [entity.md](references/entity.md), *Input capture*. Within a
captured hud, `hittest="false"` decides which panels the cursor passes through; write it on
everything and take it off the few containers that must catch the mouse.

## What a hud can actually contain

The markup vocabulary is four tags, but what you can put *through* them is wider than it looks:

- **SVG icons** (`.vsvg`) that stay sharp at any ui-scale, tinted at runtime with `wash-color`;
- **your own fonts**, by dropping `.ttf` files into the addon's `panorama/fonts/`;
- **per-player localised text**, via `text="#Token"` resolved client-side in each player's own
  language, drawing on your own strings or the whole base-game catalogue;
- **the base game's artwork**, referenced by `s2r://` path with nothing added to your download;
- **animations and transitions**, including entry/exit pairs driven by class toggles;
- **video**, as a `background-image` layer on an ordinary `Panel`.

## Quick start

A worked example: a **capture-point indicator** — a read-only strip anchored to the bottom of
the screen with a radial progress ring, driven entirely by dialog variables and class toggles.
It needs no input capture, because nothing in it is clickable.

Authored sources live under `panorama/layout/custom_game/` and `panorama/styles/custom_game/`
in the addon's content root; the compiler turns them into `.vxml_c` / `.vcss_c`:

```
panorama/layout/custom_game/capture_point.xml
panorama/styles/custom_game/capture_point.css
```

**1. Markup** — `panorama/layout/custom_game/capture_point.xml`

```xml
<root>
	<styles>
		<include src="s2r://panorama/styles/custom_game/capture_point.vcss_c" />
	</styles>

	<!--
		The outer panel must NOT carry an id — the loader assigns it, and the compiler
		rejects one. Keep it an anonymous full-screen wrapper and put the real root inside.
	-->
	<Panel class="cp-screen" hittest="false">

		<!-- Starts hidden: the entity shows this layout to EVERY player the moment it spawns, so the authored state has to be the empty one. -->
		<Panel id="CaptureRoot" class="cp-root hidden" hittest="false">

			<Panel class="cp-ring">
				<Panel class="cp-ring-track" hittest="false" />
				<!-- The server rewrites cp-fill's class to cp-fill-0 … cp-fill-100;
				     the stylesheet turns each step into a clip: radial() sweep. -->
				<Panel id="CaptureFill" class="cp-ring-fill cp-fill-0" hittest="false" />
				<Label id="CapturePct" class="cp-pct" text="{s:pct}" />
			</Panel>

			<Panel class="cp-copy">
				<Label id="CaptureName" class="cp-name" text="{s:point_name}" />
				<Label id="CaptureHint" class="cp-hint" text="{s:hint}" />
			</Panel>

		</Panel>
	</Panel>
</root>
```

**2. Styles** — `panorama/styles/custom_game/capture_point.css` (there are no inline styles at all)

```css
/* Full-screen wrapper. noclip lets the ring's glow spill past the panel edge instead of
   being cut off; the wrapper itself is never hit-tested. */
.cp-screen
{
	width: 100%;
	height: 100%;
	overflow: noclip;
}

/* The strip itself, parked above the bottom edge. It starts invisible and offset, and the
   server adds "show" to play the reveal — which is why this is an alternative to .hidden,
   not an addition: a collapsed panel is out of layout and has nothing to animate from. */
.cp-root
{
	flow-children: right;
	horizontal-align: center;
	vertical-align: bottom;
	margin-bottom: 96px;
	opacity: 0;
	transform: translateY(20px);
	transition: opacity 0.15s ease-out 0.0s, transform 0.15s ease-out 0.0s;
}

/* The revealed state. Both properties are listed in the transition above, so adding and
   removing the class animates in each direction. */
.cp-root.show
{
	opacity: 1;
	transform: translateY(0px);
}

/* Removes a panel from layout entirely, so neighbours close up rather than leaving a hole.
   Nothing else collapses a panel — opacity: 0 keeps the space reserved. */
.hidden
{
	visibility: collapse;
}

/* Square box the two ring images stack inside. Both fill it, so they line up exactly. */
.cp-ring
{
	width: 56px;
	height: 56px;
}

/* The unfilled ring underneath, dimmed so the fill on top reads as progress. Artwork comes
   from a compiled texture over s2r:// — the standard form. */
.cp-ring-track
{
	width: 100%;
	height: 100%;
	background-image: url( "s2r://panorama/images/custom_game/ring_png.vtex" );
	background-size: contain;
	background-repeat: no-repeat;
	opacity: 0.25;
}

/* The same artwork on top, recoloured by wash-color and revealed a sector at a time by the
   .cp-fill-* rules below. Tinting one grey source beats shipping a texture per team. */
.cp-ring-fill
{
	width: 100%;
	height: 100%;
	background-image: url( "s2r://panorama/images/custom_game/ring_png.vtex" );
	background-size: contain;
	background-repeat: no-repeat;
	wash-color: #4caf6dff;
}

/* One rule per step the server can select, because it can toggle a class but cannot send a
   number. clip is free of layout cost and animates, so more steps are cheap. */
.cp-fill-0
{
	clip: radial( 50% 50%, 0deg, 0deg );
}

.cp-fill-25
{
	clip: radial( 50% 50%, 0deg, 90deg );
}

.cp-fill-50
{
	clip: radial( 50% 50%, 0deg, 180deg );
}

.cp-fill-75
{
	clip: radial( 50% 50%, 0deg, 270deg );
}

.cp-fill-100
{
	clip: radial( 50% 50%, 0deg, 360deg );
}

/* Text column to the right of the ring, centred against it. */
.cp-copy
{
	flow-children: down;
	vertical-align: center;
	margin-left: 12px;
}

/* Point name — the line the player reads first. */
.cp-name
{
	font-size: 20px;
	font-weight: bold;
	color: #ffffffff;
}

/* Supporting line, held back with alpha rather than a second colour. */
.cp-hint
{
	font-size: 14px;
	color: #ffffff99;
}

/* Team accent. The server sets team-ct or team-t on the root, and the descendant selector
   repaints the fill — so team colour and progress stay independent of each other. */
.cp-root.team-ct .cp-ring-fill
{
	wash-color: #56a0ddff;
}

.cp-root.team-t .cp-ring-fill
{
	wash-color: #f0a531ff;
}
```

**3. Entity** — in Hammer or via `CreateEntityByName`

| Keyvalue | Value |
|----------|-------|
| `layout` | `panorama/layout/custom_game/capture_point.vxml` |

**4. Driving it from the server** (server-side JS under `cs_script`) — see [entity.md](references/entity.md)

```js
const hud = Instance.FindEntityByName("capture_hud"); // JS wrapper class: CustomHudLayout

function showCapture(slot, name, pct, team) {
	hud.SetHasClassForPlayer(slot, "CaptureRoot", "show", true);
	hud.SetHasClassForPlayer(slot, "CaptureRoot", "team-" + team, true);

	// swap the step class: clear the old one, set the new one
	hud.SetHasClassForPlayer(slot, "CaptureFill", "cp-fill-" + roundTo25(pct), true);

	hud.SetDialogVariableStringForPlayer(slot, "CapturePct", "pct", pct + "%");
	hud.SetDialogVariableStringForPlayer(slot, "CaptureName", "point_name", name);
	hud.SetDialogVariableStringForPlayer(slot, "CaptureHint", "hint", "Hold the point");
}

function hideCapture(slot) {
	hud.SetHasClassForPlayer(slot, "CaptureRoot", "show", false);
}
```

Because the server can only toggle classes and set strings — it cannot send a number — a
continuous value like a progress ring has to be quantised into a class per step. Five steps are
enough for a capture ring; a smoother bar just needs more rules.

## Common rejections

| What you want | Why it fails |
|---------------|---------|
| `<TextEntry>`, `<ProgressBar>`, `<Movie>`, `<ToggleButton>`, any other tag | only `Panel`, `Label`, `Image`, `Button` — for video use a `background-image` layer instead of `<Movie>` |
| `style="..."` on an element | no panel type has a `style` attribute |
| `onactivate="..."` or any markup event handler | none; clicks reach the server keyed by `id` |
| `<scripts>`, `<snippets>`, `<snippet>` | rejected at the AST node-type level |
| `<include>` of another `.xml` | only a `.vcss` resource may be referenced |
| `dialogvariable="..."` as an attribute | the server sets variables via `SetDialogVariableString` |
| `hittest` on `<Button>` | `Button` accepts only `id` and `class` |

Full permitted set: [xml.md](references/xml.md).

## Debugging

The validator logs to the **`custom_hud`** logging channel at warning severity:

| Message | Cause |
|---------|-------|
| `Layout xml is an invalid resource name "%s"` | `layout` does not resolve to a resource name |
| `Failed to load layout '%s'.` | file missing or not compiled |
| `Layout contains disallowed node '%s' (type: %d).` | `<scripts>`/`<snippets>`/`<snippet>` etc. |
| `Layout contains disallowed panel type '%s'.` | tag outside the four permitted |
| `Layout contains disallowed attribute %s for panel type '%s'.` | attribute not on that tag's list |
| `Layout contains reference to disallowed resource type '%s'.` | `<include>` targets a non-`.vcss` resource |
| `Layout xml did not pass CustomHud validation "%s"` | final rejection |
| `Layout is invalid.` | an attribute node whose parent is not a panel |

On a validation failure the panel is **never created** — a blank screen, no partial render.
Hot-reloading a live layout into an invalid state **destroys** the existing HUD.

## References

| File | Contents |
|------|----------|
| [xml.md](references/xml.md) | Complete tag/attribute whitelist, AST node types, validator rules |
| [css.md](references/css.md) | All 140 Panorama CSS properties, units, flow/align value sets, preprocessor |
| [entity.md](references/entity.md) | Entity schema, server-side JS API, click protocol, limits |
| [patterns.md](references/patterns.md) | Idioms for addressing, server-driven state, lookup tables and the pool budget |
| [internals.md](references/internals.md) | How this was derived, and the anchors to re-verify it after a game update |
