# Claude Vault — Implementation Plan

Concrete plan for turning the [idea capture](../README.md) into a working plugin. Anchored to the existing `New-Repo-From-Template-Plugin` workflow so the vault slots in as the activation layer rather than a parallel mechanism.

## Architecture: three layers

The vault decomposes into three layers, two of which already exist informally.

### 1. Vault store (already half-built)

A local marketplace (or set of marketplaces) holding every plugin and MCP server that's installed-but-dormant. Today this exists in spirit at `~/repos/github/my-repos/` — `claude-code__*`, `claude-templates__*`, `op-vault-plugin`, etc. — but it isn't registered as a Claude Code marketplace.

Formalisation: add a `.claude-plugin/marketplace.json` so `enabledPlugins` can flip entries on by name. Nothing in the vault is enabled at user level by default.

### 2. Manifest layer (the bridge)

Every template repo declares what tools it needs via a `.claude/requirements.json` file:

```json
{
  "plugins": ["op-vault", "Claude-Transcription"],
  "mcp_servers": ["greeninvoice", "gws-personal"],
  "bundles": []
}
```

This is the integration point with `New-Repo-From-Template-Plugin`. Templates know what they want; the vault knows how to provide it. The two plugins stay decoupled — each does one job.

Alternative: extend the existing `repository-templates.json` index entries with a `requirements` field. Either works; per-repo files are more discoverable, central index is easier to audit.

### 3. Activation layer (the new thing)

A meta-plugin enabled globally — the only one. Its skills edit the *project's* `.claude/settings.json` to flip `enabledPlugins` on and add `mcpServers` entries. It doesn't install anything; it activates from what's already in the vault.

## Plugin layout

```
Claude-Vault/
├── .claude-plugin/plugin.json
├── skills/
│   ├── activate/SKILL.md                        # activate <name> in CWD
│   ├── activate-bundle/SKILL.md                 # apply named bundle
│   ├── apply-template-requirements/SKILL.md     # read .claude/requirements.json, activate all
│   ├── list-vault/SKILL.md                      # show installed-but-dormant
│   ├── list-active/SKILL.md                     # show enabled in current project
│   └── update-vault/SKILL.md                    # git pull across vault entries
└── data/
    └── bundles/                                  # podcast.json, invoicing.json, etc.
```

Config lives at `~/.claude/vault/config.json` and lists vault marketplace paths and bundle locations.

## Integration with `New-Repo-From-Template-Plugin`

Add one step to that plugin's SKILL.md, after the post-creation step:

> **Step 6.** If `<DEST_PATH>/.claude/requirements.json` exists, invoke the vault's `apply-template-requirements` skill to activate the listed plugins/MCPs in the new repo.

That's the entire handoff. Templates scaffold structure and declare requirements; the vault fulfils them. No new mechanism in the template plugin — one extra step that's a no-op when the file is absent.

## Build order — smallest viable slice first

The trap to avoid: don't build a registry abstraction before activating anything. Start by editing `settings.json` directly from a skill, see how it feels, then generalise.

1. **`activate <plugin-name>` skill** — edits `.claude/settings.json` `enabledPlugins` only. ~50 lines.
2. **`apply-template-requirements` skill** — reads JSON, calls `activate` per entry.
3. **Proof of integration** — add `requirements.json` to one or two existing templates and run end-to-end.
4. **MCP activation** — extend `activate` to handle `mcpServers` entries.
5. **Bundles** — once two or three real bundles emerge from real projects, formalise the bundle file format.
6. **`update-vault`** — git pull across vault entries. Last, because it's only valuable once the vault has real population.

`list-vault` and `list-active` can ride along whenever convenient — they're trivial.

## Repo split

- **`Claude-Vault-Dev`** (private) — scratch repo for prototyping. Break things, throw out drafts, no commitment to backwards compatibility. This is where steps 1–3 happen.
- **`Claude-Vault`** (private initially, public once stable) — the production plugin. Code only lands here once it's proven in `Claude-Vault-Dev` and the surface is stable.
- **`Claude-Vault-Idea`** (this repo, public) — historical record of the idea and the underlying problem. Doesn't move; remains a reference.

## Open questions deferred to build phase

- Does `activate` write to `.claude/settings.json` or `.claude/settings.local.json`? Probably `settings.json` so it's checked in with the project; the requirements file already implies team-shared activation.
- How does the skill detect the vault location at runtime? Read `~/.claude/vault/config.json`; fall back to `~/repos/github/my-repos/` for Daniel's setup.
- Marketplace plugins vs direct git clones — handle both, with marketplace path being the cleaner case.
- Does the plugin ever touch user-level settings? No, by design. Project-level only, except the one global flag enabling the meta-plugin itself.

## When this plan becomes obsolete

If Anthropic ships native lazy loading for plugin skill descriptions ([#44536](https://github.com/anthropics/claude-code/issues/44536), [#19445](https://github.com/anthropics/claude-code/issues/19445)) and fixes user-level skill body loading ([#16616](https://github.com/anthropics/claude-code/issues/16616)), the activation layer's value collapses to templating and discovery. The manifest layer (`requirements.json`) survives as a useful template-tooling convention regardless.
