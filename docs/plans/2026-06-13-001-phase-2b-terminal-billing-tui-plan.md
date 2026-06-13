---
title: "feat: Phase 2b terminal billing — hermes-agent TUI/CLI client + E2E against NAS preview"
status: building
date: 2026-06-13
type: feat
target_repo: hermes-agent
branch: feat/billing-phase-2b-terminal-charge
worktree: sleek-gate (/home/daimon/github/worktrees/hermes-agent/sleek-gate/hermes-agent)
base: origin/main (has merged Phase 2a /credits — PR #44776)
origin: /tmp/handoff-hermes-agent-phase2b-billing.md
cross_repo: nous-account-service PR #412 (server side, preview-deployed, live-verified)
---

# feat: Phase 2b terminal billing — hermes-agent client half

## Summary

Build the **hermes-agent (TUI/CLI) client half** of Phase 2b terminal billing so the whole flow can run end-to-end against the **preview deployment** of `nous-account-service` PR #412 — before that PR merges. The NAS server side (4 `/api/billing/*` endpoints + `billing:manage` scope + per-org kill-switch + settlement polling) is built and CI-green on the preview. This is the missing client half: 5 interactive screens (CLI + TUI) + one new auth wire (`billing:manage` on the device-flow scope, lazy/step-up) + a 9-case E2E run with Stripe test cards.

This extends an **explicitly-stubbed seam**: the merged Phase 2a `/credits` handler docstring (`cli.py`) says *"the terminal never confirms or polls payment (billing phase 2a)."* Phase 2b adds exactly that. We build on `origin/main` which already contains the merged Phase 2a code (`agent/account_usage.py::build_credits_view`, `credits.view` RPC, `credits.ts`).

The 5 design-target screenshots (`/tmp/screenshots/{1..4}.png` + `SCR-20260612-ohvp.png`) are **Claude Code's** terminal billing UI — the layout/copy template. Re-skin for Nous: drop "Save 10/20/30%" volume-discount labels (Nous has no discount), swap "Anthropic"→"Nous Research" consent copy, claude.ai links → Nous portal deep-links.

---

## Step 0 — Contract is live-verified (LOCKED 2026-06-13)

The NAS side is live-verified and out of draft. Server half exercised end-to-end against the preview (real device-flow token, fork DB, sandbox Stripe) — see the [PR #412 E2E comment](https://github.com/NousResearch/nous-account-service/pull/412). The contract below is **observed behavior**.

### ✅ Live probe results (this session)

| Probe | Result |
|-------|--------|
| PR #412 | OPEN, not draft, MERGEABLE; title "feat(billing): phase 2b terminal-native billing…"; **all CI green** (Jest, E2E Browser, Migration Safety, Preview DB Fork, Vercel) |
| `GET /api/billing/state` (no/bogus bearer) | `401 {error:"invalid_token"}` — route live, auth-gated |
| `POST /api/billing/charge` (no auth) | `401` — auth fires *before* idempotency/body validation |
| `GET /api/billing/charge/{id}` (no auth) | `401` |
| `PATCH /api/billing/auto-top-up` (no auth) | `401` — all 4 routes live |
| `POST /api/oauth/device/code` `scope=inference:invoke billing:manage` | **`200`** → real `device_code`, `verification_uri → /manage-subscription`, `interval:5`, `expires_in:600`. **§3 auth wire proven live.** |
| E2E comment cross-check | matches plan: default ceiling `1000`/`isDefaultCeiling:true`, presets `["100","250","500"]`, bounds `10/10000`, strict-body 400s, kill-switch `403 cli_billing_disabled` (gate-4 before card-gate-5), happy `$50` charge → `settled` ~3s, balance 10→60, idempotency **replay** verified, rate limiter `429` on 6th charge/hr |

**Reusable NAS-side runbook:** `docs/specs/phase-2b-e2e-testing-runbook.md` (PR branch) — curl + psql recipes against the fork.

**Cosmetic gotcha:** fork columns are `TIMESTAMP WITHOUT TIME ZONE` (MigrationPilot MP040) → ~7h DB-vs-UTC display skew. If `settledAt` looks off by hours during preview testing, it's this, not a bug — render with explicit TZ handling.

---

## Architecture: reuse the `/credits` pattern verbatim

`/credits` (merged Phase 2a) already touches every layer the billing screens need. Mirror its shape (the `debugging-hermes-tui-commands` skill calls this "one surface-agnostic core + thin per-surface adapters").

| Layer | `/credits` reference (file:line on main) | Phase 2b equivalent |
|-------|------------------------------------------|---------------------|
| Shared core (fetch+parse, dataclass) | `agent/account_usage.py:341` `CreditsView`, `:358` `build_credits_view()` | new `agent/billing_view.py` — `BillingState` dataclass + `build_billing_state()`, `submit_charge()`, `poll_charge()`, `set_auto_reload()` |
| HTTP client to portal | `agent/account_usage.py:384` (ThreadPoolExecutor + `get_nous_portal_account_info`) | new `hermes_cli/nous_billing.py` httpx client (mirror `nous_account.py`); bearer = held Nous JWT |
| JSON-RPC method (TUI) | `tui_gateway/server.py:4687` `@method("credits.view")` | `billing.state`, `billing.charge`, `billing.charge_status`, `billing.auto_reload` |
| CLI handler | `cli.py:8357` `_show_credits()`; dispatch `cli.py:7454` | `_show_billing()`; dispatch `elif canonical == "billing"` |
| Ink TUI command | `ui-tui/src/app/slash/commands/credits.ts` | `ui-tui/src/app/slash/commands/billing.ts` |
| Command registry | `CommandDef("credits", …)` in `hermes_cli/commands.py` | `CommandDef("billing", …, subcommands=("buy","auto-reload","limit"))` |
| Test | `ui-tui/src/__tests__/creditsCommand.test.ts` | `billingCommand.test.ts` + python tests |

**Key reuse facts**
- Auth token: `get_provider_auth_state("nous")["access_token"]` (`account_usage.py:370`) is the bearer for all `/api/billing/*` calls.
- Portal base URL resolution order (copy from `auth.py:7515`): `HERMES_PORTAL_BASE_URL` → `NOUS_PORTAL_BASE_URL` → `PROVIDER_REGISTRY["nous"].portal_base_url`. **Pointing hermes at the preview needs zero code** — just `export HERMES_PORTAL_BASE_URL=https://nous-account-service-git-phase-2b-billing-fixtures.nousresearch.wtf`.
- Fail-open discipline: every core builder returns a logged-out/empty struct on any exception (`account_usage.py:366,373,388`). Billing does the same — a portal hiccup degrades to a clear message, never a crash.
- **Decimal strings, not 2dp**: `/state` returns `"142.5"`. Parse with `decimal.Decimal`, never float, never assume cents. Mirror money-safe parsing in `agent/credits_tracker.py:62 _safe_int` / `:83 _validate_usd`.
- TUI slash-worker is non-interactive (`_app is None`): the CLI handler must gate the prompt path on `if getattr(self, "_app", None):` and render text otherwise — see `cli.py:8398`. A handler that prompts in the worker silently produces nothing.

---

## The one new auth wire: `billing:manage` scope + lazy step-up (handoff §3)

`billing:manage` is privileged, off by default. The device-flow scope already plumbs cleanly:

```
_nous_device_code_login(scope=…)  auth.py:7502/7507/7527/7546
   → _request_device_code(scope=…) auth.py:4336
   → POST {portal}/api/oauth/device/code  data={client_id, scope}  auth.py:4339-4344
```

| Change | Location | Edit |
|--------|----------|------|
| Add scope constant | `auth.py:73` | `NOUS_BILLING_MANAGE_SCOPE = "billing:manage"` |
| Persist granted scope | already handled — `auth.py:7592` stores `token_data.get("scope") or scope` (server downscopes silently if admin didn't tick the box) |
| Step-up trigger | new in `nous_billing.py` | on `403 {error:"insufficient_scope"}` → raise typed `BillingScopeRequired`; CLI/Ink catches → re-run device-connect requesting `billing:manage` + tell user "an ADMIN must tick 'Allow terminal billing'." |
| Mid-session strip | same | scope stripped on refresh if user loses ADMIN → same 403 path re-auths |

**D-A RESOLVED → LAZY/step-up.** Normal `hermes portal` login stays byte-identical (`inference:invoke tool:invoke`, no billing checkbox shown). The first billing action that needs the scope gets `403 insufficient_scope` → triggers the device-connect requesting `billing:manage`. Relies on the step-up handler that must exist anyway (scope stripped on refresh if ADMIN lost). **No eager scope request added to the normal login path.**

---

## The HTTP contract (exact — from handoff §2, live-verified)

All under `{portal}/api/billing/*`, `Authorization: Bearer *** JWT>`. No API-key auth.

| Endpoint | Method | Scope | Notes |
|----------|--------|-------|-------|
| `/api/billing/state` | GET | none | role-tiered; `card/monthlyCap/autoReload` null for MEMBER |
| `/api/billing/auto-top-up` | PATCH | `billing:manage` | strict body `{enabled, threshold>0, topUpAmount>0}`; NO `maxMonthlySpend`/payment-method (400s) |
| `/api/billing/charge` | POST | `billing:manage` | **`Idempotency-Key` header REQUIRED**; body `{amountUsd>0, multipleOf 0.01}`; returns **`202 {chargeId}`** (NOT settled) |
| `/api/billing/charge/{id}` | GET | `billing:manage` | poll; unknown/foreign id → `{status:"pending"}` (never 404) |

`/state` shape (decimal strings): `org{id,slug,name,role}`, `balanceUsd`, `cliBillingEnabled`, `chargePresets[]`, `bounds{minUsd,maxUsd}`, and (ADMIN/OWNER only) `card{brand,last4}`, `monthlyCap{limitUsd,spentThisMonthUsd,isDefaultCeiling}`, `autoReload{enabled,thresholdUsd,reloadToUsd}`.

Charge poll terminal states: `settled{amountUsd,settledAt}` / `failed{reason}` where reason ∈ `authentication_required|payment_method_expired|card_declined|processing_error`.

### Live-run findings that change the TUI (from the PR #412 E2E pass — handoff §8)

1. **`no_payment_method` is a MAINLINE case, not an edge.** A user who only *bought credits* (one-time) has a Stripe card but **no reusable `AutoTopUpSettings.paymentMethodId`** — so `POST /charge` returns `403 no_payment_method` **even right after they paid**. Accepted existing-flow behavior. **Build impact:** Screen 4 handles `no_payment_method` as a *likely* outcome; copy = "set up a saved card on the portal" (the `portalUrl` is the funnel), never "you have no card."
2. **Charge rate limit (5/org/hr + 5/token/hr) is real and easy to trip.** Live, the 6th charge in an hour returned `429`. `429`/`503` carry `Retry-After`, are NOT payment failures — surface "try again in N min," never "charge failed," and **never auto-retry-spam**.
3. **`monthly_cap_exceeded` carries `remainingUsd` + `isDefaultCeiling`; capless orgs hit a default ceiling.** `isDefaultCeiling:true` (e.g. `"1000"`) exposed on `/state.monthlyCap` — Screen 5 shows "X of $1,000 used" with no charge attempt. (Resolves D-B: read-only.)
4. **BetterStack triangulation dead on preview** — verify settlement via the **fork DB directly**, not preview logs.
5. **Confirmed live:** decimal strings, `Idempotency-Key` mandatory (missing=400), poll `pending`→`settled` ~3s, balance credited only *after* settle.

**Error-code → UX map** (every gate denial carries `portalUrl` = `/billing?topup=open` → deep-link):

| HTTP/error | Meaning | TUI action |
|-----------|---------|-----------|
| `403 insufficient_scope` | token lacks `billing:manage` | step-up re-auth (lazy) |
| `403 role_required` | not ADMIN/OWNER | "ask an admin"; portal link |
| `403 cli_billing_disabled` | kill-switch off | "enable terminal billing on portal" + link |
| `403 no_payment_method` | no **reusable** card (live-finding #1) | "set up a saved card on the portal" + link — **expected even for users with billing history** |
| `403 monthly_cap_exceeded` | carries `remainingUsd`,`isDefaultCeiling` | show headroom + portal link |
| `409 idempotency_conflict` | same key, diff amount | bug-guard: never reuse key across amounts |
| `429 rate_limited` (+Retry-After) | limiter | back off; while polling = retry, NOT failure |
| `503 temporarily_unavailable` (+Retry-After:60) | limiter outage | distinct retry-after, NOT a failed payment |

---

## The 5 screens (screen → screenshot → endpoint → behavior)

Render style (all 5 share it): blue top rule, blue bold title, dim secondary lines, `❯` cursor on numbered options, footer `Enter to confirm · Esc to cancel`.

**D-C RESOLVED → reuse existing primitives.** CLI: `_prompt_text_input_modal` (`cli.py:8409`). TUI: existing confirm/input overlay (`patchOverlayState`, `ConfirmReq` in `ui-tui/src/types.ts`). No bespoke multi-field form component.

### Screen 1 — Overview (`1.png`) — `GET /api/billing/state`
- Title "Usage credits". Spend bar: `$X spent [████░░░] N% used` + `Resets <date> · $Y monthly limit` (from `monthlyCap`).
- Line 1 (cursor): `$<balanceUsd> balance · auto-reload <on|off>`.
- Menu: `2. Buy more` `3. Continue with usage credits` `4. Adjust monthly limit` `5. Manage on portal`.
- Render for **everyone**; gate admin actions (2/4/auto-reload) on `org.role ∈ {ADMIN,OWNER}` + `cliBillingEnabled`. MEMBER or kill-switch-off → those rows dimmed/omitted, "Manage on portal" always present.

### Screen 2 — Auto-reload config (`2.png`) — `PATCH /api/billing/auto-top-up`
- Title "Auto-reload". `Card on file: <brand> ····<last4>` (from `/state.card`).
- Two inputs: "When balance falls below: $<threshold>" and "Reload balance to: $<topUpAmount>". Validate both >0; `reloadTo > threshold`.
- Consent (Nous): "By selecting Agree, you authorize Nous Research to automatically charge <card> whenever your balance reaches the threshold… Turn off any time here or at <portal>."
- Confirm → `PATCH {enabled:true, threshold, topUpAmount}`. **No monthly-cap field** (portal-only). `200 {ok:true}` → success; `400 validation_failed{message}` → inline.

### Screen 3 — Buy credits (`3.png`) — selection only
- Title "Buy usage credits". `Payment: <brand> ····<last4>`.
- Render `chargePresets` (server-driven) as numbered rows + `Custom amount…` + `Cancel`. **NO "Save X%" labels.**
- Custom → text input, validate against `bounds{minUsd,maxUsd}`, 2dp / `multipleOf 0.01`. Select → Screen 4.

### Screen 4 — Confirm + charge (`SCR-20260612-ohvp.png`) — `POST /charge` + poll
- Totals: `Subtotal $X` / `Tax (0%) $0.00` / `Total due $X` (one-time = face amount, no tax math). `Payment <brand> ····<last4>`.
- Options: `1. Pay $X now` / `2. Go back`. Consent (Nous): "By confirming, you allow Nous Research to charge your card in the amount above."
- On "Pay": fresh **`Idempotency-Key` (UUID) per user-confirmed purchase**, reuse on retry. `POST /charge` → `202 {chargeId}`.
- **Poll loop (§5):** `GET /charge/{id}` every **2s**, **5-min cap**, **always user-cancellable** (distinct quit-vs-timeout copy).
  - `settled` → "✓ $X credits added" (confirmation = **ledger truth = poll reaching settled**).
  - `failed{authentication_required}` (3DS/SCA) → **fail to portal** w/ deep-link. Inline 3DS NOT in v1.
  - `failed{payment_method_expired|card_declined|processing_error}` → reason + portal link.
  - `403 no_payment_method` at submit (common, finding #1) → "set up a saved card on the portal" + link; likely path, not error.
  - `429`/`503` while polling → back off + continue; never auto-retry-spam (finding #2).
  - `pending` past 5-min cap → **timeout**, not error.

### Screen 5 — Monthly spend limit (`4.png`) — **read-only in TUI**
- Display cap from `/state.monthlyCap` + "Manage on portal" deep-link. TUI can **never set** the cap (D2). Screenshot's `$20` input + buttons are portal-only — do NOT wire.
- Capless orgs hit default ceiling (`isDefaultCeiling:true`, e.g. `"1000"`); render "X of $1,000 used this month" with no charge attempt (finding #3). `monthlyCap` null (MEMBER) → "managed on portal."

---

## Requirements

- R1. `/billing` (+ `buy|auto-reload|limit`) renders all 5 screens, role- and kill-switch-gated, in **CLI + TUI**, byte-identical via one shared core.
- R2. All money parsed/displayed as `Decimal` from server decimal strings — never float, never assumed-2dp.
- R3. `POST /charge` always sends fresh per-purchase `Idempotency-Key`; reused on retry of same purchase; never across amounts.
- R4. Poll loop: 2s, 5-min cap, cancellable, distinct quit-vs-timeout copy; `429/503`=retry not failure; `settled`=only success signal.
- R5. `403 insufficient_scope` triggers lazy step-up re-auth; all other gate denials deep-link to `portalUrl`.
- R6. Every core builder fail-open; no crash on portal hiccup.
- R7. TUI slash-worker path (`_app is None`) renders non-interactive text; never blocks on stdin.
- R8. Nous re-skin: no "Save %", no "Anthropic"/"claude.ai".
- R9. Tests: python (core + error mapping + decimal), TS (`billingCommand.test.ts`), §6 E2E matrix green vs preview.
- R10. No org switcher (token pins `org_id`). No first-time card capture in TUI (portal-only).

---

## Build order (commits)

1. `feat: nous_billing http client + BillingState core` (`agent/billing_view.py`, `hermes_cli/nous_billing.py`) + python unit tests.
2. `feat: billing:manage scope constant + lazy step-up re-auth path` (`auth.py`).
3. `feat: billing JSON-RPC methods` (`tui_gateway/server.py`).
4. `feat: /billing CLI handler + command registry` (`cli.py`, `hermes_cli/commands.py`) — screens 1 & 5 first.
5. `feat: /billing Ink TUI screens 1-5` (`ui-tui/...`) — reuse existing overlays. **Built in-session (D-D).**
6. `test: billingCommand.test.ts + full suites`.
7. E2E matrix vs preview; capture evidence.

---

## E2E test matrix (handoff §6)

Stripe **test** cards, sandbox. Point hermes via `HERMES_PORTAL_BASE_URL`. Unique test org; key assertions on **your** `chargeId`/`pi_` (shared sandbox webhooks fan out — never "latest row"). **Charge rate limit 5/org/hr + 5/token/hr is easy to trip — do NOT burn charges in loops; space runs out or lock yourself out for an hour (finding #2).**

| # | Case | Setup | Expected | Live status (server side) |
|---|------|-------|----------|---------------------------|
| 1 | Enroll admin | device-connect ADMIN, tick "Allow terminal billing" | `/state cliBillingEnabled:true`, card/cap/autoReload populated | ✅ verified |
| 1b | Enroll member | as MEMBER | no checkbox; billing actions 403 | ⚠️ **NOT live-tested** — my run is first |
| 2 | Happy charge | card `4242…` saved via portal first | buy preset → 202 → poll → settled → balance rises | ✅ verified ($50, ~3s, 10→60) |
| 3 | 3DS | `4000 0025 0000 3155` | failed `authentication_required` → bounce to portal | ⚠️ **NOT live-tested** (needs 3DS card) — first |
| 4 | Decline | `4000 0000 0000 9995` | failed | ⚠️ not in E2E comment — first |
| 5 | Kill-switch | flip off between 2 charges | 2nd charge 403 `cli_billing_disabled` | ✅ verified |
| 6 | Cap | low cap, exceed | 403 `monthly_cap_exceeded` + `remainingUsd` → headroom + link | ⏳ **masked by limiter**; default ceiling confirmed in `/state` |
| 7 | No-card (**mainline**) | org that only *bought credits* (Stripe card, no reusable `paymentMethodId`) | 403 `no_payment_method` even post-purchase → "saved card on portal" link | ✅ verified |
| 8 | Idempotency | double-submit same key | one charge, 2nd same `chargeId` | ✅ **replay** verified; ⏳ `409` masked by limiter |
| 9 | Step-up | start w/o `billing:manage` | 1st billing call 403 `insufficient_scope` → TUI re-auths | ⚠️ **NOT live-tested** — first |

**Legend:** ✅ server-verified · ⏳ masked by limiter (space runs out / stage values) · ⚠️ never exercised live — **the TUI E2E run is the FIRST real test** (1b, 3, 4, 9 + cross-org poll + scope-strip-on-refresh). Budget a 2nd test account + a 3DS card.

Triangulate settlement via the **fork DB directly** — BetterStack triangulation does NOT work on the preview (finding #4). See NAS `debugging-billing-and-credits` skill.

---

## Decisions (RESOLVED 2026-06-13)

- **D-A** scope timing → **LAZY/step-up**. Normal login unchanged; first billing action triggers `403 insufficient_scope` → device-connect requesting `billing:manage`.
- **D-B** screen 5 → **read-only display only** (finding #3).
- **D-C** input UI → **reuse existing primitives** (`_prompt_text_input_modal` CLI; Ink confirm/input overlay TUI). No bespoke form.
- **D-D** Ink screens → **built in-session** (not delegated). **Wire the CLI fully too** — both surfaces ship off the shared core.

---

## Gotchas (pinned)

- Decimal strings not 2dp; `Idempotency-Key` mandatory (missing=400); first-time card capture portal-only; terminal pins one org (no switcher); confirmation = poll `settled` not "Stripe charged" not balance-watch.
- **Editable-install routing trap** (`debugging-hermes-tui-commands` skill): the installed `hermes` runs from `~/.hermes/hermes-agent/`, a DIFFERENT tree than this worktree. Test CLI via `python -m hermes_cli.main` from the worktree; test TUI with `HERMES_PYTHON_SRC_ROOT=<worktree>` + rebuild Ink (`npm --prefix ui-tui run build`). Profile/preview selection via `HERMES_HOME` (not `HERMES_PROFILE`).
- Slack 50-slash cap: adding `CommandDef("billing")` may clamp a different low-priority command off Slack and break `test_telegram_parity`. If so, add `billing` (or the victim) to `_SLACK_VIA_HERMES_ONLY` in `commands.py`.
