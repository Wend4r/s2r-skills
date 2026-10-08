# Code Style, Naming and C++ Practices

Follow `.clang-format` first, then the style of the file being edited.

## Contents

- [Formatting](#formatting)
- [Naming](#naming)
- [C++ Practices](#c-practices)

## Formatting

### Indentation and Braces

Tabs for indentation; tab width and indent width are 4. Braces go on their own line (Allman):

```cpp
if ( bEnabled )
{
	DoWork();
}
```

### Spaces Inside Brackets

| Construct | Example |
| --- | --- |
| Parentheses — declarations, calls, expressions, control statements | `Foo( nArgCount );` `if constexpr ( COMPILER_CONDITION )` `while ( true )` |
| Template angle brackets — every `<x>` becomes `< x >`, including the C++ casts | `template < size_t SIZE >` `CBufferStringN< SIZE >` `CUtlVector< int >` `static_cast< int >( nValue )` `reinterpret_cast< const char * >( pData )` |
| Array element count | `int m_nChild[ 2 ];` `char m_pGameInfoPath[ MAX_PATH ];` |
| Subscripts — every `[x]` becomes `[ x ]` | `vec[ i ]` `m_pElements[ nIndex ]` `map[ map.Find( 1 ) ]` |
| Empty brackets stay tight — unsized arrays, `delete[]`, captureless lambdas | `int nValues[] = { 1, 2, 3 };` `delete[] pData;` `[]() {}` |
| C-style casts in edited code | `( int )( nValue )` `( const char * )pData` |
| Braced initializer lists and short macro bodies | `{ 1, 2, 3 }` `#define Assert_BSO( exp ) { if ( IsStackAllocated() ) Assert( exp ); }` |

### Space Before `(`

None for function declarations, definitions and calls; one for control statements and
control-like macros:

```cpp
void SetValue( int nValue );
SetValue( nValue );

if ( bReady )
for ( int i = 0; i < nCount; ++i )
FOR_EACH_VEC( vecArgs, i )
```

### Pointers and References

The marker binds to the variable name, not the type:

```cpp
const char *pString;
CBufferString &sBuffer;
void *pData;
```

### Bit-flag Enums

Write flag values as a bit shift, without parentheses:

```cpp
enum EntityClassFlags_t
{
	ECF_NOT_NETWORKED = 1 << 0,
	ECF_ALIAS = 1 << 1,
	ECF_SPAWN_GROUP_HANDLE_INVALID = 1 << 2,
};
```

Not `( 1 << 2 )`, `0x4` or `4`. Existing enums written as `(1 << n)` are left as they are unless
the task is about them — no style-only churn.

### Line Shape

- Keep a short inline function on one line only when surrounding code does and it stays readable:
  `int Length() const { return m_nLength; }`
- `ColumnLimit: 0` — keep long argument, template and call lists on one line when the surrounding
  code does. Do not wrap only because a line is long.
- No alignment-only churn. Do not realign unrelated declarations, assignments, comments or tables.
- Keep the file's existing line endings and encoding.
- Do not sort includes unless the file already uses sorted include groups.

## Naming

| Prefix / suffix | Meaning | Example |
| --- | --- | --- |
| `p` | Pointer | `pString`, `pData` |
| `n` | Integer count or size | `nLen`, `nCount` |
| `b` | Boolean | `bAllowHeapAllocation` |
| `m_` | Member | `m_nLength` |
| `C` | Class | `CUtlString` |
| `I` | Interface | `IGameEventManager2` |
| `M` | Compile-time metaprogramming structure or trait | — |
| `_t` | Many enum and struct typedef-style names | `SchemaClassInfoData_t` |

Do not rename symbols to modernize them. Public ABI and API names stay stable unless the task is
explicitly a rename.

## C++ Practices

- **Project types first.** `CBufferString`, `CUtlString`, `CUtlBuffer`, `CUtlVector`,
  `CUtlLeanVector`, `Q_*` / `V_*` string helpers, `Assert`, `Move` and the platform abstraction
  headers — when nearby code uses them. Do not add new generic utilities where a project helper
  exists. Headers, examples and tests for each are in [containers.md](containers.md).
- **Ownership and allocation.** Many modules have custom allocation, fixed-buffer, stack-buffer or
  platform-specific lifetime expectations. Read how the surrounding code allocates before adding
  to it.
- **Public contracts.** Preserve binary layout, vtable layout, exported symbols, calling
  conventions and public header contracts.
- **No exceptions or RTTI-dependent designs** unless the subsystem already uses them.
- **Platform guards stay precise.** Do not widen `_WIN32`, POSIX, Xbox, PS3 or dedicated-server
  paths casually.
- **Public header overloads and templates** cost compile time and can create ABI or overload
  ambiguity. Weigh both before adding them.
- **Modern C++ only where it already lives.** Old subsystems do not get new language facilities
  unless nearby code already uses them or the task requires it.

### Comments

Comments explain non-obvious constraints, compatibility requirements, reverse-engineered behavior
or ownership rules. Never restate the code. Comments on reconstructed code have extra rules — see
[reverse-engineering.md](reverse-engineering.md#comments-on-reconstructed-code).
