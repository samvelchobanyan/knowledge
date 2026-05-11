# Flutter-kb Log

Append-only chronicle of events in the wiki. Three purposes:
- Historical record of when wiki changes happened
- Accumulator of candidates from project work (deposited by project agents during continuous capture)
- Audit trail for lint and re-ingest operations

## Entry format

```
## [YYYY-MM-DD] <event-type> | <context>
<details over one or more lines>
```

## Event types

- `bootstrap` — initial bootstrap completion or partial bootstrap pass
- `ingest` — routine ingest of new knowledge
- `re-ingest` — update of existing pages from changed sources
- `lint` — lint pass with summary of findings
- `create` — targeted page creation
- `candidate` — deposited by a project agent (continuous capture); pending ingest

## Append rules

- Always append at the bottom; never edit existing entries
- Always use the consistent prefix format for parseability
- Always include date in YYYY-MM-DD format

---

## [2026-05-11] init | wiki repository created
Repository initialized at C:\knowledge\flutter-kb\. CLAUDE.md, four templates in _templates/, empty index.md and log.md in place. Bootstrap pending source materials.
