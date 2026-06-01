---
description: Launch a Vast.ai instance from an offer ID with sane defaults (PyTorch image, --ssh --direct --cancel-unavail)
argument-hint: <offer-id> [image] [--disk N] [--label STR] [--bid PRICE]
allowed-tools: [Bash]
---

# /vastai:launch

Launch a Vast.ai GPU instance from an offer ID. Defaults to the Vast-curated PyTorch image and the `--ssh --direct --cancel-unavail` connection mode — what most renters want.

## Steps

1. **Parse `$ARGUMENTS`** as: `<offer-id> [image] [--disk N] [--label STR] [--bid PRICE]`. The offer ID is required; everything else has a default.

2. **Ensure an SSH key is registered BEFORE launching.** Launching without a registered key produces an unreachable instance.
   ```
   vastai show ssh-keys --raw
   ```
   If the response is `[]`, register the user's default key first (public key is POSITIONAL — there is no `--ssh-key` flag):
   ```
   vastai create ssh-key "$(cat ~/.ssh/id_ed25519.pub)" --raw
   ```
   Fall back to `~/.ssh/id_rsa.pub` if no ed25519 key exists; instruct the user to generate one if neither file exists.

3. **Honor pricing-mode intent from the arguments.** `vastai search offers` does not support filtering by `id=`, so there is no clean way to "look up the offer" before launching — accept the user's stated intent:
   - If `$ARGUMENTS` includes `--bid PRICE`, the user wants interruptible pricing — pass `--bid_price PRICE` to `create instance` (step 5).
   - Otherwise, launch on-demand. Every offer can be rented either way; the on-demand price is the offer's `dph_base` and the bid floor is its `min_bid`. The user picks the mode at launch.

4. **Build the create command** with these defaults:
   - `--image` → user-provided, else `vastai/pytorch:@vastai-automatic-tag` (Vast-curated image; `@vastai-automatic-tag` only resolves on `vastai/*` images, NOT `pytorch/pytorch` or other third-party images)
   - `--disk` → user-provided, else `20`
   - `--label` → user-provided, else `claude-launch-$(date +%s)` (so the instance is greppable later)
   - **Always include `--ssh --direct --cancel-unavail --raw`.** `--cancel-unavail` fails the launch if the machine becomes unavailable mid-call instead of silently producing a stopped instance that accrues disk charges while you poll forever.
   - **If a bid price was supplied**, also add `--bid_price <PRICE>`. (No need to pass `--type bid` on `create instance` — `--bid_price` selects bid mode.)

5. **Run it (on-demand example):**
   ```
   vastai create instance <OFFER_ID> --image <IMAGE> --disk <DISK> --label <LABEL> \
     --ssh --direct --cancel-unavail --raw
   ```
   **Bid (interruptible) example:**
   ```
   vastai create instance <OFFER_ID> --image <IMAGE> --disk <DISK> --label <LABEL> \
     --ssh --direct --cancel-unavail --bid_price 0.20 --raw
   ```

6. **Parse and verify the response.** Success returns `{"success": true, "new_contract": <INSTANCE_ID>}`. Do NOT report success purely on the API response — the `create instance` call sometimes returns success for instances that fail to materialize. Confirm by querying:
   ```
   vastai show instance <INSTANCE_ID> --raw
   ```
   - If the response is `{}` or `error: no instance found with id <INSTANCE_ID>` → materialization failed. Tell the user, do NOT try to destroy the phantom (the API won't find it either). Recommend re-running `/vastai:search` on a different offer.
   - If found, report `actual_status`, `intended_status`, `cur_state`, `next_state`. Remind the user to poll for `actual_status: running` (or run `/vastai:status <INSTANCE_ID>`).

7. **Error paths:**
   - `Insufficient credits` → surface <https://cloud.vast.ai/billing/>; don't retry.
   - `error 404/3603: no_such_ask Instance type by id ... is not available` → offer was just taken or expired. Re-run `/vastai:search` and pick another.
   - `401` + `"Two Factor Authentication"` in body → the account has 2FA but no active TFA session; recommend `!vastai tfa login --method-type totp --code <CODE>` (with `!` prefix to keep the code out of the transcript), then retry.
   - Other 401 → see `/vastai:setup` step 4 triage.

## Examples

| Prompt | What runs |
|---|---|
| `/vastai:launch 26349244` | `vastai create instance 26349244 --image vastai/pytorch:@vastai-automatic-tag --disk 20 --label claude-launch-... --ssh --direct --cancel-unavail --raw` |
| `/vastai:launch 26349244 vastai/vllm:@vastai-automatic-tag` | swap image (must be a `vastai/*` curated image for `@vastai-automatic-tag` to resolve) |
| `/vastai:launch 26349244 --disk 50` | bigger disk |
| `/vastai:launch 26349244 --bid 0.20` | bid (interruptible) at $0.20/hr — adds `--bid_price 0.20` |
| `/vastai:launch 26349244 pytorch/pytorch:2.4.0-cuda12.4-cudnn9-runtime --disk 100` | third-party image — MUST use a real tag, not `@vastai-automatic-tag` |

## Critical regressions (don't drop)

- `--ssh --direct --cancel-unavail` are all required for a usable launch (skill rule 8).
- `--raw` on the create call (so `new_contract` can be parsed).
- SSH key registered *before* launch (step 2).
- Default image must be `vastai/pytorch:@vastai-automatic-tag`. NEVER fall back to `pytorch/pytorch:@vastai-automatic-tag` — `@vastai-automatic-tag` only resolves on Vast curated images (`vastai/pytorch`, `vastai/vllm`, `vastai/comfy`, `vastai/base-image`, `vastai/linux-desktop`).
- Verify the instance actually exists via `show instance` after launch; don't trust the success response alone.
- Don't auto-destroy the new instance from this command, even if it fails to launch — that's `/vastai:status` + an explicit destroy step territory.

## Notes

- Full flag reference + image catalog: `skills/vastai/SKILL.md` § "Instances".
- For SSH connection failures after launch, see skill Critical rule 11 — pull `vastai logs <id>` BEFORE attempting any other recovery.
