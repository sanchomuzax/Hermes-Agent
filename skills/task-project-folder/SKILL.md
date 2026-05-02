---
name: task-project-folder
description: "Start every user task with a unique project directory under ${PROJECTS_ROOT}; capture prompts, final responses, artifacts, web sources, tool/skill usage, logs, and task metadata in a reproducible structure."
version: 2.2.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [project-setup, filesystem, workflow, audit-log, provenance]
    related_skills: [hermes-agent]
---

# Task Project Folder

## Overview

Minden új felhasználói feladat elején hozz létre egy egyedi projektmappát a `${PROJECTS_ROOT}` alatt. A mappa legyen a feladat teljes, rendezett munkaterülete: ide kerüljön minden prompt, jegyzet, forrás, létrehozott fájl, export, webes forráslista, eszköz-/skillhasználati napló és rövid összefoglaló.

Cél: a feladat később visszakereshető, auditálható, reprodukálható és tisztán átadható legyen.

Session-specific rationale and pitfalls for final-response capture: `references/final-response-logging.md`.

## Configurability and Modes

A skill célja az auditálhatóság, de ne termeljen felesleges adminisztrációt.

- `PROJECTS_ROOT` (kötelező): a projektmappák gyökere (abszolút útvonal).
- `TASK_FOLDER_MODE`:
  - `full` (default): teljes auditstruktúra;
  - `lite`: minimál naplózás kis feladatokhoz.

Ajánlott döntési szabály:
- **lite**: egyszerű, rövid, alacsony kockázatú kérés, kevés toolhívással.
- **full**: több lépéses, fájlmódosítós, webforrásos, vagy átadásra szánt feladat.


## When to Use

Használd minden új user-task indulásakor, de a terheléshez igazított módban:
- kutatás, webes böngészés, ár- vagy termékellenőrzés;
- fájl létrehozása vagy módosítása;
- kódolás, debug, konfigurálás;
- dokumentum, wiki, riport, prezentáció, script, adatfeldolgozás;
- bármilyen többüzenetes munkafolyamat.

Ha a feladat egy korábbi task egyértelmű folytatása, használd az eredeti projektmappát, és frissítsd benne a promptokat/naplókat. Ha új cél vagy új kontextus indul, hozz létre új mappát.

## Standard Directory Layout

Full módban használd ezt az alapstruktúrát:

```text
${PROJECTS_ROOT}/YYYYMMDD-HHMMSS-short-task-slug/
├── README.md
├── task_metadata.json
├── inputs/
│   ├── prompt_log.md
│   └── prompts.json
├── outputs/
│   ├── final_response.md
│   └── final_response.json
├── notes/
│   ├── run_log.md
│   ├── decisions.md
│   └── assumptions.md
├── sources/
│   ├── web_sources.md
│   └── web_sources.json
├── logs/
│   ├── tool_usage.md
│   ├── tool_usage.json
│   ├── skill_usage.md
│   └── skill_usage.json
├── artifacts/
│   └── manifest.json
└── tmp/
```

Lite módban minimál struktúra is elfogadott:

```text
${PROJECTS_ROOT}/YYYYMMDD-HHMMSS-short-task-slug/
├── task_metadata.json
├── inputs/
│   └── prompts.json
├── outputs/
│   └── final_response.json
└── artifacts/
    └── manifest.json
```

Alapszabályok:

- `inputs/`: user promptok és user által adott bemenetek.
- `outputs/`: végleges, usernek átadható eredmények, beleértve a záró/final assistant választ is.
- `sources/`: külső források, weboldalak, API endpointok, letöltési URL-ek.
- `logs/`: toolok, skillek, végrehajtási naplók.
- `artifacts/`: létrehozott vagy módosított fájlok jegyzéke, provenance információval.
- `tmp/`: ideiglenes fájlok, cache, köztes anyagok. Ha fontos lesz, mozgasd át `artifacts/` vagy `outputs/` alá.

## Naming Rules

1. Gyökér: `${PROJECTS_ROOT}` vagy a környezetedben beállított `PROJECTS_ROOT`.
2. Projektmappa név: `YYYYMMDD-HHMMSS-short-task-slug`.
   - Példa: `${PROJECTS_ROOT}/20260429-082320-task-project-folder-skill-tuning`
3. A slug legyen rövid, kisbetűs, ékezet nélküli, kötőjeles.
4. Ütközés esetén adj hozzá sorszámot: `-02`, `-03`.

## Bootstrap Procedure

Feladat elején:

1. Állapítsd meg az aktuális timestampet eszközzel, ne emlékezetből.
2. Hozd létre a projektmappát és az alap almappákat.
3. Hozd létre a szükséges mappákat és csak a ténylegesen használt naplófájlokat (lazy-create).
4. Mentsd el az aktuális user promptot `inputs/prompt_log.md` és `inputs/prompts.json` fájlba.
5. Jegyezd fel, hogy ezt a skillt használtad `logs/skill_usage.*` alatt.
6. A záró/final választ a kiküldés előtt vagy azzal azonos tartalommal mentsd `outputs/final_response.md` és `outputs/final_response.json` fájlba.
7. A válaszban vagy záró összefoglalóban mindig add meg a projektmappa abszolút útvonalát.

Javasolt shell mappalétrehozás:

```bash
mkdir -p "$PROJECT_DIR"/{inputs,notes,sources,logs,artifacts,outputs,tmp}
```

## Logging Policy (Canonical Format)

A **JSON a kanonikus forrás**.
- Kötelező: `*.json` naplók.
- Opcionális: `*.md` emberi nézet, ami a JSON-ból készül vagy azzal konzisztens.

Ha csak egy formátumot tudsz biztosan konzisztensen vezetni, a JSON legyen az.

## Prompt Logging Requirements

Minden user promptot legalább JSON-ben ments el:

### `inputs/prompt_log.md` (opcionális, ajánlott)

Emberileg olvasható, időrendi napló:

```markdown
# Prompt log

## 001 — 2026-04-29 08:23:20 local

<user prompt teljes szövege>

## 002 — 2026-04-29 08:31:05 local

<következő user prompt teljes szövege>
```

### `inputs/prompts.json`

Géppel feldolgozható JSON tömb:

```json
[
  {
    "id": 1,
    "timestamp_local": "2026-04-29T08:23:20",
    "role": "user",
    "content": "...",
    "source": "chat",
    "notes": null
  }
]
```

Többüzenetes feladatnál minden új user üzenet után appendeld a `prompts.json`-t, és ha használsz MD nézetet, azt is frissítsd.

## Final Response Logging Requirements

Minden záró/final assistant választ ments el a projektmappába pontosan azzal a tartalommal, amit a usernek kiírsz.

### `outputs/final_response.md`

Emberileg olvasható végső válasz. Ha több részválasz vagy több final jellegű lezárás van, időrendben appendeld:

```markdown
# Final responses

## 001 — 2026-04-29T08:30:00 local

<assistant final response teljes szövege>
```

### `outputs/final_response.json`

Géppel feldolgozható JSON tömb:

```json
[
  {
    "id": 1,
    "timestamp_local": "2026-04-29T08:30:00",
    "role": "assistant",
    "type": "final_response",
    "content": "...",
    "related_prompt_ids": [1],
    "notes": null
  }
]
```

Fontos: a final válasz mentése ne maradjon ki csak azért, mert az utolsó lépés maga a usernek való válaszadás. A final szöveget először írd fájlba, majd ugyanazt add vissza a usernek.

## Web Source Logging Requirements

Ha bármilyen weboldalt, API endpointot, dokumentációt, GitHub oldalt, keresési találatot vagy letöltési URL-t megnyitsz vagy felhasználsz, mentsd el:

### `sources/web_sources.md` (opcionális, ajánlott)

```markdown
# Web sources

| # | Timestamp | URL | Title/Label | Purpose | Used in output? |
|---|---|---|---|---|---|
| 1 | 2026-04-29T08:25:00 | https://example.com | Example docs | API ellenőrzés | yes |
```

### `sources/web_sources.json`

```json
[
  {
    "id": 1,
    "timestamp_local": "2026-04-29T08:25:00",
    "url": "https://example.com",
    "title_or_label": "Example docs",
    "purpose": "API ellenőrzés",
    "used_in_output": true,
    "notes": null
  }
]
```

Kötelező logolni a ténylegesen meglátogatott/használt URL-t, nem csak a domain nevét.

## Tool Usage Logging Requirements

Minden eszközhasználatot vezesd JSON-ben; az MD nézet opcionális összefoglaló.

### `logs/tool_usage.md` (opcionális)

```markdown
# Tool usage

## Summary

| Tool | Count | Main purposes |
|---|---:|---|
| terminal | 2 | timestamp lekérés, mappalétrehozás |
| write_file | 3 | prompt logok és metadata létrehozása |

## Events

| # | Timestamp | Tool | Purpose | Inputs/Scope | Result |
|---|---|---|---|---|---|
| 1 | 2026-04-29T08:23:20 | terminal | timestamp lekérés | date command | success |
```

### `logs/tool_usage.json`

```json
{
  "summary": {
    "terminal": {
      "count": 2,
      "main_purposes": ["timestamp lekérés", "mappalétrehozás"]
    }
  },
  "events": [
    {
      "id": 1,
      "timestamp_local": "2026-04-29T08:23:20",
      "tool": "terminal",
      "purpose": "timestamp lekérés",
      "inputs_scope": "date command",
      "result": "success"
    }
  ]
}
```

Ne ments titkokat, tokeneket, jelszavakat vagy teljes credential tartalmakat a naplóba. Ilyenkor csak redaktált scope-ot írj: `credential file checked: yes/no`.

## Skill Usage Logging Requirements

Minden betöltött vagy használt skillt naplózz JSON-ben; MD opcionális:

### `logs/skill_usage.md` (opcionális)

```markdown
# Skill usage

## Summary

| Skill | Count | Used for |
|---|---:|---|
| task-project-folder | 1 | projektstruktúra létrehozása |

## Events

| # | Timestamp | Skill | Action | Purpose |
|---|---|---|---|---|
| 1 | 2026-04-29T08:23:15 | task-project-folder | loaded | projektmappa workflow betöltése |
```

### `logs/skill_usage.json`

```json
{
  "summary": {
    "task-project-folder": {
      "count": 1,
      "used_for": ["projektstruktúra létrehozása"]
    }
  },
  "events": [
    {
      "id": 1,
      "timestamp_local": "2026-04-29T08:23:15",
      "skill": "task-project-folder",
      "action": "loaded",
      "purpose": "projektmappa workflow betöltése"
    }
  ]
}
```

## Artifact and Output Tracking

Minden létrehozott vagy módosított fájlt jegyezz fel `artifacts/manifest.json` alatt, akkor is, ha végül `outputs/` mappába kerül:

```json
[
  {
    "path": "${PROJECTS_ROOT}/.../outputs/report.md",
    "type": "markdown_report",
    "created_or_modified": "created",
    "timestamp_local": "2026-04-29T08:40:00",
    "purpose": "végső riport",
    "source_inputs": ["inputs/prompts.json#1"],
    "notes": null
  }
]
```

Ha a task külső fájlokat módosít a projektmappán kívül, azokat is rögzítsd a manifestben `external_path: true` mezővel, és a final válaszban külön emeld ki.

## Metadata, Notes, and Reproducibility

### `task_metadata.json`

Tartalmazza legalább:

```json
{
  "project_dir": "${PROJECTS_ROOT}/YYYYMMDD-HHMMSS-slug",
  "created_at_local": "2026-04-29T08:23:20",
  "task_title": "Rövid cím",
  "task_slug": "short-task-slug",
  "status": "in_progress",
  "language": "hu",
  "owner": "user",
  "agent": "Hermes Agent",
  "summary": null
}
```

### `README.md`

Legyen rövid emberi áttekintés:
- feladat címe;
- létrehozás ideje;
- státusz;
- fontos mappák;
- végső eredmények helye.

### `notes/run_log.md`

Rövid, időrendi végrehajtási napló: mit csináltál és miért.

### `notes/decisions.md`

Fontos döntések: miért egy adott megoldás, fájlnév, forrás vagy megközelítés lett választva.

### `notes/assumptions.md`

Feltételezések és nyitott kérdések. Ha nincs ilyen, írd bele: `Nincs rögzített feltételezés.`

## Organization Rules During Execution

- Ne ments feladatfájlt közvetlenül a home gyökérbe, `/tmp`, aktuális random cwd vagy Desktop alá, kivéve ha a user kifejezetten kéri.
- Ha egy tool ideiglenesen máshova hoz létre fájlt, mozgasd vagy másold a releváns eredményt a projektmappába.
- Végleges user-facing fájlok az `outputs/` alá kerüljenek.
- A usernek kiírt záró/final válasz mindig mentődjön `outputs/final_response.md` és `outputs/final_response.json` alá is.
- Forrásadatok és bemenetek az `inputs/` vagy `sources/` alá kerüljenek.
- Kísérletek, intermediate fájlok, scratch anyagok a `tmp/` alá kerüljenek.
- Ha repo/projekt módosítás történik máshol, a projektmappa akkor is maradjon audit/worklog központként.

## Final Response Requirements

A záró válasz elküldése előtt:
- mentsd a teljes final választ `outputs/final_response.md` alá;
- mentsd ugyanazt strukturáltan `outputs/final_response.json` alá;
- jegyezd fel ezeket az `artifacts/manifest.json` fájlban.

A záró válaszban szerepeljen:
- projektmappa abszolút útvonala;
- fő kimeneti fájlok abszolút útvonala, ha vannak;
- külső módosított fájlok, ha vannak;
- rövid státusz: kész / részben kész / további teendő.

## Common Pitfalls

1. Prompt csak MD-ben mentődik, JSON-ben nem. A JSON kötelező, az MD opcionális nézet.
2. Webes források kimaradnak. Minden megnyitott vagy felhasznált URL-t logolj.
3. Tool/skill usage csak fejben marad. Kötelező JSON-ben eseményszinten menteni.
4. Szétszórt fájlok. Minden taskhoz tartozó fájl a projektmappába kerüljön, vagy legyen manifestben külső fájlként megjelölve.
5. Titkok naplózása. Credentialt, tokent, cookie-t soha ne ments teljes értékben.
6. Több független task egy mappában. Új cél = új projektmappa.
7. Záró válaszból kimarad az útvonal. Mindig add meg.
8. Záró/final válasz nincs elmentve. Mindig mentsd `outputs/final_response.md` és `outputs/final_response.json` alá, mielőtt/amikor elküldöd.

## Verification Checklist

- [ ] Projektmappa létrejött `PROJECTS_ROOT` alatt (default: `${PROJECTS_ROOT}`), egyedi timestampelt névvel.
- [ ] Standard almappák létrejöttek.
- [ ] `README.md` és `task_metadata.json` létrejött.
- [ ] Minden user prompt mentve legalább `inputs/prompts.json` alatt (MD nézet opcionális).
- [ ] Webes források, ha voltak, mentve legalább `sources/web_sources.json` alatt (MD nézet opcionális).
- [ ] Toolhasználat mentve `logs/tool_usage.json` alatt (MD nézet opcionális).
- [ ] Skillhasználat mentve `logs/skill_usage.json` alatt (MD nézet opcionális).
- [ ] Létrehozott/módosított fájlok szerepelnek `artifacts/manifest.json` alatt.
- [ ] Végső kimenetek `outputs/` alatt vannak, ha releváns.
- [ ] Final válasz mentve `outputs/final_response.md` és `outputs/final_response.json` alatt.
- [ ] Külső fájlmódosítások külön jelölve vannak.
- [ ] Final válasz tartalmazza a projektmappa abszolút útvonalát.
