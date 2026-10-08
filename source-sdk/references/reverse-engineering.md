# Reverse Engineering and Binary Work

Applies to anything involving signatures, offsets, vtables, calling conventions, binary
compatibility, gamedata, disassembly, or behavior reconstructed from a binary.

## Contents

- [Evidence Rules](#evidence-rules)
- [Working with IDA](#working-with-ida)
- [Reconstructing Structures and Classes](#reconstructing-structures-and-classes)
- [Vtables](#vtables)
- [Gamedata and Signatures](#gamedata-and-signatures)
- [Comments on Reconstructed Code](#comments-on-reconstructed-code)

## Evidence Rules

- Prefer direct evidence — functions, xrefs, names, types, strings, decompiler output — over
  memory or inference.
- Never invent addresses, offsets, signatures, symbol names or vtable indexes. If IDA is
  unavailable, say so explicitly and use repository evidence only.
- Record important IDA-derived assumptions in the final response, or in a nearby technical
  comment when the code would otherwise be hard to justify.

## Working with IDA

First check whether an `ida-pro-mcp` server is available among the active MCP tools.
When available, use it as the primary source for IDA-derived facts before editing code or gamedata.
Its tool schemas may be deferred and need loading before the first call. Use the following
instance-selection workflow when those tools are exposed by the server.

1. **List instances.** One IDA instance is open per module. Call `list_instances` first.
2. **Select one.** Call `select_instance`, then `server_health` to confirm which binary it
   actually has loaded. The instance-to-binary mapping shifts between game builds — never carry
   it over from a previous session.
3. **Query.** Decompile, disassemble, follow xrefs, search strings.
4. **Script when needed.** IDAPython runs through the same server: `py_eval` for expressions,
   `py_exec_file` for scripts.

### Orientation by Strings

String references are the primary orientation when reconstructing fields: nearby literals, xrefs,
logging and assert messages, schema names, RTTI/typeinfo, constructor and destructor references.
Do not infer a semantic name from an offset alone when string evidence exists.

### Template Names from the Schema System

Game modules are stripped, but the schema system embeds Itanium-mangled `typeid( T ).name()`
strings. A regex search over strings recovers exact template signatures.

A `_ZTS` string with no matching `_ZTI` object means the class is not polymorphic — do not give it
a vtable.

## Reconstructing Structures and Classes

- Preserve exact field offsets and padding.
- Use explicit padding members only when the real field type or purpose is unknown, and name them
  so the offset range is clear.

### Size Asserts

For every binary-reconstructed structure or class with a known size, add a
`COMPILE_TIME_ASSERT` near the declaration or definition that owns the layout.

- Use the size verified from the target binary.
- Write the verified size in hexadecimal. For example, if the binary confirms a size of `0x40`:

```cpp
COMPILE_TIME_ASSERT( sizeof( TypeName ) == 0x40 );
```

- Guard platform- or branch-specific sizes with the matching preprocessor conditions.

## Vtables

- Preserve exact slot order. Never reorder, remove or collapse virtual methods because their
  purpose is unknown — every slot in the binary keeps a corresponding slot in the SDK.
- **Plausible name, unknown signature:** `Unk_IntendedMethodName( void *p )`, in its exact slot,
  with a single opaque `void *p` until the signature is verified.
- **Neither name nor signature known:** a slot-preserving name carrying the vtable index or offset.
  Rename only once binary evidence supports the real meaning.
- **Partially known signature:** do not "improve" it with guessed argument types. Keep opaque
  placeholders and document what evidence would replace them.

## Gamedata and Signatures

- Verify game, engine branch, platform and binary version before changing an entry.
- Keep platform-specific entries separated.
- Do not broaden a signature without evidence.
- Validate JSON syntax after editing.

## Comments on Reconstructed Code

Keep them to what the reader cannot get from tooling.

| Do not | Why |
| --- | --- |
| Annotate members with their offsets (`// 0x38`) | clangd in VS Code and CLion shows offsets on hover; the comment goes stale when the layout shifts |
| Name the library or module a reconstruction came from | Belongs in the reply or commit message |
| Point at other headers by filename | Describe what the declaration is; the include graph says where it lives, and the reference rots when files move |
| Add `AMNOTE:` markers | Reserved for repositories under the AlliedModders LLC organization; this fork does not add new ones |
