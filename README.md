# Source 2 Skills

A collection of agent skills for working with Source 2 resources and developing related code.
It covers resource formats, engine APIs, tooling and practical workflows for maps and plugins.


## Custom Hud Layout

[Skill](custom-hud-layout/SKILL.md) covers the `custom_hud_layout` entity (`CCSCustomHudLayout`): how to create a HUD, update it for all players or one player, and handle button clicks on the server.

Use it when building timers, scoreboards, progress indicators, shops, voting dialogs or menus. It includes a capture-point indicator example, API references and common validation errors.

### How it works

XML defines the panels. A Panorama stylesheet defines their appearance. Server code updates text through dialog variables and changes visual states by toggling classes.

The layout accepts four panel types: `Panel`, `Label`, `Image` and `Button`. Each has a fixed set of attributes. Inline styles, client scripts and XML event handlers are not supported.

A HUD starts as an overlay. Enabling input capture for a player gives them a cursor and allows interaction. Button clicks arrive on the server through `OnCustomHudClicked`, with the player, layout and button ID

### Installation

Install the skill globally for your agent. This requires Node.js and npm.

Restart the current agent session after installation. The skill is available in every project and can be selected automatically when the task matches its description.

To install it only for the current project, run the command from the project root without `--global`.

<details>
<summary>Installation commands per agent</summary>

#### Claude Code (CLI and VS Code)

```sh
npx skills add Wend4r/s2l-skills --skill custom-hud-layout --global --agent claude-code
```

#### Codex

```sh
npx skills add Wend4r/s2l-skills --skill custom-hud-layout --global --agent codex
```

#### Cursor

```sh
npx skills add Wend4r/s2l-skills --skill custom-hud-layout --global --agent cursor
```

#### GitHub Copilot

```sh
npx skills add Wend4r/s2l-skills --skill custom-hud-layout --global --agent github-copilot
```

#### Cline

```sh
npx skills add Wend4r/s2l-skills --skill custom-hud-layout --global --agent cline
```

#### OpenClaw

```sh
npx skills add Wend4r/s2l-skills --skill custom-hud-layout --global --agent openclaw
```

</details>

### What the skill covers

- **Layout:** supported XML tags and attributes, panel IDs, visibility and scrolling
- **Styling:** sizing, alignment, colours, images, SVG icons, custom fonts and video backgrounds
- **Interaction:** input capture, button clicks, hover effects and per-player state
- **Animation:** transitions and keyframes, with visual states selected through classes
- **Text:** dialog variables and localisation tokens
- **Practical patterns:** reusable slots, tabs, menus and image selection without growing the layout for every item in a catalogue
- **Limits and debugging:** resource paths, rejected layouts and the 1024-entry pools for panel IDs, class names and dialog-variable names referenced by the server

### Files

| File | Contents |
|------|----------|
| [SKILL.md](custom-hud-layout/SKILL.md) | Entry point, quick-start example and common errors |
| [xml.md](custom-hud-layout/references/xml.md) | Allowed panels, attributes and resource references |
| [css.md](custom-hud-layout/references/css.md) | Panorama style properties, assets and animations |
| [entity.md](custom-hud-layout/references/entity.md) | Server API, input capture, click events and entity schema |
| [patterns.md](custom-hud-layout/references/patterns.md) | Practical layout and state-management patterns |
| [internals.md](custom-hud-layout/references/internals.md) | Research sources, confidence notes and checks after game updates |

## Resource Compiler

[Skill](resource-compiler/SKILL.md) covers compiling addon sources with the Resource Compiler shipped in a Source 2 game's Workshop Tools on Windows: finding the installation, running the compiler, locating the outputs and reading a failure.

Use it when Panorama XML and CSS, materials, models, textures, particles or sounds have to become `_c` resources the game can load, or when a Hammer map build needs more than a bare `-i` call. It does not package a VPK or publish a Workshop item.

### How it works

The game installation is resolved from the registry rather than hardcoded. `HKCU:\SOFTWARE\Valve\Steam` gives the Steam path, `steamapps/libraryfolders.vdf` gives the library holding the app ID, and that library's `appmanifest_<appId>.acf` gives the folder under `steamapps/common`. Only the `steamapps\...` tail is fixed — the drive and library name differ per machine.

Sources live under the addon's `content/` tree and compiled resources appear under its `game/` tree, keeping the same resource-relative directories. Each input is passed to `resourcecompiler.exe -i` from the compiler's own directory, with the exit code checked between files.

Success is judged from the compiler summary, the exit code and the files actually written — not from the process merely returning. A trailing `Leaked KeyValues blocks` diagnostic accompanies successful runs and is not a failure on its own.

### Installation

Same requirements and options as above: Node.js and npm, a session restart afterwards, and no `--global` to install it only for the current project.

<details>
<summary>Installation commands per agent</summary>

#### Claude Code (CLI and VS Code)

```sh
npx skills add Wend4r/s2l-skills --skill resource-compiler --global --agent claude-code
```

#### Codex

```sh
npx skills add Wend4r/s2l-skills --skill resource-compiler --global --agent codex
```

#### Cursor

```sh
npx skills add Wend4r/s2l-skills --skill resource-compiler --global --agent cursor
```

#### GitHub Copilot

```sh
npx skills add Wend4r/s2l-skills --skill resource-compiler --global --agent github-copilot
```

#### Cline

```sh
npx skills add Wend4r/s2l-skills --skill resource-compiler --global --agent cline
```

#### OpenClaw

```sh
npx skills add Wend4r/s2l-skills --skill resource-compiler --global --agent openclaw
```

</details>

### What the skill covers

- **Installation lookup:** Steam registry keys, `libraryfolders.vdf`, app manifests and the CS2 paths relative to a library root
- **Compilation:** a PowerShell loop with input checks, exit-code handling and a restored working directory
- **Resource types:** a worked addon tree covering Panorama layouts, stylesheets, SVG and textures, plus materials, models, particles and sound events — with the raw images, meshes and audio each one is built from
- **Hammer maps:** why a full `.vmap` build is more than one compiler call
- **Verification:** summaries, exit codes, output timestamps and the source-to-output naming, including the `.vxml` name a `custom_hud_layout` entity expects
- **Diagnostics:** missing tools, missing dependencies, unsupported inputs, validation errors and access-denied writes

### Files

| File | Contents |
|------|----------|
| [SKILL.md](resource-compiler/SKILL.md) | Entry point, the pipeline and the rules that hold everywhere |
| [installation.md](resource-compiler/references/installation.md) | Steam registry lookup, library roots, app IDs, `content/` vs `game/`, per-game addon directories |
| [resources.md](resource-compiler/references/resources.md) | Definitions vs raw assets, a worked addon tree, the per-type table and the compile order |
| [compile.md](resource-compiler/references/compile.md) | The compile loop, choosing inputs, Hammer builds, verification and failure diagnosis |

## License

[MIT](LICENSE)
