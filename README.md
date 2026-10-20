# Email8 - Sentin8 email vertical

> Secure Post-Quantum-Safe Email Platform That Heals and Scales Itself.

A modern mail server built on Sentin8: four Rust services providing
SMTP, JMAP/IMAP/POP3 mail access, queueing, and client
auto-configuration - with identity, storage, events, TLS, and tenancy
delegated to the platform.

## Services

| Service | Port(s) | Role |
|---|---|---|
| `email8-mta` | 587/25 (SMTP) | Submission + reception, delivery queue, DKIM, reports |
| `email8-store` | HTTPS | Mail data (mailboxes/emails/threads/changelog), JMAP core, blob management |
| `email8-access` | 143/993 (IMAP), 110/995 (POP3), 4190 (Sieve) | Classic protocol sessions over Store's data API |
| `email8-apps` | HTTPS | autoconfig/Autodiscover, well-known redirects, RSVP |

## Layout

```
email8/
├── crates/
│   ├── contracts/       # shared types, events, errors, Store API DTOs
│   ├── smtp-proto/      # RFC 5321 wire protocol
│   ├── imap-proto/      # IMAP4rev2 wire protocol
│   ├── jmap-proto/      # RFC 8620/8621 JSON shapes
│   └── mail-core/       # RFC 5322 parsing + threading
├── services/
│   ├── store/           # mail data service + JMAP
│   ├── mta/             # SMTP + queue
│   ├── access/          # IMAP/POP3/Sieve/DAV
│   └── apps/            # auto-config + well-knowns
├── design/              # technical design docs (read these first)
├── BACKLOG.md           # epics & stories
└── vertical-registry.json
```

## Build & test

```bash
make build-rust   # cargo build --workspace --release
make test-rust    # cargo test --workspace
make lint         # clippy + fmt
```

Deployment is platform-managed:

```bash
sentin8-cli vertical init email8/vertical-registry.json
sentin8-cli vertical enroll email8/vertical-registry.json
make start        # via root Makefile → Sentin8:Captain
```

## Design docs

- `design/sentin8-email8-technical-design.md` - master design
- `design/sentin8-email8-store-technical-design.md` - data model
- `design/sentin8-email8-mta-technical-design.md` - SMTP/queue
- `design/sentin8-email8-access-technical-design.md` - IMAP/POP3/Sieve/DAV
- `design/sentin8-email8-integration-and-migration-technical-design.md` - platform gaps + rollout

## Status

P0/P1 complete (skeleton + core flows). See `BACKLOG.md` for the Epic 3-6
roadmap (DKIM signing, remote delivery, full IMAP parity, POP3/Sieve,
groupware, webmail).

---

**Design attribution:** email8 and its underlying concepts were designed by Hamid Jahanzad <Hamid@Jahanzad.com>.

## Website

The public website at **email8.dev** lives in the sibling
`email8-site` repo - Docusaurus v3 on `@sentin8/site-engine`, with
product/app/SDK pages and a Reference section harvested at build time
from these repositories (program design:
`../design/sentin8-websites-design.md`).
