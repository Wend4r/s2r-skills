# Custom hud XML contract

## 1. Permitted tags and attributes

The client builds a whitelist once, on first use: a hash table mapping panel-type name to a
list of allowed attribute names. It has exactly four entries:

| Tag | Allowed attributes | Count |
|-----|--------------------|-------|
| `Panel`  | `id`, `class`, `hittest` | 3 |
| `Label`  | `id`, `class`, `hittest`, `text` | 4 |
| `Image`  | `id`, `class`, `hittest`, `src`, `texturewidth`, `textureheight` | 6 |
| `Button` | `id`, `class` | 2 |

That is the **entire** markup vocabulary. Note:

- `Button` has **no** `hittest` and no `text` — put a nested `<Label>` inside it for a caption;
- `text` exists only on `Label`;
- no panel type has a `style` attribute → **inline CSS is structurally impossible**;
  all styling goes through `.vcss`;
- there are no event handlers at all (`onactivate`, `onload`, `onmouseover`, …);
- absent and therefore forbidden: `dialogvariable`, `visible`, `enabled`, `tabindex`,
  `acceptsinput`, `hittestchildren`, `defaultsrc`, `scaling`, `tooltip`, `args`, `snippet`.

Unknown tag → `Layout contains disallowed panel type '%s'.`
Unknown attribute → `Layout contains disallowed attribute %s for panel type '%s'.`

### Attribute values are not validated

When the walker reaches an attribute node it checks the attribute *name* and returns success
immediately, **without descending into the child node that holds the value**. Consequently the
strings in `class`, `text`, `src` and `id` are unconstrained.

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

## 3. Resource references — `.vcss` only

A second whitelist holds exactly one allowed resource type: `vcss`. Any node that carries a
compiled-resource reference has its name run through the resource-name normaliser, its
extension lowercased and truncated at the first `_` (so `vcss_c` collapses to `vcss`), and the
result compared against that list.

- ✅ `<styles><include src="s2r://<path>.vcss" /></styles>` — any path; no prefix check,
  so an addon may ship its stylesheet anywhere in its content tree;
- ❌ every other resource: `.vjs`, a nested `.vxml`, `.vsnd`, or `.vtex` via `s2r://`.

Rejection → `Layout contains reference to disallowed resource type '%s'.`

> For `<Image>` this restriction does not bite in practice: `src` is an attribute *value*, and
> values are not walked (see §1). Use non-compiled forms such as `file://{images}/...`, which
> compile to a plain string node rather than a resource reference.

## 4. Shape of a valid document

```xml
<root>
    <styles>
        <include src="s2r://panorama/styles/my_hud.vcss" />
    </styles>

    <Panel id="Root" class="Root" hittest="false">
        <Label  id="..." class="..." hittest="..." text="..." />
        <Image  id="..." class="..." hittest="..." src="..."
                texturewidth="..." textureheight="..." />
        <Button id="..." class="..." />
        <!-- arbitrary nesting of these four -->
    </Panel>
</root>
```

There are **no** limits on depth, node count, attribute count or file size — the walk is
directly recursive and unbounded. A document whose root resolves to nothing validates as a pass.

## 5. The host document you are mounted into

`panorama/layout/hud/customhuds.vxml`:

```xml
<root>
    <styles>
        <include src="s2r://panorama/styles/base.vcss" />
    </styles>
    <Panel class="WindowRoot" hittest="false">
        <CSGOCustomHud id="CustomHud" style="width: 100%; height: 100%; flow-children: none;" />
    </Panel>
</root>
```

- the top-level window is named `CSGOCustomHuds` and loads
  `file://{resources}/layout/hud/customhuds.xml`;
- `base.vcss` is the only stylesheet included, and it consists of exactly one rule:
  `.WindowRoot{width: 100%;height: 100%;}`. You inherit **no** class or token vocabulary;
- the `CSGOCustomHud` mount point is full-screen with `flow-children: none`, so your root
  panel is positioned/aligned rather than flowed;
- beneath it the engine creates a `CSGOCustomHudLayoutRoot`, one per entity, **with no `id`
  and no `class`**. It is still usable as a type selector in your stylesheet
  (`CSGOCustomHudLayoutRoot { ... }`).

## 6. When validation runs

Validation is invoked from three places:

1. loading the layout named by the replicated `m_strLayout` field — the main path;
2. the panel factory, re-validating before allocating the panel object;
3. the resource hot-reload handler: valid and no panel yet → create;
   invalid and a panel exists → **destroy it**.

The panel is created and the `CustomHudLayout` JS global registered only after validation
passes. There is no fallback layout — a rejected document yields a blank screen, never a
partial render.
