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

## Observations about the shipped assets

- Across the whole game content tree, the string `CustomHud` occurs **only** in
  `panorama/layout/hud/customhuds.vxml_c`; `CCSCustomHudLayout` occurs nowhere.
  Valve ships the mount point but **no example** of a custom hud.
- `panorama/styles/base.vcss` — the only stylesheet the host document includes — contains
  exactly one rule: `.WindowRoot{width: 100%;height: 100%;}`.
- Across the ~225 shipped stylesheets, only the units `px`, `%`, `s` and `deg` appear.

## Verification status

This material came from a multi-agent analysis: 8 analysis dimensions plus 40 independent
adversarial verifications of individual claims (12 produced substantive corrections, which are
folded in here). Not completed, due to an account session limit: the 4 completeness critics
(xml, css, workflow, contradictions) and part of the verification for the `netvars-state`,
`input-click`, `js-api` and `assets` dimensions.

Open questions:

1. **How a layout is actually shipped** (addon / workshop / map vpk) and which convars affect
   it — this dimension was not finished. `m_strLayout` is set by the server and resolved by
   the client resource resolver; the validator enforces no path prefix, but the resource-name
   normaliser rewrites the name before the extension is compared.
2. **`REFERENCE_PASSTHROUGH` is denied**, yet the walker never visits attribute values — so in
   practice the denial applies to references reachable from `ROOT` / `STYLES` / `INCLUDE`, not
   to `src` on an `<Image>`. Not confirmed dynamically.
3. The full set of client-side effects of `m_bInputCaptureEnabled = true` (cursor, focus
   capture, movement blocking) needs a dedicated pass.
