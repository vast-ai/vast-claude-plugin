---
description: First-time Vast.ai setup — store API key, register SSH key, verify auth
argument-hint: [api-key]
allowed-tools: [Bash, Read]
---

# /vast:setup

First-time setup for the Vast.ai CLI inside Claude Code. Runs the three onboarding steps documented in the `vastai` skill's Quick Start: store the API key, register an SSH key, and verify the credential by querying the user record.

## Steps

1. **Check for an existing `VAST_API_KEY` env var first.** Run:
   ```
   env | grep ^VAST_API_KEY= || echo "not set"
   ```
   If it IS set, warn the user: *"`$VAST_API_KEY` is already set in your shell. It will silently override any key I store with `vastai set api-key`. If you want the stored key to be the active one, run `unset VAST_API_KEY` before continuing."* Wait for the user's decision before proceeding.

2. **Store the API key.** Get the key from <https://console.vast.ai/manage-keys/> (NOT `cloud.vast.ai/account` — that URL doesn't exist).

   - **If `$ARGUMENTS` is non-empty**, treat it as the API key. But the value is already in the transcript by that point, so warn the user it's been logged and they should rotate it after setup.
   - **If `$ARGUMENTS` is empty**, ask the user to paste the command themselves with the `!` prefix so the key stays out of the conversation history:
     ```
     !vastai set api-key <YOUR_KEY>
     ```
     Explain that `!` runs the command in their shell without sending the line to Claude. Do not ask them to paste the key into chat.

3. **Register an SSH key BEFORE the first instance launch.** Check `~/.ssh/id_ed25519.pub` (preferred) then `~/.ssh/id_rsa.pub`. If neither exists, instruct the user to run `ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519` and re-run `/vast:setup`. Once a key file is found, register it (public key is POSITIONAL — there is no `--ssh-key` flag):
   ```
   vastai create ssh-key "$(cat ~/.ssh/id_ed25519.pub)" --raw
   ```
   This step is critical — launching an instance before registering a key produces an unreachable host.

4. **Verify authentication.** Run:
   ```
   vastai show user --raw
   ```
   On success, report the user's email and current balance.

   **Triage `401` responses by reading the body text — the remediation differs:**
   - Body contains `"requires you to have logged in using Two Factor Authentication"` → the account has 2FA enabled and there is no active TFA session. Ask the user to paste the login with `!` prefix to keep the code out of the transcript:
     ```
     !vastai tfa login --method-type totp --code 123456
     ```
     (Use `sms` or `email` instead of `totp` if that's the configured method.) Then re-run step 4. Do NOT ask for a new API key — that will not fix it.
   - Body contains `"permission"` / `"machine_read"` / scope language → the key is valid but lacks the needed permission group. Run `vastai show api-keys --raw` to inspect scope; the user should use their primary key or recreate the scoped key with broader permissions.
   - Otherwise (the env var is shadowing the stored key, or the key is genuinely wrong) → re-check step 1, then suggest `!vastai set api-key <NEW_KEY>` with a fresh key.

## Notes

- All `vastai` invocations include `--raw` so the response is parseable JSON.
- The full command reference, image catalog, and error table live in `skills/vastai/SKILL.md` — load it if any step's behavior is unclear.
- Authentication precedence (used throughout the plugin): `--api-key` flag > `$VAST_API_KEY` > stored key at `~/.config/vastai/vast_api_key`. See the skill's Setup section for the shadowing trap.
