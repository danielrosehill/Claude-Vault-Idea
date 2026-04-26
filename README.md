# Claude Vault — idea capture

A proposal for a meta-plugin that solves the "plugins don't lazy-load" problem at the user level by holding a personal vault of plugins and MCP servers that get **selectively activated per project**, rather than enabled globally.

Status: idea capture, not yet built.

## The problem (as actually understood)

The initial framing of this idea assumed plugins broadly suffer the eager-loading problem MCPs used to have. Research into the current Claude Code docs ([see `research/plugin-loading-research.md`](./research/plugin-loading-research.md)) sharpens the picture: the problem is more localised but no less real.

What's actually going on at user level today:

- **Plugin skills only half-lazy-load.** When a plugin is enabled, the full skill body is deferred — that part works. But skill **descriptions (frontmatter)** are eagerly injected into the system prompt at every session start, for every skill in every enabled plugin. With 50 plugins averaging 5 skills each at ~80 tokens of frontmatter, that's **~20,000 tokens of skill descriptions in every session**, regardless of relevance.
- **There is no discovery mechanism for skills.** This is the load-bearing point. MCP solved its bloat problem with `ToolSearch`: tool names live in a lightweight index and the orchestrator calls a search tool to fetch definitions on demand. Plugin skills have no equivalent. The orchestrator picks skills based on descriptions it already has in context — so all descriptions must be frontloaded, by design. There's no "ask for a skill and have the orchestrator find it" path.
- **User-level skills are worse still.** Skills in `~/.claude/skills/` (not inside a plugin) eager-load their **full bodies**, not just frontmatter — bug [#16616](https://github.com/anthropics/claude-code/issues/16616). Single skills measured at 5,000+ tokens that should have cost ~50. Three problems stacked: full body loaded, description loaded, no discovery.
- **MCP servers**: solved. `ToolSearch` defers tool definitions and provides on-demand discovery (~95% context reduction).
- **Plugin enablement is binary and global.** A plugin enabled at user level is enabled in every session. There's no per-component granularity, no "installed but dormant", no repo-scoped plugins. The only way to keep a plugin's descriptions out of a session's context is to not have it enabled at user level — i.e., enable per project.
- **Open Anthropic issues** [#44536](https://github.com/anthropics/claude-code/issues/44536) and [#19445](https://github.com/anthropics/claude-code/issues/19445) ask for the `ToolSearch` pattern to extend to skills and agents — acknowledged, not roadmapped.

So the problem to mitigate is three things, in order of severity:

1. **User-level skills loading their bodies** (#16616) — fixable today by converting them into local plugins.
2. **Every enabled plugin's skill descriptions loading eagerly, every session, with no discovery mechanism** — this is the structural problem, and it scales linearly with how many plugins you have enabled at user level.
3. **No clean per-project activation flow**, even though the binary `enabledPlugins` flag exists.

The first is a bug. The second is the real architectural gap and the reason plugins don't scale the way MCP now does. The third is what the vault would fix.

## A context-engineering view: where should load live at user level?

Treat context cost as a ladder. Each tier is roughly an order of magnitude cheaper than the one above it:

| Tier | Cost | What belongs here |
|------|------|-------------------|
| **Always-loaded** — system prompt, CLAUDE.md body, eager-loaded user skills | Highest; paid every session | Identity, ironclad rules, memory index pointer |
| **Eager metadata** — plugin skill frontmatter, agent descriptions, command names | Moderate; paid every session but small per item | Tools genuinely useful in >50% of sessions |
| **Deferred / lazy** — MCP via ToolSearch, plugin skill bodies | Low; paid only when invoked | Anything specialised — i.e. most things |
| **Per-project activation** — `enabledPlugins` per repo | Zero global cost | Domain-specific tooling (invoicing MCP, podcast skills, etc.) |
| **Not installed** | Zero | Self-explanatory |

Principle: the rarer the use, the further down the ladder it should sit. Anything in tier 1 or 2 should justify itself across nearly every session. Two misalignments today put things in those tiers that shouldn't be there:

- User-level skills land in tier 1 *by accident* (the #16616 bug) instead of tier 2.
- Every enabled plugin's skill descriptions land in tier 2 *by design* (no discovery mechanism) instead of tier 3, where they belong. This is why even a well-bundled plugin library scales badly at user level: each plugin you enable globally is paying tier-2 rent on every skill it ships.

For my own setup: CLAUDE.md is already disciplined — split into `~/.claude/context/*.md` files referenced on demand. The unbounded growth happens in `~/.claude/skills/`, `~/.claude/agents/`, and `~/.claude/commands/`.

## Workarounds available today (no vault required)

Practical mitigations that need no new tooling:

1. **Convert user-level skills into local plugin format.** Add a `.claude-plugin/plugin.json` manifest and the skill loads frontmatter-only, restoring expected behaviour. Single biggest win — directly bypasses #16616. Heavy users should treat `~/.claude/skills/` as deprecated for any non-trivial skill.
2. **Audit and prune.** Periodically review `~/.claude/skills/`, `~/.claude/agents/`, `~/.claude/commands/` and delete what hasn't been invoked in months.
3. **Prefer hooks for automations.** Hooks live in `settings.json` and don't sit in context — they fire on harness events. "Always do X when Y happens" is a hook, not a skill.
4. **Use sub-agents for heavy specialised work.** A sub-agent's system prompt doesn't bloat the parent context; only its summary returns.
5. **Per-project `enabledPlugins`.** Disable noisy plugins at user level, re-enable them only in repos where they're relevant. Manual but already supported.
6. **Keep CLAUDE.md as an index, not a manual.** Reference `context/*.md` on demand rather than inlining.

## Where the vault still adds value

After applying the above, the residual gaps are:

- **Manual orchestration.** Per-project enablement is still paper-shuffling. The vault's contribution isn't a new mechanism — it's *systematising* workarounds 1 and 5: install everything to a known path, enable nothing globally, activate per project through a single skill.
- **Templating.** Once activation is itself a skill, it composes into project templates ("scaffolding a podcast project? activate these three plugins and this MCP"). That's hard by hand.
- **Discovery.** `list-vault` / `list-active` beats remembering what you have.

If Anthropic ships native lazy loading for user-level skills (#16616) and per-component enablement (#44536, #19445), the vault's value collapses to templating and discovery only. Until then, it's a real pattern, but it should be sold as a *systematiser of known workarounds*, not as a fundamental fix.

## The idea: a Claude Vault meta-plugin

A user-level plugin whose only job is to manage a personal vault of plugins and MCP servers, and to selectively activate the right ones for the project you're working on.

### Core concept

The vault is a local clone of:
- All the plugins you might possibly want
- All the MCP servers you might possibly want
- Their templates, scaffolding, marketplace metadata

These are kept in a known location, kept up to date (the vault has its own update workflow), and **none of them are enabled at user level**. Except perhaps a tiny handful of genuinely cross-cutting essentials.

The vault meta-plugin is the only thing enabled globally. Its skills are the activation interface.

### How it would work in practice

A typical session:

1. User starts a new project and runs `/claude-vault:setup-workspace` (or similar).
2. Skill asks: what kind of project is this? Or: which template do you want?
3. Skill consults a small registry — either user-curated or template-driven — that maps project types to plugins and MCP servers.
4. Skill edits the project's `.claude/settings.json` to:
   - Enable the relevant plugins from the vault
   - Add the relevant MCP server entries
5. User restarts Claude Code in that workspace and the right tools are present, with no global tax.

A second mode — direct request:

1. User says "I want my podcast plugin and the GreenInvoice MCP for this repo."
2. Skill grabs both from the vault, activates them in the project settings.
3. Done.

A third mode — template integration:

1. User scaffolds a new project from a template (existing scaffolding plugin).
2. Template specifies its desired tools.
3. Vault meta-plugin reads the spec and activates them.

### What "vault" actually means here

Two layers:

**Marketplace layer** — a local marketplace (or set of marketplaces) registered with Claude Code, containing all candidate plugins. This already exists as a primitive; users with private marketplaces effectively have the start of a vault. The vault is just the deliberate decision to *have* one and to keep nothing enabled at user level by default.

**Activation layer** — the meta-plugin's skills. This is the new thing. It edits project-level `.claude/settings.json` to flip plugins on, and edits the MCP server config to mount MCP servers. It doesn't install anything new — it just activates from what's already in the vault.

### Why this is "have your cake and eat it"

The whole point of plugins was that you could install useful tools once and have them available. Per-project manual enablement breaks that — every new project is cold and you're back to the bad old days of remembering which tools you have.

The vault mechanism preserves the original intuition (install once, persistently *available*) while changing the meaning of "available" from "loaded into every session's context" to "ready to be activated in one command for the right project." You get:

- One canonical place for everything you've installed.
- Zero global context tax (except for the meta-plugin itself, which would be tiny).
- One-command activation per project.
- Updates centralised — the vault stays current, projects don't drift.
- A natural surface for templates to declare their tool requirements.

### Why this is explicitly a workaround

To be clear: this is a workaround for a current Claude Code limitation, not a permanent design proposal. The right long-term fix is for plugins themselves to lazy-load descriptions the way MCP tools now do — `description` summaries kept terse and full descriptions deferred until the orchestrator decides a skill is potentially relevant. If/when that lands, the vault meta-plugin becomes redundant.

But until then, the vault pattern lets heavy users keep installing useful plugins without paying the per-session tax for plugins they aren't currently using.

## Sketch of skills the vault plugin would ship

- `setup-workspace` — bootstrap a new project with a chosen template + tools
- `activate-plugin <name>` — turn on a vault plugin in the current project
- `activate-mcp <name>` — mount a vault MCP server in the current project
- `activate-bundle <name>` — apply a named bundle (e.g. "podcast-production")
- `list-vault` — show what's in the vault
- `list-active` — show what's currently active in this project
- `update-vault` — pull updates across all vault plugins and MCP servers
- `bundle-create` — save the current project's tool set as a reusable bundle

## Open questions

- How does the meta-plugin discover the vault location? A config file in `~/.claude/` listing vault paths feels right.
- How does it handle marketplace plugins vs. direct git clones? Probably both, with the marketplace path being the cleaner case.
- Does it touch user-level settings ever? Probably not, by design. Project-level only, except for one global flag enabling the meta-plugin itself.
- How does it interact with templates from existing scaffolding plugins? Templates declare requirements; vault fulfils them.
- Is this better as one plugin with skills, or multiple plugins (vault-plugins, vault-mcps, vault-templates)? Likely starts as one and grows.

## Why log this here

Captured as a public repo because the underlying problem (eager plugin description loading) is general, the workaround is general, and others may want to reproduce the pattern before any official fix lands.

If/when the harness adds lazy plugin loading natively, this repo becomes a historical record. Until then, it's a design sketch for something worth building.
