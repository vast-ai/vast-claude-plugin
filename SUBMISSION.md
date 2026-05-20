# Distribute to Claude Code users

Claude Code's plugin system supports three install paths: the **Anthropic-managed Official Directory**, **community directories**, and **direct GitHub marketplace add** from any public repo.

## Pre-flight checklist

- [ ] `.claude-plugin/plugin.json` exists with `name` (kebab-case)
- [ ] `version` (semver) — bump on every release; users only get updates when this changes
- [ ] `description`, `author`, `license`, `homepage`, `repository`, `logo` all set
- [ ] `skills/vastai/SKILL.md` with YAML frontmatter (`name`, `description`)
- [ ] Every file under `commands/` has frontmatter (`description`, `argument-hint`, `allowed-tools`)
- [ ] `LICENSE` file at repo root
- [ ] Public Git repo
- [ ] `README.md` describes usage
- [ ] Plugin name not on Anthropic's reserved-name list (e.g. `claude-plugins-official`, `anthropic-plugins`)

## Users install from this repo today

End users add this repo as a marketplace and install via Claude Code's CLI:

```bash
/plugin marketplace add vast-ai/vast-claude-plugin
/plugin install vastai
```

Works without review. Updates pull whenever `version` in `plugin.json` is bumped (or, if no `version` is set, every commit counts as a new release).

## Submit to the Anthropic Official Directory

For inclusion in the Anthropic-curated directory at [github.com/anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official):

1. Submit at **https://claude.ai/settings/plugins/submit** or **https://platform.claude.com/plugins/submit**.
2. Paste the repo URL: `https://github.com/vast-ai/vast-claude-plugin`.
3. Anthropic reviews for quality and security; expect a manual review queue.

Auto-discovery also crawls GitHub for valid manifests, but indexing can take days — direct submission is faster.

## Community directories

Optional secondary listings:

- [claudemarketplaces.com](https://claudemarketplaces.com) — submit via their form
- [claudepluginhub.com/tools/submit-plugin](https://www.claudepluginhub.com/tools/submit-plugin)

These are independent of Anthropic's official directory.

## Updating after distribution

1. Bump `version` in `.claude-plugin/plugin.json` (semver). Keep this in lockstep with any git tags and CHANGELOG entries — version mismatch is the most common marketplace-rejection reason.
2. Push to `main`.
3. Users running `/plugin update vastai` pull the new version.

## References

- [Create and distribute a plugin marketplace (official docs)](https://code.claude.com/docs/en/plugin-marketplaces)
- [Anthropic's official plugin directory](https://github.com/anthropics/claude-plugins-official)
- [Publishing a plugin to the Claude Marketplace](https://systemprompt.io/guides/publish-plugin-claude-marketplace)
