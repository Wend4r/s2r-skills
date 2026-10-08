# Schema Classes, Meta Tags and `noschema` Fields

Applies when declaring or reconstructing a class, field, enum or atomic that the Source 2 schema
system describes, when adding `META` / `TYPEMETA` markup, or when deciding whether a member is
`noschema`. The evidence rules from [reverse-engineering.md](reverse-engineering.md) apply
throughout: the binary decides, never the name or the habit of neighbouring classes.

## Contents

- [Markup in SDK Headers](#markup-in-sdk-headers)
- [Is the Class Schema](#is-the-class-schema)
- [Finding Records in a Binary](#finding-records-in-a-binary)
- [What the Class Record Gives](#what-the-class-record-gives)
- [Class Meta Tags](#class-meta-tags)
- [Field Meta Tags](#field-meta-tags)
- [Enums, Enumerators and Atomics](#enums-enumerators-and-atomics)
- [Which Fields can be `noschema`](#which-fields-can-be-noschema)
- [Declaring a Missing Tag](#declaring-a-missing-tag)
- [Checklist](#checklist)

## Markup in SDK Headers

The markup keywords are defined empty in `public/tier0/basetypes.h`. They compile to nothing, but
they record what the binary registers and are read by schema tooling, so they must stay truthful.

| Markup | Placement | Meaning |
| --- | --- | --- |
| `schema` | Before `class`, `struct` or `enum` | The type is registered in the schema system |
| `TYPEMETA( ... )` | Inside the class or enum body, before the members | Tags on the class, enum or atomic itself |
| `META( ... )` | After a field or enumerator declaration | Tags on that field or enumerator |
| `noschema` | Before a member declaration | Member exists in memory but is not a schema field |
| `META_USE_CODEGEN_TAG( Name )` | Inside the class body | Schema compiler runs the codegen tag script for the class |
| `UNSCHEMATIZED_METHOD( decl )` | Around a method | Hidden from the schema compiler (`COMPILING_SCHEMA`) |

Several tags go into one macro, separated by `;`, with `=` for a value:

```cpp
schema class CExampleData
{
public:
	TYPEMETA( MVDataRoot; MVDataAssociatedFile = "scripts/example.vdata" );

	int m_nCount; META( MPropertyFriendlyName = "Count"; MPropertySortPriority = 100 );
	float m_flScale; META( MNotSaved );
	noschema void *m_pRuntimeCache;
};
```

`public/mathlib/femodel.h` also uses `DECLARE_SCHEMA_DATA_CLASS`. It is not built; do not copy
that macro.

## Is the Class Schema

Use the strongest evidence available, in this order:

1. **Class record in the owning module.** Search the class name string, follow its data xrefs to
   a `SchemaClassInfoData_t` (layout in `public/schemasystem/schematypes.h`): binding pointer,
   name, project (module) name, C++ name, size, field and metadata counts, alignment, base count,
   then field, base, datamap and metadata arrays. The module registers it through its
   `CSchemaRegistration` list during `SCHEMA_REGISTRATION_PHASE_CLASSES`.
2. **Runtime lookup.** `ISchemaSystem` → the module's type scope → `FindDeclaredClass`.
3. **Schema dumps** (for example source2gen output). Secondary: confirm the dump matches the game
   build and module you are working on.

Hints that are **not** proof on their own: the class appears as a field type of a schema class,
it has a datamap, it uses `CLASS_USES_KV3TRANSFER_*`, it is an entity class, or a `typeid` string
mentions it (atomic registration embeds template names, not declared classes).

No class record means a plain C++ class: no `schema`, no `META`, no `TYPEMETA`, no
`noschema`. Its size and layout come from constructors, allocations and accessors instead.

The same class name may be registered by several modules, and a class may be registered in the
global or a module-local type scope. Use the record of the module the declaration describes.

## Finding Records in a Binary

The records are static data of the module and have exactly the layout of the SDK structs
in `public/schemasystem/schematypes.h`, so read them through those structs, never through
hand-counted offsets:

1. Declare the SDK structs in the database:
   `SchemaClassInfoData_t`, `SchemaClassFieldData_t`, `SchemaBaseClassInfoData_t`,
   `SchemaMetadataEntryData_t`, `SchemaEnumInfoData_t` and `SchemaEnumeratorInfoData_t`. Types
   they only point at (`CSchemaType`, `CSchemaSystemTypeScope`, `datamap_t`) can stay opaque.
2. Find the exact name string, then its data xrefs.
3. The xref from `SchemaClassInfoData_t::m_pszName` or `SchemaEnumInfoData_t::m_pszName` is the
   record. Apply the struct there and read it.
4. Keep the candidate that reads sanely: `m_pfnManipulator` is a function, `m_pFields`,
   `m_pBaseClasses` and `m_pStaticMetadata` point at data, `m_nSize` is plausible. Other xrefs of
   the name — code, string tables, unrelated data — read as garbage under the struct.
5. Apply the element structs to the arrays and read them: `m_pFields` as `m_nFieldCount`
   `SchemaClassFieldData_t`, `m_pBaseClasses` as `m_nBaseClassCount`
   `SchemaBaseClassInfoData_t`, every `m_pStaticMetadata` as its count of
   `SchemaMetadataEntryData_t`; for an enum, `m_pEnumerators` as `m_nEnumeratorCount`
   `SchemaEnumeratorInfoData_t`.
6. Resolve each `SchemaMetadataEntryData_t::m_pszName` and read `m_pData` as the tag's
   `Storage_t`.

The static record is read before registration, so some members are not filled in yet:

| Struct | Members not filled in the static record |
| --- | --- |
| `SchemaClassInfoData_t` | `m_pSchemaBinding`, `m_pszProjectName`, `m_pszCPPName` (null); `m_nMultipleInheritanceDepth`, `m_nSingleInheritanceDepth` (`0xFFFF`); `m_pTypeScope`, `m_pDeclaredClass` |
| `SchemaClassFieldData_t` | `m_pType`: a packed `0xF...` type index, or null until the manipulator fills it |
| `SchemaBaseClassInfoData_t` | — (`m_pClass` points at the base's static record) |
| `SchemaEnumInfoData_t` | `m_pSchemaBinding`, `m_pszProjectName`, `m_pTypeScope` (null); `m_nMinEnumeratorValue`, `m_nMaxEnumeratorValue` (`-1`) |

Only the module-owned names and counts are reliable there; take the project name and resolved
field types from a runtime lookup or a dump when they matter.

### Worked Examples

From the CS2 `libserver.so` 1.41.8.8 schema records of SDK types. Addresses and values change
between builds; re-read them for the build you work on.

**`CEntityInstance`** (`public/entity2/entityinstance.h`): size `0x30`, no bases, no metadata
entries, `m_nFlags1` `0x20241`.

| Offset | Record field | SDK member | Verdict |
| --- | --- | --- | --- |
| `0x00` | — | vtable pointer | `SCHEMA_CF1_HAS_VIRTUAL_MEMBERS` is set; not a member |
| `0x08` | `m_iszPrivateVScripts` | `CUtlSymbolLarge m_iszPrivateVScripts` | schema |
| `0x10` | `m_pEntity` | `CEntityIdentity *m_pEntity` | schema |
| `0x18` | — | `CEntityPrivateScriptScope m_hPrivateScope` | `noschema` |
| `0x20` | — | `CEntityKeyValues *m_pKeyValues` | `noschema` |
| `0x28` | `m_CScriptComponent` | `CScriptComponent *m_CScriptComponent` | schema |

`m_pStaticMetadata` is empty, yet `m_nFlags1` has `SCHEMA_CF1_INFO_TAG_MConstructibleClassBase`:
the class carries `TYPEMETA( MConstructibleClassBase )` through the flag alone.

**`CEntityIdentity`** (`public/entity2/entityidentity.h`): size `0x70`, one class tag
`MGetKV3ClassDefaults` with a non-null value (a cell holding the defaults getter).

- Fields from `0x14` on: `m_nameStringTableIndex`, `m_name`, `m_designerName`, `m_flags`,
  `m_worldGroupId`, `m_fDataObjectTypes`, `m_PathIndex`, `m_pAttributes`, `m_pPrev`, `m_pNext`,
  `m_pPrevByClass`, `m_pNextByClass`. All but `m_name` and `m_pAttributes` carry `MNotSaved`
  with null data — tag-only, and still schema fields.
- Not in the record, so `noschema`: `m_pInstance` (`0x00`), `m_pClass` (`0x08`), `m_EHandle`
  (`0x10`), `m_hPublicScope` (`0x28`) and `m_hSpawnGroup` (`0x34`). The SDK code reads and
  writes all of them, which is the evidence the rules below require.

**`RenderMode_t`** (`public/const.h`): size 1, no metadata; enumerators `kRenderNormal` 0,
`kRenderTransAlpha` 1, `kRenderNone` 2, `kRenderModeCount` 3. The count enumerator is in the
record, so it is a schema enumerator like the others.

## What the Class Record Gives

| Record member | Use |
| --- | --- |
| `m_nSize`, `m_nAlignment` | Verified size for the `COMPILE_TIME_ASSERT`, and the alignment |
| `m_pBaseClasses` / `m_nBaseClassCount` | Bases in order, with their offsets; the first is primary |
| `m_pFields` / `m_nFieldCount` | The class's own fields only, not inherited ones |
| `m_pStaticMetadata` | Class tags (see below) |
| `m_nFlags1` | `SCHEMA_CF1_HAS_VIRTUAL_MEMBERS` (vtable), `SCHEMA_CF1_IS_ABSTRACT`, construction flags and info-tag bits |
| `m_pfnManipulator` | Construct, destruct and registration hooks; action `SCHEMA_CLASS_MANIPULATOR_ACTION_REGISTER` fills unresolved field types |

Each field record holds the name, the type, `m_nSingleInheritanceOffset` (from the start of this
class) and its metadata array. Before registration the type slot holds `0xF` in the top nibble
plus an index into the module's registered type table, or null until the manipulator fills it;
after registration it is a `CSchemaType *`.

## Class Meta Tags

- Read the class metadata array: each `SchemaMetadataEntryData_t` is `{ m_pszName, m_pData }`.
  A tag may repeat; names are case sensitive. Write every entry into `TYPEMETA`, in record order.
- `m_pData` points at a value of the tag's `Storage_t`, and is null for a tag-only tag. For a
  `const char *` tag it points at a cell holding the string pointer; for an `int` tag at the int;
  for a function tag at a cell holding the function. Find `Storage_t` with
  `rg "DECLARE_SCHEMA_META_TAG\( MTagName"`.
- On CS2, Dota 2 and Deadlock some class tags are also, or only, encoded as bits in `m_nFlags1`
  (`SCHEMA_CF1_INFO_TAG_*`: `MNetworkAssumeNotNetworkable`, `MNetworkNoBase`,
  `MClassHasCustomAlignedNewDelete`, `MConstructibleClassBase` and others). A set bit means the
  tag is on the class even when the metadata array lacks it.
- The array covers the class itself only. Tags on a base stay on the base; lookups reach them
  through `SchemaBaseClassTraversal_t`. Never copy base tags onto a derived class.
- Codegen tags (`MEmit*`, `MEmitKV3Transfer`) and `META_USE_CODEGEN_TAG` never reach the binary.
  Their evidence is generated code, e.g. a `KV3TransferSave_<Class>` body chaining to the base.
- Recent CS2 server binaries no longer carry most `MNetwork*` tags as metadata strings. Absence
  there is not evidence that the tag is gone; say which source the network tags came from.

## Field Meta Tags

- Read each field's metadata array the same way and write it into `META( ... )` after the field.
- A tagged field is still a schema field. `MNotSaved`, `MPropertySuppressField`,
  `MPropertyHideField` and `MFgdFromSchemaCompletelySkipField` hide a field from save, editors or
  FGD — they never make it `noschema`.
- Static fields (`SchemaStaticFieldData_t`) carry metadata the same way.
- Inherited fields belong to the base's record; never redeclare or retag them in the derived class.

## Enums, Enumerators and Atomics

An enum is schema when the owning module has a `SchemaEnumInfoData_t` record for it; find it the
same way as a class record, from the name string. Without a record it is a plain enum: no
`schema`, no `TYPEMETA`, no `META`.

| Record member | Use |
| --- | --- |
| `m_nSize`, `m_nAlignment` | Underlying type size: 1, 2, 4 or 8 bytes |
| Enumerator values | A negative value means a signed underlying type; `m_nMinEnumeratorValue` / `m_nMaxEnumeratorValue` are not filled in the static record |
| `m_pEnumerators` / `m_nEnumeratorCount` | Every enumerator: name, `int64` value and metadata |
| `m_pStaticMetadata` | Tags on the enum itself |
| `m_nFlags` | `SCHEMA_EF_*` registration and type scope flags, not tags |

- Mark the enum `schema` and give it the underlying type the size and sign call for, unless the
  existing declaration already fixes it.
- Declare every enumerator from the record with its exact value. Keep a source-only enumerator,
  such as a count or mask, only with code evidence; never renumber to fill gaps.
- Enum tags (e.g. `MEnumFlagsWithOverlappingBits`, `MTreatUnknownEnumeratorsAsZero`) go into
  `TYPEMETA( ... )` before the first enumerator, without a trailing `;` — an enum body cannot hold
  an empty declaration.
- Enumerator tags (e.g. `MPropertyFriendlyName`, `MPropertySuppressEnumerator`,
  `MAlternateSemanticName`, `MEnumeratorIsNotAFlag`) go into `META( ... )` after the enumerator's
  comma.
- `noschema` does not apply to enumerators.

```cpp
schema enum ExampleFlags_t : uint8
{
	TYPEMETA( MEnumFlagsWithOverlappingBits )
	EXAMPLE_FLAG_NONE = 0, META( MPropertySuppressEnumerator )
	EXAMPLE_FLAG_VISIBLE = 1 << 0, META( MPropertyFriendlyName = "Visible" )
	EXAMPLE_FLAG_SOLID = 1 << 1, META( MPropertyFriendlyName = "Solid" )
};
```

- Atomic types (registered templates such as `CUtlVector< T >` or `CHandle< T >`) carry tags such
  as `MAtomicTransfersAsPlainString` or `MNetworkSerializeAs`; read them from
  `SchemaAtomicTypeInfo_t`.
- Method-location tags (`META_TAG_ON_METHOD`, e.g. `MPulse*`) come from script binding tables,
  not from schema class records.

## Which Fields can be `noschema`

A `noschema` member occupies bytes of a schema class but has no entry in that class's field
array. Find the candidates by laying the record out:

1. Start after the primary base subobject (its record size), or after the vtable pointer when
   `SCHEMA_CF1_HAS_VIRTUAL_MEMBERS` is set and no base already provides one.
2. Sort the fields by offset; each covers its type's size (`CSchemaType::GetSizeAndAlignment`).
3. Every uncovered range up to `m_nSize` is either alignment padding or `noschema` members.

An uncovered range is `noschema` only when code shows a member there: constructor or destructor
writes, accessors, xrefs, datamap entries or debug strings. Otherwise it is padding, declared
under the padding rules in [reverse-engineering.md](reverse-engineering.md). An explicit padding
member inside a schema class is itself not a schema field, so mark it `noschema` as well.

Typical `noschema` members, still subject to the gap evidence above:

- types the schema system cannot describe: function pointers, references, unions and anonymous
  structs, standard library or third-party types;
- template instances not registered as atomics, and values of non-schema classes;
- runtime-only state: caches, locks, engine object handles, intrusive list links.

Never `noschema`:

- a member at an offset the field array lists, whatever its tags;
- a networked field — the network system builds its serializers from schema fields;
- the vtable pointer or a base subobject — they are not members at all;
- a member of a class that is not schema — there is nothing to exclude it from.

Older branches set `SCHEMA_CF1_HAS_NOSCHEMA_MEMBERS` on classes with such members. On CS2,
Dota 2 and Deadlock that bit means `SCHEMA_CF1_LIMITED_METADATA`, so the layout walk above is
the only evidence.

## Declaring a Missing Tag

If a record carries a tag the SDK does not declare yet:

- declare it with `DECLARE_SCHEMA_META_TAG( Name, locations, META_TAG_ONLY() )` or
  `META_VALUE( type )` in the header of its subsystem (`schemasystem/schema.h` for generic schema
  tags, `networksystem/networkvar.h`, `vdata/vdatametadata.h`, `pulse/pulsemetadata.h` and so on);
- set the location flags to the owners where the tag was actually found;
- take `Storage_t` from what `m_pData` points at; keep it opaque if the value type is unknown;
- write a short comment with an observed example value, or `meaning unknown` — never a guessed
  meaning.

## Checklist

1. Class record found in the owning module, or the class is plain C++.
2. Size, alignment, bases and the vtable flag match the declaration; `COMPILE_TIME_ASSERT` added.
3. Every own field from the record is declared at its offset with its `META` tags.
4. Class tags from the metadata array and `SCHEMA_CF1_INFO_TAG_*` bits are in `TYPEMETA`.
5. Every uncovered range is either padding or a `noschema` member backed by code evidence.
6. The final response names the module, game build and evidence used.
