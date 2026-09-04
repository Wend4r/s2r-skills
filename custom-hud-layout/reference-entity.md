# The `custom_hud_layout` entity and its server API

## 1. Registration and keyvalues

| | |
|---|---|
| Classname (Hammer / `CreateEntityByName`) | `custom_hud_layout` |
| C++ / schema class | `CCSCustomHudLayout` |
| Base class | `CBaseEntity` |
| JS wrapper class | `CustomHudLayout` (derives from `Entity`) |

### Exactly one keyvalue of its own

| Keyvalue | Field | Type |
|----------|-------|------|
| `layout` | `m_strLayout` | `CUtlSymbolLarge` |

All other keyvalues (`targetname`, `origin`, ...) and IO (`Kill`, `AddOutput`, `FireUser1`, ...)
are inherited from `CBaseEntity`.

> **The entity declares ZERO Hammer inputs and ZERO Hammer outputs of its own.**
> A pure Hammer map with no scripting can only set `layout` at spawn time.
> Everything else must be driven from server-side JS.

## 2. Data model

### `CCSCustomHudLayout`

| Field | Type |
|-------|------|
| `m_strLayout` | `CUtlSymbolLarge` |
| `m_vecPlayerLayoutStates` | `CUtlVectorEmbeddedNetworkVar<CCSCustomHudLayoutState>` |
| `m_globalLayoutState` | `CCSCustomHudLayoutState` |
| `m_vecPanelIds` | `CNetworkUtlVectorBase<CUtlString>` |
| `m_vecClassNames` | `CNetworkUtlVectorBase<CUtlString>` |
| `m_vecDialogVariableNames` | `CNetworkUtlVectorBase<CUtlString>` |

Three additional non-networked hash tables (string → index) back the interning described below.

### `CCSCustomHudLayoutState`

| Field | Type |
|-------|------|
| `m_playerSlot` | `CPlayerSlot` (int32) |
| `m_bInputCaptureEnabled` | `bool` |
| `m_vecHasClasses` | `CNetworkUtlVectorBase<HUDPanelHasClass_t>` |
| `m_vecDialogVariableStrings` | `CNetworkUtlVectorBase<HUDPanelDialogVariableString_t>` |

The constructor pre-populates one state **per player slot**; the vector index equals the
`CPlayerSlot`, and `m_playerSlot` is set to that index. `m_globalLayoutState` uses
`m_playerSlot = -1`.

### Payload structs

`HUDPanelHasClass_t`:

| Field | Type |
|-------|------|
| `m_nPanelIdIndex` | `uint16` |
| `m_nClassNameIndex` | `uint16` |
| `m_eClassStatus` | `EHudPanelClassStatus_t` |

`EHudPanelClassStatus_t`: `k_eHudPanelClassStatus_Undefined = -1`,
`k_eHudPanelClassStatus_DoesNotHaveClass = 0`, `k_eHudPanelClassStatus_HasClass = 1`.

`HUDPanelDialogVariableString_t`:

| Field | Type |
|-------|------|
| `m_nPanelIdIndex` | `uint16` |
| `m_nDialogVariableIndex` | `uint16` |
| `m_sValue` | `CUtlString` |
| `m_bIsSet` | `bool` |

### String interning and limits

Panel ids, class names and dialog-variable names are **interned** into three networked
vectors; the payload structs reference them by `uint16` index rather than by string.

| Vector | Cap | Overflow message |
|--------|-----|------------------|
| `m_vecPanelIds` | 1024 | `The maximum number of panel ids has been reached, no more can be referenced.` |
| `m_vecClassNames` | 1024 | `The maximum number of class names has been reached, no more can be referenced.` |
| `m_vecDialogVariableNames` | 1024 | `The maximum number of dialog variables has been reached, no more can be referenced.` |

The client resolves a panel by its string id via `FindChildTraverse`, so **every panel you
intend to drive from the server must carry an `id` in the XML**.

## 3. Server-side JS API

The entity object is exposed to server-side JS as class `CustomHudLayout`:

| Method | Signature | Scope |
|--------|-----------|-------|
| `SetHasClass` | `(panelId: string, className: string, hasClass?: boolean)` | all players |
| `SetHasClassForPlayer` | `(playerSlot: number, panelId: string, className: string, hasClass?: boolean)` | one slot |
| `SetDialogVariableString` | `(panelId: string, variableName: string, value?: string)` | all players |
| `SetDialogVariableStringForPlayer` | `(playerSlot: number, panelId: string, variableName: string, value?: string)` | one slot |
| `SetInputCaptureEnabled` | `(playerSlot: number, enabled: boolean)` | **per-slot only** |
| `IsInputCaptureEnabled` | `(playerSlot: number)` | **per-slot only** |

Omitting the optional trailing argument:

- on `SetHasClass*` — resets the class to `Undefined` / `DoesNotHaveClass`;
- on `SetDialogVariableString*` — unsets the variable (`m_bIsSet = false`).

There is no global variant of `SetInputCaptureEnabled`; input capture is per player.

### Dialog variables in markup

The value is substituted into a `Label`'s text. Panorama spells a dialog variable as
`{s:name}` inside `text`:

```xml
<Label id="Timer" text="Time left: {s:value}" />
```

```js
hud.SetDialogVariableString("Timer", "value", "01:23");
```

## 4. The click protocol end to end

```
client: panel clicked
   |  user message CS_UM_CustomHudClicked (id 390)
   |  CCSUsrMsg_CustomHudClicked {
   |      optional uint32 custom_hud_layout = 1 [default = 16777215];  // CEHandleNetworkableInt
   |      optional string button_id         = 2;
   |  }
   v
server: user-message handler
   |- decodes the entity handle (index and serial packed into the uint32)
   |- dynamic_cast to CCSCustomHudLayout (silently dropped otherwise)
   \- dispatches OnCustomHudClicked to every registered script
        payload { player: CSPlayerController, layout: CustomHudLayout, buttonId: string }
```

`OnCustomHudClicked` is a **script event**, not a Hammer output. There is no activator/caller
in entity-IO terms.

> **Security:** the server does **not** re-check `m_bInputCaptureEnabled` when handling the
> message. The gate is client-side only, so a modified client can send any `button_id` for any
> `custom_hud_layout` entity. Validate `buttonId` and the player's authority server-side.

## 5. Client-side classes

- `CCSGO_CustomHud` — singleton panel (`CUI_Root` → `panorama::CPanel2D`), XML tag
  `<CSGOCustomHud>`;
- `CCSGO_CustomHudLayoutRoot` — XML tag `<CSGOCustomHudLayoutRoot>`, stores the entity handle;
  created only from C++, one per entity, **with no id and no class**;
- neither adds any XML attribute or JS method of its own — they do not override the property
  template, so their surface is plain `CPanel2D`;
- `CCSGO_CustomHud` subscribes to exactly two events: `UIGameShutdown` and
  `CSGOHudSafeZoneChanged` (the latter converts the `safezonex` / `safezoney` convars into
  percentage margins plus width/height on itself);
- from `CUI_Root` it inherits automatic aspect-ratio class assignment (16:9, 17:9, 16:10, 4:3,
  21:9) and ui-scale updates — applied to the root panel, not to your markup.
