# James's Handover Checklist

**Purpose:** the concrete actions only James can take to actually execute the handover — mostly
granting access and making introductions. This is distinct from
[`open-items.md`](open-items.md), which is what David inherits and works through once he *has*
that access — this file is what needs to happen for him to get it in the first place.

---

## 0. First: create David's Google Workspace login

- [ ] **Create David a Google Workspace account** (e.g. `david@numberhive.org`/`.app` —
  whichever domain the Workspace is actually on). Do this before anything else below: it's the
  identity everything else gets granted to, several of the systems below support "Sign in with
  Google" against it directly (GCP, Firebase, and possibly others), and it means every
  subsequent invite goes to one address David already controls rather than a personal one.
- [ ] Decide separately whether David also needs Workspace **admin** rights (billing, user
  management, domain settings) — that's a distinct, bigger grant from just having an account,
  and doesn't need to happen on day one unless he's taking over Workspace administration itself.

## Access to grant David

Once the Workspace login above exists, grant it (or invite that address) into each of the
following:

- [ ] **Render dashboard** — add David. Covers all live hosting/deploy for
  `number-hive-complete`, `number-hive-newvis`, and `number-hive-admin`. The single most
  operationally important grant on this list.
- [ ] **GCP console** — add David. Needed to resolve whether the abandoned GKE/Pulumi
  infrastructure was ever actually torn down ([`open-items.md`](open-items.md) #1) and whether
  Firebase Hosting is still billing (#2).
- [ ] **MongoDB Atlas** — add David. Needed to confirm/rotate the production credential (#4).
- [ ] **GitHub org (NumberHive)** — confirm David has whatever org role a CTO needs (repo
  admin, ability to archive the two zombie repos — #5).
- [ ] **Stripe dashboard** — add David.
- [ ] **Firebase console** — add David (ties into #2 above).
- [ ] **WordPress admin** (`www.numberhive.app/login`) — add David, or hand over the login.
- [ ] **SendGrid** / **Mailchimp** dashboards — add David if he's taking these over directly.
- [x] **Plane** (project tracker — completed/in-progress/deferred work context) — invite issued;
  export delivered at handover; hosting ends 30 September, so the export is the durable record
  once the tool itself goes away.

## Introductions

- [ ] Introduce David into the GoDaddy/DNS delegation chain — currently James → account holder
  "Chris", authority passed via "Fletch", and doesn't name David at all (#7). Either get him
  added to the account directly, or make sure Chris/Fletch know he's the new point of contact.

## Handing over the credentials themselves

- [ ] Share/transfer the Google Doc referenced throughout `tech-inventory.md` (the
  `[see Google Doc: ...]` placeholders) — access to the systems above doesn't replace David
  needing the doc itself for anything not covered by direct console access.
- [ ] Confirm David has (or is added to) whatever password manager/vault the team actually
  uses, if anything exists beyond that doc.

## Already in motion — separate from this handover, listed for completeness

These surfaced during a separate incorporation/security review (ahead of the
NumberHive/Indigo Tide assignment deed) and are James's to finish regardless of this handover's
timeline — not new items, just worth keeping on one list:

- [ ] Rotate the GitHub PAT embedded in `number-hive-complete`'s local `.git/config` remote URL.
- [ ] Rotate the credentials that were in `server/.env.tmp` in `number-hive-admin`'s git history.

---

## Not on this list

Everything in [`open-items.md`](open-items.md) — the GKE teardown decision, staging ambiguity,
cost pull, security policy, backups, and so on — is David's to work through once he has the
access above, not something to resolve before handover. This file is only the mechanics of
actually getting him set up.

---

*Compiled 2026-08-29, companion to [`README.md`](README.md), [`tech-inventory.md`](tech-inventory.md),
and [`open-items.md`](open-items.md).*
