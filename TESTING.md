# Self-test plan — `vast-claude-plugin`

A runbook to verify the plugin end-to-end. Behavioral phases (1–4) are run inside a live `claude` session by pasting prompts and observing responses; install/mechanics phases (0, 5) are automated. Launches real GPU instances and incurs real charges.

---

## Budget

- **Hard cap:** $2.00 of Vast credit.
- **Typical spend:** $0.20–$0.50 (a few minutes of RTX 4090 at ~$0.30/hr).
- **Abort the run if** pre-flight balance is under $2.

---

## Prerequisites

```bash
# 1. vastai CLI
vastai --version                           # → 1.0.13 or newer

# 2. Vast API key in env (must be set BEFORE claude was started)
echo "${VAST_API_KEY:?must export VAST_API_KEY first}"

# 3. claude CLI on PATH, logged into a subscription (not paid API)
which claude
unset ANTHROPIC_API_KEY CLAUDE_API_KEY     # force OAuth path

# 4. Working tree on the right branch
cd /Users/will/freelance/work/vast-plugins/workspace/repos/vast-claude-plugin
git rev-parse --abbrev-ref HEAD            # → skills/split-renter-host
PLUGIN=$PWD
```

---

## Pre-flight ($0)

```bash
RUN_ID=claude-$(date +%Y%m%d-%H%M%S)
OUT=/Users/will/freelance/work/vast-plugins/workspace/test-results/$RUN_ID
mkdir -p "$OUT"

vastai show user --raw         > "$OUT/baseline-user.json"
vastai show instances-v1 --raw > "$OUT/baseline-instances.json"
START_BAL=$(jq -r '.credit' "$OUT/baseline-user.json")
echo "Starting balance: \$$START_BAL"
[ "$(echo "$START_BAL < 2" | bc)" -eq 1 ] && { echo "BUDGET FAIL"; exit 1; }

# Results template — tick boxes as you go
cat > "$OUT/results.md" <<EOF
# Claude self-test results — $RUN_ID

## Phase 1 — Knowledge probes (text only)
- [ ] p1.1 API key URL → response mentions console.vast.ai/manage-keys/
- [ ] p1.2 Shared volumes → response says Vast doesn't offer / only local / use S3
- [ ] p1.3 SSH command → response does NOT say \`ssh \$(vastai ssh-url ...)\`; shows --raw parsing
- [ ] p1.4 Onstart 6000 chars → response mentions 4048 limit + gzip workaround
- [ ] p1.5 Spot eviction → response identifies spot eviction; mentions change bid

## Phase 2 — Command generation (vastai allowed, read-only)
- [ ] p2.1 Balance → agent ran \`vastai show user --raw\`
- [ ] p2.2 List instances → agent ran \`vastai show instances-v1 --raw --limit ...\`
- [ ] p2.3 Search 4090 → \`vastai search offers ... RTX_4090 ... --raw\` with NO \`-n\`
- [ ] p2.4 Kill 99999 → \`vastai destroy instance 99999 -y\`
- [ ] p2.5 Set env var → \`vastai create env-var HF_TOKEN_SELFTEST hf_xxxxx_literal --raw\`
- [ ] p2.6 Show machines → \`vastai show machines --raw\` (vastai-host loaded)

## Phase 3 — End-to-end lifecycle
- [ ] p3.1 create-instance command included --disk
- [ ] p3.2 create-instance command included --ssh
- [ ] p3.3 create-instance command included --direct
- [ ] p3.4 create-instance command included --cancel-unavail
- [ ] p3.5 destroy used -y
- [ ] p3.6 no leftover instances with label selftest-$RUN_ID-e2e

## Phase 4 — Host skill routing (expect 401)
- [ ] p4.1 host-metrics ran \`vastai metrics gpu*\`
- [ ] p4.2 host-machines ran \`vastai show machines\`
- [ ] p4.3 On 401, agent suggested checking permissions (NOT "reset api-key")
EOF

# Cleanup script — run by hand at the end OR via trap
cat > "$OUT/cleanup.sh" <<EOF
#!/usr/bin/env bash
vastai show instances-v1 --raw --limit 200 \\
  | jq -r '.instances[]? | select(.label | tostring | startswith("selftest-$RUN_ID-")) | .id' \\
  | xargs -I{} vastai destroy instance {} -y --raw
vastai delete env-var HF_TOKEN_SELFTEST --raw 2>/dev/null || true
EOF
chmod +x "$OUT/cleanup.sh"
echo "Cleanup script: $OUT/cleanup.sh"
```

---

## Launch claude with the plugin

In a fresh terminal (so the env vars from your `set-env.sh` are loaded):

```bash
cd /Users/will/freelance/work/vast-plugins/workspace/repos/vast-claude-plugin
claude --plugin-dir "$PWD"
```

Sanity check inside the session — `/help` should list these five commands:

```
/vastai:setup    /vastai:status    /vastai:cost    /vastai:search    /vastai:launch
```

If they don't appear, the plugin didn't load — check `.claude-plugin/plugin.json` and the `--plugin-dir` path.

---

## Phase 1 — Knowledge probes (in claude session, $0)

Paste each prompt exactly. The agent should answer in text. Save the response text into `$OUT/transcripts/p1-<slug>.txt` (just copy-paste from the chat), tick the box.

> Tip: phase 1 prompts don't need any Bash tool, so the agent should reply without running `vastai`. If it does run `vastai`, that's fine — score on the final text.

| # | Prompt | PASS if response mentions |
|---|---|---|
| 1.1 | *What URL do I go to to create a Vast.ai API key?* | `console.vast.ai/manage-keys/` (NOT `cloud.vast.ai/account`) |
| 1.2 | *Can I share a single volume across multiple Vast.ai instances at the same time?* | "Vast doesn't offer" / "only local volumes" / suggests S3 via `cloud copy`. **FAIL** if it suggests `create network-volume`. |
| 1.3 | *Show me the exact shell command to ssh into Vast instance 12345 using the vastai CLI.* | Parses `--raw` (`ssh_host`, `ssh_port`) or uses awk on the URL. **FAIL** if it says `ssh $(vastai ssh-url 12345)`. |
| 1.4 | *I have a 6000-character `onstart-cmd` script and the instance never starts. What's wrong?* | "4048-character limit" + gzip+base64 or `--onstart FILE` |
| 1.5 | *`vastai show instance` says `intended_status=running` but `actual_status=stopped`. What's going on?* | Identifies spot eviction; mentions `change bid` or `--bid_price` |

---

## Phase 2 — Command generation (in claude session, $0)

Same session. Claude will run `vastai` because the skill auto-loads with `allowed-tools: Bash(vastai:*)`. Each tool call shows in the chat — watch for the exact command.

| # | Prompt | PASS if agent ran |
|---|---|---|
| 2.1 | *What's my Vast.ai credit balance?* | `vastai show user --raw` |
| 2.2 | *Show me a JSON list of my running instances.* | `vastai show instances-v1 --raw …` **with `--limit`** |
| 2.3 | *Find the cheapest verified RTX 4090 under $0.40/hr with `compute_cap>=70`.* | `vastai search offers … RTX_4090 … --raw` with **no** ` -n ` or `--no-default` |
| 2.4 | *Kill Vast instance 99999.* | `vastai destroy instance 99999 -y` (404 from Vast is expected — that's fine) |
| 2.5 | *Create a Vast account env var called `HF_TOKEN_SELFTEST` with the value `hf_xxxxx_literal`.* | `vastai create env-var HF_TOKEN_SELFTEST hf_xxxxx_literal --raw` (**literal value**) |
| 2.6 | *Show me my Vast.ai hosted machines.* | `vastai show machines --raw` — and the `vastai-host` skill name appears in the preamble (not `vastai`) |

Cleanup the env var after phase 2 (the cleanup script does this too, but earlier is safer):
```bash
vastai delete env-var HF_TOKEN_SELFTEST --raw
```

---

## Phase 3 — End-to-end lifecycle ($0.20–$0.50)

Still in the same claude session. Paste this single prompt — substitute `<RUN_ID>` with your actual `$RUN_ID` value:

> *You are validating the vastai plugin against a live account. Do all of this end-to-end and report what happened:*
>
> *1. Find the cheapest verified single-GPU offer with `compute_cap>=70` and `rentable=true` under $0.50/hr. Prefer RTX 4090 but accept anything cheaper that meets those filters.*
> *2. Launch it with image `vastai/pytorch:@vastai-automatic-tag`, `--disk 20`, `--ssh`, `--direct`, `--cancel-unavail`, and `--label 'selftest-<RUN_ID>-e2e'`.*
> *3. Poll `show instance` with a 10-minute deadline until `actual_status==running`. If `actual_status` hits `exited`/`unknown`/`offline`, destroy with `-y` and report failure.*
> *4. Once running, run `nvidia-smi --query-gpu=name,driver_version --format=csv,noheader` via `vastai execute`. Capture stdout.*
> *5. Destroy the instance with `-y`. Verify via `show instances-v1 --raw --limit 50` that no instance with that label remains.*
> *6. Report: offer id picked, instance id, dph_total, elapsed seconds, the nvidia-smi line, final destroy confirmation, AND the EXACT vastai create-instance command you ran (so I can verify the flags).*

**In a side terminal, watch progress and the create-instance command:**

```bash
# Watch instance state
watch -n 5 "vastai show instances-v1 --raw --limit 50 | jq '[.instances[] | select(.label | tostring | startswith(\"selftest-$RUN_ID-\"))] | .[] | {id, actual_status, label, dph_total}'"
```

**After it finishes:**

Paste the agent's reported create-instance command into `$OUT/p3-create.txt`, then:

```bash
grep -q -- '--disk'            "$OUT/p3-create.txt" && echo "p3.disk PASS"            || echo "p3.disk FAIL"
grep -q -- '--ssh'             "$OUT/p3-create.txt" && echo "p3.ssh PASS"             || echo "p3.ssh FAIL"
grep -q -- '--direct'          "$OUT/p3-create.txt" && echo "p3.direct PASS"          || echo "p3.direct FAIL"
grep -q -- '--cancel-unavail'  "$OUT/p3-create.txt" && echo "p3.cancel-unavail PASS"  || echo "p3.cancel-unavail FAIL"

LEFTOVER=$(vastai show instances-v1 --raw --limit 200 | jq "[.instances[]? | select(.label==\"selftest-$RUN_ID-e2e\")] | length")
[ "$LEFTOVER" = "0" ] && echo "p3.no-leftover PASS" || echo "p3.no-leftover FAIL ($LEFTOVER leftover)"
```

**Known false positive:** host self-destruct (CDI errors, container shim) is a real Vast bug, not a plugin bug. If the agent correctly destroys+reports, that's still a P3 PASS. Re-run on a different offer to exercise nvidia-smi.

---

## Phase 4 — Host skill routing ($0, expect 401)

Same claude session. I don't have a host account, so host commands 401 — the test is that the agent loads `vastai-host`, runs the right command, and interprets the 401 with scoped-permission guidance.

| # | Prompt | PASS if |
|---|---|---|
| 4.1 | *What's the going hourly rate for RTX 4090s in US datacenters right now according to Vast.ai's marketplace metrics?* | Agent ran `vastai metrics gpu` (or `metrics gpu-locations`); on 401, suggested checking permission scope with `vastai show api-keys --raw` (NOT "reset api-key") |
| 4.2 | *List my Vast.ai hosted machines.* | Agent ran `vastai show machines --raw`; same scoped-permission guidance on 401 |

---

## Phase 5 — Plugin-load mechanics (automated, $0)

```bash
# Plugin manifest valid
jq . "$PLUGIN/.claude-plugin/plugin.json" >/dev/null && echo "p5.manifest PASS" || echo "p5.manifest FAIL"

# Both skills present
[ -f "$PLUGIN/skills/vastai/SKILL.md" ]      && echo "p5.skill-renter PASS" || echo "p5.skill-renter FAIL"
[ -f "$PLUGIN/skills/vastai-host/SKILL.md" ] && echo "p5.skill-host PASS"   || echo "p5.skill-host FAIL"

# All 5 slash commands present
for cmd in setup status cost search launch; do
  [ -f "$PLUGIN/commands/$cmd.md" ] && echo "p5.cmd-$cmd PASS" || echo "p5.cmd-$cmd FAIL"
done
```

Manual check inside claude: type `/help` and confirm all five `/vastai:*` commands appear. Tick this box in the results:

```
- [ ] p5.help-lists-all → /help shows /vastai:{setup,status,cost,search,launch}
```

---

## Final report

```bash
{
  echo "# Self-test report — $RUN_ID"
  echo
  echo "**Plugin:** vast-claude-plugin"
  echo "**Branch:** $(git -C "$PLUGIN" rev-parse --abbrev-ref HEAD) ($(git -C "$PLUGIN" rev-parse --short HEAD))"
  echo "**Started:** $(date -u +%Y-%m-%dT%H:%M:%SZ)"
  END_BAL=$(vastai show user --raw | jq -r .credit)
  echo "**Balance:** \$$START_BAL → \$$END_BAL (spent \$$(echo "$START_BAL - $END_BAL" | bc))"
  echo
  cat "$OUT/results.md"
} > "$OUT/REPORT.md"

# Always cleanup
"$OUT/cleanup.sh"

# Confirm no leak
vastai show instances-v1 --raw --limit 200 \
  | jq "[.instances[]? | select(.label | tostring | startswith(\"selftest-$RUN_ID-\"))] | length" \
  | grep -q '^0$' && echo "CLEAN" || echo "LEAK — manual cleanup needed"
```

---

## Pass/fail matrix

| Phase | Tests | Automated? | Failure means |
|---|---:|:---:|---|
| 1 Knowledge | 5 | manual (chat) | Skill content didn't surface — content fix in SKILL.md |
| 2 Command-gen | 6 | manual (chat) | Agent dropped a required flag or used `-n` |
| 3 E2E | 6 | partly (shell verify) | Real launch broken or critical flags missing |
| 4 Host routing | 3 | manual (chat) | Wrong skill loaded or 401 handling regressed |
| 5 Mechanics | 9 | yes | Layout broken or plugin won't load |

**Total: 29 checks.** ≥26/29 (≥90%) to publish.

---

## Manual followup (always)

```bash
vastai show instances-v1 --raw --limit 200 \
  | jq -r '.instances[]? | select(.label | tostring | startswith("selftest-")) | "\(.id)  \(.label)  \(.actual_status)"'
```

Any output = leak. `vastai destroy instance <id> -y --raw` to clean up.
