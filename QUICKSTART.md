# Quickstart — Fill WL Forms with X Accounts

Copy-paste skeleton. Replace `<placeholders>`.

```bash
# 1. creds (once per account) — plain KEY=value, NO export, NO quotes
#    ~/.x_creds_alice :  X_AUTH=<40 hex> / X_CT0=<160 hex>
chmod 600 ~/.x_creds_alice

# 2. verify
X_CREDS=~/.x_creds_alice python3 ~/x.py whoami

# 3. health probe
X_CREDS=~/.x_creds_alice python3 ~/x.py follow <target_handle> 2>&1 | head -1

# 4. X actions
X_CREDS=~/.x_creds_alice python3 ~/x.py like   <tweet_id>
X_CREDS=~/.x_creds_alice python3 ~/x.py rt     <tweet_id>
X_CREDS=~/.x_creds_alice python3 ~/x.py quote  <tweet_id> "text variant"
X_CREDS=~/.x_creds_alice python3 ~/x.py reply  <tweet_id> "text variant"

# 5. proof URL = https://x.com/<handle>/status/<tweet_id>

# 6. submit the form (per account, 2-5s apart)
```

## Error codes

| Code | Meaning |
|---|---|
| 32 | token dead |
| 64 | suspended |
| 108 | follow header gate (use REST) |
| 187 | duplicate text |
| 226 | anti-bot cooldown |
| 326 | account locked |
| 344 | daily write limit |

## Golden rules

- **No secrets in git** — env/placeholder only, `chmod 600`.
- **Parse x.py output as JSON**, never regex.
- **Check server-verified vs client-trusted** before spending writes.
- **Vary text per account** (avoid 187).
- **Never invent a tweet ID.**
