---
description: Launch a Vast.ai instance from an offer ID with sane defaults (PyTorch image, --ssh --direct)
argument-hint: <offer-id> [image] [--disk N] [--label STR]
allowed-tools: [Bash]
---

# /vastai:launch

Launch a Vast.ai GPU instance from an offer ID. Defaults to PyTorch and the `--ssh --direct` connection mode — what most renters want.

## Steps

1. **Parse `$ARGUMENTS`** as: `<offer-id> [image] [--disk N] [--label STR]`. The offer ID is required; everything else has a default.

2. **Ensure an SSH key is registered BEFORE launching.** This is a critical regression: launching without a registered key produces an unreachable instance.
   ```
   vastai show ssh-keys --raw
   ```
   If the response is `[]`, register the user's default key first:
   ```
   vastai create ssh-key --ssh-key "$(cat ~/.ssh/id_ed25519.pub)" --raw
   ```
   (Fall back to `~/.ssh/id_rsa.pub` if no ed25519 key exists; instruct the user to generate one if neither file exists.)

3. **Build the create command** with these defaults:
   - `--image` → user-provided, else `pytorch/pytorch:@vastai-automatic-tag`
   - `--disk` → user-provided, else `20`
   - `--label` → user-provided, else `claude-launch-$(date +%s)` (so the instance is greppable later)
   - Always include `--ssh --direct --raw`

4. **Run it:**
   ```
   vastai create instance <OFFER_ID> --image <IMAGE> --disk <DISK> --label <LABEL> --ssh --direct --raw
   ```

5. **Parse the response.** Success returns `{"success": true, "new_contract": <INSTANCE_ID>}`. Report the new instance ID and remind the user to poll `vastai show instance <INSTANCE_ID> --raw` for `actual_status: running` (or run `/vastai:status <INSTANCE_ID>`).

6. **Error paths:**
   - `Insufficient credits` → surface the billing link; don't retry.
   - Offer no longer rentable → tell the user to re-run `/vastai:search` and pick another.

## Examples

| Prompt | What runs |
|---|---|
| `/vastai:launch 26349244` | `vastai create instance 26349244 --image pytorch/pytorch:@vastai-automatic-tag --disk 20 --label claude-launch-... --ssh --direct --raw` |
| `/vastai:launch 26349244 huggingface/transformers-pytorch-gpu` | swap image, keep other defaults |
| `/vastai:launch 26349244 --disk 50` | bigger disk |
| `/vastai:launch 26349244 pytorch/pytorch --label training-1 --disk 100` | full custom |

## Critical regressions (don't drop)

- `--ssh --direct` is required for a usable SSH connection.
- `--raw` on the create call (so `new_contract` can be parsed).
- SSH key registered *before* launch (step 2).
- Default image must fall back to `pytorch/pytorch:@vastai-automatic-tag` — never ask the user for an image if they didn't specify one.
- Don't auto-destroy the new instance from this command, even if it fails to launch — that's `/vastai:status` + an explicit destroy step territory.

## Notes

- Full flag reference + image catalog: `skills/vastai/SKILL.md` § "Instances".
