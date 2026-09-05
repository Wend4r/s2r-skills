# How this was derived, and how to re-verify it

No public documentation for `CCSCustomHudLayout` exists, and the game ships no example custom
hud. Everything in this skill was recovered from the shipped client/server modules and from
the compiled Panorama assets.

This page deliberately contains **no addresses**. Addresses move on every game update; the
identifiers below — log strings, class and field names, method names, enum names — are what
actually survive, so they are the anchors to search on.

## Durable anchors

Search for these strings to re-locate each mechanism after an update.

### The validator

| Anchor string | Leads to |
|---------------|----------|
| `Layout xml did not pass CustomHud validation "%s"` | the layout loader, which calls the validator |
| `Layout xml is an invalid resource name "%s"` | the same loader, resource-name resolution step |
| `Failed to load layout '%s'.` | the validator entry point |
| `Layout contains disallowed node '%s' (type: %d).` | the AST node-kind check |
| `Layout contains disallowed panel type '%s'.` | the panel-type whitelist lookup |
| `Layout contains disallowed attribute %s for panel type '%s'.` | the per-type attribute check |
| `Layout contains reference to disallowed resource type '%s'.` | the `.vcss`-only resource check |
| `Layout is invalid.` | attribute node whose parent is not a panel node |
| `Validation failed for layout '%s'.` | the panel factory's re-validation |

The whitelist initialiser is the function that references, in order, the literals
`id`, `class`, `hittest`, `Panel`, then `… text`, `Label`, then `… src`, `texturewidth`,
`textureheight`, `Image`, then `id`, `class`, `Button`. Finding those four groups gives you
the complete tag/attribute table without needing any address.

### Panel types and the host window

| Anchor string | Meaning |
|---------------|---------|
| `CSGOCustomHud` | XML tag of the mount panel (`CCSGO_CustomHud`) |
| `CSGOCustomHudLayoutRoot` | XML tag of the per-entity wrapper (`CCSGO_CustomHudLayoutRoot`) |
| `CSGOCustomHuds` | name of the top-level Panorama window |
| `file://{resources}/layout/hud/customhuds.xml` | the host layout that window loads |
| `CustomHudLayout` | the JS global registered after successful validation |
| `CPanel2DFactory:  Factory for '%s' already exists!!!!` | the panel-type factory registrar |

To confirm the game still ships no example custom hud, grep the content tree for `CustomHud`:
it should hit exactly one file, `panorama/layout/hud/customhuds.vxml_c`.

### Entity and server API

| Anchor string | Meaning |
|---------------|---------|
| `custom_hud_layout` | entity classname, paired with `CCSCustomHudLayout` in the factory descriptor |
| `CCSCustomHudLayout` | schema class name |
| `CCSCustomHudLayoutState` | per-player / global state schema class |
| `HUDPanelHasClass_t`, `HUDPanelDialogVariableString_t` | payload structs |
| `EHudPanelClassStatus_t` | class-status enum |
| `SetHasClass`, `SetHasClassForPlayer`, `SetDialogVariableString`, `SetDialogVariableStringForPlayer`, `SetInputCaptureEnabled`, `IsInputCaptureEnabled` | the six registered JS methods |
| `OnCustomHudClicked` | the script event, alongside `player` / `layout` / `buttonId` |
| `CCSUsrMsg_CustomHudClicked`, `CS_UM_CustomHudClicked` | the click user message |
| `The maximum number of panel ids has been reached, no more can be referenced.` | the 1024-entry interning cap (and its class-name / dialog-variable siblings) |
| `OnInputCaptureEnabledChanged`, `OnHasClassesChanged`, `OnDialogVariableChanged`, `OnPanelIdsChanged`, `OnClassNamesChanged` | the five networked-field change callbacks the client reacts to |

### CSS

The property table is the single function that references every property name literal in
sequence. Two assertion strings mark it unambiguously:

```
CStyleSymbol must become larger, cannot fit all style properties in uint8 anymore
Need to increase size of static g_StylePropertyRegistrations (MAX_PANORAMA_STYLE_SYMBOLS) before registering more styles, failed on %s
```

Value vocabularies (`flow-children`, `horizontal-align`, `vertical-align`, `overflow`) are
hard-coded in the corresponding parsers — read them there rather than inferring from shipped
stylesheets, which use only a subset.

### AST enums

`EPanelNodeType` and `EStyleNodeType` are schema enums. In the Panorama module they appear as
a descriptor: an array of `{const char* name; int64 value}` pairs. Locate it by searching for
a pointer to the `ROOT` string; two such descriptors exist, distinguished by their second
entry (`STYLES` for the XML enum, `EXPRESSION` for the CSS enum). Reading the descriptor gives
the name↔value mapping directly, which is far more reliable than inferring it by elimination.

## Reading compiled resources

`.vxml_c` / `.vcss_c` are Source 2 resources (`RED2`) whose KV3 data block is LZ4-compressed.

> **A naive `strings` pass yields truncated literals.** LZ4 back-references splice text, so
> `class="WindowRoot"` appears only as `Window`, and `PANEL_ATTRIBUTE_VALUE` as `_VALUE`.
> Always decompress the block (e.g. Python `lz4.block`) before reconstructing markup —
> otherwise the reconstruction will be silently wrong.

In the decompressed KV3 the AST is stored under `m_AST` / `m_pRoot`, with each node carrying
`eType` (the enum name as a **string**), `name`, `vecChildren`, `child` and `sourceLineColumn`.
Reading the node names in document order reconstructs the original XML.

## How far to trust each part

**Independently re-derived and cross-checked.** The tag and attribute whitelist, the AST
node-type mapping, the entity schema (field names, order, per-module sizes and offsets), the
`.vcss`-only resource rule, and the CSS property table. The click message additionally matches
`csgo/cstrike15_usermessages.proto` verbatim, so it is canonical rather than inferred.

**Read from the engine's own text**, so only as good as that text: every property description
marked *(engine doc)* in the CSS reference.

**Derived from control flow, not observed at runtime.** That attribute *values* are never
walked, and therefore that a rejected resource type cannot bite an `<Image src>`. If an
`s2r://` value in `src` is rejected in practice, this is the claim that was wrong.

**Weakest, and most likely to have gaps.** The completeness of the value vocabularies for
properties with no shipped help text (`flow-children`, `align`, `margin`, `padding`, the
`animation` family), and the replication path in the entity reference.

Where the engine's own documentation disagrees with its parsers, the parser wins. The clearest
case is `background-size`: the help text says `contains`, but the value parser accepts `contain`
(with `cover`, `stretch`, `stretchx`, `stretchy`, `none` and the long-form
`stretch-to-*-preserve-aspect` synonyms). A keyword the parser does not know is dropped silently,
so following the help text here would break the rule with no diagnostic.

## Shipping a layout

Authored sources live under `custom_game` subdirectories of an addon's content root. Two shapes
are in use, differing only in whether a further per-addon folder is nested under `custom_game`.

Flat, one layout per workshop addon:

```
<addon>/
└── panorama/
    ├── layout/custom_game/<name>.xml
    └── styles/custom_game/<name>.css
```

Nested, several layouts and their artwork grouped under the addon's own folder:

```
<addon>/
└── panorama/
    ├── layout/custom_game/<addon>/<name>.xml
    ├── styles/custom_game/<addon>/<name>.css
    └── images/
        ├── <addon>/
        └── icons/<addon>/<state>/<size>/
```

The layout and its stylesheet share a base name in both, which is what makes the pairing readable
at a glance. Both nest freely: nothing in the validator enforces the `custom_game` prefix, and
nothing constrains where artwork sits, since a stylesheet's `url()` is not subject to the XML
resource whitelist. `custom_game` is convention, matching Valve's own `custom_game` layouts.

The compiler produces `.vxml_c` / `.vcss_c` / `.vtex_c` alongside the sources. Three names cross a
file boundary, and each is written differently:

| Reference | Written as | Held by |
|-----------|------------|---------|
| layout | resource name of the `.vxml` | the entity's `layout` keyvalue |
| stylesheet | `s2r://…/<name>.vcss_c` — **compiled** spelling | `<include src>` in the layout |
| raster texture | `s2r://…/<name>_png.vtex` — **source** spelling | `url()` or `<Image src>` |
| vector | `s2r://…/<name>.vsvg` — no format infix | `<Image src>` |

Both stylesheet spellings pass, because the extension check lowercases and truncates at the first
`_`. The format infix inside a raster name is the image importer's doing — `logo_small.png`
becomes `logo_small_png.vtex` and `logo_small.webp` becomes `logo_small_webp.vtex`, so the
reference carries its *source* format mid-name. **SVG is the exception**: it compiles to its own
resource type and takes no infix, so `icon.svg` is referenced as `icon.vsvg`. These spellings are
read back out of compiled resources, so they are the compiler's canonical forms rather than
necessarily what the author typed.

> **A hazard when working from decompiled content.** A decompiler writes the *decoded* image back
> out under the resource name, so `logo_webp.vtex` lands on disk as `logo_webp.png`. Recompiling
> that directory as-is produces `logo_webp_png.vtex` — a second infix — and every reference in the
> layouts silently resolves to nothing. Rename the exports back to their true sources
> (`logo_webp.png` → `logo.webp`) before building.

### The rest of the addon

A full addon ships more than layouts and stylesheets, and four of those directories *are* reachable
from a custom hud. Which ones, and by what route:

| Directory | Reachable | How |
|-----------|-----------|-----|
| `panorama/layout/`, `panorama/styles/` | ✅ | the entity's `layout`, then `<include src>` |
| `panorama/images/`, `panorama/materials/` | ✅ | `url()` in a stylesheet, `<Image src>` in a layout |
| `panorama/fonts/` | ✅ | implicitly — a `.ttf` here registers itself, see css.md |
| `panorama/videos/` | ✅ | `background-image: url( "file://{resources}/videos/…" )` |
| `resource/` | ✅ | `text="#Token"` in a `Label` |
| `soundevents/`, `sounds/` | ⚠️ | not from the hud — only from the server, by event name |
| `materials/`, `models/`, `particles/`, `maps/` | ❌ | world content; nothing in a hud can name it |

`materials/` is the one to watch, because there are two of them: `panorama/materials/` is a normal
place to keep hud textures, while a top-level `materials/` is world content and unreachable.

### Localisation files

`resource/<name>_<language>.txt`, one per language, in Valve's KeyValues format — not KV3, and
with no preamble:

```
"lang"
{
	// Comments are legal, but only INSIDE the braces. See the encoding rule below.
	"Language"	"English"
	"Tokens"
	{
		"myhud_title"		"Server rules"
		"myhud_close"		"Close"
	}
}
```

Ship the same key set in every language file and prefix your keys with something specific to the
addon, so they cannot collide with the game's own. Then `text="#myhud_title"` in a layout resolves
per player, client-side, with no server call.

Two rules are worth writing down because both fail quietly:

- **The file must begin with a `"` character** — or a UTF-8 / UTF-16LE byte-order mark. The loader
  sniffs the encoding from the first bytes and accepts nothing else, so a comment on line 1 makes
  the entire file unreadable. Put comments inside the `"lang" {` block, never above it.
- **A token's value is not rescanned for tokens.** There is no composition and no substitution
  into a token — every variant has to be spelled out as its own key. This is the same restriction
  that stops a dialog variable from carrying a `#Token`.

An unknown token renders as its own literal text (`#myhud_title` on screen), which is the symptom
to look for when a file failed to load.

## Open questions, and what would settle each

1. Which convars, if any, affect layout resolution. The packaging half of this is settled
   (see *Shipping a layout* above); what remains is whether anything besides the entity's
   `layout` keyvalue influences which resource is picked.
2. Whether any node kind other than an element, an attribute or a text node can appear in a
   compiled layout at all. The walker accepts a fixed set of node kinds and rejects the rest,
   but a compiled `.vxml_c` produced by the normal toolchain has only ever been seen to contain
   the three. Nothing here depends on the answer; it only bounds how much the node-kind check
   can ever reject.
