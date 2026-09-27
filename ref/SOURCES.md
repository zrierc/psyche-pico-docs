# ref/ — local references (never tracked)

Everything here is downloaded or cloned for reading. It is deliberately outside the git whitelist in `.gitignore`, so nothing in this directory is committed. Re-download or re-clone rather than relying on these bytes surviving.

## repos/

Shallow clones, read 2026-09-24. Used for architecture study; all three are AGPL-3.0, so read, do not copy.

| Directory | Upstream | Commit | Date |
| --- | --- | --- | --- |
| `repos/pico-fido` | https://github.com/polhenarejos/pico-fido | `c260981` | 2026-09-22 |
| `repos/pico-openpgp` | https://github.com/polhenarejos/pico-openpgp | `50d024d` | 2026-09-18 |
| `repos/pico-keys-sdk` | https://github.com/polhenarejos/pico-keys-sdk | `d075b18` | 2026-09-22 |

Refresh with:

```bash
git -C ref/repos/pico-fido pull
```

## specs/

| File | Source | Used by |
| --- | --- | --- |
| `rp2350-datasheet.pdf` | https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf | notes 07, 08, 11; threat model |
| `pico-2-datasheet.pdf` | https://datasheets.raspberrypi.com/pico/pico-2-datasheet.pdf | note 09 |
| `ccid-rev110.pdf` | https://www.usb.org/sites/default/files/DWG_Smart-Card_CCID_Rev110.pdf | note 06 |
| `openpgp-card-3.4.1.pdf` | https://gnupg.org/ftp/specs/OpenPGP-smart-card-application-3.4.1.pdf | notes 02, 05; threat model |
| `webauthn-3.html` | https://www.w3.org/TR/webauthn-3/ | notes 01, 02, 03 |
| `rfc4226.html` | https://datatracker.ietf.org/doc/html/rfc4226 | notes 02, 04 |
| `rfc6238.html` | https://datatracker.ietf.org/doc/html/rfc6238 | notes 01, 02, 04 |
| `rfc9580.html` | https://datatracker.ietf.org/doc/html/rfc9580 | note 05 |
| `rfc5869.html` | https://datatracker.ietf.org/doc/html/rfc5869 | note 07 |
| `sp800-38d.pdf` | https://csrc.nist.gov/pubs/sp/800/38/d/final | note 07 |

Blocked at the source, retry later or use a mirror: the FIDO Alliance CTAP2 specification page returns HTTP 403 to automated fetches. CTAP2 details in these notes are cited from the `pico-fido` implementation instead.
