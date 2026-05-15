# Manual testing — `vast-claude-plugin`

How to take this plugin for a real spin against your Vast.ai account from the shell. Three modes, from cheapest to most realistic.

---

## Prerequisites

- `vastai` 1.0.x on PATH:
  ```bash
  uv tool install --force vastai
  vastai --version       # → 1.0.13 or newer
  ```
- A working Vast.ai account with credit. Get a key from <https://cloud.vast.ai/account>.
- `claude` CLI on PATH, logged in (`claude /login` once) so OAuth credentials live at `~/.claude/.credentials.json`.
- This repo cloned somewhere; we'll call its path `$PLUGIN`:
  ```bash
  PLUGIN=/path/to/vast-claude-plugin
  ```
- An env file with your keys (use the project's `set-env.sh` or build your own). At minimum it should export `VAST_API_KEY`.

### Critical gotcha — unset the Anthropic API key before running

If your env file exports `ANTHROPIC_API_KEY=sk-ant-api03-…` (a real API key), `claude` will prefer it over your OAuth credentials. That means **burning paid API credits** and getting an "Invalid API key" error if the key isn't current. Always do this before launching `claude`:

```bash
source ./set-env.sh
unset ANTHROPIC_API_KEY CLAUDE_API_KEY
```

OAuth-token-shaped `ANTHROPIC_API_KEY` values (those starting with `sk-ant-oat`) are safe to leave set — they go through your subscription.

---

## Mode 1 — Interactive session (start here)

This is the way a real user will use the plugin.

```bash
source ./set-env.sh
unset ANTHROPIC_API_KEY CLAUDE_API_KEY
claude --plugin-dir "$PLUGIN"
```

Inside the session, try:

| Try | What you should see |
|---|---|
| `/help` | The five `vastai` plugin commands listed: `setup`, `status`, `cost`, `search`, `launch` |
| `/vastai:status` | Either a list of running instances or "you have 0 instances" |
| `/vastai:cost` | Your account email, credit balance, active $/hr, projected 24h cost |
| `/vastai:setup` *(or `/vastai:setup <new-key>`)* | The 3-step setup walkthrough: store key → register SSH key → verify auth |
| `/vastai:search gpu_name=RTX_4090` | Top 10 cheapest verified RTX 4090 offers |
| `/vastai:launch <offer-id>` | Launch with defaults: pytorch image, 20 GB disk, `--ssh --direct`, auto label `claude-launch-<ts>` |
| **Natural language: "what's my balance?"** | Skill auto-loads, agent runs `vastai show user --raw`, reports the balance |
| **Natural language: "find me a cheap 4090"** | Either fires `/vastai:search` or runs `vastai search offers ... gpu_name=RTX_4090 ...` directly |
| **Natural language: "kill instance 99999"** | Agent runs `vastai destroy instance 99999 -y --raw` (Vast returns "not found" since the ID is fake — that's fine) |

> If `/help` doesn't show the commands, the plugin didn't load. Check the `--plugin-dir` path and that `$PLUGIN/.claude-plugin/plugin.json` exists.

---

## Mode 2 — Single-prompt non-interactive (good for scripted checks)

Useful for one-off probes or piping output into other tools.

```bash
source ./set-env.sh
unset ANTHROPIC_API_KEY CLAUDE_API_KEY

echo "what's running on my vast account?" | claude -p \
  --plugin-dir "$PLUGIN" \
  --allowed-tools "Bash(vastai:*) Bash(jq:*) Read"
```

Swap the prompt for any of these to spot-check:

| Prompt | Should fire |
|---|---|
| `what's running on my vast account?` | `vastai show instances-v1 --raw` |
| `what's my balance?` | `vastai show user --raw` |
| `find me a cheap 4090` | `vastai search offers 'gpu_name=RTX_4090 ...' --raw` |
| `list my volumes` | `vastai show volumes --raw` |
| `show my serverless endpoints` | `vastai show endpoints --raw` |
| `set HF_TOKEN to hf_abc123 as an env var` | `vastai create env-var HF_TOKEN hf_abc123 --raw` (sometimes `update env-var`) |

To see exactly which CLI calls the agent made, pipe through `--output-format stream-json --include-partial-messages --verbose` and filter for `tool_use`:

```bash
echo "what's my balance?" | claude -p \
  --plugin-dir "$PLUGIN" \
  --allowed-tools "Bash(vastai:*) Bash(jq:*) Read" \
  --output-format stream-json --include-partial-messages --verbose 2>/dev/null \
  | jq -r 'select(.type=="assistant") | .message.content[]? | select(.type=="tool_use") | (.input.command // (.input | tostring))'
```

---

## Mode 3 — End-to-end live lifecycle (spends ≤ $0.50)

The real test: launch a GPU, SSH in, run a command, destroy it. Use a unique label so the cleanup script can find leftovers.

```bash
source ./set-env.sh
unset ANTHROPIC_API_KEY CLAUDE_API_KEY
TS=$(date +%s)

echo "I want to verify a Vast.ai instance lifecycle end-to-end. Do all of this in one go and report what happened:

1. Find the cheapest verified RTX 4090 on-demand offer that is rentable (filter: gpu_name=RTX_4090 num_gpus=1 verified=true rentable=true), under \$0.40/hr.
2. Launch it with the pytorch/pytorch image, --disk 20, --ssh, --direct, and label 'test-rivet-manual-$TS'.
3. Poll show instance until actual_status is 'running' (max 5 minutes). If it hits exited/unknown/offline/created+stopped, stop and report.
4. Once running, get the ssh-url and run 'nvidia-smi' on the instance via vastai execute.
5. Destroy the instance with -y. Confirm it's gone via show instances-v1.
6. Report: offer ID picked, instance ID, total elapsed time, nvidia-smi output (just the GPU line), and final destroy status." \
  | claude -p \
    --plugin-dir "$PLUGIN" \
    --allowed-tools "Bash(vastai:*) Bash(jq:*) Bash(ssh:*) Bash(sleep:*) Bash(date:*) Read"
```

**What to look for in the output:**

- Offer ID + price (should be ≤ $0.40/hr).
- Instance ID and label that matches `test-rivet-manual-$TS`.
- A polling loop that exits cleanly (no infinite loop on a terminal state).
- `nvidia-smi` output containing your GPU's name (RTX 4090 expected).
- A destroy confirmation and a final `instances_found: 0`.
- Total spend: typically $0.02–$0.10 for 5–15 minutes.

### If anything goes sideways

The host machine might be broken (CDI device errors, container shim failures — that's a real Vast.ai failure mode, not a plugin bug). Re-run with the next-cheapest offer.

### Manual cleanup safety net

Run after every Mode 3 test, and any time you suspect a leak:

```bash
vastai show instances-v1 --raw \
  | jq -r '.instances[] | select(.label | tostring | startswith("test-rivet-")) | .id' \
  | xargs -I{} vastai destroy instance {} -y --raw
```

---

## Verifying state without running the plugin

Quick health checks that don't go through `claude`:

```bash
# auth + balance
vastai show user --raw | jq '.email, .credit'

# running instances
vastai show instances-v1 --raw | jq '.instances_found, [.instances[] | {id, actual_status, label}]'

# registered SSH keys
vastai show ssh-keys --raw | jq '.[] | {id, public_key: (.public_key | .[0:50])}'
```

If `show user` returns `{"error": true, "msg": "Invalid user key"}`, your saved key is bad — restore it:

```bash
source ./set-env.sh
vastai set api-key "$VAST_API_KEY" --raw
```

---

## Known false-positive in the test corpus

When you try natural-language prompts, expect these to behave conservatively (they are **not** plugin bugs):

- **"tear down all my instances"** — agent will list instances first and ask for confirmation rather than bulk-destroying. Safety reflex.
- **"rotate my SSH key"** — agent will ask which key (host SSH vs Vast SSH key id) before acting.
- **"snapshot 12345 to myrepo"** — agent will ask whether `myrepo` is a Docker Hub repo or a Vast volume target.

If you want non-interactive behavior for these, prefix the prompt with *"just do it, no clarifying questions"*.

---

## Filing bugs

If the plugin emits a call that drops `--raw`, drops `-y` on destroy, or tries to launch without first registering an SSH key, that's a real bug. File against:

- **Plugin behaviour** → [CWORK-1028](https://linear.app/teraflop/issue/CWORK-1028)
- **`SKILL.md` content** (most "agent did the wrong vastai call" issues land here) → upstream PR to [`vast-ai/vast-cli`](https://github.com/vast-ai/vast-cli) (see `PATCHES.md` in this repo for our pending local patches).
