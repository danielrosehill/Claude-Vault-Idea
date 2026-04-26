# Claude Vault — idea capture

A proposal for a meta-plugin that solves the "plugins don't lazy-load" problem at the user level by holding a personal vault of plugins and MCP servers that get **selectively activated per project**, rather than enabled globally.

Status: idea capture, not yet built.

## The problem

We've seen the same pain point with MCP servers and plugins, in that order.

**MCP servers, originally**: every tool definition for every connected server was loaded into the orchestrator's context. User-level MCP servers were notorious for this — connect a few, and your context budget was eaten before any work began. The community workaround was to define MCPs on a per-project basis and accept the friction of editing config every time.

That has since been mitigated by deferred tool loading (`ToolSearch`-style on-demand schema fetch). MCP servers now lazy-load. The pain has receded.

**Plugins, today**: plugins have the same pain point that MCPs used to have. Skill and agent `description` fields are eagerly loaded into the orchestrator at the start of every conversation, regardless of whether the plugin is invoked. A heavy plugin install (50+ user-level plugins) easily costs 30,000–40,000 tokens of upfront context per session. See companion repo [`Claude-Plugins-Skills-Context-Notes`](https://github.com/danielrosehill/Claude-Plugins-Skills-Context-Notes).

The intuitive value of plugins is that you can ship a useful skill once and have it persistently available across all your sessions at user level. That's exactly what plugins were designed for. The tension is that "persistently available" currently means "persistently loaded into context", and that doesn't scale.

The current built-in workaround is **per-project enablement** — list the plugins you want enabled in `.claude/settings.json` per repo. This works, but it's manual, cumbersome, and easy to forget. Every new project starts cold; you have to remember which plugins matter and edit settings by hand.

## The proposed selective-availability mechanism is not (yet) as developed as MCP's

Plugins currently have an `enabled` flag in settings, and that's about it. There's no equivalent of the MCP deferred-load pattern for plugin descriptions. There's no skill that knows "for this kind of project, I need plugins X, Y, Z" and turns them on automatically.

So the gap is: we want **selective availability without manual per-project config**.

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
