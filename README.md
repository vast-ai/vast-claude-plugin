# vast-claude-plugin

A [Claude Code](https://claude.com/claude-code) plugin for [Vast.ai](https://vast.ai). Rent, launch, monitor, and tear down GPU instances — plus volumes, serverless endpoints, and billing — without leaving your editor.

The plugin combines two layers:

- **Five slash commands** wrap the highest-frequency renter operations with safe defaults.
- **Two bundled skills** give Claude full command-reference knowledge of the `vastai` CLI, so anything the slash commands don't cover works through natural language:
  - **`vastai`** — renter operations: search and launch instances, SSH, copy, logs, exec, destroy, volumes, serverless, env vars, billing.
  - **`vastai-host`** — GPU provider operations: list/unlist machines, pricing, maintenance windows, self-tests, earnings, marketplace metrics. Auto-loads on host-intent prompts.

## Slash commands

| Command | What it does |
|---|---|
| `/vastai:setup [api-key]` | Stores your API key, registers your SSH public key with Vast, and verifies the credential against `vastai show user`. Run this once before anything else — launching an instance before an SSH key is registered produces an unreachable host. |
| `/vastai:status [instance-id]` | Snapshot of a single instance or all of yours. Flags terminal states (`exited`, `unknown`, `offline`) so polling loops actually terminate. |
| `/vastai:cost` | Account balance, current burn rate ($/hr across active instances), and projected 24-hour spend. Useful as a pre-flight check before launching anything large. |
| `/vastai:search [filter]` | Finds the cheapest **rentable** offers matching a filter, sorted by total $/hr. Recognises `"spot"` / `"bid"` in the filter and switches to interruptible bid-mode pricing automatically. |
| `/vastai:launch <offer-id> [image] [--disk N] [--label STR]` | Launches an offer with sensible defaults: `pytorch/pytorch:@vastai-automatic-tag`, 20 GB disk, direct SSH, JSON output. Parses the returned `new_contract` ID so subsequent commands can reference it. |

Every `vastai` invocation includes `--raw` so responses come back as parseable JSON.

## Natural language (the skills)

Anything outside the slash-command set works conversationally. The right skill auto-loads based on intent.

**Renter prompts** load `vastai`:

> *"Find a verified 4090 under $0.40/hr and launch it with pytorch."*
>
> *"SSH into instance 12345 and tail `/var/log/nvidia-installer.log`."*
>
> *"Set `HF_TOKEN` to `hf_xxxxx` on my running instance."*
>
> *"Destroy everything except instance 12345."*

**Host prompts** load `vastai-host`:

> *"List my machine 98765 at $0.30/GPU/hr."*
>
> *"Schedule maintenance on machine 98765 next Tuesday at 02:00 UTC for 4 hours."*
>
> *"Show my host earnings for last month."*
>
> *"What's the going rate for RTX 4090s in the US right now?"*

To force a skill to load explicitly, mention it by name (*"using the vastai skill, …"* or *"using the vastai-host skill, …"*).

## Install

### Prerequisites

```bash
pip install vastai
```

### From the Claude Plugins marketplace (recommended)

```
/plugin install vastai@claude-plugins-official
```

### From source

```bash
git clone https://github.com/vast-ai/vast-claude-plugin.git
claude --plugin-dir ./vast-claude-plugin
```

## First run

```
/vastai:setup
```

You'll be prompted for an API key from <https://console.vast.ai/manage-keys/>. The command then registers your SSH public key (`~/.ssh/id_ed25519.pub` by default, falling back to `id_rsa.pub`) and verifies the credential by querying your user record.

## Layout

```
vast-claude-plugin/
├── .claude-plugin/plugin.json    # marketplace manifest
├── commands/
│   ├── setup.md                  # /vastai:setup
│   ├── status.md                 # /vastai:status
│   ├── cost.md                   # /vastai:cost
│   ├── search.md                 # /vastai:search
│   └── launch.md                 # /vastai:launch
├── skills/
│   ├── vastai/SKILL.md           # renter skill
│   └── vastai-host/SKILL.md      # GPU provider / host skill
└── TESTING.md                    # acceptance test plan
```

## License

MIT — see [LICENSE](./LICENSE).
