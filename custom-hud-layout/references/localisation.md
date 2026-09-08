# Localisation

A `text` value beginning with `#` is a localisation token, resolved on the client against the
string catalogue for that player's language. It is the only mechanism in a custom hud that
produces per-player text with **no server call at all**, and in a text-heavy hud it carries the
majority of every label.

Two sources feed it: the catalogues the game already ships, which cost you nothing, and a file
of your own. This document covers both — what exists, how to add to it, and the four ways it
fails quietly.

## Contents

- [Using a token](#using-a-token)
- [Shipping your own strings](#shipping-your-own-strings)
- [Naming keys](#naming-keys)
- [The catalogues the game ships](#the-catalogues-the-game-ships)
- [What is in each file](#what-is-in-each-file)
- [Tokens worth knowing](#tokens-worth-knowing)
- [Format parameters a hud cannot fill](#format-parameters-a-hud-cannot-fill)
- [Languages](#languages)
- [How it fails](#how-it-fails)

## Using a token

```xml
<Label class="weapon-name" hittest="false" text="#SFUI_WPNHUD_AK47" />
```

The `#` is part of the attribute value, not part of the key: the file declares `SFUI_WPNHUD_AK47`
and the layout writes `#SFUI_WPNHUD_AK47`.

Substitution happens **once, client-side, at apply time**. Two consequences follow, and both are
covered at length in [xml.md](xml.md), *Localisation tokens*:

- a dialog variable's value is not rescanned, so `SetDialogVariableString(id, "v", "#SFUI_…")`
  renders the literal `#SFUI_…` on screen;
- a token's own value is not rescanned either, so tokens do not compose out of other tokens.

Together they mean a string that must be **both localised and chosen at runtime** cannot be sent
from the server. Enumerate the candidates in the markup and reveal one with a class — the idiom is
written out in xml.md.

## Shipping your own strings

`resource/platform_<language>.txt` inside the addon, one file per language, in Valve's KeyValues
format — not KV3, and with no preamble:

```
"lang"
{
	// Comments are legal, but only INSIDE the braces. See the encoding rule below.
	"Language"	"English"
	"Tokens"
	{
		"MyHud_Title"		"Server rules"
		"MyHud_Close"		"Close"
	}
}
```

Ship the same key set in every language file. A key missing from one language falls back to
nothing useful — see *How it fails*.

> **The file must begin with a `"` character**, or with a UTF-8 / UTF-16LE byte-order mark. The
> loader sniffs the encoding from the first bytes and accepts nothing else, so a comment on line 1
> makes the entire file unreadable. Put comments inside the `"lang" {` block, never above it. The
> shipped files take the second route: `csgo_english.txt` starts with a UTF-8 BOM.

## Naming keys

Prefix your keys with something specific to the addon so they cannot collide with the game's, then
match the shipped convention: a namespace prefix, then PascalCase words joined by underscores.

```
GameUI_Brightness
SFUI_WPNHUD_DesertEagle
Cstrike_TitlesTXT_Terrorists_Win
Rarity_Mythical_Weapon
```

Whatever case you choose, the key and the `#` reference in the layout have to agree. Matching the
convention costs nothing and keeps your keys from reading as foreign next to the ones you borrow.

## The catalogues the game ships

Two directories hold them, and the split is game content versus engine layer:

| Directory | Files | What it is |
|---|---|---|
| `<game>/resource/` | `csgo_<language>.txt` | everything CS2-specific: weapons, maps, items, finishes, round outcomes, menus |
| `<core>/resource/` | `valve_<language>.txt` | engine-level UI: key bindings, dates, store, generic buttons |
| `<core>/resource/` | `countries_<language>.txt` | Steam country and region names |
| `<core>/resource/` | `toolhelp_*_english.txt`, `dmecontrols_english.txt`, `keybindings_english.txt` | Workshop Tools help text — English only, not game UI |

`csgo_*` is the one the game's own interface is built on, so a token found there is as safe a bet
as a shipped string gets. For the rest, and for anything in `toolhelp_*`, **check that the token
renders before depending on it** — an unresolved token displays as its own literal text, which
makes the check a two-second one.

## What is in each file

Counts are from one CS2 install and will drift with updates; the shape is what matters.

| File | Keys | Largest prefixes |
|---|---:|---|
| `csgo_english.txt` | ~44 400 | `StickerKit_` (20 000), `SFUI_` (5 600), `CSGO_` (4 500), `PaintKit_` (2 700), `HighlightReel_` / `HighlightDesc_` (930 each), `GameUI_` (590), `coupon_` (300), `Cstrike_` (250), `StoreItem_` (240) |
| `valve_english.txt` | ~860 | `Valve_` (420), `LOC_` (135), `Native_` (37), `StoreCheckout_` (32), `Attrib_` (31), `hud_` (30) |
| `countries_english.txt` | ~250 | `Steam_Country_`, `Steam_Region_` |
| `toolhelp_particles_english.txt` | ~490 | `Attribute.C…`, `Element.C…` |
| `toolhelp_cs2_item_editor_english.txt` | ~1 800 | `Attribute.PaintKit`, `Attribute.StickerKit` |
| `toolhelp_modeldoc_editor_english.txt` | ~200 | `modeldocattr.`, `clothchain.` |
| `toolhelp_vmixtool_english.txt` | ~160 | `Attribute.CMix…`, `Attribute.VMix…` |
| `toolhelp_hammer_commands_english.txt` | ~110 | `Command.` |
| `toolhelp_lights*_fgd_english.txt` | ~130 total | `Attribute.light` |
| `toolhelp_sndtool_english.txt` | ~105 | `Attribute.public.` |
| `toolhelp_surfacepropertyeditor_english.txt` | ~100 | `surfaceattribute.` |
| `dmecontrols_english.txt` | ~40 | `Dme…`, `elementpropertiestree` |
| `resource/ui/radiopanel.txt` | ~75 | `title`, `label`, `hotkey`, `cmd` — a menu definition, not a token file |

Nearly half of `csgo_english.txt` is `StickerKit_` and `PaintKit_` names. That is the practical
argument for the catalogue: a skin browser gets every finish name in thirty languages for free,
and the addon ships no localisation file at all.

## Tokens worth knowing

Sampled from `csgo_english.txt` unless marked. Values are the English ones; the same key resolves
per player.

**Weapons** — the whole arsenal, in the short form the HUD uses:

| Token | English |
|---|---|
| `SFUI_WPNHUD_AK47` | AK-47 |
| `SFUI_WPNHUD_AWP` | AWP |
| `SFUI_WPNHUD_DesertEagle` | Desert Eagle |
| `SFUI_WPNHUD_Knife` | Knife |

**Maps** — `SFUI_Map_<mapname>`, keyed by the map's own file name:

| Token | English |
|---|---|
| `SFUI_Map_de_dust2` | Dust II |
| `SFUI_Map_de_mirage` | Mirage |
| `SFUI_Map_de_inferno` | Inferno |

**Round outcomes and sites** — for scoreboards, end-of-round banners and defuse timers:

| Token | English |
|---|---|
| `SFUI_RoundWin_Defused` | Bomb Defused |
| `SFUI_RoundWin_Bomb` | Bomb Detonated |
| `SFUI_RoundWin_Hostage` | Hostage Rescued |
| `SFUI_RoundWin_Time` | Time Expired |
| `SFUI_BombZoneA` / `SFUI_BombZoneB` | Bomb Site A / B |
| `SFUI_Rounds` | rounds |

**Item rarity** — the eight-step ladder, ready-made for a shop or an inventory grid:

| Token | English |
|---|---|
| `Rarity_Default_Weapon` | Stock |
| `Rarity_Common_Weapon` | Consumer Grade |
| `Rarity_Uncommon_Weapon` | Industrial Grade |
| `Rarity_Rare_Weapon` | Mil-Spec Grade |
| `Rarity_Mythical_Weapon` | Restricted |
| `Rarity_Legendary_Weapon` | Classified |
| `Rarity_Ancient_Weapon` | Covert |
| `Rarity_Contraband_Weapon` | Contraband |

**Buttons and navigation:**

| Token | English | File |
|---|---|---|
| `SFUI_Accept` | ACCEPT | `csgo_*` |
| `SFUI_Back` | BACK | `csgo_*` |
| `SFUI_Continue` | CONTINUE | `csgo_*` |
| `Valve_OK` | OK | `valve_*` |
| `Valve_Cancel` | Cancel | `valve_*` |
| `Valve_Close` | Close | `valve_*` |

**Dates** — twelve months and the weekday set, in `valve_*`:

| Token | English |
|---|---|
| `LOC_Date_Month0` … `LOC_Date_Month11` | January … December |

**Countries** — `Steam_Country_<ISO code>`, in `countries_*`:

| Token | English |
|---|---|
| `Steam_Country_US` | United States |
| `Steam_Country_DE` | Germany |
| `Steam_Country_BR` | Brazil |
| `Steam_Country_JP` | Japan |

**Movement and key names** — useful for a controls or key-hint panel, in `valve_*`:
`Valve_Move_Forward`, `Valve_Move_Back`, `Valve_Move_Left`, `Valve_Move_Right`, `Valve_Jump`,
`Valve_Duck`, `Valve_Look_Up`, `Valve_Look_Down`.

**Bulk sets** worth knowing exist rather than listing: `StickerKit_*` and `PaintKit_*` (every
sticker and finish name, and their descriptions), `CSGO_*` (missions, operations, store),
`HighlightReel_*` / `HighlightDesc_*` (highlight captions), `Cstrike_TitlesTXT_*` (the classic
round and chat messages), `GameUI_*` (settings labels).

To find one, search the English file for the phrase you want and read the key beside it. The
search is the workflow — no index of these exists.

## Format parameters a hud cannot fill

Roughly 830 of the CS2 keys carry `%s1` / `%s2` / `%d` placeholders, because they were written for
call sites that pass arguments:

```
Cstrike_TitlesTXT_Game_teammate_attack    " %s1 attacked a teammate"
hud_telemetry_frametime_fps               "Max %s1ms | Avg %s2FPS"
Cstrike_Chat_CT_Loc                       " [CT] %s1﹫%s3: %s2"
```

**A custom hud has no way to supply those arguments.** Panorama's own substitution is `{s:name}`
against dialog variables, and a token's value is not rescanned for anything — so a token with
`%s1` in it renders with the `%s1` still there.

Check the value before adopting a key. When a parameterised string is what you need, write your
own key with a `{s:…}` placeholder in it and set the variable from the server; that composes,
whereas the shipped one does not.

## Languages

Thirty language files ship for both `csgo_*` and `valve_*`, twenty-eight for `countries_*`. The
suffix is the language's own name, not an ISO code:

```
brazilian  bulgarian  czech     danish    dutch      english
finnish    french     german    greek     hungarian  indonesian
italian    japanese   koreana   latam     norwegian  polish
portuguese romanian   russian   schinese  spanish    swedish
tchinese   thai       turkish   ukrainian vietnamese
```

Three oddities in the same install: `csgo_*` adds `schinese_pw` and has no `korean`, `valve_*`
ships both `korean` and `koreana`, and `countries_*` has neither `korean` nor `indonesian`. Do not
assume a language present in one catalogue is present in all three.

Name your own files with the same suffixes. A player whose language has no file falls back to
English, so `<name>_english.txt` is the one file you cannot omit.

## How it fails

Every failure here is silent on the authoring side. Nothing in the build reports any of them.

| Symptom | Cause |
|---|---|
| `#MyHud_Title` rendered literally on screen | the token is unknown — file failed to load, key misspelled, or the key is missing from that player's language |
| The whole file ignored | it does not begin with `"` or a BOM — usually a comment on line 1 |
| `%s1` visible in the text | a shipped key that expects arguments; see above |
| Text correct in English, English in every other language | the key was added to `_english.txt` only |
| `#SFUI_…` shown where a dialog variable was set | a token passed through `SetDialogVariableString`; values are not rescanned |

The literal-text symptom is the useful one: it makes verifying a borrowed token a matter of
putting it in a label and looking at the screen.
