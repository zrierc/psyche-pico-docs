# psyche-pico

Documentation and research repo for a DIY USB hardware key on a Raspberry Pi Pico 2 (RP2350, non-wireless board). The goal is to learn the microcontroller, learn what hardware keys actually do, and then build one. No hardware or electrical-engineering background assumed.

This repo holds notes, reviews, plans, and local reference material. Firmware lives in a separate repository and is mounted here at `code/` as a git submodule, so the docs and the code can be versioned, reviewed, and licensed independently.

## Current stage: research first

No firmware exists yet, and no toolchain is selected. Work proceeds in three modes:

1. **Learn** — understand the RP2350, the Pico 2 board, and USB security-device basics. Notes go in `learn/`.
2. **Research and decide** — compare options (features, technology, libraries) with security, compatibility, and efficiency trade-offs. Write-ups go in `docs/reviews/`. Decisions are recorded here or in docs before any implementation.
3. **Implement** — only after an explicit decision, following the task matrix and verification gate in `AGENTS.md`. Implementation lands in the firmware repository mounted at `code/`, not in this docs repo.

## Hardware

Raspberry Pi Pico 2 with the RP2350 microcontroller. Verified MCU facts from the RP2350 datasheet, chapter 1:

- Dual Cortex-M33 or Hazard3 processors at 150 MHz
- 520 KB on-chip SRAM in 10 independent banks
- 8 KB one-time-programmable storage (OTP)
- External QSPI flash or PSRAM
- USB 1.1 controller with device support
- Hardware TRNG (section 12.12) and SHA-256 accelerator (section 12.13)
- Secure boot, OTP key protection, bus filtering, and glitch-detector mitigations (chapters 10 and 13)

Board-level details are cited in [`learn/09-PICO2-BOARD.md`](learn/09-PICO2-BOARD.md) from the Pico 2 datasheet: 4 MB flash, micro-USB B, GPIO fixed at 3.3 V, and the power chain limits.

## Candidate features

Nothing is committed yet. Under consideration:

- **Passkey** — FIDO2/CTAP2/WebAuthn authentication, in the style of a YubiKey
- **OpenPGP** — sign, decrypt, and authenticate with on-device keys
- **TOTP/HOTP** — one-time-password credentials
- **Beyond** — backup/restore, multiple USB interfaces, secure boot for the key itself; to be decided after research

Constraints, in order: **security**, **compatibility** (host OS, browsers, tools), **efficiency** (the device is small — RAM, flash, CPU, and USB endpoints are all budgeted).

## Prior art

Study before building. These projects run on the same class of hardware:

- [`polhenarejos/pico-fido`](https://github.com/polhenarejos/pico-fido) — FIDO2/CTAP2/U2F passkeys, OATH TOTP/HOTP, Yubico-app compatibility on Pico/ESP32. FIDO side only, not OpenPGP.
- [`polhenarejos/pico-hsm`](https://github.com/polhenarejos/pico-hsm) — on-device key generation and storage, AES/RSA/ECDSA operations, smartcard-style interface. The closest reference for the OpenPGP-style sign/decrypt/auth role.
- `pico-keys-sdk` — shared glue used by the above projects.

Take architecture and USB-device lessons from them. Do not copy private keys, attestation certificates, PINs, or test secrets.

## Toolchain: decided

**Rust with Embassy, built in a Nix dev shell** (decided 2026-09-27). Rationale and the porting plan: [`docs/reviews/toolchain.md`](docs/reviews/toolchain.md).

The firmware is a port of [`r17x/universe`'s `r17c`](https://github.com/r17x/universe) to the RP2350, used with the author's permission and credited in the firmware repo's `NOTICE`. `r17c` already implements FIDO2/CTAP2, OpenPGP 3.4, and OATH as a composite HID plus CCID device in Rust on Embassy, on RP2040.

Work still ahead before features: switch target and HAL feature to `thumbv8m.main-none-eabihf` and `rp235xa`, replace ROSC entropy with the RP2350 TRNG, re-derive the flash storage map for 4 MB, and design a GPIO user-presence button. C prior art is read for protocol cross-checks only. Build and check commands are locked in the firmware repo's `AGENTS.md`.

## Layout

- `AGENTS.md` — operating rules for agents (and the human): constraints, task levels, verification gate.
- `learn/` — durable learning notes, one topic per note.
- `docs/plans/` — implementation plans.
- `docs/reviews/` — research comparisons and decisions.
- `code/` — mount point for the firmware repository (git submodule). The repo is named `psyche-pico` and is not created yet; the layout decision and wiring checklist are in [`docs/plans/2026-09-27-repo-layout.md`](docs/plans/2026-09-27-repo-layout.md).
- `ref/` — local reference clones and datasheets, gitignored. Provenance in `ref/SOURCES.md`.
- `dump/` — reference agent files from unrelated services (style source only, not project content).
- `.agents/skills/` — project-level skills.

## Status

- [x] Repo scaffolded, agent rules written
- [x] Board documentation checked (`learn/09-PICO2-BOARD.md`, board datasheet cited)
- [x] First `learn/` notes: RP2350, USB transport, FIDO2 vs OpenPGP vs OATH, MCU crypto, prior art
- [x] Threat model written (`docs/reviews/threat-model.md`): backup decision made (no export, reset and re-enroll), release policy made (signed plus OTP debug disable), PIN hardening deferred to spike measurements, four items still pending
- [x] Toolchain decided: Rust with Embassy in a Nix dev shell, porting [`r17c`](https://github.com/r17x/universe) to the RP2350 with the author's permission (`docs/reviews/toolchain.md`)
- [ ] RP2350 port spike: target and HAL switch, TRNG, 4 MB storage map, user-presence button, then measure
- [ ] Feature shortlist with threat-model and resource justification
- [ ] First firmware implementation
