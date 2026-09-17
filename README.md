# Flytedesk Claude Plugins

Claude Code plugins carrying Flytedesk-specific engineering knowledge — SSP/ad-ops workflows, domain conventions, and anything else that's faster to teach an agent once than to re-explain in every session.

## Plugins

| Plugin | Purpose |
|---|---|
| `ad-ops` | SSP ad unit placement diagnosis and configuration (masthead, sticky bottom, interstitial, in-content/ICV). |

## Install

```bash
claude plugin marketplace add github:Flytedesk/claude-plugins
claude plugin install ad-ops@flytedesk-plugins
```

Or declare in a project's `.claude/settings.json`:

```json
{
  "plugins": ["ad-ops@flytedesk-plugins"]
}
```

## Editing a plugin — bump the version

Every edit to `<plugin>/skills/**/SKILL.md` or any other plugin asset must bump `<plugin>/.claude-plugin/plugin.json`'s `version` in the same commit. Plugin caches are keyed by version directory (`~/.claude/plugins/cache/flytedesk-plugins/<plugin>/<version>/`) — if the version doesn't move, installed copies keep reading the stale cached content even after the push lands. Patch-bump for wording/content fixes, minor for new skills, major for breaking structure changes.

## Design notes

- One `SKILL.md` per skill, YAML frontmatter (`name`, `description`) + a plain markdown body. See [`ad-ops/skills/ad-placement/SKILL.md`](ad-ops/skills/ad-placement/SKILL.md) for the format — same shape as [Flytedesk/flytedesk-skills](https://github.com/Flytedesk/flytedesk-skills).
- Unlike `flytedesk-skills` (a plain skills folder meant to be copied/submoduled into a project), this repo is a proper **Claude Code plugin marketplace** (`.claude-plugin/marketplace.json` at the root, one subfolder per plugin) — install it once per machine/account rather than copying files into each project.
