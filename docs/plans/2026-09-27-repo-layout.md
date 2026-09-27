# Repo layout: docs repo plus a separate firmware repo

Date: 2026-09-27
Status: decided by owner, not yet executed
Level: S2 (decision and plan, no code)

## Decision

Two repositories, not one:

| Repo | Purpose | State |
| --- | --- | --- |
| `psyche-pico-docs` (this repo) | learn notes, reviews, plans, decisions, local reference material | exists, remote configured |
| `psyche-pico` | firmware source | **not created yet** |

The firmware repository mounts in this one at `code/` as a **git submodule**. Until that repository exists, `code/` does not exist either, and no placeholder directory is committed.

Trigger to create it: **fired 2026-09-27** — the toolchain decision landed (Rust with Embassy, see [`docs/reviews/toolchain.md`](../reviews/toolchain.md)). Create `psyche-pico` when the RP2350 port work starts.

Constraint until further notice: both repositories stay **private** while the ported code rests on a personal permission rather than an upstream license. Recorded in the toolchain review under attribution and credit requirements.

## Why separate

- Firmware needs its own CI (build, and later flash and interop tests) while docs stay fast and simple.
- Licenses differ by artifact. Docs and firmware each carry their own `LICENSE`, and the C prior art is AGPL-3.0, so reuse decisions belong next to the firmware.
- Releases are firmware artifacts (UF2 images, tags). Coupling them to documentation history adds ceremony with no benefit.
- A future firmware-only reader should not have to wade through research notes to build the project.

Trade-off accepted: submodule handling costs a little friction (`git clone --recurse-submodules`, detached HEAD, manual pin bumps). The rules below are what keep that friction from turning into drift.

## Rules once the submodule exists

1. **Pin discipline.** This repo pins one firmware commit. Moving the pin is a deliberate commit whose message names the old and new hash and the reason.
2. **A bump updates claims.** If the new firmware commit changes behavior described in `learn/` or `docs/reviews/`, the affected note is updated in the same task.
3. **Boundary.** Firmware build, check, and test commands live in `code/AGENTS.md`. This repo does not duplicate them.
4. **Evidence.** Firmware evidence (commands, firmware hash, board identity, resource numbers) is recorded in the firmware repo and summarized here only when a claim in these docs depends on it.
5. **No side effects.** A docs task never edits files inside `code/`, and nothing is committed inside the submodule unless the owner asked for a firmware commit.
6. **Clone instructions.** `README.md` states the clone command with `--recurse-submodules` when the submodule lands.

## Wiring checklist (run when the firmware repo exists)

```bash
# from the docs repo root
git submodule add git@github.com:zrierc/psyche-pico.git code
git commit -m "chore(repo): add firmware submodule at code/"
```

1. Add `!code/` to the ignore whitelist in `.gitignore` so the gitlink can be staged (`.gitmodules` is already whitelisted).
2. Update the `README.md` layout section and clone instructions with the real URL.
3. Confirm the firmware repo has its own `AGENTS.md`, `LICENSE`, and `.gitignore` before adding code.
4. Record the pinned hash in the first plan or review that references firmware behavior.

## Open questions

- Does the firmware repo need any docs of its own beyond `AGENTS.md` and a `README.md`, given this repo holds the research? Current answer: `README.md` plus `AGENTS.md` is enough; deep material stays here and is linked.
- Will the firmware repo be public from the start? It affects whether secrets, board serial numbers, or personal hostnames ever appear in its history. Assume public unless the owner says otherwise, and keep such material out of both repos.

## Sources

- `AGENTS.md` in this repo — firmware repository boundary, submodule discipline, verification gate.
- [`docs/reviews/toolchain.md`](../reviews/toolchain.md) — the decision that gates creating the firmware repo.
- Owner decision, 2026-09-27: firmware repository named `psyche-pico`.
