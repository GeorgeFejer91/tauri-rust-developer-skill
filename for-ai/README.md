# For AI

This is the mandatory first-read control plane for **tauri-rust-developer-skill**. It routes agent work; it does not duplicate product documentation or contain product output.

## Boundary

- Keep product source, shipped assets, end-user documentation, and final deliverables outside this folder.
- Keep only durable agent routing, verification policy, decision pointers, and project-memory rules here.
- Existing repository documents remain authoritative for their subjects; link to them instead of copying them.

## Authority and reading order

1. Direct user and system instructions.
2. The nearest applicable `AGENTS.md`.
3. This file.
4. The root `README.md`, task-specific documentation, source, and tests.

If these disagree, stop and reconcile the conflict explicitly. Do not silently weaken a project contract.

## Skills

- For every implementation, fix, refactor, code review, or technical design, use `$ponytail` when available.
- Apply its ladder in order: reuse the repository, then the standard library, then native platform features, then existing dependencies, and add the minimum new code only when those do not solve the task.
- Load other skills only when the task actually matches them. Follow the nearest task-specific skill route and do not create a skill manager or speculative skill inventory.
- YAGNI is a standing rule: do not add behavior, files, dependencies, abstractions, compatibility layers, or configuration without a current requirement and consumer.

## Workflow and self-update

- Define one bounded outcome and its verification before editing.
- Prefer the smallest coherent diff; avoid unrelated refactors, speculative abstractions, duplicate systems, and “for later” scaffolding.
- Update this control plane in the same change only when a durable goal, constraint, workflow, skill route, decision, or readiness gate changes.
- Do not store chat transcripts, daily logs, duplicate status ledgers, generated evidence, or speculative backlogs here.
- Add a new file here only when it has a distinct current owner and consumer. Otherwise extend this router or link to the existing authority.

## Readiness and Git

Work is ready only when the requested behavior is present, focused checks pass, applicable full checks run, the diff is reviewed, and durable instructions still match reality. Report every check that could not run; do not claim CI, deployment, device, or user acceptance without evidence.

Inspect Git state before and after work. Stage only intended paths, keep commits coherent and itemized, and push completed validated work when repository policy and branch state allow it. Never force-push, bypass protection, publish secrets, or absorb unrelated local changes.

## Project routes

- Continue with the remaining project-specific rules in the root `AGENTS.md`.
- Read the root `README.md` when present for product purpose and supported usage.
- Read only the task-relevant build, architecture, testing, and release documents.
