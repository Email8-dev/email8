# Changelog

All notable changes to the email8 vertical are documented here.

## [Unreleased]

### Changed
- **mimalloc global allocator in every service binary (story 6.8, design §6
  D13)** — all five binaries (`email8-store`, `email8-mta`, `email8-access`,
  `email8-apps`, `email8-gate`) now install `mimalloc` via
  `#[global_allocator]` in `main.rs`; the mimalloc dependency is declared
  once in `[workspace.dependencies]` (`default-features = false` — no
  `override`/`local_dynamic_tls`, no secure mode) and referenced with
  `mimalloc.workspace = true` in each service. Background workers (MTA
  queue dispatch/delivery, FTRL training, DMARC/TLS report windows, DKIM
  rotation sweep, Store JMAP push fan-out + event consumer, Apps WebPush
  poller, Gate connection pump) run inside the same binaries and inherit
  the allocator with no extra wiring. No API, data-model, or security
  change — process-local allocator swap only.

## [0.4.0] — 2026-09-04

### Added
- **Legacy import tool (story 6.6)** — `sentin8-cli vertical import-email8
  <export.json>` imports a legacy Email8 export document (domains, accounts,
  groups, sieve scripts, quota/spam policy) into the email8 schema:
  idempotent `ON CONFLICT` upserts everywhere; per-account bootstrap of the
  standard 6 mailboxes via the same idempotent path the provisioning pipeline
  uses; sieve scripts stored with a blake3 content hash and imported inactive
  (activate via ManageSieve after review); message blobs are deliberately not
  imported — historic `.eml` re-ingest goes through the MTA queue (see the
  operator runbook §import). Gated behind the `email8` cargo feature of the
  CLI (default-on; compile out with `--no-default-features`).
- **Operator runbook (story 6.7)** — `design/sentin8-email8-operator-runbook.md`:
  gate port matrix, full DNS record set per hosted domain (MX, SPF, DKIM,
  DMARC, MTA-STS, TLSRPT, SRV/autoconfig client discovery), domain
  provisioning procedure (+ no-UI product-gap note), quota policy semantics
  (env default vs per-principal, `0` = inherit), spam tuning workflow
  (thresholds, FTRL training, DNSBL, outbound sending quotas), legacy import
  procedure with smoke-test SQL, troubleshooting quick reference.

### Changed
- **Binary-sourced vertical registration (CLI REG-1..8)** — vertical
  registry files slimmed to services+apps (+ legitimate declarative
  `valkey` ACL / `event_triggers` where used) — deployment topology only;
  registration data comes from the binaries via `--print-registration`:
  email8 dropped 15 duplicated `admin_scopes` + `sql_dirs` +
  `event_triggers: []`; trade8 + exchange8 dropped `sql_dirs` (all 20
  binaries embed their SQL) + empty `admin_ui`/`consent_flows`/
  `cli_commands`. `vertical init` now sources pg_roles (roles BEFORE SQL —
  2026-08-28 incident rule) + SQL from the binaries with
  `sentin8.vertical_migrations` tracking; the filesystem `sql_dirs` scan
  remains only as a legacy fallback. Fresh-DB scratch proofs: email8 38
  tables / 38 policies / 0 uncovered / 4 roles; trade8 58 / 232 / 0 / 6;
  exchange8 105 / 420 / 0 / 15 — all exact parity with the live schemas
  (live exchange8 has one `rls_probe` leftover debug table not from any
  init.sql). email8 fresh-DB init exposed + fixed a pre-existing
  `init.sql` ordering bug: `sentin8_current_account_id()`/
  `sentin8_is_env_admin()` were defined after the first policy referencing
  them — worked only because other verticals had created them earlier on
  the shared DB. REG-7 deduplicated the service identity itself:
  `binary` and `subdomain` default to `service_type` (identical in 47/47
  and 35/35 services platform-wide) — the six vertical registries now
  declare bare `service_type` + topology overrides; only the sentin8
  platform registry keeps explicit short-subdomain overrides
  (`auth`, `db`, `sys`). Verified: init+enroll green for all 6 verticals,
  subdomains resolved in `sentin8.vertical_services`, captain catalog
  topology (gate fixed_port 8103) intact. REG-8 removed
  `start_priority` from all vertical registries (26 declarations; tier
  ordering is platform-core machinery — verticals lazy-init their
  dependencies, so the boot-tier barrier was dead time; default 40 and
  the captain field remain for platform-core use).

### Fixed
- **Provisioning pipeline end-to-end** — the env-init consumer path
  (`sentin8:sys:service:updated` → email8-store provisioner) was
  non-functional: ConsumeMessage `env_id` was dropped (bootstrap ran with a
  nil env), successful handling never ACKed (messages stuck in the Valkey
  PEL forever — the XAUTOCLAIM sweep resets the entry idle clock, so
  unACKed entries were never redelivered), and resolved principals skipped
  mailbox bootstrap. All fixed: `event_consumer.rs` parses `env_id` from the
  frame, ACKs on success / NACKs (30 s retry) on error, and
  `provision_principal` runs the idempotent `bootstrap_mailboxes` backfill
  in BOTH branches. Verified end-to-end by event replay (also used to
  provision the e2e persona without a new signup).
- **UUID refactor completed** — the platform-wide i64→Uuid entity-id
  refactor was only half-applied; stale `i64` reads (e.g. `r.get::<i64,_>("id")`
  on UUID columns) crashed Mailbox/get with connection resets. Completed
  across `store_api.rs` contracts, `jmap_objects.rs` (all /get, /query,
  /changes, /set paths), `store_ops.rs` (ingest, changelog, id allocation),
  `data_api.rs`, mta queue/submit, and `init.sql` (thread_id, email_id,
  mailbox_id, changelog record ids → UUID; `blob_links.owner_account_id`
  now nullable). Schema re-applied via `DROP SCHEMA email8 CASCADE` +
  `sentin8-cli vertical init` (no-migration rule). 184 tests green.
- **SMTP plain-DATA mode** — `smtp-proto::receiver` had no BDAT-less DATA
  path: implemented `begin_data()`/dot-unstuffing accumulation until the
  lone `.` terminator, emitting `DataChunk{last}`; accept-data finalizes on
  `bdat_last` regardless of empty chunks.
- **M2M service-to-service auth** — replaced empty static dev tokens with
  DCR-registered M2M clients (`register_or_load_m2m_client` + cached
  `TokenSource`): store↔Storage (`storage_auth.rs`), mta↔store/Storage
  (internal `service_token` + storage client), access↔store (IMAP/POP3/
  Sieve fetch paths), gate unaffected (L4 passthrough). DCR rows land in
  `sentin8.auth_m2m_dcr_credentials`; `EMAIL8_*_TOKEN` env vars remain as
  dev fallbacks.
- **IMAP FETCH BODY[...] sections** — access now slices the raw message:
  `header_block`/`header_fields` free functions emit proper literals for
  `BODY[HEADER]`/`BODY[HEADER.FIELDS (...)]`; separate `BODY[]` and
  `RFC822` arms emit full raw literals (fixes an E0408 or-pattern binding
  error from the same change).
- **Queue retry stall** — due-recipients query matched only
  `status='scheduled'`; `temporary_failure` rows past `next_attempt_at`
  never re-fired (deliveries stalled after one transient error). Now
  matches both. Also: stale `claimed_at` rows from dead workers are
  cleared on scan.
- **Inbound MX tenant resolution** — unauthenticated MX traffic had no
  token/env; `finalize_message` now resolves the environment from
  `email8.domains` over the first local recipient (domain rows currently
  seed via SQL — provisioning UI is a documented product gap).
- **Ingest 422** — `ham_override` in `IngestMessage` lacked `#[serde(default)]`,
  rejecting store→mta ingest payloads that omit it; `blob_links` inserts
  for queue submissions bound NULL owner instead of a scout 0.

### Added (testing)
- **IMAP conformance battery** — committed `e2e-ts/tests/protocol/
  imap_battery.ts` + `test:imap` runner: 11 checks (greeting, AUTHENTICATE
  XOAUTH2 SASL-IR, LIST, SELECT, FETCH `BODY[HEADER.FIELDS]` literal,
  SEARCH, UID FETCH, STORE, ENABLE UTF8=ACCEPT, STATUS, UNSELECT) through
  the gate with PROXY v2; 11/11 passing.

## [0.3.0] — 2026-09-03

### Added (design, pre-implementation)
- **PROXY protocol v2 both sides (story 7.4)** — `email8-contracts::proxy_parse`
  parser (signature/version/family validation, v4+v6 address blocks, byte-layout
  tests); mta SMTP session consumes the gate's header pre-banner into `client_ip`;
  access IMAP/POP3/ManageSieve sessions consume it pre-greeting via shared
  `strip_proxy_v2`; per-listener `EMAIL8_{MTA,ACCESS}_ACCEPT_PROXY` env gates;
  listener-map `unpublish` on SIGTERM/Ctrl-C (spawned watcher); captain
  `spawn_command` prefix-search branch now injects catalog `extra_env` (bugfix —
  only the `binary_path` branch did); `EMAIL8_GATE_PROXY_PROTO=true` shipped.
  Verified: clean banners on SMTP/Submission/IMAP/POP3/ManageSieve through the
  gate, implicit-TLS 465/993/995 forwarded opaquely, and a rolling mta upgrade
  with 60/60 :25 probes OK.
- **Email8:Gate implemented (stories 7.1-7.3, 7.5)** — stateless L4 TCP
  passthrough owning the well-known mail ports (25/587/465/143/993/110/995/4190).
  Upstreams bind OS-ephemeral raw listeners and publish per-instance listener
  maps to Valkey (`captain:listeners:*`, publisher in `email8-contracts`);
  the gate reads Active instances + published maps and dials the right port.
  Verified: SMTP/IMAP through the gate, and a rolling mta upgrade with 60/60
  :25 probes OK (zero refused). `EMAIL8_GATE_PROXY_PROTO` ships **false**
  until 7.4 lands upstream PROXY-v2 parsing. Also: captain CLI
  `RegistryService` now carries `fixed_port`/`upgrade_stop_first`/`extra_env`
  through enroll into the live catalog; mta `queue_events` claim query
  parameter-index bug fixed.
- **Email8:Gate design approved** (Epic 7 stories 7.1–7.6 in BACKLOG;
  `design/sentin8-email8-gate-technical-design.md`) — a stateless L4 TCP
  passthrough owning the well-known mail ports (25/587/465/143/993/110/995/4190),
  forwarding to Active mta/access instances from the captain Valkey registry.
  mta/access revert to auto-pool ports + standard rolling upgrades
  (`upgrade_stop_first` remains only on the gate); implicit-TLS 993/995/465
  included from day one; PROXY protocol v2 for client-IP fidelity.

### Added
- **Platform Epic 0 prerequisites** (sentin8 stories 0.1–0.5 wired to email8):
  - Raw TCP + STARTTLS upgrade path in `sentin8-tls` (inbound STARTTLS wrap live
in the MTA listeners).
  - Storage `put_if_absent` dedup + `head()` (blob de-dup on ingest).
  - DNS resolver profile (`SENTIN8_DNS_*` env) in `sentin8-config`.
  - App passwords in Sentin8:Auth (`/v1/me/app-passwords`, downscoped tokens) —
the IMAP/POP3 SASL bridge accepts them.
  - Generic per-user bus event filter in the BFF subscribe layer.
- **DANE (RFC 7672, story 3.6)** — TLSA lookup at `_25._tcp.{host}`, selector=0
full-cert SHA-256 verification with enforced/opportunistic modes, STARTTLS wrap
integration in the remote dialogue (selector=1 fails closed until platform G3 A
SN.1 lands).
- **JMAP EmailSubmission chaining (story 3.11)** — `onSuccessUpdateEmail` / `on
SuccessDestroyEmail` applied after submit success (draft patch / destroy with ch
angelog + tombstones).
- **iMIP inbound (story 4.6)** — text/calendar detection at ingest with METHOD p
arsing (REQUEST/REPLY/CANCEL), event filing into `email8_calendar_events` (UID u
psert, CANCELLED status), and groupware fan-out events `email8:groupware:event:
{created,updated,cancelled}`.
- **DAV LOCK/UNLOCK/PROPPATCH/ACL (story 4.7)** — exclusive write locks (timeout
parsing, depth, 423 on conflict, Lock-Token header), PROPPATCH set/remove with m
ultistatus responses, ACL method grant parsing; Allow header advertised.
- **JMAP over WebSocket (RFC 8887, story 4.8)** — `GET /jmap/ws` upgrade with `
access_token` query auth, request/response frames through the shared JMAP dispat
ch, 30 s StateChange push polling, plus `PushSubscription/set` (create/update/de
stroy with verification codes) alongside `PushSubscription/get`.
- **Per-env DNSBL zone config (story 5.2)** — `email8.email8_dnsbl_zones` table
(RLS env-scoped) + 60 s TTL zone cache in the MTA; unauthenticated inbound sesse
ions run the DNSBL gate (ip/helo/from/to scopes) against configured zones via th
e system resolver.

### Fixed
- All previously stale BACKLOG marks corrected to their actual implementation st
ate (1.7 Valkey limits, 2.7 ACL grant API, 3.4 ARC, 3.9 ARF viewer, 4.5 groupwar
e JMAP, 4.9 VAPID ES256, 4.11 milters, 5.2 DNSBL wiring, 5.4 training worker, 5.
5 greylist+ham, 5.8 audit, 5.9 stat, 2.13 BFF proxy, 3.6 DANE, 3.11 chaining).

## [0.2.0] — 2026-09-02

### Added
- **New protocol/auth crates** (5, all unit-tested):
  - `email8-mail-auth` — SPF (all mechanisms + qualifiers + macros + redirect), DKIM verify/sign (RSA + Ed25519, relaxed/simple canon, body-hash), DMARC (alignment eval), Authentication-Results builder.
  - `email8-mail-queue` — queue-kind schedule tables (retry/notify/TTL), DSN builder (RFC 3464 multipart/report), reply classification.
  - `email8-sieve` — RFC 5229/5429/5430 compiler + interpreter (header/address/size/body/envelope tests; fileinto/redirect/reject/vacation/keep/discard/stop actions).
  - `email8-spam` — tag rule engine with thresholds, URL/phishing scanner (zero-width, mixed-script, homograph, IP-literal, redirector unrolling), FTRL-Proximal classifier with serde model artifacts.
  - `email8-dav-proto` — namespace-aware XML parse/build, Depth/ETag/If-Match/lock-token helpers.
- **Mta outbound engine** — MX resolution (null-MX + implicit-MX), SmtpClient dialogue (EHLO/LHLO caps, STARTTLS-ready, MAIL SIZE/REQUIRETLS, RCPT, DATA dot-stuffing, BDAT), MTA-STS policy fetch/cache + MX authorization, TLSRPT policy lookup, per-recipient classification with retry/temp/permanent outcomes.
- **Mta inbound pipeline** — SPF check, DKIM verify, DMARC evaluation, Authentication-Results + Received-SPF + Received/Return-Path prepends, spam gate (unauthenticated sessions; tag stack + URL findings + DMARC disposition + heuristics) emitting X-Spam-Status/Score/Result, protocol-level reject on spam Reject / DMARC p=reject.
- **DKIM signing + rotation** — secret-ref keys via sentin8-secrets, active-signer load per env domain, chained-header signing (Received/AR/From/To/Cc/Reply-To/Subject/Date/Message-ID/Content-Type/MIME-Version), pending→active→retiring→retired rotation state machine.
- **DSN + expiry** — RFC 3464 failure DSNs on permanent-fail, delayed DSNs on expiry, double-bounce guard for DSN/report queues.
- **DMARC/TLS aggregate reports** — per-message DMARC evaluation recorded into 24h aggregate windows (RFC 7489 §7.2), window-close worker building RFC 7489 App. C / RFC 8460 §4 JSON and enqueuing to the report queue.
- **Queue quotas** — sender/rcpt/rcpt-domain count+bytes buckets, enqueue gate (`452 4.3.1`), release at terminal recipient state.
- **Access protocol battery** — full POP3 (CAPA/USER-PASS/STAT/LIST/UIDL/RETR/TOP/DELE/RSET/QUIT expunge), ManageSieve RFC 5804 (PUTSCRIPT/CHECKSCRIPT with literal capture + compile validation, LIST/GET/SETACTIVE/DELETE/RENAME), DAV dispatch (PROPFIND d0/d1, REPORT calendar/addressbook/sync-collection, MKCOL/GET/PUT/DELETE), IMAP op battery (FETCH UID/FLAGS/INTERNALDATE/RFC822.SIZE/BODY/ENVELOPE, STORE, SEARCH, EXPUNGE with untagged responses, COPY/MOVE with APPENDUID-style responses, APPEND literal capture via ingest, IDLE acknowledge, ENABLE, CLOSE/UNSELECT).
- **Store JMAP set methods** — Email/set (keywords + mailbox re-place + destroy with tombstones), Mailbox/set (create with id allocation, update name/subscribe, destroy with mailboxHasEmail guard), Identity/get, PushSubscription/get.
- **Store internal endpoints** — mailbox email listing (imap_uid order), email keywords set/clear with changelog, email copy with uid allocation, raw append (blob + ingest in one call), Sieve script CRUD + activate.
- **Real WebPush (RFC 8291)** — ECDH P-256 + HKDF-SHA256 content-key derivation, aes128gcm framing (salt|rs|idlen), AEAD seal with header AAD; polling delivery worker over `email8_push_subscriptions`.
- **Mta admin pages** — queue, dkim, reports, spam (policy row), hooks (stage scripts/milters/HTTP), domains — RFW definitions + data endpoints, registered in the vertical catalog.

### Fixed
- FTRL classifier learned inverted weights (missing z-accumulator; sign error in the L1 solve) — now a correct FTRL-Proximal with persisted `zs` state.
- IMAP receiver now surfaces literal bytes (`Element::Literal`) instead of discarding them — APPEND bodies reach the session layer.

### Known limitations (tracked in BACKLOG.md)
- Raw-TCP STARTTLS wrap is platform gap G1 (TLS terminates via the listener shim).
- VAPID ES256 signer is stubbed (header layout pinned by test); full JWT signing with the env's VAPID key is pending.
- DANE (DNSSEC chain validation), JMAP groupware objects (Calendar/ContactCard), milters client, greylisting, audit events, and stat dashboards remain P2.
- Webmail Flutter app is Epic 6.

## [0.1.0] — 2026-09-02

### Added
- **Email8 workspace scaffold** — 5 protocol/data crates + 4 services:
  - `email8-contracts` — shared enums, events, errors, queue types, Store API DTOs.
  - `email8-smtp-proto` — SMTP command/receiver/reply (RFC 5321, BDAT/CHUNKING, enhanced codes).
  - `email8-imap-proto` — IMAP command/receiver/response (IMAP4rev2 subset, literals, sequence sets).
  - `email8-jmap-proto` — JMAP request/response parsing, capability URNs, back-reference resolution (RFC 8620).
  - `email8-mail-core` — RFC 5322 message parsing, preview generation, thread-key computation.
  - `email8-store` — mail data service (principals, mailboxes, emails, threads, changelog, ACLs, quotas, blobs via Sentin8:Storage) + JMAP core (session, Email/Mailbox/Thread methods, SSE push).
  - `email8-mta` — SMTP submission/reception (SASL XOAUTH2/OAUTHBEARER bridge), queue manager with retry/DSN/expiry schedules, DKIM key + report tables.
  - `email8-access` — IMAP session server over Store data API, POP3/ManageSieve listener scaffolds, DAV capability surface.
  - `email8-apps` — Thunderbird autoconfig, Outlook Autodiscover, JMAP/CALDAV/CARDDAV well-knowns, calendar RSVP stub.
- **Sentin8 platform integration** — RLS on every tenant table (`env_id` + `account_id` with admin bypass), per-service PG roles, Sentin8:Bus events (`email8:smtp:message:received`, `email8:mail:email:ingested`, `email8:mail:thread:merged`), Sentin8:Storage for all blobs, Sentin8:Auth SASL bridge (no local passwords).
- **Vertical registry** — catalog.json + vertical-registry.json for `sentin8-cli vertical init/enroll`.

### Known limitations (tracked in BACKLOG.md)
- IMAP FETCH/SEARCH/STORE/APPEND, POP3/ManageSieve sessions, DKIM signing, DMARC/TLS-Report generation, and remote delivery — all delivered in 0.2.0.
- Webmail Flutter app is Epic 6.