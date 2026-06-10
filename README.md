# Cognos → Sigma

Migrate **IBM Cognos Analytics** to **Sigma**:

- **Data Module JSON → Sigma data model** — query subjects → tables, query items → columns/metrics, calculations → Sigma formulas, relationships → DM relationships.
- **Report-spec XML → Sigma workbook** — lists → table elements, dataItems → columns, prompts → controls, filters surfaced for re-creation.

It translates the Cognos expression DSL where it maps cleanly and **flags what it can't** (runtime macros, running-totals, localization helpers) rather than emitting wrong logic — the same "flag, never fake" philosophy as the other `sigma-migration-skills` converters.

> **Status:** working MVP, validated on real IBM Cognos samples (Great Outdoors, GO sales performance, Telco churn, …). Laid out as a Claude Code **plugin** so it drops into [`sigma-migration-skills`](https://github.com/twells89/sigma-migration-skills)`/plugins/` when ready; the `converter/` is destined to vendor into `sigma-data-model-mcp` as `convert_cognos_to_sigma` MCP tools.

---

## The migration flow

```mermaid
flowchart TB
    COG["🟦 <b>IBM Cognos Analytics</b><br/><i>CA REST /bi/v1 · session cookie + X-XSRF-Token</i>"]

    subgraph DISCOVER["1 · DISCOVER"]
        D1["<b>List content</b><br/>scripts/cognos-discover.sh<br/>GET /objects/{id}/items"]
        D2["<b>Pull artifacts</b><br/>Data Module → GET /metadata/modules/{id}<br/>Report spec → GET /objects/{id}?fields=specification"]
        D1 --> D2
    end

    subgraph CONVERT["2 · CONVERT (translate, never fake)"]
        C1["<b>Data Module → Sigma DM</b><br/>converter/cognos.ts<br/>query subjects→tables · items→cols/metrics<br/>calcs→formulas · relationships→DM rels"]
        C2["<b>Report XML → Sigma workbook</b><br/>converter/cognos-report.ts<br/>lists→tables · dataItems→cols · prompts→controls<br/>⚠ flags macros · running-totals · GetResourceString"]
    end

    subgraph BUILD["3 · BUILD in Sigma"]
        B1["<b>POST data model</b><br/>/v2/dataModels/spec → readback element ids"]
        B2["<b>Wire + POST workbook</b><br/>plug DM ids into the workbook spec<br/>/v2/workbooks/spec"]
        B1 --> B2
    end

    subgraph VERIFY["4 · VERIFY / DATA"]
        V1["<b>Resolve + query</b><br/>columns/metrics/rels resolve (0 errors)"]
        V2["<b>Parity</b> (optional)<br/>land the Cognos source DB in the warehouse<br/>→ Cognos report numbers == Sigma"]
        V1 --> V2
    end

    SIG["🟩 <b>Sigma</b><br/><i>data model + workbook via REST · warehouse via connection</i>"]
    WH[("⛁ Warehouse<br/>Snowflake / BigQuery / Databricks …")]

    COG --> DISCOVER --> CONVERT
    C1 -. "Sigma formulas + ⚠ warnings" .-> C2
    CONVERT --> BUILD --> VERIFY
    B1 ==> SIG
    B2 ==> SIG
    SIG --- WH
    V1 -. "live queries" .-> SIG
```

---

## Translation coverage

| Cognos | Sigma | Status |
|---|---|---|
| Data Module · query subject | data model · table element | ✅ |
| Query item (attribute / fact + `regularAggregate`) | column / metric (`Sum`/`Avg`/…) | ✅ |
| Relationship (`link[].leftRef/rightRef` + cardinality) | DM relationship (source = many side) | ✅ |
| `total(x for a,b)` / aggregates | `SumOver(x,[a],[b])` / `Sum(x)` … | ✅ |
| `if (c) then (a) else (b)` | `If(c, a, b)` | ✅ |
| `_add_days/_add_months/_days_between`, `extract` | `DateAdd` / `DateDiff` / `DatePart` | ✅ |
| `substitute`, `substr`, `cast(x as char)`, `\|\|` | `RegexpReplace`, `Mid`, `Text`, `&` | ✅ |
| Report list + columns + `Summary()`/`Total()` footers | table element + columns + aggregate columns | ✅ |
| `prompt('p')` | control element | ✅ |
| Detail / summary filters | surfaced as warnings to re-create | ⚠ flagged |
| Runtime **macros** (`#…# prompt(…,'token',…)`) | `Switch([control])` placeholder | ⚠ flagged |
| `running-total`, `rank`, `lag/lead`, `GetResourceString` | — (no clean static analog) | ⚠ flagged |
| **Crosstab → pivot-table** (rows edge → rowsBy, columns edge → columnsBy, measure → values) | pivot-table element | ✅ |
| **Charts (RAVE2 `vizControl`)** — clustered/stacked bar & column, line, area, pie, donut, combo, bubble/scatter | bar / line / area / pie / donut / combo / scatter chart | ✅ |
| Cognos map / network / word-cloud / packed-bubble / treemap (no native Sigma chart) | flagged → table fallback | ⚠ flagged |
| Drill-through → actions, Framework Manager `.cpf` | — | ⛔ roadmap |

---

## Quickstart

```bash
cd skills/cognos-to-sigma/converter
npm install
# Data Module JSON → Sigma data model
node --import tsx/esm cli.ts ../fixtures/great-outdoors.module.json --database CSA --schema PUBLIC
# Report XML → Sigma workbook
node --import tsx/esm cli.ts ../fixtures/go-sales-performance.report.xml
npm test     # convert every bundled fixture
```

See **`skills/cognos-to-sigma/SKILL.md`** for the full discover→convert→build→verify workflow, **`QUICKSTART.md`** for a runnable walkthrough, and **`refs/`** for the Cognos REST/format notes + the expression-DSL mapping table.

## Layout

```
.claude-plugin/plugin.json          plugin manifest
skills/cognos-to-sigma/
  SKILL.md                          phased workflow (discover → convert → build → verify)
  QUICKSTART.md                     runnable on the bundled fixtures
  converter/                        the converter (→ vendors into sigma-data-model-mcp)
    cognos.ts                       Data Module JSON → Sigma data model
    cognos-report.ts                report-spec XML → Sigma workbook
    sigma-ids.ts                    shared Sigma id/name/format helpers
    cli.ts · test.ts · package.json
  scripts/cognos-discover.sh        CA REST discovery (session token)
  refs/                             design notes, format shapes, DSL mapping
  fixtures/                         real IBM Cognos sample modules + report
```
