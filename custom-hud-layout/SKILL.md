---
name: custom-hud-layout
description: Build a custom HUD in CS2 with the custom_hud_layout entity (CCSCustomHudLayout) — Panorama XML markup, VCSS styling, server-side JS API, and click handling. Use when authoring a custom HUD for a CS2 map or plugin, when you need to know which tags and attributes a custom hud allows, why a layout fails validation, how to toggle panel classes and text from the server, or how to receive button clicks
---

# custom_hud_layout — custom HUD in CS2

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

## Quick start

**1. Markup** — `content/<addon>/panorama/layout/my_hud.xml`

```xml
<root>
    <styles>
        <include src="s2r://panorama/styles/my_hud.vcss" />
    </styles>

    <Panel id="Root" class="Root">
        <Label id="Timer" class="Timer" text="00:00" />
        <Image id="Logo" class="Logo" src="file://{images}/my_logo.png" texturewidth="128" textureheight="128" />
        <Button id="BtnReady" class="Btn" />
    </Panel>
</root>
```

**2. Styles** — `content/<addon>/panorama/styles/my_hud.vcss` (there are no inline styles at all)

```css
#Root      { width: 100%; height: 100%; flow-children: down; }
.Timer     { font-size: 32px; color: #ffffff; horizontal-align: center; }
.Btn       { width: 160px; height: 40px; background-color: #202020; }
.Btn:hover { background-color: #303030; }

/* state variants the server toggles via SetHasClass */
.Timer.warning { color: #ff4040; transition: color 0.2s ease-in-out 0.0s; }
```

**3. Entity** — in Hammer or via `CreateEntityByName`

| Keyvalue | Value |
|----------|-------|
| `layout` | path to the compiled layout, e.g. `panorama/layout/my_hud.vxml` |

**4. Driving it from the server** (server-side JS, `point_script`) — see [reference-entity.md](reference-entity.md)

```js
const hud = Instance.FindEntityByName("my_hud");   // JS wrapper class: CustomHudLayout

hud.SetDialogVariableString("Timer", "value", "01:23");
hud.SetHasClass("Timer", "warning", true);
hud.SetInputCaptureEnabled(playerSlot, true);      // enable cursor/clicks for that player

Instance.OnCustomHudClicked = (ev) => {
    // ev.player, ev.layout, ev.buttonId
    if (ev.buttonId === "BtnReady") { /* ... */ }
};
```

## Common rejections

| What you want | Reality |
|---------------|---------|
| `<TextEntry>`, `<ProgressBar>`, `<Movie>`, `<ToggleButton>`, any other tag | ❌ only `Panel`, `Label`, `Image`, `Button` |
| `style="..."` on an element | ❌ no panel type has a `style` attribute |
| `onactivate="..."` or any markup event handler | ❌ none; clicks reach the server keyed by `id` |
| `<scripts>`, `<snippets>`, `<snippet>` | ❌ rejected at the AST node-type level |
| `<include>` of another `.xml` | ❌ only a `.vcss` resource may be referenced |
| `dialogvariable="..."` as an attribute | ❌ the server sets variables via `SetDialogVariableString` |
| `hittest` on `<Button>` | ❌ `Button` accepts only `id` and `class` |

Full permitted set: [reference-xml.md](reference-xml.md).

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

On a validation failure the panel is **never created** — a blank screen, no partial render.
Hot-reloading a live layout into an invalid state **destroys** the existing HUD.

## References

| File | Contents |
|------|----------|
| [reference-xml.md](reference-xml.md) | Complete tag/attribute whitelist, AST node types, validator rules |
| [reference-css.md](reference-css.md) | All 140 Panorama CSS properties, units, flow/align value sets, preprocessor |
| [reference-entity.md](reference-entity.md) | Entity schema, server-side JS API, click protocol, limits |
| [reference-internals.md](reference-internals.md) | How this was derived, and the anchors to re-verify it after a game update |
