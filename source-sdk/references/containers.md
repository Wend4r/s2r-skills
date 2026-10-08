# Data structures

The engine's own containers, strings and buffers, roughly in order of how often SDK code uses them.
Reach for these before the standard library. Paths are relative to the SDK root; every example is
taken from the CTest suite in `tests/`, which is the quickest way to check exact behavior.

## Contents

- [Choosing a type](#choosing-a-type)
- [Strings](#strings)
- [Vectors](#vectors)
- [Ordered maps and trees](#ordered-maps-and-trees)
- [Hash tables](#hash-tables)
- [Linked lists](#linked-lists)
- [Symbols and tokens](#symbols-and-tokens)
- [Byte buffers](#byte-buffers)
- [KeyValues3](#keyvalues3)
- [Rules that hold for all of them](#rules-that-hold-for-all-of-them)
- [Tests](#tests)

## Choosing a type

| Need | Type | Header |
|---|---|---|
| Growable array | `CUtlVector< T >` | `public/tier1/utlvector.h` |
| Compact array: count, capacity and pointer in one small object | `CUtlLeanVector< T >` | `public/tier1/utlleanvector.h` |
| Array with inline storage that may spill to the heap | `CUtlVectorFixedGrowable< T, N >` | `public/tier1/utlvector.h` |
| Fixed-size array | `CUtlArray< T, N >` | `public/tier1/utlarray.h` |
| Kept-sorted array | `CUtlSortVector< T, LessFunc >` | `public/tier1/UtlSortVector.h` |
| Owning heap string | `CUtlString` | `public/tier0/utlstring.h` |
| String with stack storage, formatting and path helpers | `CBufferString`, `CBufferStringN< SIZE >` | `public/tier0/bufferstring.h` |
| Incremental string building | `CUtlStringBuilder` | `public/tier1/utlstringbuilder.h` |
| Ordered key/value map | `CUtlMap< K, V >`, `CUtlOrderedMap< K, V >` | `public/tier1/utlmap.h` |
| Ordered set, or a map's underlying tree | `CUtlRBTree< T, Less, I >` | `public/tier1/utlrbtree.h` |
| String-keyed map | `CUtlDict< T >` | `public/tier1/utldict.h` |
| String-keyed map over a `CUtlSymbol` table, case-insensitive by default | `CUtlStringMap< T >` | `public/tier1/utlstringmap.h` |
| Hash map or set | `CUtlHashtable< K, V >` | `public/tier1/utlhashtable.h` |
| Large open-addressing hash map | `CUtlHashMapLarge< K, V >` | `public/tier1/utlhashmaplarge.h` |
| Thread-safe hash | `CUtlTSHash< T >` | `public/tier1/utltshash.h` |
| Index-linked list with stable handles | `CUtlLinkedList< T >` | `public/tier1/utllinkedlist.h` |
| Several lists sharing one pool | `CUtlMultiList< T, I >` | `public/tier1/utlmultilist.h` |
| Intrusive list | `CUtlIntrusiveList< T >` | `public/tier1/utlintrusivelist.h` |
| Stack, queue, priority queue | `CUtlStack< T >`, `CUtlQueue< T >`, `CUtlPriorityQueue< T >` | `public/tier1/utlstack.h`, `utlqueue.h`, `utlpriorityqueue.h` |
| Interned string, pointer-sized | `CUtlSymbolLarge` + `CUtlSymbolTableLarge` | `public/tier1/utlsymbollarge.h` |
| Interned string, 16-bit id | `CUtlSymbol` | `public/tier0/utlsymbol.h` |
| Case-insensitive 32-bit string hash | `CUtlStringToken` | `public/tier0/utlstringtoken.h` |
| Binary or text byte stream | `CUtlBuffer` | `public/tier0/utlbuffer.h` |
| Typed flag set | `CUtlFlags< T >` | `public/tier1/utlflags.h` |
| Bit vector | `CBitVec< N >` | `public/bitvec.h` |
| Callback | `CUtlDelegate< Signature >` | `public/tier1/utldelegate.h` |
| Reference-counted pointer | `CSmartPtr< T >` | `public/tier1/smartptr.h` |
| Tree-shaped config and resource data | `KeyValues3` | `public/kv3lib/keyvalues3.h` |
| Raw element storage under the containers | `CUtlMemory`, `CUtlBlockMemory`, `CUtlFixedMemory` | `public/tier0/utlvectormemory.h`, `public/tier1/utlblockmemory.h`, `utlfixedmemory.h` |

## Strings

`CUtlString` owns a heap copy. `Get()` never returns null:

```cpp
CUtlString sString( "alpha" );

sString += "-";
sString += "beta";

V_strcmp( sString.Get(), "alpha-beta" ); // 0

sString = nullptr;
sString.IsEmpty(); // true
```

`CBufferString` starts on a stack buffer and moves to the heap only when it outgrows it.
`CBufferStringN< SIZE >` sets the stack size. Use it for temporary text, formatting and paths:

```cpp
CBufferString sBuffer;

sBuffer += "alpha";
sBuffer += '-';
sBuffer += 42;

sBuffer.StartsWith( "alpha" ); // true
sBuffer.IsStackAllocated(); // true while it fits

CBufferString sLiteral = "literal"_bs;
CBufferString sFromUtl( CUtlString( "utl" ) );
```

## Vectors

`CUtlVector` is the default array. Indices are `int`; `AddToTail` returns the new index:

```cpp
CUtlVector< int > vec;

vec.AddToTail( 2 ); // 0
vec.AddToHead( 1 ); // 0
vec.InsertAfter( 1, 3 ); // 2

vec.Remove( 1 );
vec.IsValidIndex( 2 ); // false

FOR_EACH_VEC( vec, i )
{
	Msg( "%d\n", vec[i] );
}
```

`CUtlLeanVector` has the same interface in a smaller object. Prefer it where the binary layout
already uses it, and for `FindAndFastRemove`-style unordered removal:

```cpp
CUtlLeanVector< int > vec;

vec.AddToTail( 1 );
vec.AddToTail( 3 );
vec.FindAndFastRemove( 1 ); // true; the last element fills the gap
```

## Ordered maps and trees

`CUtlMap` is a red-black tree. Lookups return an index, compared against `InvalidIndex()`:

```cpp
CUtlMap< int, int > map;

map.Insert( 2, 20 );
map.InsertOrReplace( 2, 25 );

auto i = map.Find( 2 );

if ( i != map.InvalidIndex() )
{
	int nValue = map[i]; // 25
}

FOR_EACH_MAP( map, i )
{
	Msg( "%d = %d\n", map.Key( i ), map.Element( i ) );
}
```

`CUtlRBTree` is the same tree without separate keys — an ordered set:

```cpp
CUtlRBTree< int, CDefLess< int >, int > tree;

tree.Insert( 2 );
tree.Insert( 1 );

for ( int i = tree.FirstInorder(); i != tree.InvalidIndex(); i = tree.NextInorder( i ) )
{
	Msg( "%d\n", tree[i] );
}
```

`CUtlDict` maps strings to values and keeps a copy of each key:

```cpp
CUtlDict< int > dict;

int nAlpha = dict.Insert( "alpha", 10 );

dict.Find( "alpha" ); // nAlpha
dict.GetElementName( nAlpha ); // "alpha"
dict.Remove( "alpha" );
```

## Hash tables

`CUtlHashtable` works with `UtlHashHandle_t`. Inserting an existing key keeps the old value and
reports it through the optional `bool *`:

```cpp
CUtlHashtable< int, int > table;

bool bDidInsert = false;
UtlHashHandle_t hOne = table.Insert( 1, 10, &bDidInsert ); // bDidInsert == true

table.Insert( 1, 20, &bDidInsert ); // bDidInsert == false, value stays 10
table.Element( hOne ); // 10
table.HasElement( 1 ); // true
table.Remove( 1 );
```

## Linked lists

`CUtlLinkedList` stores nodes in an array and links them by index, so handles stay valid while
other nodes are added and removed. The node storage `M` must provide a static `INVALID_INDEX`;
the default `CUtlLeanVector` does not, so the test supplies its own:

```cpp
class CTestLinkedListMemory : public CUtlLeanVector< UtlLinkedListElem_t< int, int >, int >
{
public:
	using BaseClass = CUtlLeanVector< UtlLinkedListElem_t< int, int >, int >;
	using BaseClass::BaseClass;

	static inline const int INVALID_INDEX = -1;
};

CUtlLinkedList< int, int, false, int, CTestLinkedListMemory > list;

auto iHead = list.AddToHead( 1 );
list.AddToTail( 3 );
list.InsertAfter( iHead, 2 );

FOR_EACH_LL( list, i )
{
	Msg( "%d\n", list[i] );
}
```

## Symbols and tokens

`CUtlStringToken` is a case-insensitive 32-bit hash of a string, used for fast name comparisons:

```cpp
CUtlStringToken token( "Alpha" );

token == "ALPHA"; // true
token.GetHashCode() == MakeStringToken< true, false >( "Alpha" ); // true
```

`CUtlSymbolLarge` is an interned string handle from a `CUtlSymbolTableLarge`. Equal strings get the
same handle, and `String()` returns the stored text:

```cpp
CUtlSymbolTableLarge symbols;
CUtlSymbolLarge sym = symbols.AddString( "weapon_ak47" );

sym.String(); // "weapon_ak47"
```

`CUtlSymbol` is the older 16-bit id form; check `IsValid()` before use.

## Byte buffers

`CUtlBuffer` is a growable byte stream with separate put and get positions, in binary or text mode:

```cpp
CUtlBuffer buffer( 0, 0, CUtlBuffer::NONE );

buffer.PutInt( 1234 );
buffer.PutFloat( 12.5f );
buffer.PutString( "value" );

buffer.TellPut(); // bytes written
buffer.GetInt(); // 1234
```

## KeyValues3

`KeyValues3` is a typed tree value: null, scalar, string, array, table or binary blob:

```cpp
KeyValues3 kv;

kv.SetToEmptyTable();
kv.SetMemberInt( "answer", 42 );
kv.SetMemberString( "name", "source" );

kv.GetMemberInt( "answer" ); // 42
kv.GetMemberInt( "missing", -1 ); // -1
kv.FindMember( "missing" ); // nullptr

KeyValues3 array;

array.ArrayAddToTail()->SetInt( 1 );
array.GetArrayLength(); // 1
```

Arrays and tables are implemented by `CKeyValues3Array` and `CKeyValues3Table` in
`kv3lib/keyvalues3_array.h` and `kv3lib/keyvalues3_table.h`.

## Rules that hold for all of them

- **Indices are not pointers.** Maps, trees, lists and hash tables return an index or handle. Test
  it against `InvalidIndex()` / `IsValidIndex()` / `IsValidHandle()`, never against a literal `-1`.
- **Iterate with the project macros** — `FOR_EACH_VEC`, `FOR_EACH_MAP`, `FOR_EACH_LL` — or range
  `for` where the container supports it and nearby code uses it.
- **Removal invalidates differently.** Removing from a `CUtlVector` shifts later indices;
  `FastRemove` moves the last element into the gap. Linked-list and tree handles survive removal of
  other elements.
- **Layout is part of the ABI.** A container member in a binary-matched class keeps the exact type,
  template arguments and index type the binary uses. Do not swap `CUtlVector` for
  `CUtlLeanVector`, or the reverse, to tidy up.
- **No standard-library replacements** in SDK code that already uses these types.

## Tests

Each type has its own test executable in `tests/`, registered as `<name>_tests`. Run one with:

```sh
ctest --preset Debug -R utlmap
```

| Test | Covers |
|---|---|
| `tests/utlvector.cpp` | `CUtlVector`: insert, remove, iteration, copy and move, sorting, tracked lifetimes |
| `tests/utlleanvector.cpp` | `CUtlLeanVector`: insert, remove, fast remove, swap, copy and move |
| `tests/utlstring.cpp` | `CUtlString`: copy, compare, concat, substring, replace, trim, move |
| `tests/bufferstring.cpp` | `CBufferString`: concat, insert, replace, format, case, capacity, ownership, paths, UTF-8, `CUtlString` interop |
| `tests/utlbuffer.cpp` | `CUtlBuffer`: binary and text mode, seek, capacity, copy and move |
| `tests/utlmap.cpp` | `CUtlMap`: insert, replace, in-order iteration, closest-key search, copy and move |
| `tests/utlrbtree.cpp` | `CUtlRBTree`: insert, find, in-order traversal, duplicates, depth |
| `tests/utldict.cpp` | `CUtlDict`: insert, rename, find, remove |
| `tests/utlhashtable.cpp` | `CUtlHashtable`: insert, duplicate detection, swap, compact, purge |
| `tests/utlhash.cpp` | `CUtlHash` |
| `tests/utllinkedlist.cpp` | `CUtlLinkedList`: head, tail and middle insertion, iteration, removal |
| `tests/utlmultilist.cpp` | `CUtlMultiList` |
| `tests/utlsortvector.cpp` | `CUtlSortVector` |
| `tests/utlstack.cpp`, `tests/utlqueue.cpp`, `tests/utlpriorityqueue.cpp` | Stack, queue, priority queue |
| `tests/utlarray.cpp`, `tests/utlpair.cpp`, `tests/utlflags.cpp` | `CUtlArray`, `CUtlPair`, `CUtlFlags` |
| `tests/utlmemory.cpp`, `tests/utlblockmemory.cpp`, `tests/utlfixedmemory.cpp`, `tests/utlscratchmemory.cpp` | Element storage |
| `tests/utlsymbol.cpp`, `tests/utlstringtoken.cpp`, `tests/utlstringmap.cpp` | `CUtlSymbol`, `CUtlStringToken`, `CUtlStringMap` |
| `tests/utlsignalslot.cpp` | Signal and slot |
| `tests/keyvalues3.cpp` | `KeyValues3`: primitives, arrays, tables, operators, arenas, loading KV1, KV3, JSON and binary |
| `tests/tier0_utl_headers.cpp`, `tests/tier1_utl_headers.cpp` | Every header compiles on its own |
