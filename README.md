# Fill WL Forms Using Your X Accounts

A practical, multi-account playbook for submitting NFT **whitelist (WL) / allowlist forms** that require X/Twitter social tasks (follow / like / repost / quote / reply) plus a wallet address.

> **Built for Hermes Agent ops.** This is a workflow guide, not a spam kit. Automating engagement can break platform rules — prefer real accounts, honest proofs, and human pacing.

---

## ⚠️ Security rules (read first)

This repo contains **no secrets**. By design, everything below uses placeholders. When you apply it to your own setup:

| Rule | Why |
|---|---|
| **Never commit credentials** | `auth_token`, `ct0`, private keys, seed phrases, session cookies → keep OUT of git, always in gitignored local files |
| **Env vars, not literals** | Point tools at a creds file via `X_CREDS=...`, never paste tokens into scripts or chat |
| **`chmod 600`** | Creds files and wallet keys must be owner-only (`-rw-------`) |
| **No real addresses in repos** | Use `0xABC…` / `0x…1234` placeholders; real wallets stay in local `~/wallets/` |
| **Revoke if leaked** | If a token appears in chat/logs, revoke it at `x.com/settings/sessions` and re-extract |

### `.gitignore` (use this)

```gitignore
.x_creds*
*.key
wallets_pk.txt
.env
```

---

## What this covers

```text
01 X social tasks   → follow / like / repost / quote / reply (real actions)
02 Proof URL        → capture the tweet URL that proves the action
03 Wallet address   → the EVM address that receives the WL spot
04 SUBMIT           → POST to the form backend (Google Form / Supabase / custom API / GraphQL)
```

A WL form is really two halves glued together:

- **X half** — perform the real social actions, get proof URLs.
- **Form half** — submit handle + wallet + proof URLs to the project's backend.

This guide walks both halves, per account, for N accounts.

---

## 1) Identity mapping (X account ↔ wallet)

Maintain a single source of truth. One X account = one wallet = one form submit (most campaigns).

```json
{
  "accounts": [
    { "name": "alice", "x_handle": "alice_handle",   "creds_file": "~/.x_creds_alice", "wallet": "0xABC…01" },
    { "name": "bob",   "x_handle": "bob_handle",     "creds_file": "~/.x_creds_bob",   "wallet": "0xABC…02" }
  ]
}
```

Never hardcode handle → wallet mappings inside script logic — read them from this file.

---

## 2) Creds setup (placeholder pattern)

One file per account:

```bash
# ~/.x_creds_alice   (chmod 600 — plain KEY=value, no export, no quotes)
X_AUTH=<40-hex-characters>
X_CT0=<160-hex-characters>
```

```bash
chmod 600 ~/.x_creds_alice
ls -la ~/.x_creds_alice   # expect -rw-------
```

Switch identity per command by pointing the env var at a different file:

```bash
X_CREDS=~/.x_creds_alice python3 ~/x.py whoami
X_CREDS=~/.x_creds_bob   python3 ~/x.py post "some text"
```

**Never** write the token inline in a command (it leaks into shell history and chat logs).

---

## 3) Pre-flight: token health check

Before any campaign, classify each account's token state with one cheap probe (a `follow` is the cheapest — a 403 costs nothing):

```bash
for acc in alice bob; do
  echo "=== $acc ==="
  X_CREDS=~/.x_creds_$acc python3 ~/x.py follow <real_target_handle> 2>&1 | head -1
  sleep 1
done
```

| Response | State | Implication |
|---|---|---|
| `401 code:32 "Could not authenticate you"` | **Token dead** | All writes fail — re-extract from browser |
| `403 code:108 "Cannot find specified user"` | v1.1 follow header gate | Follow fails, but **quote/RT still work** |
| `{"ok": true, "following": ...}` | Healthy | Full automation available |

Don't conflate codes: `32` = dead token, `108` = header gate only, `344` = daily budget, `226` = anti-bot cooldown.

---

## 4) The X actions (x.py)

```bash
# Follow (via v1.1 REST — needs browser headers; use the follow_via_rest helper)
python3 scripts/follow_via_rest.py ~/.x_creds_alice <target_user_id> --screen-name <handle>

# Like / repost / quote / reply / post
X_CREDS=~/.x_creds_alice python3 ~/x.py like   <tweet_id>
X_CREDS=~/.x_creds_alice python3 ~/x.py rt     <tweet_id>
X_CREDS=~/.x_creds_alice python3 ~/x.py quote  <tweet_id> "text variant"
X_CREDS=~/.x_creds_alice python3 ~/x.py reply  <tweet_id> "text variant"
X_CREDS=~/.x_creds_alice python3 ~/x.py post   "text"
```

**Always parse output as JSON, not regex** — the response is `{"ok": true, "tweet_id": "…", ...}`. Build the proof URL as `https://x.com/<handle>/status/<tweet_id>`. Never invent a tweet ID.

### Engagement reliability hierarchy (highest first)

| Rank | Action | Reliability |
|---|---|---|
| 1 | `like` | near 100% |
| 2 | `follow` (REST + headers) | high |
| 3 | `reply` | ~3–5 / 7 per wave |
| 4 | `quote` | ~1–3 / 7 per wave |
| 5 | `rt` | intermittent 404 → retry |
| 6 | `post` (new tweet) | most restricted |

### Write-path asymmetry

One endpoint failing ≠ all writes dead. Try **reply → post → quote** (or reverse) before marking an account blocked. `post` 344 may still allow `reply`; `reply` 226 may still allow `post`/`quote`.

---

## 5) Daily write budget (344)

X caps writes per account per day, **shared across post/reply/quote/RT**. Once exhausted you get:

```json
{"code": 344, "message": "…daily limit for sending Tweets and messages…"}
```

Strategy: order accounts by headroom, stagger 30–60s between accounts, vary text per account (see below).

---

## 6) Unique text per account (avoid 187)

`code:187 "Status is a duplicate"` fires on identical text within a window. Vary wording, emoji, and angle:

```text
alice: "learning a lot from your threads @target 🙌 keep them coming"
bob:   "stumbled on your posts @target — solid takes 🔥"
carol: "always good catching up on your work @target 🫡"
```

Length 60–200 chars, different sentence structure per account.

---

## 7) Form-type triage (first 30 seconds)

| Signal | Backend | Submit path |
|---|---|---|
| `docs.google.com/forms` | Google Forms | `formResponse` POST |
| `*.supabase.co/rest/v1/...` | Supabase | REST insert + anon JWT |
| `/api/submit`, `/api/wl/apply` | Custom Next.js / SPA | JSON POST (reverse from Network / JS bundle) |
| GraphQL (`/graphql` mutations) | Custom | mutation call, often client-trusted |
| Apps Script `script.google.com/macros` | GAS | FormData POST |
| Only `localStorage` "registered" | Cosmetic client WL | Real X tasks only — no server row |

**Key insight: "client-trusted" vs "server-verified".** Read the flow before spending writes:

- **Client-trusted** (checkboxes / flags only, no proof URL) → the submit is theater; skip the real X actions the backend ignores.
- **Server-verified** (requires a real tweet URL, checks handle ownership) → do the REAL action, capture the URL.
- **Deferred / snapshot check** (accepts flags now, verifies at a later `closesAt`) → do the real actions before the deadline or get silently dropped.

---

## 8) Submit workflow (per account)

1. **Confirm scope** — which accounts, same/different wallet per account, mint needed?
2. **Pre-flight** — token health check (§3), skip dead/locked accounts.
3. **X actions** — follow → repost → quote/reply per what the form requires (§4), capture proof URL.
4. **Submit** — post the form payload per account, **2–5s apart** (never parallel).
5. **Report** — which accounts submitted + their proof URLs + the form's `ref_slug`/HTTP 200. Never claim "submitted" without a ref/200.

---

## 9) Error code cheat sheet

| Code | Meaning | Fix |
|---|---|---|
| `32` | Token dead / not authenticated | Re-extract `auth_token` + `ct0` |
| `64` | Account suspended | Manual appeal only |
| `108` | v1.1 follow header gate | Use REST + browser headers |
| `187` | Duplicate status | Vary text |
| `226` | Anti-bot cooldown | Wait 15–30 min |
| `326` | Account locked (access control) | Manual unlock (`account/access`) |
| `344` | Daily write limit | Use another account / wait a day |
| empty `tweet_results: {}` | Transient rate limit | Retry once after 60s |

---

## 10) Transport: browser vs bare REST

- **Bare `requests`/`curl`** — OK only for public Supabase / Google Form / Typeform with no session cookie and no burst throttle.
- **Browser in-page `fetch`** (CloakBrowser-class stealth browser) — the default for custom `/api/*` sites that throttle datacenter IPs (`429` / "too many sessions") or sit behind Cloudflare.
- **Browser UI clicks** — only when Turnstile/CAPTCHA forces real clicks, or a simple 1–2 account form.

On the first `429` mid-wave, switch transport immediately — waiting alone is often not enough on the same IP.

---

## 11) Pre-submit checklist

- [ ] No credential in any file, command, or chat (env/placeholder only)
- [ ] Creds files `chmod 600`
- [ ] Identity map read from file, not hardcoded
- [ ] Token health probed per account
- [ ] Proof URLs are real (`x.com/<handle>/status/<id>`), not invented
- [ ] Server-verified vs client-trusted established **before** spending writes
- [ ] Text varied per account (no 187)
- [ ] Submits spaced 2–5s, sequential

---

## References

- X cookie-auth + x.py: see the companion repo `agent-x-twitter-cookie-auth`
- Form-half playbook: `hermes-web-wl-gate`
- Multi-account tactics: `hermes-x-multi-account`
- Hermes Agent: https://github.com/NousResearch/hermes-agent

---

## Disclaimer

Community-oriented ops notes. Platform rules vary — always prefer real accounts and honest proofs. Verify commands against current X behavior; endpoints and rate limits change.
