---
description: Snapshot of Vast.ai spend — account balance, active $/hr, recent invoices
argument-hint:
allowed-tools: [Bash]
---

# /vast:cost

Report current Vast.ai spend at a glance.

## Steps

1. **Account balance.** Run:
   ```
   vastai show user --raw
   ```
   Report `credit` (current balance) and `email`. On a 401 with `"Two Factor Authentication"` in the body, the account has 2FA enabled and no active TFA session — recommend `!vastai tfa login --method-type totp --code <CODE>` (with `!` prefix so the code stays out of the transcript), then retry.

2. **Active spend rate.** Run:
   ```
   vastai show instances-v1 --raw -a
   ```
   `-a` / `--all` auto-fetches every page, sidestepping the interactive `Fetch next page? (y/N)` prompt that otherwise blocks non-interactive sessions even under `--raw`. (If `-a` is undesired for some reason, pass `--limit <N>` instead — read the current per-page cap from `vastai show instances-v1 --help`.)

   From the response object, sum `instances[].dph_total` (dollars per hour) across rows whose `actual_status` is `running`. Report as `$/hr` and project a 24h cost.

3. **Recent invoices.** Run:
   ```
   vastai show invoices-v1 -c --raw --limit <N> --latest-first
   ```
   - `-c` (`--charges`) is required to select charges (use `-i` / `--invoices` for paid invoices).
   - `--limit <N>` short-circuits the same pagination prompt; read the current cap from `vastai show invoices-v1 --help`.
   - `--latest-first` is supported on `show invoices-v1` (NOT on `show instances-v1` — different flag sets).

   Summarize `results[]` over the last three months. Group by `charge_type` (`i`=instance, `v`=volume, `s`=serverless) if the user wants a breakdown.

## Notes

- `--raw` is required on every call.
- Use `show invoices-v1`, **not** the legacy `show invoices`.
- Pagination gotcha (skill rule 9): always pass `--limit` on `show instances-v1` and `show invoices-v1`. Each command's `--help` lists its supported flags — they differ.
- See `skills/vastai/SKILL.md` § "Account & Billing" for the full billing surface.
