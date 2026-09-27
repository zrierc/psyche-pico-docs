# Toolchain decision: Rust with Embassy, plus Nix

**Decision (2026-09-27, owner): the firmware is Rust with Embassy, built inside a Nix dev shell.** C with the Pico SDK is not used. This supersedes the earlier comparison in this file, which recommended C because Rust lacked a CCID transport on RP2350; that gap is now closed by a working reference.

The sections below keep the evidence, the reason the earlier recommendation flipped, the RP2350 porting work this implies, and what still gates implementation.

## Decision summary

| Item | Choice |
| --- | --- |
| Language | Rust, `no_std`, edition 2021 |
| Async runtime and HAL | Embassy (`embassy-executor`, `embassy-rp`, `embassy-time`, `embassy-sync`, `embassy-futures`) |
| USB | `embassy-usb` with its HID class, plus a hand-written CCID class |
| Crypto | RustCrypto crates (`p256`, `sha2`, `sha1`, `hmac`, `aes`, `cbc`, `zeroize`) |
| Build and dev environment | Nix dev shell and Nix apps; `cargo` for compile, `cargo clippy -D warnings` for lint |
| Target | `thumbv8m.main-none-eabihf`, `embassy-rp` feature `rp235xa` |
| Nushell | not adopted. The firmware needs only `cargo` and Nix; a device CLI can be a small Rust binary later |
| Starting point | port of r17c, used with the author's explicit permission and credited |

## Why the earlier recommendation flipped

The previous version of this document recommended C with the Pico SDK because:

1. C prior art (`pico-fido`, `pico-openpgp`) runs the full feature set on RP2350, including composite USB with HID and CCID.
2. Rust's CCID story was unproven: `embassy-usb` ships HID, CDC, MIDI, DFU, and other classes but **no CCID**, and `usbd-ccid` depends on the synchronous `usb-device` stack.

Both points are still factually true, but point 2 is no longer a blocker. `r17c` (in `r17x/universe`, `apps/r17c`) implements a **custom CCID class directly on `embassy-usb`** in 208 lines (`src/usb/ccid.rs`), and the same firmware implements CTAP2 with Client PIN, an OpenPGP 3.4 card, and OATH TOTP/HOTP as a USB composite device. It runs on RP2040, in Rust, on Embassy, with RustCrypto.

That is a full working reference for this project's feature set in the chosen language. The reason to accept C's different risk profile (manual memory safety in parsers that face an untrusted host) disappeared with it.

```mermaid
flowchart TD
    A["r17c reference found"] --> B{"CCID on embassy-usb proven"}
    B -->|"yes, custom class"} --> C["Rust has no transport gap left"]
    C --> D["Rust chosen: memory safety, one language, Nix"]
    D --> E["RP2350 port work becomes the spike"]
    A --> F["C prior art still useful for protocol cross-checks"]
```

## What r17c gives us

`cited` — `dump/universe/apps/r17c` at commit `12a0bca`, GitHub user `r17x`, 2026-09-22. About 6,200 lines of Rust.

| Area | Detail |
| --- | --- |
| FIDO2 | CTAP2 with discoverable credentials, Client PIN protocol v1 (ECDH P-256 plus AES-256-CBC plus HMAC-SHA256), 8 PIN retries, U2F/CTAP1 compatibility, credential management |
| OpenPGP | Card 3.4 with three P-256 slots (sign, decrypt, auth), on-card keygen and import, works with `gpg --card-status` |
| OATH | TOTP and HOTP, 16 slots, `ykman` compatible, challenge-response |
| USB | Composite: HID interface for CTAPHID, CCID interface for the smartcard applications, IAD declared, ATR provided |
| Storage | Sector-aligned flash regions, staging sectors for compaction, monotonic counter with 1024 32-bit slots, credentials survive reflash |
| Crypto | RustCrypto only, no Rust `unsafe` crypto of their own |
| RNG | ROSC bit sampling on RP2040 (which has no TRNG) |
| Limitations | No RSA, no secure element, user presence via a BOOTSEL-button trampoline, RP2040 only |

Credit and permission: the author granted permission to copy and modify. There is **no `LICENSE` file upstream** (the README badge links to a 404), so the permission itself is the basis for reuse. Requirements that follow from that are listed below.

## RP2350 porting work (the spike)

`cited` — notes 06, 07, 08 and 09 for the chip and board facts.

| Task | Detail |
| --- | --- |
| Target and features | `thumbv6m-none-eabi` to `thumbv8m.main-none-eabihf`; `embassy-rp` feature `rp2040` to `rp235xa`; r17c pins Embassy at a git rev, we move to a released version that supports RP235x |
| USB driver | `embassy-rp` provides the RP235x USB device driver; confirm the HID and boxed CCID path works unchanged |
| RNG | replace ROSC sampling with the RP2350 TRNG (about 7.5 kb/s, added entropy checks, occasional very slow runs), keep a pool and software mixing, never fail open |
| Storage map | re-derive sector offsets for 4 MB flash; 520 KB SRAM instead of 264 KB; re-validate the magic and version scheme; keep staging and compaction |
| User presence | r17c reads BOOTSEL through a QSPI-CS trampoline that does not apply to RP2350; a dedicated GPIO button is needed, wired and checked against the board schematic |
| Gains available | hardware TRNG, secure boot and OTP debug lock, TrustZone-M, more RAM and flash, FPU and DSP. RSA becomes worth re-measuring but is not in scope for the first slice |
| Errata | check the errata list for the chip revision in hand before trusting pads, SIO, or USB behavior (note 08) |
| Spike acceptance | device enumerates as the composite HID plus CCID device, `GetInfo` answers `FIDO_2_0` and `U2F_V2`, `lsusb` shows the expected descriptors, and, as a second gate, `gpg --card-status` reaches the OpenPGP application |

```mermaid
flowchart LR
    S1["1 change target and HAL feature"] --> S2["2 TRNG replaces ROSC"]
    S2 --> S3["3 rebuild storage map for 4 MB"]
    S3 --> S4["4 GPIO user-presence button"]
    S4 --> S5["5 measure flash, RAM, latency"]
    S5 --> S6["6 host interop: lsusb, browser prompt, gpg"]
```

## Attribution and credit requirements

These are mandatory, not optional, and they apply to every derived file:

1. `NOTICE` in the firmware repository: upstream project, repository, and directory, the GitHub user, the commit ported (`12a0bca`), which files were derived, what changed, and the permission basis with its date. Contact details beyond the public handle are not recorded.
2. Per-file header on derived files: origin plus the upstream commit, alongside our own SPDX identifier once a license is chosen.
3. `README.md` credits section: short statement of lineage, link, and author.
4. Commit provenance: the import commit names the upstream repository, directory, and commit, for example `Refs: r17x/universe apps/r17c @12a0bca`. Do not embed personal contact details in commit metadata or history.
5. Preserve upstream file comments and headers. No rename-and-strip.
6. Permission record: a dated summary of what was granted, who granted it, and for which repositories.
7. Reciprocity: port improvements back upstream, starting with the RP2350 port itself.

Open risk, recorded rather than solved: without an upstream license file, permission to us does not automatically extend to downstream users of our repositories. If either repository becomes public with derived code, the written permission must cover redistribution, or upstream must carry a license. Owner decision (2026-09-27): asking upstream for a `LICENSE` is not possible right now, so both repositories stay private until that changes or a permissive grant is recorded in writing.

## Rejected options

| Option | Why not |
| --- | --- |
| C with the Pico SDK (`pico-fido`, `pico-openpgp`) | Still the best protocol cross-check material, and AGPL-3.0. Rejected as the primary because Rust now has a working reference for the same feature set, and parser-facing memory safety matters for a device that faces an untrusted host |
| Rust with the synchronous `usb-device` stack plus `usbd-ccid` | Works, but abandons Embassy's async model for the whole USB layer; r17c shows the async path is viable |
| Trussed-based stack (`fido-authenticator`, `opcard`) | Strong prior art on other chips, but a framework to adopt and a second architecture to learn; r17c's direct Embassy approach is closer to our needs |
| Nushell CLI | Not needed to build or flash; `cargo` plus Nix apps cover the workflow. Revisit only if a device CLI is wanted later |

## Consequences for the repository plan

The trigger in `docs/plans/2026-09-27-repo-layout.md` for creating the firmware repository has fired: the toolchain is decided. The firmware repository (`psyche-pico`) can be created when the port work starts, and this docs repo then pins it as the `code/` submodule.

## Sources

Local working copies are gitignored: specs in `ref/specs/`, reference clones in `ref/repos/`.

- `r17c` — `https://github.com/r17x/universe`, directory `apps/r17c`, commit `12a0bca` (2026-09-22), GitHub user `r17x`. Facts read from its `README.md`, `Cargo.toml`, `src/` tree, and `src/usb/ccid.rs`.
- [embassy-rp on crates.io](https://crates.io/crates/embassy-rp) 0.10.0 — RP2040 and RP235x support, `rp235xa` and `rp235xb` features on `thumbv8m.main-none-eabihf`.
- [embassy-usb on crates.io](https://crates.io/crates/embassy-usb) 0.6.0 — class list, no CCID class.
- [pico-fido source code](https://github.com/polhenarejos/pico-fido) and [pico-openpgp source code](https://github.com/polhenarejos/pico-openpgp), both AGPL-3.0, kept as protocol cross-check material.
- RP2350 datasheet §10.5, §12.12, §12.13 and the errata list — secure boot and OTP flags, TRNG behavior, SHA-256 accelerator.
- Notes in this repo: `learn/06-USB-TRANSPORT.md`, `learn/07-MCU-CRYPTO.md`, `learn/08-RP2350.md`, `learn/09-PICO2-BOARD.md`, `learn/11-PRIOR-ART.md`.
