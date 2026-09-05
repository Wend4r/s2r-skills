# Patterns for a custom hud

A custom hud has no client script, no inline styles, and no way to send a number. Everything it
does is a class toggled or a string substituted, on a panel found by `id`. Those constraints do
not merely restrict a design — they dictate one, and the shapes below are what a working hud
converges on.

## Contents

- [The constraints that force everything else](#the-constraints-that-force-everything-else)
- [Write the smallest markup that can be addressed](#write-the-smallest-markup-that-can-be-addressed)
- [Addressing discipline](#addressing-discipline)
- [Repeated rows differ by id, by class, or by both — pick deliberately](#repeated-rows-differ-by-id-by-class-or-by-both-pick-deliberately)
- [Own dialog variables on an ancestor](#own-dialog-variables-on-an-ancestor)
- [Put screen-wide state on one panel](#put-screen-wide-state-on-one-panel)
- [Hide with visibility: collapse, and author the hidden state](#hide-with-visibility-collapse-and-author-the-hidden-state)
- [The class vocabulary is the server's API](#the-class-vocabulary-is-the-servers-api)
- [The stylesheet cannot do exclusion](#the-stylesheet-cannot-do-exclusion)
- [When you cannot build a value, enumerate it](#when-you-cannot-build-a-value-enumerate-it)
- [Split the feedback: hover is free](#split-the-feedback-hover-is-free)
- [Procedural effects from static markup](#procedural-effects-from-static-markup)
- [Panels the engine makes for you](#panels-the-engine-makes-for-you)
- [Habits that break here](#habits-that-break-here)
- [Budget arithmetic](#budget-arithmetic)
- [Open questions](#open-questions)

## The constraints that force everything else

Each is enforced by a mechanism in the shipped modules, named here so it can be re-checked after a
game update. No addresses are involved; the identifiers are the anchors.

| Constraint | What enforces it |
|------------|------------------|
| Four tags, fixed attribute sets, no `style`, no handlers | the whitelist the validator builds on first use |
| `<scripts>`, `<snippets>` and inline script bodies rejected | an AST node-kind check, not a tag-name check |
| Only a `.vcss` may be referenced from the layout | a second whitelist holding exactly one resource type |
| A stylesheet's `url()` is not subject to that rule | the resource check belongs to the XML AST and never descends into a stylesheet |
| Panels are addressed by `id` | the server's calls resolve the panel with `FindChildTraverse` |
| A click returns the `Button`'s `id`, nothing else | the click user message and the script event it raises |
| Three interning pools, 1 024 entries each | the cap and its warning, one per pool: panel ids, class names, dialog-variable names |
| Exactly 140 style properties exist | a single compile-time registration table, bounded by its own assertion |

Full derivation and the strings to search for: [internals.md](internals.md).

## Write the smallest markup that can be addressed

The tempting design is to enumerate the content in markup — one `Button` per item, each item's
name written out as literal `text`. The design that scales keeps a **fixed window of slots** and
pushes every string into a dialog variable, letting the stylesheet absorb the variation, because
the lookup table it holds is the only mechanism for choosing an image (below).

**Put the variation in the stylesheet and keep the markup addressable.**

But be precise about *why*, because the cost is not what it looks like. A large enumerated markup
table is not itself expensive: file size is cheap, the validator's walk is linear and unbounded,
and a panel that carries only static classes and no `id` interns **nothing**. Huge markup tables
are a normal, working shape.

What does not scale is enumerating panels **the server must address individually**. The moment
each row needs its own `id` — because the server sets a class or a dialog variable on that
specific row — the table is spending pool entries, and the pool is 1 024.

So the rule is narrower and more useful than "keep the markup small":

| Shape | Cost |
|-------|------|
| Many panels, static classes, no `id` | free — enumerate as much as you like |
| Many panels, each with an `id` the server drives | one pool entry each; this is the thing that runs out |
| Few panels, driven by classes on a shared ancestor | one entry for the whole table |

Write as much markup as the design needs; be stingy only with the ids and class names the server
actually touches.

## Addressing discipline

`id` is the server's handle; `class` is the styling handle. Keep them disjoint — an id selector
should be a last resort, for the rare panel that needs a rule with no class to hang it on.

Express structure by giving every element a class rather than by walking the tree. Selectors stay
one segment deep, occasionally two; combinators (`>`, `+`, `~`), attribute selectors,
`!important`, variables, `@media` and `@import` have no place here and mostly do not exist.

Give **every** `Button` an `id`. A `Button` without one cannot be identified when it comes back
from the click event — there is no other field to tell buttons apart.

## Repeated rows differ by `id`, by `class`, or by both — pick deliberately

A row in a repeated block needs an `id` if the server must reach *that row*, and its own `class`
if the stylesheet must style *that row differently*. Those are separate questions, and the three
answers are all correct in different places:

```xml
<!-- id only. Rows are interchangeable; the server picks one and fills it.
     Cheapest to style: one rule covers every row. -->
<Panel id="Row0" class="row" hittest="false"> ... </Panel>
<Panel id="Row1" class="row" hittest="false"> ... </Panel>

<!-- class only. Rows are decorative and the server never addresses them; a class on an
     ancestor decides which one shows. Costs nothing from any pool. -->
<Panel class="row slot-0" hittest="false"> ... </Panel>
<Panel class="row slot-1" hittest="false"> ... </Panel>

<!-- both. The row is clickable AND individually styled — a grid cell that reports its
     own click and sits at its own coordinates. -->
<Button id="Cell0" class="cell pos-0"> ... </Button>
<Button id="Cell1" class="cell pos-1"> ... </Button>
```

The mistake is reaching for `id` reflexively because it feels like the row's name. An `id` the
server never passes to `SetHasClass` or `SetDialogVariableString` is free, but it is also
pointless — and the moment you *do* start driving it, it is a pool entry. Decide which of the two
questions the row actually answers.

## Own dialog variables on an ancestor

A `{s:name}` substitution set on a panel reaches that panel's whole subtree. So the label carrying
the placeholder does not need an `id` — the panel above it does, and one call fills several labels
at once.

```xml
<Button id="Slot0" class="slot hidden">
	<Panel class="slot-box">
		<Panel id="SlotIcon0" class="slot-icon" />
		<Label class="slot-badge" text="{s:badge}" />
	</Panel>
	<Label class="slot-title" text="{s:title}" />
	<Label class="slot-sub" text="{s:subtitle}" />
</Button>
```

Three variables, one owner, read by labels one and two levels down — and the **same three names**
are reused verbatim in every slot. There is no `title0 … titleN`.

> **Repeat the name when you repeat the object; repeat the number when the repetition is inside
> one object.** Every slot uses `{s:title}`; the rows of a single multi-line readout use
> `{s:row0} … {s:rowN}`, because they are one object rather than many.

Two ids per slot is worthwhile when the levers are typed differently: one panel receives the
dialog variables and the state classes, the other receives the image-selecting classes.

Avoid using `text` as a format string. Keep it a bare `{s:x}` filling the whole value and push the
literal wording into the string the server sends — otherwise the wording is frozen in a compiled
resource and cannot be changed, localised or corrected without a rebuild.

## Put screen-wide state on one panel

The cheapest lever in the system. Toggle a class on the addressable root and let descendant
selectors restyle everything beneath it:

```css
/* One bit on the root reveals a badge anywhere in the tree. The server never touches the badge
   itself — it only sets "owned" on the root panel. */
.hud-root.owned .badge
{
	visibility: visible;
}

/* Two bits compose. Neither class knows about the other, and the pair selects a variant that
   neither could select alone. */
.hud-root.owned.compact .badge-text
{
	color: #6080ff;
}
```

Because such bits compose, four of them give sixteen states for a pool cost of **one panel id and
four class names**. Reach for this before toggling classes per panel.

## Hide with `visibility: collapse`, and author the hidden state

```css
/* The whole show/hide vocabulary, in one rule. collapse drops the panel out of layout AND out of
   hit-testing, so a hidden Button cannot be clicked. */
.hidden
{
	visibility: collapse;
}
```

Nothing else conceals anything. `opacity: 0` keeps the space **and keeps the panel
hit-testable** — invisible click targets over a grid are a defect, not a style choice. Reserve
opacity for animation.

Ship the hidden state **in the file**. The entity presents the layout to every player the moment
it spawns, so whatever the markup says is what everyone sees before the server has spoken; the
authored state has to be the empty one, and the server reveals by removing the class.

The mirror idiom is worth having too — the child collapses itself and an ancestor reveals it,
which lets the server's lever and the client's polish share a panel without colliding:

```css
/* Authored state: out of layout and fully transparent. The two properties do different jobs —
   visibility is the server's lever, opacity is the client's. */
.overlay
{
	visibility: collapse;
	opacity: 0;
	transition-property: opacity;
	transition-duration: 0.15s;
}

/* The server's half: a state bit on the root plus a per-item flag put the overlay into layout. */
.hud-root.locked .slot.gated .overlay
{
	visibility: visible;
}

/* The client's half: once it is in layout, hover fades it in using the transition the base rule
   already declared. No round trip, no pool entry. */
.hud-root.locked .slot.gated:hover .overlay
{
	opacity: 1.0;
}
```

## The class vocabulary is the server's API

Classes defined in the stylesheet but absent from the markup are not dead code — they are the
contract between the server and the hud. A state can be added to that contract by shipping a
stylesheet alone, with the markup untouched, which makes the class list an interface worth
versioning deliberately.

Only a handful of those names are boolean state. Most are **enumerated values** — a numeric data
channel, described next.

## The stylesheet cannot do exclusion

Nothing links a tab to its page. A stylesheet can highlight the active tab and it can hide a page,
but it cannot express "exactly one of these is visible", so the server holds the current selection
and issues every call itself:

```js
hud.SetHasClass("PageSettings", "hidden", false);
hud.SetHasClass("PageMain", "hidden", true);
hud.SetHasClass("TabMain", "on", false);
hud.SetHasClass("TabSettings", "on", true);
```

Two calls to move a highlight, and a third to hide the outgoing page — because sibling pages in
normal flow would otherwise stack. Lifting a page out of flow removes that third call, which is
why an overlay costs less to show than a flowed page:

```css
/* A pane out of flow: showing it needs one call, because nothing has to move aside for it. */
.modal
{
	ignore-parent-flow: true;
	z-index: 10;
	width: 100%;
	height: 100%;
	background-color: #0000008c;
}
```

## When you cannot build a value, enumerate it

The server toggles a class and sets a string. It cannot send a number, build a URL, or compute
anything, so **every value the hud can ever display must already exist as a class.** Expect the
enumerated tables to dwarf the hand-written styling; a screen of real complexity holds a couple of
hundred authored selectors and thousands of generated ones.

### An image chosen by a compound class

```css
/* One cell of a two-axis table. Neither class appears in the markup — the server sets both on
   the same panel, and their intersection is the only way to choose a texture. */
.kind-4.variant-12
{
	background-image: url( "s2r://panorama/images/custom_game/icons/blade_4_12_png.vtex" );
}
```

The cost is `rows + cols` class names rather than `rows × cols`: a 47 × 48 table needs 2 256 rules
but only 95 class names.

Artwork is not limited to your own content. A stylesheet's `url()` sits outside the XML resource
whitelist, so it may reach anywhere in the mounted tree — including everything the base game
ships, which is often the entire library a hud needs.

### Dense ordinals, not domain identifiers

The subtlest rule here, and the easiest to get wrong. Key the table on **0-based ordinals**, not
on identifiers borrowed from the game's item schema.

| Scheme | Distinct class names |
|--------|----------------------|
| sparse domain ids | over 1 500 |
| dense ordinals | 95 |

Both cover the same catalogue. Sparse identifiers are harmless while they sit statically in the
markup, because the server never passes them to `SetHasClass` and nothing is interned. Drive the
same scheme from the server and it needs more than 1 500 interned class names **against a cap of
1 024** — and entries past the cap are dropped in silence.

> Sparse identifiers are safe baked into markup and dangerous the moment the server drives them.

### No `calc()`, so tables get duplicated

```css
/* The base declares the motion once. Every stop below supplies only a target, so a single class
   swap plays the whole eased slide with no further server involvement. */
.reel
{
	transform: translateX( 0px );
	transition-property: transform;
	transition-duration: 0.30s;
	transition-timing-function: ease-out;
}

/* One of N stops, each written out by hand. */
.reel.stop-0
{
	transform: translateX( 368px );
}

/* And written out again for a narrower layout mode, because the geometry differs and nothing can
   derive one table from the other. */
.hud-root.compact .reel.stop-0
{
	transform: translateX( 92px );
}
```

### A variable-length list without a loop

Declare the maximum number of rows in markup and let the stylesheet collapse the tail. A class
naming the current length hides everything past it:

```css
/* Rows beyond the current length collapse. One rule per (length, row) pair covers every size the
   hud can display. */
.readout.len-8 .row-8
{
	visibility: collapse;
}
```

The server sets the length class on the wrapper and fills only the variables it needs.

## Split the feedback: hover is free

Anything the client can observe by itself — pointer position, press — belongs in a pseudo-class
and costs nothing. Spend a server class only on facts the client cannot derive: persistent
selection, ownership, entitlement. Where a hover effect depends on server state, write
`.state:hover` rather than a second server-toggled class.

```css
/* Free: the client already knows where the pointer is. */
.slot:hover
{
	brightness: 1.12;
}

/* Paid: whether this slot is the selected one is a fact only the server holds. */
.slot.selected .slot-ring
{
	visibility: visible;
}

/* Composed: server state refined by a pseudo-class, at no additional server cost. */
.slot.selected:hover .slot-ring
{
	opacity: 1.0;
}
```

A whole screen's interactive surface comes to a couple of dozen `:hover` rules. Anything more
usually means state that should have stayed on the client.

**Turning a hover response back off needs no second class.** A disabled button should not light up
under the pointer, and the way to stop it is to re-declare the same value on the disabled hover
rule — the transition then has nowhere to go:

```css
.btn
{
	background-color: #3c4248;
}

.btn:hover
{
	background-color: #4d545b;
}

/* One server class disables it. The third rule is what kills the hover, by making the hovered
   value identical to the resting one. */
.btn.disabled
{
	background-color: #282d32;
}

.btn.disabled:hover
{
	background-color: #282d32;
}
```

Without the last rule `.btn:hover` still matches and the button still lights up while disabled.
This is the general shape: a state class that must suppress a pseudo-class effect has to say so,
because the pseudo-class rule does not stop matching.

## Procedural effects from static markup

Animated ornament has to exist as panels, because there is nothing to generate them at runtime.
Budget for that: a screen with real motion can spend a quarter of its elements on nodes that exist
only so keyframes have something to animate.

Two desynchronisation techniques are worth knowing, and the difference between them matters.

A **one-shot burst** staggers on several coprime moduli at once — delay, duration, keyframe, size
and colour each cycling on a different count — so the least common multiple far exceeds the number
of panels and the run never visibly repeats.

An **infinite loop** needs no delay at all: unequal periods decorrelate on their own, so a set of
idle animations gets its shimmer from durations alone.

Hoist everything shared into the base class and write only the varying part per index. The
keyframe library and the staggering recipe are in [css.md](css.md).

## Panels the engine makes for you

The whitelist governs what you may write, not what may exist. A scrolling panel gets a scrollbar
built from panel types no layout may name — and they are still addressable from the stylesheet, by
type and by class:

```css
/* fill-parent-flow is the flex-grow equivalent and right-wrap makes the grid wrap. The second
   overflow axis is what makes the engine build a scrollbar at all. */
.list
{
	height: fill-parent-flow( 1.0 );
	flow-children: right-wrap;
	overflow: clip scroll;
}

/* A panel type no layout may name, reached as a type selector. layout-position: fixed pins it
   outside the scrolled content so it does not travel with the rows. */
.list VerticalScrollBar
{
	width: 6px;
	layout-position: fixed;
}

/* The draggable part, reached as a class. min-height keeps it grabbable when the list is long. */
.list VerticalScrollBar .ScrollThumb
{
	min-height: 32px;
	border-radius: 2px;
	opacity: 0.4;
}

/* A pseudo-class on an engine-created panel behaves exactly as it does on your own. */
.list VerticalScrollBar:hover .ScrollThumb
{
	opacity: 0.6;
}
```

A scrolling list needs nothing else, and this is the only way to get one.

## Habits that break here

All documented in [css.md](css.md).

- **`background-size: contain`**, not `contains`. The engine's own help string is wrong, the value
  parser reads `contain`, and an unknown keyword is dropped silently.
- **Gradients are a `background-color`, never a `background-image`.** `background-image` is
  exclusively `url()`, one per declaration, never comma-layered.
- **`@keyframes` names are quoted at the definition and unquoted at the reference.** Both forms are
  required.
- **Neither `animation:` nor `transition:` has a usable shorthand** — write the longhands.
- Colours are hex only, with alpha in the eight-digit form; `rgba()` never appears.

## Budget arithmetic

Three pools of 1 024, counting only what the **server** references. A hud that exceeds one does not
slow down; the excess references simply stop working, with a warning in the log and no symptom on
screen. That is the failure mode to design against, because nothing on screen will tell you.

A hud built around a **fixed window of slots** is safe by construction: its id count is decided by
the layout, not by the size of the catalogue behind it.

A hud built around a **catalogue** is not, and its two apparent overflows are not equally real:

- **Class names are usually a false alarm.** If they sit statically in the markup the server never
  passes them to `SetHasClass` and nothing is interned.
- **Panel ids are the real exposure.** Structural ids are bounded and cheap. What is unbounded is
  *selection*: a per-item class must be set on an individual item, so every distinct item a player
  touches interns one more id.

That gives a soft cliff partway through a long session, with no failure mode except silently
ceasing to update. Two habits keep the budget small: **address an ancestor** so one id serves a
subtree, and **toggle a mode on the root** so one class name serves a screen.

Three further properties of the pools are worth knowing, because each changes a decision.

- **A name in the stylesheet costs nothing.** Only names the server actually passes to
  `SetHasClass` or `SetDialogVariableString` are interned. A generated sheet may declare tens of
  thousands of class names against a cap of 1 024 and never come near it — what counts is how many
  *distinct* ones the server names over the entity's lifetime.
- **There is no eviction.** An entry is added on first reference and stays for as long as the
  entity lives, so the number that matters is a high-water mark, not a concurrent count. A hud
  that walks a large catalogue over a long map will keep climbing. Recreating the entity is the
  only reset.
- **The pools are replicated.** All three are networked vectors of strings, so every distinct name
  the server references is sent to clients as text. Short names are therefore worth something
  beyond tidiness — though only in bandwidth. They do not buy pool entries: `a` and
  `SlotIconHighlight` each cost exactly one.

## Open questions

- Whether Panorama replays a one-shot `@keyframes` when a panel goes `collapse` → `visible`. This
  decides whether an animation can be re-triggered by re-hiding and re-showing its host.
- The realistic interning cost of a catalogue-shaped hud depends on player behaviour across a
  session, so only the ceiling and the structural floor can be reasoned about in advance.
