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
    "kind": "db_scan | repo_scan | file_scan",
    "scanner": "crucible.scan.db@0.1.0",
    "evidence_ref": "content-addressed pointer or file/line locator"
  },
  "subject": {
    "kind": "table | column | view | procedure | job | report | feed | artifact | consumer | mapping",
    "id": "stable subject id"
  },
  "property_type": "stable archaeology vocabulary value",
  "value": {},
  "confidence": 0.0
}
```

Rules:

- `claim_id` is content-addressed from normalized payload
- every claim carries a concrete provenance pointer
- confidence is scanner-owned, not model-inferred
- no hidden enrichment

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
