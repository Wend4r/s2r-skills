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

## License

[MIT](LICENSE)
