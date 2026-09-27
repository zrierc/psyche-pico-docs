# AGENTS.md

Personal learning repo for a DIY USB hardware key on Raspberry Pi Pico 2 (RP2350, non-wireless). The owner has no hardware or electrical-engineering background. The current stage is research-first: learn the RP2350 and Pico 2, learn what hardware keys support, compare technologies, then implement. This file holds operating rules. `README.md` must stay the human-readable project overview: hardware, current stage, toolchain decision, and layout.

## Authoritative references

- Use the official RP2350 datasheet for MCU facts. Verify board-specific facts, including flash size, USB connector, pinout, and power, against Raspberry Pi Pico 2 board documentation before relying on them.
- Use protocol specifications and host tools for compatibility claims: FIDO2/CTAP2/WebAuthn, OpenPGP card, OATH TOTP/HOTP, and relevant vendor apps.
- Candidate prior art: `polhenarejos/pico-fido` for FIDO2/OATH and a Pico USB device stack, `polhenarejos/pico-hsm` for smartcard key storage and crypto operations, and `pico-keys-sdk` as shared glue in those projects. Study them. Do not copy private keys, attestation certificates, PINs, or test secrets. Do not treat a demo build as production-ready.
- Keep a short source list in every research note and plan. Cite datasheet or specification sections for hardware and protocol facts.

## Stage and work modes

- Default to learn and research. Do not start implementation, wiring, soldering, flashing, ordering parts, or release work unless the user explicitly asks.
- Learn mode: explain simply, link sources, save durable notes under `learn/`. Separate measured or tested facts from inference.
- Research and decide mode: write `docs/reviews/<slug>.md` with options, security/interop/resource/toolchain impact, a recommendation, and open questions. Record the decision in `README.md` or docs before implementation.
- Implement mode: follow the task matrix and verification gate below.

## Constraints become design rules

The target is a small USB security device. Already verified in the RP2350 datasheet, chapter 1: dual Cortex-M33 or Hazard3 processors at 150 MHz, 520 KB SRAM in 10 banks, 8 KB OTP, external QSPI flash or PSRAM, USB 1.1 device support, hardware TRNG in section 12.12, SHA-256 accelerator in section 12.13, and secure-boot plus OTP protections in chapters 10 and 13. Do not hard-code board-level details without board documentation.

Derived rules:

- Write a threat model before crypto features: assets, attackers, trust boundaries, PIN and retry-counter behavior, backup and restore, lockout, wipe, and decommissioning.
- Measure before adding: RAM, flash, CPU and interrupt load, USB endpoints, power, and host-compatibility budget. Do not add a protocol, dependency, or USB interface without size, performance, and interop justification.
- Treat compatibility as evidence. Claim support only for host OS, browser, firmware, and tool versions actually tested. Stay within USB 1.1 full-speed device limits until proven otherwise.
- Use hardware randomness for secrets. Prefer the SHA-256 accelerator where available. Use secure boot or OTP-backed key protection when the design needs it.
- Never store keys, PINs, or seeds in plaintext. Zeroize sensitive RAM and flash on failure and reset paths.
- Keep cryptographic code constant-time where applicable. Avoid branching on secrets and avoid secret-dependent memory-access patterns.
- Do not leave SWD or another debug interface enabled in a release image unless the user explicitly accepts that risk in the decision document.
- Do not perform unverified electrical work: check schematic and pinout before applying power, use USB power unless a reviewed external-supply design exists, and keep mains or high-voltage work out of this repo.

## NEVER DO THIS

- Never assume. Check code, schematic, datasheet, or prior art first, then ask when still unclear.
- Never claim Passkey, OpenPGP, or TOTP support from host UI alone. Verify protocol messages, key slots, PIN behavior, and negative paths.
- Never commit or stage unless the user explicitly asks.
- Never add a dependency, crypto library, USB stack, or build tool without justification and a license check.
- Never hard-code secrets, PINs, test keys, or attestation certificates. Never log secrets.
- Never store research speculation as fact. Label findings as `tested`, `measured`, `cited`, or `inferred`.
- Never flash or release from unchecked wiring or from an unsigned release image after release signing exists.
- Never expose private keys except through an explicit encrypted-backup design with a threat model.

## ALWAYS DO THIS

- Ask when requirements are unclear.
- Cite sources for hardware and protocol facts.
- Record decisions in `README.md` or docs, and update plans as facts change.
- Check `skills-lock.json` and inventory `.agents/skills/` before work. Use installed skills when relevant. Ask before installing new skills, and keep skills project-level.
- Keep changes surgical. Match the existing repo style.
- Define verification criteria upfront and loop until verified.

## Task-level matrix

Classify every task by the highest level any factor reaches before planning.

| Factor | S0 Trivial | S1 Small | S2 Medium | S3 Major | S4 Critical |
| --- | --- | --- | --- | --- | --- |
| Scope | Typo, note, or link fix | One note, config value, or small snippet | Research spike, decision review, or small code change across a few files | New protocol or feature, wiring change, or toolchain selection | Crypto core, authentication, PCB, release firmware, or security incident |
| Evidence | None | Source link | Cited research plus reproduction steps | Tested host interop plus resource and security review | Independent verification with risk and rollback review |
| Rollout risk | None | None | Low | Medium | High |

Rules by level:

- S0 and S1: no plan document. Gate: sources cited and note saved.
- S2: `docs/reviews/<slug>.md` or `docs/plans/<slug>.md` required. Gate: research sources, reproduction steps, and plan updated.
- S3: full plan with affected hardware, firmware, hosts, and resources. Gate: clean build where code exists, host interop tests, and plan updated.
- S4: full plan with risk and rollback section, reviewed before implementation. Gate: everything above plus independent verification where feasible.

## Verification gate and Definition of Done

Finish every task with, in order:

1. Sources cited. Label every important finding as `tested`, `measured`, `cited`, or `inferred`.
2. Reproduction steps recorded. For firmware, record commands, firmware hash or version, and board identity in the firmware repository, and summarize the evidence here when a claim in these docs depends on it.
3. Host interop tested before claiming compatibility: USB enumeration plus the relevant app, such as browser WebAuthn, `gpg` or smartcard tools, or an OATH app.
4. Resource report included for firmware changes when available: flash, RAM, CPU, and USB endpoint use.
5. Security checklist done: no secrets committed, retry and lockout behavior tested or explicitly deferred, debug-interface disposition recorded.
6. S2 and above: review or plan updated. S3 and above: plan reflects what actually changed.

Firmware build, check, and test commands live in `code/AGENTS.md` in the firmware repository. This repo's gate covers documentation, decisions, and the evidence they rest on.

## Toolchain

**Decided 2026-09-27: Rust with Embassy, built in a Nix dev shell.** C with the Pico SDK is not used. Evidence and the RP2350 porting plan are in [`docs/reviews/toolchain.md`](docs/reviews/toolchain.md).

- Language and runtime: Rust `no_std` with Embassy (`embassy-executor`, `embassy-rp`, `embassy-usb`, `embassy-time`, `embassy-sync`). Crypto from RustCrypto crates. Target `thumbv8m.main-none-eabihf` with the `embassy-rp` feature `rp235xa`.
- Build and check live in the firmware repo: `cargo build --release`, `cargo clippy --release -- -D warnings`, `cargo test` where host-testable. Nix provides the toolchain and any `nix run` shortcuts. Firmware commands are locked in `code/AGENTS.md`, not here.
- Nushell is not adopted. The firmware needs `cargo` plus Nix only.
- Do not mix toolchains in one firmware image. C prior art (`pico-fido`, `pico-openpgp`) is read for protocol cross-checks only, never compiled into our image.
- Derived code: anything ported from `r17c` carries an origin header, an entry in the firmware repo's `NOTICE`, and a commit that names the upstream commit and author. The credit requirements are mandatory and listed in the toolchain review.
- Rewrite of the port: the RP2350 work implied by the decision (target switch, TRNG, storage map, user-presence button) is the Phase 9 spike scope. Measure before adding features on top.

## Repo layout and ignore policy

This is the documentation repo. Firmware lives in a separate repository named `psyche-pico`, mounted at `code/` as a git submodule once that repository exists. Until then `code/` does not exist and nothing stands in for it.

- `learn/` holds durable learning notes.
- `docs/plans/` holds implementation plans.
- `docs/reviews/` holds research comparisons and decisions.
- `code/` is a git submodule, not a directory of files tracked here. The parent repo pins one commit of the firmware repository.
- `ref/` holds local reference clones and datasheets. It is gitignored and never committed; provenance lives in `ref/SOURCES.md`.
- This repo ignores almost everything by default. Whitelist exact paths in `.gitignore` before tracking new artifacts, including `/.gitmodules` when the submodule is added.

## Firmware repository boundary

- The firmware repository carries its own `AGENTS.md` with its build, check, and test commands. This file governs documentation and decision records; it does not duplicate firmware commands.
- Firmware changes are classified and gated in the firmware repo, then summarized here only when they change a decision or a claim in these docs.
- Code review of firmware belongs in the firmware repository. Reviews, plans, and measurements referenced by these docs may still live here.
- License is per repository. Add a `LICENSE` to each before any code or vendored material lands; the C prior art is AGPL-3.0, so reuse decisions belong with that file.

## Notes and documentation conventions

- `learn/` notes are numbered in roadmap order: `learn/00-ROADMAP.md`, `learn/01-FOUNDATIONS.md`, and so on. One note per phase, named after the roadmap exit.
- Diagrams use **Mermaid** in ```mermaid fences. This is the standard for every protocol exchange, auth flow, state machine, memory or byte layout, and architecture explanation. Do not ship an explanation of a flow as prose alone, and do not use ASCII art.
- Pick the diagram type by job: `sequenceDiagram` for exchanges between parties, `flowchart` for architecture and decision paths, `stateDiagram-v2` for state machines such as PIN retry and lockout, `flowchart` chains for byte and memory layouts.
- Mermaid must render on GitHub with no plugins. Keep node labels short and quote labels that contain punctuation; avoid `click` handlers and external images; avoid raw HTML.
- Written notes stay in normal prose with complete sentences. Links use `[title](url)` markdown form. Label important claims `tested`, `measured`, `cited`, or `inferred`.

## Planning and Git

- Check git status before major changes.
- S2 and above: create `docs/plans/<date>.md` before implementation and update it as requirements clarify.
- When planning, consider edge cases, error handling, failure and recovery behavior, host compatibility, and breaking changes before implementing.
- Never commit or stage unless explicitly asked.
- When asked to commit, use Conventional Commits as `type(scope): subject`, keep the subject at or under 80 characters, and add a body only when it helps.
- Submodule discipline: advancing the `code/` pin is a deliberate change, recorded with the commit hash and the reason in a plan or review. Never edit files inside `code/` as a side effect of a docs task, and never commit inside the submodule unless the user asks for a firmware commit.
- A submodule bump updates a claim: if the new firmware commit changes behavior described in `learn/` or `docs/reviews/`, the affected note is updated in the same task.

## General guidelines

- Think before coding: state assumptions, surface tradeoffs, verify against current code or hardware, and ask when uncertain.
- Simplicity first: minimum code that solves the problem; nothing speculative.
- Surgical changes: touch only what the task requires; do not refactor what is not broken.
- Goal-driven: define verification criteria upfront and loop until verified.
- When fixing bugs: find and analyze the root cause. Fix the cause, not the symptom. No temporary patches.
