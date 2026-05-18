# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, no-build, vanilla-JS web app for logging competitive Pokémon Champions VGC matches. The entire app — HTML, CSS, JS, embedded reference data — lives in `index.html` (~225 KB, ~6.5k lines). Source lives at https://github.com/NathanGugliotta/PokePicker; deploys to https://pokepicker.netlify.app via Netlify auto-build on every push to `main` (no CI step, the repo root *is* the published site). Used on iPad and Mac Safari; designed to also work as an iOS home-screen standalone app.

Built by and for Nathan, a competitive VGC player and senior trademark attorney — knows the game deeply, newer to web dev. Explain dev concepts (Git, modules, build steps) when introducing them; don't over-explain things he clearly already gets. He has ADHD and gets overwhelmed by jargon — keep technical terms grounded.

There is no `package.json`, bundler, test suite, or lint config in this folder. The parent directory is an Obsidian vault — this folder is a self-contained code project inside it and does not inherit the vault's note-writing conventions.

## About Pokémon Champions (critical — do NOT assume Scarlet & Violet mechanics)

Pokémon Champions launched April 2026 as the official competitive VGC platform, replacing Scarlet & Violet. **It is not SV.** If a future session writes code that references IVs, EV yields, Tera types, etc., something has gone wrong.

- **No IVs.** All Pokémon are treated as if 31 IVs in every stat. Do not surface IV fields anywhere.
- **No EVs.** Replaced by **Stat Points (SP)**: 66 total per Pokémon, max 32 per stat, 1 SP = 1 stat point (direct addition, no EV-style ÷4 formula).
- **Natures** replaced by **Stat Alignments** (functionally identical: +10% / −10%).
- **Item Clause** and **Species Clause** are baseline rules in Champions. Don't enforce them in-app (no value for solo use), but don't violate them in suggestions either.
- **187 species** at launch — final evolutions only (Pikachu is the sole exception); no Legendaries or Mythicals.
- Current regulation: **Reg M-A** (April 8 – June 17, 2026). Mega Evolution is the active gimmick.
- **Items NOT available**: Choice Band, Choice Specs, Life Orb, Assault Vest, Rocky Helmet, Heavy-Duty Boots, Toxic Orb, Flame Orb, Weakness Policy, Eviolite.
- **Items available**: Focus Sash, Leftovers, Choice Scarf, White Herb, Lum Berry, Sitrus Berry, all 18 type-boosting items, super-effective resist berries, Mental Herb, Shell Bell, Scope Lens, Mega Stones.

## Hard constraints

- **Stay single-file.** Don't split into modules, add a `package.json`, introduce a bundler, or pull in npm dependencies without an explicit request. External resources are limited to Google Fonts via `<link>`. The app must work offline after first load.
- **No framework.** UI handlers are inline `onclick="..."` attributes calling top-level functions. New interactions follow the same pattern — do not introduce React/Vue/etc. without asking.
- **iOS Safari is the primary target.** Avoid APIs unavailable in standalone-mode Safari; keep `localStorage` writes resilient (all storage calls are already try/catch-wrapped — preserve that).
- **Don't break the single-file deploy path** unless we're explicitly transitioning to a modular structure.

## File layout inside `index.html`

Major boundaries (line numbers approximate):
- `<style>` block: 11–2099 — CSS custom-property theme tokens for `dark` / `light` (set via `:root[data-theme="..."]`), plus all component styles
- Early theme bootstrap `<script>`: 2100–2118 — sets `data-theme` before paint to prevent flash
- `<body>` markup: 2120–2840 — three mode panels (Capture / Reflect / Research) plus modal backdrops
- Main `<script>`: 2841–6398 — all application logic, organized by `// ============ SECTION ============` banner comments. Search for that marker to navigate.

## Architecture

### Three modes share one shell
`setMode('quick' | 'reflect' | 'research')` toggles a `data-mode` attribute on the app root; CSS rules show/hide cards accordingly. The mode tabs in the UI are **⚡ Capture / 🧠 Reflect / 🛡️ Research**. Header, match-info card, and team slots are visible in Capture and Reflect; Research has its own 6-slot team builder and analysis panel.

### Two top-level state objects
- `state` — the current match draft (lead/back/opp slots, brought toggles, move log, reflection text, rating, per-opponent scouting via `state.oppScout`). Shape defined by `blankState()` at ~4709. Mutations go through `scheduleSave()` which debounces a write 400 ms later.
- `researchTeam` — the current research-mode team (6 slots, moves, abilities, items, alignments, SP spreads, nicknames). Persisted independently via `saveResearchCurrent()`.

Always mutate these in place and call the appropriate save function; never replace the reference, because many DOM handlers close over the binding.

### localStorage key namespaces
Each subsystem owns a prefix — keep them disjoint:
- `champions-match:current-draft` — last-edited match
- `champions-match:<draft-id>` — every saved match draft (id is `draft-<timestamp>`)
- `champions-team:current` — research-mode working team
- `champions-team:saved-<id>` — saved team snapshots
- `champions-roster` — single JSON array of saved Pokémon builds reusable across teams
- `theme-preference` — `'system' | 'light' | 'dark'`

`listDrafts()` and the saved-teams loader iterate `localStorage` and filter by prefix, so any new persisted entity needs its own non-colliding prefix.

### Regulation-keyed reference data
All format-specific data (legal Pokémon roster, items, abilities, moves, per-species types) lives in `REG_DATA[reg]`. Currently only `'M-A'` is populated (`M_A_POKEMON`, `M_A_ITEMS`, `M_A_ABILITIES`, `M_A_MOVES`, `MATCH_POKEMON_TYPES`). Every lookup routes through `getCurrentRegData()` — never reference `M_A_*` constants directly from feature code.

To add a future regulation (e.g. M-B):
1. Append to `REGULATIONS` and optionally update `DEFAULT_REGULATION` (~2848)
2. Add `REG_DATA['M-B'] = { ...REG_DATA['M-A'], pokemon: M_B_POKEMON, items: M_B_ITEMS }` — spread M-A and override only what changed
3. Add the option to the HTML `<select id="f-reg">`
4. Existing saved matches/teams/builds are migrated lazily — `loadRoster()` tags pre-regulation entries with `DEFAULT_REGULATION` on load

The roster picker filters by regulation by default; saved builds carry a `regulation` field set at save time via `getCurrentRegulation()`.

### Type analysis (Research mode)
`TYPE_CHART` (defender-keyed × attacker → multiplier) plus per-species `MATCH_POKEMON_TYPES` feed `analyzeMon()` and `analyzeTeam()`. The team analyzer produces per-mon weakness/resist/immunity buckets and flags **glaring weaknesses** — attacking types that hit ≥3 team members for ≥2×, with `severe` set when ≥1 of those is 4×. A separate, smaller threat-bar implementation under Capture mode (`computeThreats` / `renderThreatBar`) scores the opponent's brought mons against your team.

Both analyzers use a local `normalizeName()` (lowercase, strip non-alphanumerics) for fuzzy lookup so users can type "alolan-ninetales", "Alolan Ninetales", etc.

### Pokémon roster (cross-team build library)
`champions-roster` stores reusable builds keyed by `buildSignature()` for dedup. The signature is `species + ability + alignment + sorted moves + SP` — **item and nickname are intentionally excluded**, so the same competitive build with different items (e.g. different Mega Stones) collapses to one entry. Mega Stones are a per-team choice, not a build identity. `openRosterPicker(slotIdx)` opens the picker filtered to the current slot's species; `useBuildFromRoster()` writes the build into `researchTeam.slotsDetails[idx]`.

Teams keep **snapshots** of the builds they reference — editing a roster build does not propagate to teams that already pulled it in.

### Per-opponent scouting
`state.oppScout` runs parallel to `state.opp` — one entry per opponent slot holding `{item, mega, moves[4], notes}`. The 📝 button on each opp slot opens the scout modal; saved intel renders as colored chips below the slot and is included in the Markdown export.

### Markdown + Pokepaste I/O
- `buildMarkdown()` / `downloadMd()` / `copyMarkdown()` render the current match to a Markdown report (filename derived from date + opponent + result). Output is **Obsidian-targeted**: YAML frontmatter, `inbox_target: "0 Inbox/"` (NOT `1 Inbox` — that's the old vault convention), `[[wikilinks]]` for cross-refs.
- `importPokepaste()` parses Showdown-format paste blocks into `researchTeam` slots, including SP parsing via `parseSPLine` (Champions native SP format, not EV÷4).
- `buildPokepasteFromResearch()` is the inverse.

When extending output formats, keep the same pattern: a pure builder function plus separate download/copy wrappers.

## Tier roadmap (active backlog)

Inspired by Phil Wingett / THATSAplusONE coaching frame:

- ✅ **Tier 1 — Per-opponent scouting** (shipped). `state.oppScout[]` parallel to `state.opp[]`; 📝 button per slot; chips render under each slot; scouting intel flows into Markdown export.
- ⏳ **Tier 2 — Bo3 game tracking.** Opt-in toggle; structural refactor. Each match becomes 1–3 games, each with its own lead/back, opp lead, result, log, between-games plan. Match result derives from games.
- ⏳ **Tier 3 — Lead matchup planning.** Pre-match free-text per their-lead × my-lead.
- ⏳ **Tier 4 — Conditioning / set notes.** Across-games strategic theme field.

## Pending / known issues

- **Light mode theme cache.** A prior deploy showed "Theme: Light" on the toggle while the page stayed dark. CSS was restructured to use `@media (prefers-color-scheme: light) { :root:not([data-theme="dark"]) }` as the fallback baseline plus explicit `:root[data-theme]` overrides. Safari cache can still mask this — hard refresh when verifying.
- **iOS 26 polish "Turn 2"** deferred: spring physics on press, mode-tab pill morph animation, sheet-style modal slide-up from bottom, skeleton loading states.

## Conventions Nathan cares about

- **No IV fields anywhere.** Champions has no IVs.
- **No EV math.** SP only: 66 total cap, 32 per stat cap. Direct addition, no ÷4.
- **Mega Stones are per-team items**, NOT part of the roster build signature — same build can use different megas on different teams.
- **Teams keep snapshots.** Roster build edits don't propagate to teams that referenced an older snapshot.
- **Nicknames are preserved** when relevant. Nathan's teams: **Tinka Trap!** (Whitney = Sinistcha, Heartbreaker = Mega Gengar, Worm = Kommo-o, plus Tinkaton, Politoed, Incineroar) and **TrickSand** (Scott = Whimsicott, Cersei = Tyranitar, Mr. Worldwide = Excadrill, Gold Rush = Steelix, Shogun = Kingambit, Orthworm).
- **Markdown export targets Obsidian.** YAML frontmatter, `inbox_target: "0 Inbox/"` (NOT `1 Inbox` — that's the old convention). `[[wikilinks]]` for cross-refs.
- **Toast brand is yellow gradient with dark text** (`#1a1410`) in both themes — brand consistency.
- **Item Clause / Species Clause** are baseline — don't enforce in-app, don't violate in suggestions.

## Workflow

- **Deploy:** `git push` to `main` on https://github.com/NathanGugliotta/PokePicker — Netlify auto-builds from the repo root and publishes to https://pokepicker.netlify.app within ~1 minute. No build step; no staging. The earlier Netlify Drop workflow is superseded — don't drag files into Drop anymore or it'll conflict with the Git pipeline.
- **Edit cycle:** Claude Code edits the file directly. Show diffs before applying non-trivial changes — Nathan likes to see what's changing.
- **Checkpoints:** small, focused commits with clear messages when a feature spans multiple steps.
- **Verification:** Nathan reviews on iPad Safari + Mac Safari; cache is aggressive, hard refresh sometimes needed. Test light AND dark mode when touching CSS.
- **No regex-driven Python migration scripts** — old workflow. Edit the file directly with file tools.
- **No build step yet.** When we modularize, we'll add tooling then.

## Voice / coaching style

Nathan engages best with:
- Sharp, practical, direct prose
- Reasoning made visible (why X, not just what X)
- Honest pushback when he's converging too fast or missing something
- Sports/business analogies when concepts get abstract
- Millennial register — "cooking" (doing well), "fire" (excellent), occasional profanity for emphasis

Avoid:
- Over-explaining basics he clearly understands
- Wall-of-text technical jargon
- "Great question!" sycophancy
- Excessive bullet-list formatting in conversational replies

## Common edits

- **Add a new field to the match draft:** extend `blankState()`, add the input to the HTML, bind it in `init()` (use `bindInput(id, key)` for plain inputs), and surface it in `buildMarkdown()` if it should appear in exports.
- **Add a new modal:** follow the existing `.modal-backdrop` / `.modal` pattern and add a `closeModal(id)` wiring — backdrop-click-to-close is already handled globally in `init()`.
- **Add a new section of reference data:** put it next to the existing `M_A_*` arrays and expose it through `REG_DATA[reg]` so it stays regulation-scoped.
- **Add a new persisted entity:** pick a fresh `localStorage` prefix, wrap reads/writes in try/catch, and migrate older entries lazily on load (see `loadRoster()` for the pattern).

## Useful project links

- Live app: https://pokepicker.netlify.app
- Deploy: https://app.netlify.com/drop (Nathan's Netlify account)
- Parent vault: `/Users/nathangugliotta/Documents/Gugliotta & Gugliotta, LPA/` (this app sits under `1 Areas/Poke Picker App/`)
