# Documentation Index

`CLAUDE.md` in the repository root holds the short rules loaded in every Claude Code session. These files hold the detail and are read only when needed.

**New here? Start with [SETUP.md](SETUP.md).**

| File | What it is |
|---|---|
| `SETUP.md` | Install, clone, the task loop, common fixes |
| `PLAN.md` | Milestones M0 to M14, team lanes, waves, decision gates and their state, exit criteria |
| `tasks/BE1.md`, `BE2.md`, `FE1.md`, `FE2.md` | One file per member: task IDs, checkboxes, dependencies, Log |
| `REQUIREMENTS.md` | What must be built (FR/NFR codes), roles, phases, acceptance criteria, open questions |
| `ARCHITECTURE.md` | How the parts fit, which app owns which table, the failure matrix |
| `CONTRACTS.md` | API conventions and the error format (the OpenAPI schema is the source of truth) |
| `API.md` | Every planned API operation: headers, path, query and body in, response out, permission, errors |
| `PAGES.md` | Every frontend page: inputs, outputs, and which part calls which API operation and when |
| `DECISIONS.md` | Design decisions and their reasons |
| `ER-DIAGRAM.md` | Keys and relationships per domain, with constraints and rules |
| `DATA-DICTIONARY.md` | Every table, why it exists, every column |

Evidence for launch (isolation, load, restore, UAT) goes in `results/`, created when the first result exists.

## Rules for the docs

- **Codes are stable.** FR/NFR codes and task IDs (`B1-05`) are referenced from code, tests and commits. Do not renumber.
- **Docs and code change together.** A change to a table, rule or decision updates the matching doc in the same pull request.
- **Decisions are recorded.** Replace a decision with a new entry and mark the old one superseded. Never delete entries.
- **Checkboxes live only in the task files.**
- **One table list.** `ER-DIAGRAM.md`, `DATA-DICTIONARY.md` and the module map in `ARCHITECTURE.md` must always name the same tables. Columns live only in `DATA-DICTIONARY.md`.
- **English only.** Bangla appears only as example data or interface text.