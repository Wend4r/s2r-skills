# Finding the game and its directories

Everything else depends on one absolute path: the installation root of the target game.
Nothing in this file guesses it. Resolve it, verify it, then build every other path from it.

## Contents

- [Resolve the installation root](#resolve-the-installation-root)
- [The lookup, end to end](#the-lookup-end-to-end)
- [Library roots vary per machine](#library-roots-vary-per-machine)
- [Finding the app ID](#finding-the-app-id)
- [content/ and game/](#content-and-game)
- [Addon directories per game](#addon-directories-per-game)
- [The compiler binary](#the-compiler-binary)
- [When the lookup fails](#when-the-lookup-fails)

## Resolve the installation root

Use the path the user gave. When they gave none, resolve it — do not hardcode one and do not
assume the game sits next to Steam itself, because a Steam library can live on any drive.

| Step | Source | What it yields |
|---|---|---|
| 1 | `HKCU:\SOFTWARE\Valve\Steam` → `SteamPath` | where Steam itself is installed |
| 2 | `HKLM:\SOFTWARE\WOW6432Node\Valve\Steam` → `InstallPath` | the same, machine-wide fallback |
| 3 | `<steam>\steamapps\libraryfolders.vdf` | every library root, each with an `apps` block |
| 4 | `<library>\steamapps\appmanifest_<appId>.acf` → `installdir` | the folder name under `steamapps\common` |
| 5 | `<library>\steamapps\common\<installdir>` | the installation root |

Details that matter:

- Prefer the per-user key and fall back to the machine-wide one. `WOW6432Node` appears because
  Steam is a 32-bit application on 64-bit Windows.
- `SteamPath` is stored with forward slashes and in lower case. Normalize the separators before
  joining paths onto it.
- There is no registry value for a game's own directory. Step 3 exists precisely because the
  game need not be in the same place as Steam.
- Paths inside `.vdf` and `.acf` files are escaped with doubled backslashes; unescape them.

## The lookup, end to end

```powershell
$appId = '730'  # Replace with the target game's Steam app ID.

$steamPath = (Get-ItemProperty 'HKCU:\SOFTWARE\Valve\Steam' -ErrorAction SilentlyContinue).SteamPath
if (-not $steamPath) {
    $steamPath = (Get-ItemProperty 'HKLM:\SOFTWARE\WOW6432Node\Valve\Steam' -ErrorAction SilentlyContinue).InstallPath
}
if (-not $steamPath) { throw 'Steam installation not found in the registry.' }
$steamPath = $steamPath -replace '/', '\'

$libraryFolders = Join-Path $steamPath 'steamapps\libraryfolders.vdf'
if (-not (Test-Path -LiteralPath $libraryFolders -PathType Leaf)) {
    throw "Steam library index not found: $libraryFolders"
}

# Collect the libraries whose apps block lists the requested app ID.
$libraries = @()
$currentPath = $null
foreach ($line in Get-Content -LiteralPath $libraryFolders) {
    if ($line -match '"path"\s+"(.+)"') {
        $currentPath = $matches[1] -replace '\\\\', '\'
    }
    elseif ($currentPath -and $line -match "`"$appId`"\s+`"") {
        $libraries += $currentPath
        $currentPath = $null
    }
}

$installationRoot = $null
foreach ($library in $libraries) {
    $manifest = Join-Path $library "steamapps\appmanifest_$appId.acf"
    if (-not (Test-Path -LiteralPath $manifest -PathType Leaf)) { continue }
    $installDir = (Select-String -LiteralPath $manifest -Pattern '"installdir"\s+"(.+)"').Matches[0].Groups[1].Value -replace '\\\\', '\'
    $candidate = Join-Path $library "steamapps\common\$installDir"
    if (Test-Path -LiteralPath $candidate -PathType Container) {
        $installationRoot = $candidate
        break
    }
}
if (-not $installationRoot) { throw "Installation for app $appId not found in any Steam library." }

$installationRoot
```

Report the resolved root back to the user before compiling, so a wrong installation can be
corrected before anything is written.

## Library roots vary per machine

Only the default library sits at the Steam path itself. Every other library is a separate
root — at most one per drive, but with a folder name the user chose when creating it. All of
these are equally plausible for the same game:

```text
C:\Program Files (x86)\Steam\steamapps\common\Counter-Strike Global Offensive
D:\SteamLibrary\steamapps\common\Counter-Strike Global Offensive
E:\Games\Steam\steamapps\common\Counter-Strike Global Offensive
```

Only the `steamapps\...` tail is fixed; everything before it differs per machine. Steam itself
commonly stays in `C:\Program Files (x86)\Steam` even when the game lives elsewhere.

Therefore: never append `steamapps\common` to the Steam path, and never assume a library folder
is called `SteamLibrary`. Enumerate every entry in `libraryfolders.vdf`, take the root from the
entry whose `apps` block lists the app ID, and treat that root as the only source of the drive
and folder names.

## Finding the app ID

CS2 is `730` and Dota 2 is `570`. For anything else, do not guess:

- read it from the game's Steam store URL (`store.steampowered.com/app/<appId>/`), or
- list `steamapps\appmanifest_*.acf` across the libraries and read each manifest's `name`.

The second option is authoritative for what is actually installed on this machine.

## content/ and game/

A Source 2 installation separates authoring sources from loadable resources:

| Tree | Holds | Written by |
|---|---|---|
| `content/` | sources: `.css`, `.xml`, `.vmat`, `.vmdl`, `.vmap`, images, meshes | the author and the Workshop Tools editors |
| `game/` | compiled `_c` resources the engine loads | the Resource Compiler |

The compiler mirrors the resource-relative directory structure from one tree into the other.
A source at `content/<addons>/<addon>/panorama/layout/custom_game/ADDON/main.xml` becomes
`game/<addons>/<addon>/panorama/layout/custom_game/ADDON/main.vxml_c`.

A `content/` path in the user's request identifies the **source tree**. It is not a request to
put compiled output there. Do not relocate outputs because the user said to compile "in" that
directory.

## Addon directories per game

Addons live under a game-specific directory name that appears in both trees. The pattern is the
game's mod directory plus `_addons`:

| Game | Addon sources | Compiled addon output |
|---|---|---|
| CS2 | `content\csgo_addons\ADDON` | `game\csgo_addons\ADDON` |
| Dota 2 | `content\dota_addons\ADDON` | `game\dota_addons\ADDON` |

For any other game, do not extrapolate the name — list `content\` inside the resolved
installation root. The directory that is actually there is the authoritative answer.

For CS2, relative to the library root that holds it:

| Item | Path relative to the library root |
|---|---|
| Installation root | `steamapps\common\Counter-Strike Global Offensive` |
| App manifest | `steamapps\appmanifest_730.acf` |
| Resource Compiler | `steamapps\common\Counter-Strike Global Offensive\game\bin\win64\resourcecompiler.exe` |
| Addon sources | `steamapps\common\Counter-Strike Global Offensive\content\csgo_addons\ADDON` |
| Addon outputs | `steamapps\common\Counter-Strike Global Offensive\game\csgo_addons\ADDON` |

CS2's `installdir` is still the CS:GO folder name; that is expected, not a stale install.

## The compiler binary

`game\bin\win64\resourcecompiler.exe` under the installation root, for CS2 and for Source 2
games generally. Test for it before compiling anything — its absence usually means the Workshop
Tools were never installed for that game, not that the path is wrong.

Use the matching game's compiler. Do not substitute another Source 2 game's binary: the supported
resource types and build options come from the installed game.

## When the lookup fails

Ask the user for the installation root rather than guessing a drive letter. A game installed
outside Steam, or a library the running user cannot read, will not appear through this lookup at
all, and a wrong root silently compiles into the wrong game.
