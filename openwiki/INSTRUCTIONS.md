---
type: Repository guide
title: PowerContext Wiki Instructions
description: Brief for an agent-facing wiki of PowerContext, scoped by the project's official documentation and grounded in its source at commit b8b18909.
tags: [documentation, repository, code-wiki, powercontext]
---

A code wiki for this local repository. Prioritize a concise quickstart, architecture overview, source map, key workflows, domain concepts, operations/runbook notes, testing guidance, and integration points. Inspect git history to understand reasoning behind code changes and the progression of the repository. Keep pages grounded in the repository structure and recent code changes. Prefer practical navigation for engineers over generic summaries.

Readers are AI coding agents that install, configure, use and debug PowerContext and need exact answers from the source. Cover, in this order of priority (the sections of the project's official documentation, `docs/en/index.md`):

1. Quickstart and onboarding: install, configuration wizard, start the Server, connect an Agent and verify memory (docs/en/docs/get-started/quickstart.md, install-and-run.md, configure-server-environment.md, manual-start.md; onboarding steps in docs/en/index.md) — src/, scripts/, docker/, deploy/
2. Continuity: memory, handoff, and resuming work across sessions (docs/en/docs/workflows/memory-and-handoff.md, memory-and-context.md, use-handoff-report.md; docs/en/rfcs/0048_handoff_artifact.md) — src/
3. Ecosystem: connecting agents and integrations (docs/en/docs/integrations/index.md, capabilities.md, claude-code.md, codex.md, hermes.md, openclaw.md, zcode.md) — integrations/, gateway/, .claude-plugin/
4. Artifacts: Memory, Handoff, Experience and Skill (docs/en/docs/workflows/artifacts.md, experience-and-skill-lifecycle.md, create-and-export-skill.md) — src/, openapi/

Name settings, tool parameters and API fields exactly as the code and the wire format spell them, with their defaults.

Do not document: docs/en/blog/ (marketing posts), website/ (marketing site), docs/en/benchmarks/ and evaluation/ (benchmark results), .github/ and docs/en/development/releasing.md (contributor process), LICENSES/, assets/ (images), tests/ and e2e/ fixture data, and generated output that is not hand-written.

Conditions: describe behaviour with its conditions, exceptions and fallback paths. Name the branch taken in each case, for example unless a setting is set, or when a setting is unset, it falls back to its default. Give an order of steps only as the code runs it.

Evidence: cite this repository at commit b8b18909; prefer bounded line ranges (`repo://path#Lx-Ly`) over whole files. A claim about what happens in a function cites the lines of every branch it describes, including the early returns and error paths, not only the main path.
