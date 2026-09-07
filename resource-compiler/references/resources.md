# Resource types and their dependencies

What is a compiler input, what is only an ingredient, and what has to be built first.

## Contents

- [Definitions and raw assets](#definitions-and-raw-assets)
- [A worked addon tree](#a-worked-addon-tree)
- [The type table](#the-type-table)
- [What a definition file looks like](#what-a-definition-file-looks-like)
- [Compile order](#compile-order)
- [Panorama naming](#panorama-naming)
- [Decompiled files are not sources](#decompiled-files-are-not-sources)

## Definitions and raw assets

An addon's `content/` tree holds two different kinds of file, and only one of them is something
you hand to the compiler.

- **Definitions** are small text files — KeyValues, KV3 or DMX — that describe a resource and name
  the assets it is built from. `.vmat`, `.vmdl`, `.vtex`, `.vsnd`, `.vsndevts`, `.vpcf` and the
  Panorama `.xml` / `.css` are all definitions. Each compiles to the same name plus `_c`.
- **Raw assets** are the images, meshes and audio the definitions point at: `.png`, `.gif`,
  `.tga`, `.dmx`, `.wav`, `.mp3`. They have no `_c` counterpart of their own. They reach the game
  *through* a definition.

This is the concrete form of the rule that an arbitrary raw file is not a standalone compiler
input. Pointing `-i` at a `.png` that belongs to a material, or at a `.dmx` that belongs to a
model, is a category error — compile the `.vmat` or the `.vmdl` that names it.

The exception worth knowing: a material can name a raw image directly and let the compiler
generate the texture for it, so many `.png` files have no `.vtex` beside them. An explicit
`.vtex` exists when the texture needs settings of its own — colour space, output format,
processing steps — or when something references the texture by name without going through a
material.

## A worked addon tree

One addon covering every common resource type. Two placeholders: `GAME_addons` is the game's
addon directory, and `ADDON` is the addon's name. `ADDON` appears twice over — once as the addon
directory itself, and again as the namespace its resources sit under inside `materials/`,
`models/`, `panorama/` and the rest. Keeping the two the same is the convention that stops one
addon's resources from colliding with another's.

```text
content/GAME_addons/ADDON/
├── materials/ADDON/
│   ├── surface.vmat                 ← definition: shader + texture references
│   ├── surface_color.png            ← raw, named by surface.vmat
│   ├── surface_normal.png           ← raw, named by surface.vmat
│   └── surface_color_png.vtex       ← optional explicit texture definition
├── models/ADDON/
│   ├── prop.vmdl                    ← definition: meshes, physics, materials
│   ├── prop_render.dmx              ← raw render mesh, named by prop.vmdl
│   └── prop_phys.dmx                ← raw physics mesh, named by prop.vmdl
├── panorama/
│   ├── images/ADDON/
│   │   ├── logo_png.png             ← raw, named by logo_png.vtex
│   │   ├── logo_png.vtex            ← definition: the texture the hud references
│   │   ├── icon.svg                 ← definition and asset in one file
│   │   └── spinner.gif              ← raw
│   ├── layout/custom_game/ADDON/
│   │   └── main.xml                 ← definition: panels, includes the stylesheet
│   └── styles/custom_game/ADDON/
│       └── main.css                 ← definition: the stylesheet
├── particles/ADDON/
│   └── effect.vpcf                  ← definition: emitters, references materials
├── soundevents/
│   └── soundevents_ADDON.vsndevts   ← definition: named events over sounds
└── sounds/ADDON/
    ├── click.wav                    ← raw
    ├── theme.mp3                    ← raw
    └── click.vsnd                   ← definition: the sound resource
```

Every compiled counterpart lands under `game/GAME_addons/ADDON/` at the same resource-relative
path — `materials/ADDON/surface.vmat_c`, `panorama/layout/custom_game/ADDON/main.vxml_c`,
and so on.

## The type table

| Source | Output | What it is | Built from | Referenced as |
|---|---|---|---|---|
| `.css` | `.vcss_c` | Panorama stylesheet | textures, fonts | `s2r://…/main.vcss_c` in an `<include>` |
| `.xml` | `.vxml_c` | Panorama layout | the compiled stylesheet | `.vxml` logical name (see below) |
| `.svg` | `.vsvg_c` | vector image | — | `s2r://…/icon.vsvg`, no format infix |
| `.vtex` | `.vtex_c` | texture definition | one raw image | `s2r://…/logo_png.vtex` |
| `.vmat` | `.vmat_c` | material | raw images, or a `.vtex` | by material path, from models, particles and maps |
| `.vmdl` | `.vmdl_c` | model | `.dmx` meshes, materials | by model path, from entities and maps |
| `.vpcf` | `.vpcf_c` | particle system | materials | by particle path |
| `.vsnd` | `.vsnd_c` | sound resource | one `.wav` or `.mp3` | from a sound event |
| `.vsndevts` | `.vsndevts_c` | sound event list | the compiled sounds | by event name, from code and maps |
| `.vmap` | `.vmap_c` | map | everything above | by map name |
| `.png` `.gif` `.tga` | — | raw image | — | only through a `.vtex` or a `.vmat` |
| `.dmx` | — | raw mesh | — | only through a `.vmdl` |
| `.wav` `.mp3` | — | raw audio | — | only through a `.vsnd` |

`.vmap` is in the table for completeness only. A real map build is more than one `-i` call — see
[compile.md](compile.md), *Hammer maps*.

Treat this table as the shape to expect, not as a closed list. The installed game and its
Workshop Tools decide which types exist; check the compiler's help output and the relevant
editor before assuming a type is supported.

## What a definition file looks like

The formats differ, but every one of them names its ingredients by resource-relative path. That
is what makes dependencies findable — read the definition to learn what has to be compiled first.

A texture definition, DMX, naming one image:

```text
<!-- dmx encoding keyvalues2_noids 1 format vtex 1 -->
"CDmeVtex"
{
    "m_inputTextureArray" "element_array"
    [
        "CDmeInputTexture"
        {
            "m_name" "string" "InputTexture0"
            "m_fileName" "string" "panorama/images/ADDON/logo_png.png"
            "m_colorSpace" "string" "srgb"
        }
    ]
    "m_outputFormat" "string" "BGRA8888"
}
```

A material, KeyValues, naming a shader and its textures:

```text
"Layer0"
{
	"shader"	"csgo_projected_decals.vfx"
	"TextureColor"	"materials/ADDON/surface_color.png"
	"TextureNormal"	"materials/ADDON/surface_normal.png"
}
```

A model, KV3, naming its meshes:

```text
<!-- kv3 … format:modeldoc28… -->
{
	rootNode =
	{
		children =
		[
			{ _class = "RenderMeshFile"  filename = "models/ADDON/prop_render.dmx" },
			{ _class = "PhysicsMeshFile" filename = "models/ADDON/prop_phys.dmx" },
		]
	}
}
```

A sound event list, KV3, naming events rather than files:

```text
<!-- kv3 … format:generic… -->
{
	ADDON.click =
	{
		type = "csgo_mega"
		volume = 1.0
		vsnd_files = [ "sounds/ADDON/click.vsnd" ]
	}
}
```

## Compile order

Dependencies point downwards, so build from the bottom up. Within a level, order does not matter.

```
        .vsndevts          .vmap            .vxml
            │            (full build)         │
         .vsnd          ┌──┴───┐           .vcss
            │           │      │
       .wav / .mp3   .vmdl   .vpcf
                        │      │
                     .dmx   .vmat
                               │
                         .vtex / raw images
```

Practical consequences:

- Compile a stylesheet before the layout that includes it. A layout built against a missing
  `.vcss_c` fails as a dependency error, and one built against a stale `.vcss_c` succeeds while
  producing the wrong appearance.
- Compile textures and materials before models and particles that use them.
- Compile sounds before the `.vsndevts` that lists them.
- When only one file changed, compile that file. The order above matters when a dependency
  changed too — not as a reason to rebuild the whole addon every time.

## Panorama naming

Panorama is where the logical name and the file on disk differ most, so it is worth stating
separately:

| Source | File on disk after compiling | Written in a reference as |
|---|---|---|
| `main.css` | `main.vcss_c` | `s2r://…/main.vcss_c` — with `_c` |
| `main.xml` | `main.vxml_c` | `…/main.vxml` — without `_c`, in a `custom_hud_layout` `layout` keyvalue |
| `logo.png` | `logo_png.vtex_c` | `s2r://…/logo_png.vtex` — the source format becomes an infix |
| `icon.svg` | `icon.vsvg_c` | `s2r://…/icon.vsvg` — vector is its own type, no infix |

An explicit `.vtex` compiles under its own name, so a definition called `logo_png.vtex` yields
`logo_png.vtex_c` regardless of what its input image is called. Naming the explicit definition
the way the implicit convention would have named it keeps every reference identical either way.

Derive names for other resource types from their own compiler or editor output rather than by
analogy with these.

## Decompiled files are not sources

A tree extracted from shipped game resources looks like a source tree but is not one. Tell-tale
markers show up inside the text definitions — an exported-sound wrapper around what should be a
plain sound definition, a `Compiled Textures` block appended to a material, generated file names
carrying a hash infix such as `surface_8b02488_trans.png`.

Such files usually recompile, but they are a round trip through a decompiler, not the author's
input, and the result can differ from the original in ways the compiler will not report. When a
file shows these markers, say so rather than presenting the rebuild as equivalent to the original
asset.
