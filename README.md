# Cerebrin Dev Team

Tracker repo for the multi-agent dev team coordinated by **Cerebrin** (PM, Claude Opus 4.7) with local specialists running on Ollama.

## Board

[Cerebrin Team Tracker](https://github.com/users/raulandresmoch/projects/1) — Backlog / To-Do / WIP / Review / Done.

## Team

| Agent | Role | Model | Host |
|---|---|---|---|
| `cerebrin` | PM, code review, complex reasoning | Claude Opus 4.7 (Anthropic) | OpenClaw `main` agent |
| `unicorn` | Specialist — coding, refactor, tests | `qwen2.5-coder:latest` | Ollama, CPU-only |
| `human` | CodeVader | — | CDMX |

## Ticket contract

Every issue must have:

- **Title:** short imperative ("Add X", "Fix Y", "Investigate Z").
- **Description:** context + what / why.
- **Acceptance Criteria:** explicit, checkable list.
- **Project fields:** `Status`, `Agent`, `Skill`, `Priority` (P0–P3).
- **Labels:** at least one of `setup`, `bug`, `feature`, `chore`, `research`, `blocked`.

A ticket is **Done** only when the AC are checked, code is merged (if applicable), and the agent who closed it posts a one-line update in `#tracker-updates` on Discord.

## Specialists workflow

1. `cerebrin` refines a Backlog item into a To-Do with AC.
2. Specialist (`unicorn`) picks the top To-Do tagged with their skill.
3. Specialist moves it to WIP, works in its own workspace, opens a PR.
4. `cerebrin` reviews → Review column.
5. Human approves → Done.

## Repo layout (planned)

- `/tickets-archive/` — markdown copies of important closed tickets, for searchable history.
- `/specs/` — design docs that outlive a single ticket.
- `/playbooks/` — runbooks (deploy, rotate keys, debug ollama, etc.).
