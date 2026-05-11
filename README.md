# Flutter-kb

Personal cross-project knowledge base for Flutter mobile development. Captures engineering experience that compounds across projects: principles, patterns, decisions, and pitfalls.

## What this is

A persistent playbook that grows with each Flutter project. Instead of re-deriving the same architectural insights every time a new project starts, this knowledge base accumulates them in a structured form that both humans and AI agents can read.

The knowledge here is:
- **Cross-project** by default — applies to any Flutter project
- **Project-scoped** where appropriate — historical record of decisions specific to a project, kept as a precedent library

## How to navigate

- **`index.md`** — the catalog. Start here to see what is in the wiki.
- **`log.md`** — chronological record of changes, candidates, and lint passes.
- **`CLAUDE.md`** — operational instructions for the AI agent that maintains this base.
- **`_templates/`** — page templates for each content type.
- **`projects/<name>/`** — project-scoped ADRs.

## How it works

Three operations keep the wiki alive:

- **Ingest** — adding new knowledge from project work
- **Lint** — health check for internal consistency
- **Continuous capture** — project agents flag candidates for ingestion in `log.md` during regular development work

All operations are performed by Claude Code under instruction; the human reviews substantive changes.

## Source of truth

When source materials and project code disagree, code wins. This wiki describes how things are, not how they were intended to be.

## Inspired by

This knowledge base follows the pattern described by Andrej Karpathy as LLM Knowledge Bases — using LLMs to maintain personal knowledge that compounds over time. Adapted here for software engineering experience instead of research notes.
