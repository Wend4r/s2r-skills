# Custom Hud Skills

Skills for working with custom HUDs in CS2 (Source 2 / Panorama).

## Contents

| Skill | Purpose |
|-------|---------|
| [custom-hud-layout](custom-hud-layout/SKILL.md) | The `custom_hud_layout` entity (`CCSCustomHudLayout`): XML markup, VCSS styling, server-side JS API, click handling |

## Provenance

This material was obtained by reverse engineering, not from documentation — no public
documentation for `CCSCustomHudLayout` exists, and the game ships no example of a custom hud.
It was recovered from the shipped client and server modules together with the compiled
Panorama assets (`.vxml_c` / `.vcss_c`).

The documentation is written against **durable identifiers** — log strings, class and field
names, method names, enum names and values — rather than addresses or offsets, which change
with every game update. If something needs re-checking after a patch, the anchors to search
for and the method to use are listed in
[reference-internals.md](custom-hud-layout/reference-internals.md).

## Key conclusions in two paragraphs

A custom hud is a heavily reduced subset of Panorama. The validator admits **only** the tags
`Panel`, `Label`, `Image` and `Button`, each with a fixed attribute set (`id`, `class`,
`hittest`, `text`, `src`, `texturewidth`, `textureheight`). There is no `style` attribute, so
inline styling is impossible; there are no event handlers in markup; and `<scripts>`,
`<snippets>` and `<snippet>` are rejected at the AST node-kind level. The only resource that
may be referenced is a `.vcss`.

All dynamic behaviour comes from the server. The entity replicates interned tables of panel
ids, class names and dialog-variable names (capped at 1024 entries each), and its state —
global plus one per player slot — toggles classes and substitutes strings. Clicks travel back
as the `CS_UM_CustomHudClicked` user message and surface in server-side JS as the
`OnCustomHudClicked` event carrying `{ player, layout, buttonId }`.
