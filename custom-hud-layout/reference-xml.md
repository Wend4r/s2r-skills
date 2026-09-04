# Custom hud XML contract

## 1. The four permitted tags

The client builds a whitelist once, on first use: a hash table mapping panel-type name to a
list of allowed attribute names. It has exactly four entries — this is the **entire** markup
vocabulary available to a custom hud.

| Tag | Panorama class | Allowed attributes |
|-----|----------------|--------------------|
| `Panel`  | `panorama::CPanel2D`    | `id`, `class`, `hittest` |
| `Label`  | `panorama::CLabel`      | `id`, `class`, `hittest`, `text` |
| `Image`  | `panorama::CImagePanel` | `id`, `class`, `hittest`, `src`, `texturewidth`, `textureheight` |
| `Button` | (button panel type)     | `id`, `class` |

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
transparent to the mouse, so clicks fall through to whatever is behind it. This matters only
while input capture is enabled for the player.

Note `Button` does **not** accept `hittest` — a button is always hit-testable.

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

### `Image` — `src`, `texturewidth`, `textureheight`

`src` is the image to display. Because attribute *values* are never inspected by the validator
(§3), the string itself is unconstrained — but the resource rules still apply to how it
resolves. Use the non-compiled URL forms:

```xml
<Image id="Logo" class="Logo" src="file://{images}/my_logo.png" />
```

`texturewidth` / `textureheight` are documented in the engine as:

> texturewidth and textureheight can be used to override the size of vector graphics.
> Default value of -1 indicates texture width/height to be determined from source file

So they control the **rasterisation size of vector sources (`.svg`)**, not the on-screen size.
On-screen size is CSS (`width` / `height`). A value of `-1` means "take it from the file".
They accept a pixel value, e.g. `texturewidth="16px" textureheight="-1"`.

> An `<Image>` is not the only way to show a picture: any panel can carry a
> `background-image` in CSS, which additionally gives you `background-size`,
> `background-position` and `background-repeat`.

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

### What no tag has

No `style` (inline CSS is impossible), no event handlers (`onactivate`, `onload`,
`onmouseover`, `onmouseout`, `oncancel`), no `dialogvariable`, `visible`, `enabled`,
`tabindex`, `acceptsinput`, `hittestchildren`, `defaultsrc`, `scaling`, `tooltip`, `args`,
`snippet`, `html`. And no other panel type exists — no `TextEntry`, `ProgressBar`, `Movie`,
`ToggleButton`, `RadioButton`, `DropDown`, `Carousel`, `Slider`, `NumberEntry`.

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

- ✅ `<styles><include src="s2r://<path>.vcss" /></styles>` — any path; no prefix check,
  so an addon may ship its stylesheet anywhere in its content tree;
- ❌ every other resource: `.vjs`, a nested `.vxml`, `.vsnd`, or `.vtex` via `s2r://`.

Rejection → `Layout contains reference to disallowed resource type '%s'.`

> For `<Image>` this does not bite in practice, because attribute values are not walked (§3).
> Prefer `file://{images}/...`, which compiles to a plain string node rather than a resource
> reference.

## 5. Shape of a valid document

```xml
<root>
    <styles>
        <include src="s2r://panorama/styles/my_hud.vcss" />
    </styles>

    <Panel id="Root" class="Root" hittest="false">
        <Label  id="Timer" class="Timer" text="{s:value}" />
        <Image  id="Logo"  class="Logo"  src="file://{images}/logo.png"
                texturewidth="-1" textureheight="-1" />
        <Button id="BtnReady" class="Btn">
            <Label class="BtnCaption" text="Ready" />
        </Button>
    </Panel>
</root>
```

There are **no** limits on depth, node count, attribute count or file size — the walk is
directly recursive and unbounded. A document whose root resolves to nothing validates as a pass.

## 6. The host document you are mounted into

`panorama/layout/hud/customhuds.vxml`:

```xml
<root>
    <styles>
        <include src="s2r://panorama/styles/base.vcss" />
    </styles>
    <Panel class="WindowRoot" hittest="false">
        <CSGOCustomHud id="CustomHud"
                       style="width: 100%; height: 100%; flow-children: none;" />
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
