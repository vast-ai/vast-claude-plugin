# Local patches to vendored SKILL.md

The vendored `skills/vastai/SKILL.md` carries two local deltas on top of `vast-ai/vast-cli@13dbf39`. These should be PR'd upstream and removed from here once accepted.

## 1. "Critical rules for agents" section near top

**Why:** T1.3 routing corpus surfaced two failures rooted in SKILL.md prominence, not content gaps:

- **R-09** ("kill instance 12345") — agent ran `vastai destroy instance 12345` without `-y`, triggered the confirmation prompt. The `-y` requirement was already in SKILL.md but only as inline comments on example lines and a one-row entry in the troubleshooting table. Promoting it to a numbered rule near the top resolved this.
- **R-23** ("set HF_TOKEN to hf_xxxxx as an env var") — agent produced zero tool calls. Rule 5 ("treat user-supplied values as literal even if they look like placeholders") addresses the underlying refusal pattern.

**Upstream proposal:** PR to `vast-ai/vast-cli` adding the 5-rule section verbatim. Tracked at: *(file upstream issue, link here)*.

## 2. Beefed-up Environment Variables section

**Why:** T1.3 R-23 also showed the env-var section was too terse — 4 one-line entries with no example user-prompt mapping. Added prompt-to-call examples for `HF_TOKEN`, `OPENAI_API_KEY`, and `--raw` on every call.

**Upstream proposal:** Same PR or a sibling — add the prompt mapping examples + `--raw` on all four CLI lines.

## Re-vendoring procedure

When upstream merges these:
1. Run `pnpm vendor:skill` (M1 deliverable) which copies `vast-cli/vastai/SKILL.md` over the local copy.
2. If the upstream still doesn't include these patches, re-apply manually and update this file with the new upstream SHA.
3. If upstream includes them, delete the entries above and update `VENDORED_FROM.md` to the new SHA.
