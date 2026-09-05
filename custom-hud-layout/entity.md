# The `custom_hud_layout` entity and its server API

## 1. Registration and keyvalues

- Classname (Hammer / `CreateEntityByName`): `custom_hud_layout`
- C++ / schema class: `CCSCustomHudLayout`
- Base class: `CBaseEntity`
- JS wrapper class: `CustomHudLayout` (derives from `Entity`)

### Exactly one keyvalue of its own

| Keyvalue | Field | Type |
|----------|-------|------|
| `layout` | `m_strLayout` | `CUtlSymbolLarge` |

All other keyvalues (`targetname`, `origin`, ...) and IO (`Kill`, `AddOutput`, `FireUser1`, ...)
are inherited from `CBaseEntity`.

> **The entity declares ZERO Hammer inputs and ZERO Hammer outputs of its own.**
> A pure Hammer map with no scripting can only set `layout` at spawn time.
> Everything else must be driven from server-side JS.

## 2. Schema declaration

C++ definitions for a Source 2 modification — a server plugin or mod that talks to the engine
directly rather than through server-side JS. They mirror the engine's own schema records, so a
plugin can resolve these classes and reach their fields and methods at runtime.

Brace initialisers are the defaults the engine's constructor writes; `CPlayerSlot` needs none,
since its own default constructor already yields `INVALID_PLAYER_SLOT_INDEX`.

Networked fields are wrapped, and each wrapper carries a per-field tag type the engine
generates. The macros below stand for these templates:

| Macro | Expands to |
|-------|-----------|
| `CNetworkVar( T, m_x )` | `CNetworkVarBase< T, NetworkVar_m_x >` |
| `CNetworkUtlVector( T, m_x )` | `CNetworkUtlVectorBase< T, NetworkVar_m_x, -1, int >` |
| `CNetworkUtlVarEmbedded( T, m_x )` | `CUtlVectorEmbeddedNetworkVar< T, NetworkVar_m_x, -1, int >` |

`m_playerSlot` and `m_globalLayoutState` are the only fields not wrapped. **The client spells two
of these templates differently** — `C_NetworkUtlVectorBase` and `C_UtlVectorEmbeddedNetworkVar`,
with a leading `C_` — while `CNetworkVarBase` keeps its name on both sides.

```cpp
schema enum EHudPanelClassStatus_t : int32
{
	k_eHudPanelClassStatus_Undefined = -1,
	k_eHudPanelClassStatus_DoesNotHaveClass = 0,
	k_eHudPanelClassStatus_HasClass = 1,
};

// One "panel X carries CSS class Y" entry. Panel ids and class names are pooled on
// CCSCustomHudLayout; entries reference the pools by index.
schema struct HUDPanelHasClass_t
{
	TYPEMETA( MGetKV3ClassDefaults )
	DECLARE_SCHEMA_DATA_CLASS( HUDPanelHasClass_t );

	uint16 m_nPanelIdIndex;
	uint16 m_nClassNameIndex;
	EHudPanelClassStatus_t m_eClassStatus { k_eHudPanelClassStatus_DoesNotHaveClass };
};

// One dialog variable value. The schema fields start after a vtable pointer.
schema struct HUDPanelDialogVariableString_t
{
	uint16 m_nPanelIdIndex;
	uint16 m_nDialogVariableIndex;
	CUtlString m_sValue;
	bool m_bIsSet { false };
};

// The part of a layout that applies to one recipient: everyone (m_globalLayoutState)
// or one player slot (m_vecPlayerLayoutStates, indexed by CPlayerSlot).
schema class CCSCustomHudLayoutState
{
	CPlayerSlot m_playerSlot;
	CNetworkVar( bool, m_bInputCaptureEnabled );
	CNetworkUtlVector( HUDPanelHasClass_t, m_vecHasClasses );
	CNetworkUtlVector( HUDPanelDialogVariableString_t, m_vecDialogVariableStrings );
};

// The "custom_hud_layout" entity. m_strLayout is the KeyValue "layout".
schema class CCSCustomHudLayout : public CBaseEntity
{
	CNetworkVar( CUtlSymbolLarge, m_strLayout );
	CNetworkUtlVarEmbedded( CCSCustomHudLayoutState, m_vecPlayerLayoutStates );
	CCSCustomHudLayoutState m_globalLayoutState;
	CNetworkUtlVector( CUtlString, m_vecPanelIds );
	CNetworkUtlVector( CUtlString, m_vecClassNames );
	CNetworkUtlVector( CUtlString, m_vecDialogVariableNames );

	// Member functions the server exposes to cs_script / V8 on the script class "CustomHudLayout".
	bool IsInputCaptureEnabled( CPlayerSlot nPlayerSlot );
	void SetInputCaptureEnabled( CPlayerSlot nPlayerSlot, bool bEnabled );

	void SetHasClass( const CUtlString &strPanelId, const CUtlString &strClassName, EHudPanelClassStatus_t eClassStatus );
	void SetHasClassForPlayer( CPlayerSlot nPlayerSlot, const CUtlString &strPanelId, const CUtlString &strClassName, EHudPanelClassStatus_t eClassStatus );

	void SetDialogVariableString( const CUtlString &strPanelId, const CUtlString &strVariableName, const CUtlString &strValue );
	void SetDialogVariableStringForPlayer( CPlayerSlot nPlayerSlot, const CUtlString &strPanelId, const CUtlString &strVariableName, const CUtlString &strValue );
};
```

### Types from the Source SDK

Definitions live in [Wend4r/sourcesdk](https://github.com/Wend4r/sourcesdk):

| Type | Header |
|------|--------|
| `CPlayerSlot`, `INVALID_PLAYER_SLOT_INDEX` | [`public/playerslot.h`](https://github.com/Wend4r/sourcesdk/blob/main/public/playerslot.h) |
| `CUtlString` | [`public/tier0/utlstring.h`](https://github.com/Wend4r/sourcesdk/blob/main/public/tier0/utlstring.h) |
| `CUtlSymbolLarge` | [`public/tier1/utlsymbollarge.h`](https://github.com/Wend4r/sourcesdk/blob/main/public/tier1/utlsymbollarge.h) |
| `CNetworkVarBase` | [`public/networkvar.h`](https://github.com/Wend4r/sourcesdk/blob/main/public/networkvar.h) |
| `CNetworkUtlVectorBase`, `CUtlVectorEmbeddedNetworkVar` | [`public/networksystem/networkvar.h`](https://github.com/Wend4r/sourcesdk/blob/main/public/networksystem/networkvar.h) |
| `CEntityHandle` | [`public/entityhandle.h`](https://github.com/Wend4r/sourcesdk/blob/main/public/entityhandle.h) |
| `CCSUsrMsg_CustomHudClicked_t` | [`game/shared/cstrike15/usermessages.h`](https://github.com/Wend4r/sourcesdk/blob/main/game/shared/cstrike15/usermessages.h) |

`CBaseEntity` is game code — the SDK only forward-declares it. Mind the two near-identical
header names: `CNetworkVarBase` lives in `public/networkvar.h`, the other two in
`public/networksystem/networkvar.h`.

### Notes for anyone mirroring these types

- **`m_playerSlot` is easy to miss.** `CCSCustomHudLayoutState` has four schema fields, not three;
  the slot index is the first one. `m_globalLayoutState` carries `m_playerSlot = -1`, and each
  entry of `m_vecPlayerLayoutStates` carries its own index.
- **The client and server structs differ in size and field order.** `CCSCustomHudLayout` and `CCSCustomHudLayoutState` are both
  larger on the server, and the three pool vectors sit at different offsets on each side. The
  server-only tail is the three non-networked hash tables (string → pool index). Never share a
  mirrored struct between the two — resolve offsets per module.
- **Element stride.** `m_vecPlayerLayoutStates` holds `CCSCustomHudLayoutState` at the engine's
  schema size, which a mirrored class does not reproduce. Read the count, but reach per-player
  state through the member functions rather than by indexing.
- **Metadata is nearly absent here, though not in general.** The modules do ship a substantial
  schema-tag vocabulary — on the order of 50 distinct `M*` tags are actually attached to some class
  or field, including the `MNotSaved`, `MPulse*`, `MVData*` and `MProperty*` families. None of them
  land on this group: the only tag attached anywhere across these four types is
  `MGetKV3ClassDefaults` on `HUDPanelHasClass_t`. There is no `MNetworkEnable` string in these
  modules at all, so per-field network metadata is not recoverable statically — it is not shipped.
- **`SetDialogVariableStringForPlayer` is registered with the correct spelling.** A misspelled
  `SetDialogVariableStringorPlayer` (missing `F`) also exists in the binary, but only inside the
  wrapper's own diagnostic strings. The name registered on the script class is the correctly
  spelled one, so that is the name to bind against.

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

Server-side JS runs under `cs_script` (V8); a map attaches a script with a `point_script`
entity. The entity object is exposed to it as class `CustomHudLayout`:

| Method | Signature | Scope |
|--------|-----------|-------|
| `SetHasClass` | `(panelId: string, className: string, hasClass?: boolean)` | all players |
| `SetHasClassForPlayer` | `(playerSlot: number, panelId: string, className: string, hasClass?: boolean)` | one slot |
| `SetDialogVariableString` | `(panelId: string, variableName: string, value?: string)` | all players |
| `SetDialogVariableStringForPlayer` | `(playerSlot: number, panelId: string, variableName: string, value?: string)` | one slot |
| `SetInputCaptureEnabled` | `(playerSlot: number, enabled: boolean)` | one slot |
| `IsInputCaptureEnabled` | `(playerSlot: number)` | one slot |

Omitting the optional trailing argument:

- on `SetHasClass*` — resets the class; whether the field ends up `Undefined` (-1) or
  `DoesNotHaveClass` (0) was not determined;
- on `SetDialogVariableString*` — unsets the variable (`m_bIsSet = false`).

There is no global variant of `SetInputCaptureEnabled`; input capture is per player.

### Input capture — the difference between a cursor hud and an overlay hud

A custom hud is one of two things, and the entity decides which per player:

| | Input capture **off** (default) | Input capture **on** |
|---|---|---|
| Mouse cursor | none — the crosshair keeps aiming | cursor appears over the hud |
| `Button` clicks | never reach the server | raise `OnCustomHudClicked` |
| `:hover` in CSS | never matches | matches under the cursor |
| Player movement / firing | unaffected | the hud eats the input it captures |
| Typical use | timers, scoreboards, banners, kill feeds | shop menus, vote dialogs, pickers |

The client reads the flag **only from the per-player state at its own slot**. The copy inside
`m_globalLayoutState` is never consulted for it, which is why there is no global setter — it
would have nothing to drive. Flipping the flag on registers an input-capture object with the
input system under the name `CustomHudLayout`; flipping it off releases that registration, and
the same boolean is pushed onto the layout root panel.

> **It is initialised to `false` every time the layout is built.** The panel is created with
> capture off, so after the entity spawns — and after any change that rebuilds the layout —
> you must call `SetInputCaptureEnabled(slot, true)` again for every player who should have a
> cursor. Treat it as per-player, per-build state, never as something set once at map load.

Capture is all-or-nothing for the hud; it is not per panel. Within a captured hud, `hittest`
decides which panels the cursor can actually touch — see xml.md, *`hittest`*.

### Dialog variables in markup

The value is substituted into a `Label` whose `text` contains `{s:name}` — see xml.md,
*Label — text*, for the markup rules and the unset behaviour.

```js
hud.SetDialogVariableString("Timer", "value", "01:23");
```

## 4. The click protocol end to end

```
client: panel clicked
   |  user message CS_UM_CustomHudClicked = 390
   |  message CCSUsrMsg_CustomHudClicked {
   |      optional uint32 custom_hud_layout = 1 [default = 16777215]; // packed entity handle
   |      optional string button_id = 2;
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

The message definition above matches `csgo/cstrike15_usermessages.proto` verbatim, so it can be
taken as canonical rather than reverse-engineered. The SDK also declares the message wrapper:

```cpp
class CCSUsrMsg_CustomHudClicked_t : public CUserMessagePB< CS_UM_CustomHudClicked, CCSUsrMsg_CustomHudClicked > {};
```

### Packing the entity handle

`custom_hud_layout` is a `CEntityHandle` squeezed into a uint32 by
[`CEntityHandle::ToPackedInt()`](https://github.com/Wend4r/sourcesdk/blob/main/public/entityhandle.h), and read back with
`CEntityHandle::FromPackedInt()`:

```cpp
int CEntityHandle::ToPackedInt() const
{
	if( !IsValid() )
		return 0xFFFFFF;

	return GetEntryIndex() | ( ( GetSerialNumber() & 0x3FF ) << 14 );
}
```

So the wire layout is 14 bits of entry index in the low bits and the low 10 bits of the serial
above them, with `0xFFFFFF` reserved for an invalid handle — which is exactly the proto's
declared default of `16777215`.

Two consequences worth knowing:

- **The serial is truncated.** `CEntityHandle` stores 17 serial bits in memory, but only 10
  survive the packing, so `FromPackedInt()` can only check the handle against those:
  it looks the entity up by index and compares `GetSerialNumber() & 0x3FF`. A stale handle whose
  serial collides in its low 10 bits will pass that check.
- **`FromPackedInt()` lives in game code**, not in the header — the SDK implements it in
  `entity2/entitysystem.cpp`, because unpacking needs the entity system to resolve the index.

> **Security:** the server does **not** re-check `m_bInputCaptureEnabled` when handling the
> message. The gate is client-side only, so a modified client can send any `button_id` for any
> `custom_hud_layout` entity. Validate `buttonId` and the player's authority server-side.

## 5. Client-side classes

- `CCSGO_CustomHud` — singleton panel (`CUI_Root` → `panorama::CPanel2D`), XML tag
  `<CSGOCustomHud>`; it also implements `ICSGOGameUIStateListener`, so it is notified when the
  game-ui state changes rather than polling for it;
- `CCSGO_CustomHudLayoutRoot` — XML tag `<CSGOCustomHudLayoutRoot>`, stores the entity handle;
  created only from C++, one per entity, **with no id and no class**;
- neither adds any XML attribute or JS method of its own — they do not override the property
  template, so their surface is plain `CPanel2D`;
- `CCSGO_CustomHud` subscribes to exactly two events: `UIGameShutdown` and
  `CSGOHudSafeZoneChanged` (the latter converts the `safezonex` / `safezoney` convars into
  percentage margins plus width/height on itself);
- from `CUI_Root` it inherits automatic aspect-ratio class assignment (16:9, 17:9, 16:10, 4:3,
  21:9) and ui-scale updates — applied to the root panel, not to your markup.
