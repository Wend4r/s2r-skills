# Running the compiler and reading the result

Assumes the installation root and the addon directory are already resolved — see
[installation.md](installation.md).

## Contents

- [The minimal command](#the-minimal-command)
- [A compilation run](#a-compilation-run)
- [Choosing the inputs](#choosing-the-inputs)
- [Other resource types](#other-resource-types)
- [Hammer maps](#hammer-maps)
- [Verify the result](#verify-the-result)
- [Source-to-output names](#source-to-output-names)
- [Diagnose failures](#diagnose-failures)

## The minimal command

```powershell
& $compiler -i $sourcePath
```

Three things make this work reliably:

- **PowerShell's call operator.** A path in a variable is not executed without `&`.
- **Absolute input paths.** Relative paths resolve against the working directory, which is about
  to change.
- **The compiler's own directory as the working directory.** `resourcecompiler.exe` loads engine
  libraries from beside itself.

Do not add recursive whole-addon compilation, forced rebuilds, lighting bakes, output-root
overrides, or packaging flags by default. Apply them when the requested scope requires them and
the installed tool supports them.

## A compilation run

Replace the source files with the resources the user actually asked for:

```powershell
# $installationRoot comes from the lookup in installation.md, or from the user.
$compiler = Join-Path $installationRoot 'game\bin\win64\resourcecompiler.exe'
$addonContent = Join-Path $installationRoot 'content\GAME_addons\ADDON'

$sourceFiles = @(
    (Join-Path $addonContent 'panorama\styles\custom_game\ADDON\main.css')
    (Join-Path $addonContent 'panorama\layout\custom_game\ADDON\main.xml')
)

if (-not (Test-Path -LiteralPath $compiler -PathType Leaf)) {
    throw "Resource Compiler not found: $compiler"
}
foreach ($sourcePath in $sourceFiles) {
    if (-not (Test-Path -LiteralPath $sourcePath -PathType Leaf)) {
        throw "Source file not found: $sourcePath"
    }
}

Push-Location -LiteralPath (Split-Path -Parent $compiler)
try {
    foreach ($sourcePath in $sourceFiles) {
        & $compiler -i $sourcePath
        if ($LASTEXITCODE -ne 0) {
            throw "Compilation failed ($LASTEXITCODE): $sourcePath"
        }
    }
}
finally {
    Pop-Location
}
```

What each part is for:

- **Both existence checks run before the first compile.** A typo in the fourth path is reported
  before three files have been written.
- **`$LASTEXITCODE` is checked between files, not at the end.** A failed stylesheet stops the run
  instead of letting the layout compile against a stale dependency.
- **`finally` restores the working directory.** Without it a thrown error leaves the session
  inside the compiler's `bin\win64`, and every later relative path in the session is wrong.

`GAME_addons` and `ADDON` are placeholders — substitute the game's addon directory and the
addon's own name, which is also the namespace its resources sit under.

Compile known dependencies first. In the example the layout includes the compiled stylesheet, so
the CSS goes first; a layout compiled against a missing `.vcss_c` is a dependency error, not a
syntax error. The full dependency order across resource types is in
[resources.md](resources.md), *Compile order*.

## Choosing the inputs

- Discover the files actually present rather than assuming a project layout.
- Pass **source** resources to the compiler. An existing `_c` file is output, never input.
- Pass **definitions**, not raw assets. `.png`, `.dmx`, `.wav` and `.mp3` are ingredients named
  inside a `.vtex`, `.vmat`, `.vmdl` or `.vsnd`; the definition is the input.
- Compile the scope the user asked for. Do not sweep the whole addon because it seems tidier.

## Other resource types

Which types are supported, and which build options exist, is decided by the installed game and
compiler — not by this document. [resources.md](resources.md) has the per-type table, a worked
addon tree and the dependency order.

- Materials, models, texture definitions and particles reference other files: images, meshes,
  other materials. Keep the expected directory structure and read dependency errors rather than
  compiling every file indiscriminately.
- An arbitrary raw file is not necessarily a standalone compiler input. A `.png`, `.dmx`, `.wav`
  or `.mp3` normally reaches the game through a `.vtex`, `.vmat`, `.vmdl` or `.vsnd` that names
  it; compile that definition instead. Where a raw asset does need an authoring resource built
  around it, use the corresponding Workshop Tools editor or the compiler's documented import
  workflow.
- Never invent a source-to-output extension mapping, and never rename a raw file to a compiled
  extension. A `_c` file is a compiler product with its own container format.
- For an unfamiliar resource type, inspect the installed compiler's help output and the relevant
  editor's build log before picking extra arguments. Consult documentation for the matching game
  when local evidence is not enough.

## Hammer maps

A complete `.vmap` build can involve geometry, visibility, physics, lighting, navigation and
packaging. A bare `-i` call is **not** evidence that all of those stages ran.

Use Hammer's requested build preset, or reproduce the command Hammer logs for that preset with
the same game-specific options and output paths. Reporting a one-line `-i` invocation as a
finished map build misrepresents what was produced.

## Verify the result

1. **Read the exit code and the compiler summary.** Inspect reported errors even when the process
   returned zero. A summary such as `OK: 1 compiled, 0 failed, 0 skipped` is the claim to check
   against; a skipped up-to-date resource can be a perfectly valid outcome.
2. **Confirm the output files exist at their real paths.** For a newly compiled resource, check
   timestamps and sizes — an output left over from an earlier run proves nothing about this one.
3. **Report what happened per input:** compiled, failed or skipped, and where the outputs landed.

Keep the boundaries explicit when reporting. Successful resource compilation is not runtime
validation in the game, not a complete map build, not VPK packaging, and not Workshop publication.

## Source-to-output names

The per-type mapping lives in [resources.md](resources.md), *The type table*. The Panorama cases
are repeated here because they are the ones that bite:

| Source under the addon's content root | Output under the addon's game root |
|---|---|
| `panorama/styles/custom_game/ADDON/main.css` | `panorama/styles/custom_game/ADDON/main.vcss_c` |
| `panorama/layout/custom_game/ADDON/main.xml` | `panorama/layout/custom_game/ADDON/main.vxml_c` |

For `custom_hud_layout`, the entity's `layout` keyvalue uses the **logical** resource name
`panorama/layout/custom_game/ADDON/main.vxml`, without `_c`, while the file on disk is the
compiled `.vxml_c`. That split is specific to Panorama; derive other resource names from their
own compiler or editor output rather than by analogy.

## Diagnose failures

| Symptom | Where to look |
|---|---|
| Missing executable | The selected installation and whether its Workshop Tools are installed at all |
| Missing input or dependency | Absolute paths, addon context, resource references, and whether the dependency was compiled first |
| Unsupported input or options | The compiler's help output and the matching editor's workflow |
| Syntax or validation error | The concrete file and diagnostic the compiler named |
| Access denied | The output tree under `game\`; use the environment's normal escalation mechanism |

Rules that apply to all of them:

- Do not repeatedly retry an unchanged command. Change the input, the arguments or the
  understanding first.
- Change source files only within the scope the user requested. A compiler complaint is not
  permission to rewrite the project.
- On an access-denied write, preserve the requested destination. Silently substituting another
  addon produces resources the game will never load.
- `Leaked KeyValues blocks: <n>` accompanies successful runs as well as failed ones. On its own it
  does not establish failure — judge by the summary, the exit code and the generated outputs.
