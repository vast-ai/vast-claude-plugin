---
description: Snapshot of Vast.ai instance status — single instance or all running
argument-hint: [instance-id]
allowed-tools: [Bash]
---

# /vast:status

Show the current state of one or all Vast.ai instances. Does **not** poll — this is a single snapshot. For poll-until-running behavior, ask in natural language; the `vastai` skill handles polling with a timeout.

## Steps

1. **If `$ARGUMENTS` is empty:** list all instances.
   ```
   vastai show instances-v1 --raw -a
   ```
   `-a` / `--all` auto-fetches every page, sidestepping the interactive `Fetch next page? (y/N)` prompt that otherwise fires under `--raw`. If you only need one page, pass `--limit <N>` instead — read the current per-page cap from `vastai show instances-v1 --help`.

   The response is an object `{instances: [...], instances_found: N}`. Group `instances[]` by `actual_status` (running / loading / exited / stopped / unknown / offline). Show id, label, gpu_name, num_gpus, and `$/hr` for each.

2. **If `$ARGUMENTS` is a numeric instance id:** show just that instance.
   ```
   vastai show instance "$ARGUMENTS" --raw
   ```
   Report `actual_status`, `intended_status`, `gpu_name`, `cur_state`, `next_state`, and the `$/hr` rate. If `actual_status` is `exited`, `unknown`, or `offline`, flag this as a terminal state — the user should `vastai destroy instance <id> -y` to stop disk charges accruing.

3. **Triage 401 responses.** Both `show instances-v1` and `show instance` are 2FA-gated.
   - Body contains `"Two Factor Authentication"` → recommend `!vastai tfa login --method-type totp --code <CODE>` (`!` prefix keeps the code out of the transcript), then retry.
   - Other 401 → see `/vastai:setup` step 4 triage.

## Notes

- `--raw` is required to parse the JSON response.
- The status enum and what each state means live in `skills/vastai/SKILL.md` § "Instance status values".
- Never loop on this command without a timeout — terminal states (`exited`/`unknown`/`offline`) signal the instance won't recover.
- For SSH connection failures on a `running` instance, pull `vastai logs <id>` FIRST before any other recovery (skill Critical rule 11).
