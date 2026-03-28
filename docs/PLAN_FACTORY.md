# Crucible — Phase 1

## One-line promise
**Turn a legacy estate into deterministic archaeology claims with enough
coverage and provenance to support one bounded replacement outcome.**

This repo is no longer carrying the whole end-state Crucible vision as an
implementation plan. Phase 1 is intentionally smaller:

- ship `crucible scan`
- emit stable `claim.v0` files
- support one real legacy outcome slice
- feed `decoding`

Everything else is deferred until we learn from real slices.

---

## What Phase 1 is

Phase 1 is the archaeology layer for bounded modernization work.

For Hyperion-style knockouts, the first job is not:

- decompose the full estate
- generate the full target architecture
- build twins
- run tournaments

The first job is:

1. inventory the relevant legacy surfaces
2. extract deterministic claims from them
3. preserve provenance tightly enough that humans can resolve ambiguity
4. hand those claims to `decoding`

If Phase 1 does that well, one outcome slice can be scanned, understood, and
rebuilt with a mostly manual replacement loop.

---

## What Phase 1 is not

Phase 1 is NOT:

- `crucible decompose`
- `crucible transform`
- `crucible replay`
- `crucible tournament`
- `twinning` integration
- a general orchestration framework
- a message bus
- a full migration factory

Those may become later phases. They are not part of the current implementation
wedge.

---

## Target user story

For one bounded legacy outcome:

- a governed Hyperion export
- one close report family
- one reconciliation output
- one recurring management package

an operator should be able to run:

```bash
crucible scan db ...
crucible scan repo ...
crucible scan files ...
```

and receive:

- content-addressed claim files
- an inventory of what was scanned
- a scan report showing coverage and gaps
- enough evidence for `decoding` to converge the first map of the slice

That is the Phase 1 bar.

---

## Required surfaces

Phase 1 must cover the four surfaces already identified in the Hyperion use
case:

1. configured surface
2. executed surface
3. human surface
4. consumption surface

Not every source inside those surfaces has to be parsed on day one. But the
scan layer must be able to ingest artifacts from all four classes and emit
claims or explicit "unparsed but inventoried" entries with provenance.

---

## Phase 1 commands

Phase 1 has exactly one top-level command family:

```text
crucible scan <SUBCOMMAND> [OPTIONS]
```

Subcommands:

```text
crucible scan db
crucible scan repo
crucible scan files
```

### `crucible scan db`

Purpose: inspect a legacy database or exported database metadata and emit
claims about structure, usage hints, and liveness evidence.

Minimum inputs:

- live connection string OR exported metadata files
- optional include/exclude filters for schemas and tables

Minimum outputs:

- `claim.v0` for tables, columns, constraints, views, procedures, jobs, and
  observed usage/liveness evidence
- scan report summarizing covered objects and blind spots

### `crucible scan repo`

Purpose: inspect codebases and adjacent application assets that touch the
legacy system.

Minimum Phase 1 coverage:

- SQL string extraction
- connection/config discovery
- ORM/model metadata where obvious
- report definition files where parseable
- tests and enum/constants files where they express semantic or valid-value
  hints

Minimum outputs:

- `claim.v0` for reads, writes, joins, valid values, semantic labels, and
  consumer dependencies
- file-level provenance for every emitted claim

### `crucible scan files`

Purpose: inventory and parse local artifact directories around the legacy
outcome slice.

Typical inputs:

- metadata exports
- load rules
- calc scripts
- scheduler exports
- logs
- output extracts
- downstream deliverables
- spreadsheets and templates

Minimum outputs:

- `claim.v0` where adapters exist
- inventory records plus provenance when adapters do not exist yet
- scan report listing parsed vs inventoried-only artifacts

---

## Phase 1 data model

### `claim.v0`

Phase 1 must emit one stable JSONL contract that `decoding` can consume
directly.

Required top-level fields:

```json
{
  "event": "claim.v0",
  "claim_id": "sha256:...",
  "source": {
    "kind": "db_scan",
    "scanner": "crucible.scan.db@0.1.0",
    "artifact_id": "sha256:...",
    "locator": {
      "kind": "db_object",
      "value": "fin.gl_actuals.amount"
    }
  },
  "subject": {
    "kind": "column",
    "id": "fin.gl_actuals.amount"
  },
  "property_type": "valid_values",
  "value": ["posted", "unposted"],
  "confidence": 0.92
}
```

Rules:

- `claim_id` is content-addressed from normalized payload
- every claim carries a concrete provenance pointer
- confidence is scanner-owned, not model-inferred
- no hidden enrichment

### Frozen Phase 1 field contract

Phase 1 should treat the following fields as frozen:

| Field | Type | Rules |
|-------|------|-------|
| `event` | string | exactly `claim.v0` |
| `claim_id` | string | `sha256:<64 lowercase hex>` |
| `source.kind` | enum | `db_scan`, `repo_scan`, `file_scan` |
| `source.scanner` | string | `<scanner-name>@<version>` |
| `source.artifact_id` | string | `sha256:<64 lowercase hex>` for the scanned artifact or exported metadata unit |
| `source.locator.kind` | enum | `db_object`, `file_range`, `export_row`, `log_line`, `config_key`, `manual_note` |
| `source.locator.value` | string | scanner-specific stable locator |
| `subject.kind` | enum | frozen Phase 1 subject vocabulary |
| `subject.id` | string | normalized stable identifier for the subject |
| `property_type` | enum | frozen Phase 1 archaeology vocabulary |
| `value` | JSON | normalized by `property_type` |
| `confidence` | number | `0.0` to `1.0`, inclusive |

Phase 1 should not add optional top-level fields. If a scanner has extra local
detail, it belongs in the evidence artifact or inventory output, not in the
shared claim wire format.

### Subject identifiers

`subject.kind` is fixed to:

```text
table | column | view | procedure | job | report | feed | artifact | consumer | mapping
```

`subject.id` should follow these conventions:

| Kind | ID convention | Example |
|------|---------------|---------|
| `table` | `<schema>.<table>` | `fin.gl_actuals` |
| `column` | `<schema>.<table>.<column>` | `fin.gl_actuals.amount` |
| `view` | `<schema>.<view>` | `fin.v_close_pack` |
| `procedure` | `<schema>.<procedure>` | `fin.refresh_close_pack` |
| `job` | `<system>.<job_name>` | `autosys.fdmee_load_actuals` |
| `report` | `<system>.<report_name>` | `hyperion.close_pack_ebitda` |
| `feed` | `<system>.<feed_name>` | `fdmee.actuals_load` |
| `artifact` | normalized relative path or export key | `exports/hfm/rules/account_map.csv` |
| `consumer` | `<system>.<consumer_name>` | `finance.board_pack_mailer` |
| `mapping` | `<domain>.<mapping_name>` | `accounts.ebitda_adjustments` |

DB-like identifiers should be lowercased. Paths should use `/` separators. If
the original system is case-sensitive or quoted, preserve the original spelling
in the evidence artifact, not the normalized `subject.id`.

### `value` normalization by property type

Phase 1 freezes the following `value` shapes:

| Property type | `value` shape |
|---------------|---------------|
| `exists` | `true` |
| `schema` | normalized object containing only scalar/array metadata relevant to the subject |
| `constraint` | object: `{"kind":"not_null\|pk\|fk\|check\|unique","detail":{...}}` |
| `reads` | subject ref object: `{"kind":"table","id":"fin.gl_actuals"}` |
| `writes` | subject ref object |
| `depends_on` | subject ref object |
| `used_by` | subject ref object |
| `schedule` | object: `{"kind":"cron\|event\|manual\|unknown","value":"...","timezone":"..."}` |
| `valid_values` | sorted array of strings |
| `semantic_label` | normalized string |
| `liveness` | one of `alive`, `stale`, `unknown`, `dead` |
| `authoritative_for` | subject ref object |

A Phase 1 subject ref object is always:

```json
{
  "kind": "report",
  "id": "hyperion.close_pack_ebitda"
}
```

Raw SQL text, report definitions, stack traces, and other bulky evidence stay
behind the `artifact_id` + `locator` boundary. The claim record carries only
the normalized proposition.

### Claim normalization rules

The claim contract should be reproducible byte-for-byte:

- no timestamps, hostnames, usernames, or run-local counters inside
  `claim.v0`
- object keys serialized in sorted order before hashing
- arrays sorted and de-duplicated when order is not semantically meaningful
- strings trimmed; repeated internal whitespace collapsed only where scanners
  declare the value to be free text
- `claim_id` computed from canonical JSON of every field except `claim_id`
- any proposition outside the frozen subject/property vocabulary must be
  inventoried, not emitted as `claim.v0`

### Output layout

```text
scan-results/
├── claims/
│   ├── db.claims.jsonl
│   ├── repo.claims.jsonl
│   └── files.claims.jsonl
├── inventory/
│   ├── db.inventory.json
│   ├── repo.inventory.json
│   └── files.inventory.json
└── scan-report.json
```

The exact filenames can vary, but this three-part shape may not:

- claims
- inventory
- report

---

## Phase 1 adapters

The scan layer needs a small adapter model, not a giant plugin framework.

Day-one adapters should be hand-written and explicit. Good Phase 1 candidates:

- Oracle / SQL Server / Postgres metadata adapter
- SQL literal extractor for Java and Python
- config / connection string extractor
- filesystem inventory adapter
- simple log / scheduler export adapter

Anything more elaborate should wait until repeated need is proven.

---

## Phase 1 exit criteria

Crucible Phase 1 is implementation-ready when the plan supports shipping:

1. `crucible scan db`
2. `crucible scan repo`
3. `crucible scan files`
4. stable `claim.v0`
5. scan report with coverage / blind spots
6. deterministic reruns for identical inputs

Crucible Phase 1 is functionally successful when one real legacy outcome slice
can be scanned across all relevant surfaces and the resulting claims are good
enough for `decoding` to produce a useful first canonical map.

---

## Phase 1 implementation checklist

Crucible is Phase 1 implementation-ready when this checklist is concrete enough
to code without reopening design questions:

1. **CLI shell**
   - `crucible scan db`
   - `crucible scan repo`
   - `crucible scan files`
   - shared output directory and report writer wiring

2. **Shared claim contract module**
   - frozen enums for `source.kind`, `subject.kind`, and `property_type`
   - canonical JSON serializer
   - `claim_id` builder
   - subject-id normalizer
   - subject-ref/value helpers

3. **Inventory writer**
   - deterministic inventory records for scanned inputs
   - `artifact_id` generation
   - parsed-vs-inventoried-only classification

4. **`scan db` v0**
   - table / column / constraint extraction
   - view / procedure inventory
   - accessible job metadata extraction
   - provenance locators for every emitted claim

5. **`scan repo` v0**
   - SQL literal extraction for Java and Python
   - config / connection discovery
   - enum / constant scanning for `valid_values` and `semantic_label`
   - file-range provenance

6. **`scan files` v0**
   - recursive inventory
   - high-value adapters for scheduler exports, logs, and structured extracts
   - inventory-only fallback when parsing is not implemented

7. **Scan report**
   - claim counts by source kind and property type
   - inventoried-only counts
   - blind-spot summary
   - subject coverage summary

8. **Determinism tests**
   - identical input rerun produces identical claims and report
   - mixed-source fixture test for one bounded legacy slice
   - normalization tests for subject IDs and claim hashing

Phase 1 coding should start only after the first two checklist items are
frozen.

---

## Implementation order

1. **Claim contract**
   Freeze `claim.v0`, scanner identifiers, provenance rules, and output
   directory layout.

2. **`scan db`**
   Ship the highest-confidence scanner first. Tables, columns, PK/FK, NOT NULL,
   CHECK, views, procedures, and accessible job metadata.

3. **`scan repo`**
   Ship SQL string extraction, connection/config discovery, and obvious enum /
   model scanning.

4. **`scan files`**
   Ship filesystem inventory plus a small number of high-value adapters for
   logs, exports, and report artifacts.

5. **Scan report**
   Summarize scanned sources, emitted claims, inventoried-only artifacts, and
   major blind spots.

Everything after that is Phase 2+.

---

## Deferred after Phase 1

These are explicitly parked:

- `crucible decompose`
- `crucible transform`
- `crucible replay`
- `crucible tournament`
- coverage dashboard beyond the scan report
- twin integration
- generic swarm framework
- message bus / event-stream infrastructure

If we build one or two real outcome slices and find ourselves repeating manual
steps around replacement assembly, then we promote the next abstraction into a
real Phase 2 plan.

---

## Relationship to `decoding`

Crucible Phase 1 stops at claims.

```text
legacy estate
  -> crucible scan
  -> claim.v0 JSONL
  -> decoding archaeology mode
```

Crucible does not decide truth in Phase 1. It discovers and records evidence.
`decoding` converges that evidence into a canonical map.

---

## Hyperion slice #1 readiness

We should attempt the first Hyperion slice once all of the following are true:

- one bounded outcome is chosen
- the relevant database surface can be scanned
- the relevant repos can be scanned
- surrounding artifacts can at least be inventoried, even if not all are
  parsed yet
- claim output is stable and deterministic
- `decoding` can consume the emitted claim files without translation glue

That is enough to start learning. It is not enough to automate the whole
program, and it does not need to be.
