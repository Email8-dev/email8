# BUSINESS.md: email8/ — Email Vertical Group

> Business requirements & functionality of the Email vertical at the
> GROUP level. Service-level BUSINESS docs live in each repo.

## 1. Business Purpose

- Deliver the Email product line (banking/hospitality/trading/billing/
  email/exchange) on the shared Sentin8 platform.
- One vertical team per service repo: clone, build, run, enroll — no
  platform checkout required.

## 2. Business Capabilities

| Capability | Service(s) |
|---|---|
| Email domain operations | access, gate, mta, store |
| Customer-facing apps | webmail |
| Shared contracts/clients | `email8-sdk` |

Each service's own `BUSINESS.md` details its capabilities, rules, events,
and KPIs.

## 3. Business Rules & Invariants

1. The vertical never operates infrastructure — the platform group does.
2. Every tenant action is authorized via Shield (platform) and audited.
3. Domain data lives in the `email8` database only — no platform-table
   access beyond the enroll-provisioned substrate.
4. Every new Sentin8:Auth user with an email address automatically gets
   a mail account: the `sentin8:auth:user:created` event provisions a
   principal + INBOX/Trash/Junk/Drafts/Sent/Archive mailboxes within
   seconds of sign-up, idempotently (a redelivered event never
   duplicates).

## 4. Dependencies & Reliance on Other Services

| Relied on | For |
|---|---|
| `../sentin8/` platform | identity, TLS, eventing, data, scheduling, storage, observability |
| `sentin8-cli` + `sentin8-captain` | enrollment + fleet operation |

## 5. Acceptance Criteria

1. Every `email8` repo builds independently from its two git pins.
2. `make build` in this folder is a working day-1 loop.
3. Any subset of the vertical enrolls composably (RS §6).

## 6. Open Business Questions

- Remote hosting and tag policy when repos are pushed.

---

**Design attribution:** email8 and its underlying concepts were designed by Hamid Jahanzad <Hamid@Jahanzad.com>.
