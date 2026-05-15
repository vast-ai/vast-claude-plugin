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
   vastai show instances-v1 --raw
   ```
   The response is an object `{instances: [...], instances_found: N}`. Group `instances[]` by `actual_status` (running / loading / exited / stopped / unknown / offline). Show id, label, gpu_name, num_gpus, and `$/hr` for each.

2. **If `$ARGUMENTS` is a numeric instance id:** show just that instance.
   ```
   vastai show instance "$ARGUMENTS" --raw
   ```
   Report `actual_status`, `intended_status`, `gpu_name`, `cur_state`, `next_state`, and the `$/hr` rate. If `actual_status` is `exited`, `unknown`, or `offline`, flag this as a terminal state — the user should `vastai destroy instance <id> -y` to stop disk charges accruing.

## Notes

- `--raw` is required to parse the JSON response.
- The status enum and what each state means live in `skills/vastai/SKILL.md` § "Instance status values".
- Never loop on this command without a timeout — terminal states (`exited`/`unknown`/`offline`) signal the instance won't recover.
