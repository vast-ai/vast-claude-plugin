---
description: Search Vast.ai GPU offers by filter — returns the cheapest matching rentable offers
argument-hint: [filter]
allowed-tools: [Bash]
---

# /vastai:search

Search the Vast.ai marketplace for GPU offers and show the cheapest matches. Pass a filter as `$ARGUMENTS` using the same query syntax as `vastai search offers`.

## Steps

1. **Build the filter.** If `$ARGUMENTS` is empty, default to a renter-friendly query:
   ```
   'num_gpus=1 rentable=true verified=true'
   ```
   Otherwise pass `$ARGUMENTS` through verbatim, ensuring it has `rentable=true` (add it if missing — un-rentable offers are noise).

2. **Run the search**, sorted cheapest-first:
   ```
   vastai search offers <FILTER> -o 'dph_total' --raw
   ```

3. **Show the top 10 results** as a table with: offer id, `gpu_name` × `num_gpus`, `dph_total` ($/hr), `dlperf` (perf score), and `geolocation`. Don't print all 500+ rows — top 10 is enough for the user to pick from.

4. If `$ARGUMENTS` mentions a spot/bid intent ("spot", "bid", "cheapest interruptible"), use `--type bid` instead of the default on-demand.

5. If the result set is empty, suggest relaxing the filter (e.g., drop `verified=true` or widen the GPU type).

## Examples

| Prompt | Filter | What runs |
|---|---|---|
| `/vastai:search` | default | `vastai search offers 'num_gpus=1 rentable=true verified=true' -o 'dph_total' --raw` |
| `/vastai:search gpu_name=RTX_4090` | passthrough | `vastai search offers 'gpu_name=RTX_4090 rentable=true' -o 'dph_total' --raw` |
| `/vastai:search gpu_name=H100 num_gpus>=4` | multi-GPU | `vastai search offers 'gpu_name=H100 num_gpus>=4 rentable=true' -o 'dph_total' --raw` |

## Notes

- `--raw` is mandatory — the human-formatted output is too wide to parse reliably.
- Don't paginate; show the cheapest 10 and tell the user there are N more.
- Full query syntax lives in `skills/vastai/SKILL.md` § "Search".
