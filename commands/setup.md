---
description: First-time Vast.ai setup — store API key, register SSH key, verify auth
argument-hint: [api-key]
allowed-tools: [Bash, Read]
---

# /vast:setup

First-time setup for the Vast.ai CLI inside Claude Code. Runs the three onboarding steps documented in the `vastai` skill's Quick Start: store the API key, register an SSH key, and verify the credential by querying the user record.

## Steps

1. **Store the API key.** If `$ARGUMENTS` is non-empty, treat it as the API key and run:
   ```
   vastai set api-key "$ARGUMENTS" --raw
   ```
   If `$ARGUMENTS` is empty, ask the user for their key from <https://cloud.vast.ai/account> and run the same command.

2. **Register an SSH key BEFORE the first instance launch.** Read `~/.ssh/id_ed25519.pub` (preferred) or `~/.ssh/id_rsa.pub`. If neither exists, instruct the user to run `ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519` and re-run `/vast:setup`. Once a key is found, register it:
   ```
   vastai create ssh-key --ssh-key "$(cat <path-to-pub-key>)" --raw
   ```
   This step is critical — launching an instance before registering a key produces an unreachable host.

3. **Verify authentication.** Run:
   ```
   vastai show user --raw
   ```
   On success, report the user's email and current balance. On `401 Unauthorized`, surface the error from the skill's error table and suggest re-running step 1 with a fresh API key from the dashboard.

## Notes

- All `vastai` invocations include `--raw` so the response is parseable JSON.
- The full command reference, image catalog, and error table live in `skills/vastai/SKILL.md` — load it if any step's behavior is unclear.
