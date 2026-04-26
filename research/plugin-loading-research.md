# Plugin Discovery & Loading vs. MCP Tool Loading: A Technical Briefing

**Date:** 2026-04-26  
**Purpose:** Validate the premise of the vault meta-plugin idea by documenting eager vs. lazy loading behavior in Claude Code plugins and MCP.

---

## Executive Summary

**The premise is partially correct but more nuanced than initially stated.**

Claude Code plugins have an **asymmetry in loading behavior** compared to MCP:
- **Plugin skills (best case)**: Frontmatter-only (name + description) at session start; full content loads on-demand
- **User-level skills (problematic case)**: Full content eagerly loaded at startup, consuming tokens unnecessarily
- **MCP tools (historically)**: All tool definitions eagerly loaded until ToolSearch (lazy loading) was introduced

The vault idea is worth exploring **specifically to solve the user-skill eager-loading problem** and to provide per-project plugin activation controls that work cleanly.

---

## 1. Plugin Discovery & Loading Behavior

### What Gets Loaded at Session Start?

When a Claude Code session begins with plugins installed, the following are eagerly injected into the system prompt:

| Component | Behavior | Evidence |
|-----------|----------|----------|
| **Skill descriptions** (plugin skills) | Frontmatter only (name + description) | Documented in [Extend Claude with skills](https://code.claude.com/docs/en/skills) |
| **Skill descriptions** (user skills) | **Full content** (BUG in 2.0.76+) | [Issue #16616](https://github.com/anthropics/claude-code/issues/16616) |
| **Agent definitions** | Eagerly listed | Not explicitly documented; inferred from `/agents` availability |
| **Slash commands** | Eagerly listed | Not explicitly documented; inferred from `/help` availability |
| **Hooks** | Not context-injected; loaded from config | [Hooks reference](https://code.claude.com/docs/en/hooks) shows JSON-based config |
| **MCP servers (plugin)** | Lazy-loaded with ToolSearch | See MCP section below |
| **Output styles** | Not context-injected; selectable via command | Not documented as context component |

### Disabled Plugin Behavior

The `enabled` flag in `settings.json` is **binary**:
- If `enabled: false` → plugin components are **not loaded** into context, not invocable, not discoverable
- If `enabled: true` → **all components are loaded** (no granular per-component control)

**No per-component enablement exists** at the session level. You cannot enable a plugin's skills while disabling its agents, or vice versa.

### Progressive Disclosure in Plugin Skills

Plugin skills implement a two-tier loading system:

1. **Tier 1 (session start)**: All skill frontmatter (name + description) loads into context, typically **50-100 tokens per skill**
2. **Tier 2 (on-demand)**: Full skill content loads only when:
   - Claude determines the skill is relevant to the current task, OR
   - You invoke the skill explicitly with `/plugin-name:skill-name`

This design allows many skills to remain discoverable without bloating context until needed. The description field acts as the gating mechanism for relevance detection.

**Plugin skills DO NOT have the eager-loading problem.** The bug only affects user-level skills (see below).

### User-Level Skills: A Known Bug

A critical discrepancy was reported in [Issue #16616](https://github.com/anthropics/claude-code/issues/16616):

**User skills in `~/.claude/skills/` are fully eager-loaded at session start**, consuming thousands of tokens:

```
Example measured in Claude Code 2.0.76:
- multi-agent-patterns: 5.9k tokens (should be ~50)
- subagent-driven-development: 2.7k tokens (should be ~50)
- consulate: 1.7k tokens (should be ~50)
Total: ~10.3k tokens instead of ~150 expected
```

**Workaround reported in the issue:** Converting user skills to local plugin format (creating a `.claude-plugin/plugin.json` manifest) causes them to load correctly with frontmatter-only behavior.

**Status:** This appears to be an unresolved bug in the current version. No roadmap signals indicate when (or if) it will be fixed.

---

## 2. MCP Tool Loading: The ToolSearch Evolution

### Historical Behavior (Pre-ToolSearch)

MCP tools were **eagerly loaded** into context at session start. Each tool definition includes:
- Name
- Description (full English prose)
- Parameter schema (JSON Schema with constraints)
- Return type documentation

This consumed **100-300 tokens per tool**. A five-server setup with ~100 tools could consume **55,000 tokens before typing anything** — nearly 30% of a 200K context window.

### ToolSearch: Lazy Loading (Current Default)

Claude Code now automatically enables **ToolSearch** (lazy loading) when MCP tools would consume >10% of available context. No configuration is required.

**How it works:**

1. **Session start**: Claude sees only the ToolSearch tool itself and tool names (lightweight index)
2. **During conversation**: When Claude needs a tool, it invokes ToolSearch to fetch the 3-5 most relevant tool definitions
3. **Automatic expansion**: Matching tool definitions are loaded into context on-demand

**Performance gains:**

- Context usage reduction: **95%** (from ~55k tokens to ~2.7k for equivalent setups)
- Accuracy improvement: Opus 4.5 accuracy on tool-use tasks jumped from 79.5% to 88.1%

**Evidence:** [MCP Tool Search: Save 95% Context](https://claudefa.st/blog/tools/mcp-extensions/mcp-tool-search), [Tool search tool - Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)

### Behavior Difference Summary

| Aspect | MCP (ToolSearch enabled) | Plugin Skills |
|--------|------------------------|----------------|
| Initial load | Tool names + lightweight index | Skill frontmatter (name + description) |
| Full definitions | Loaded on-demand via ToolSearch | Loaded on-demand when skill invoked or deemed relevant |
| Transparency | User invisible; automatic | User invisible; automatic for plugins, broken for user skills |
| Can be disabled | Yes, via settings, but not recommended | Not applicable (plugins always use lazy loading) |

---

## 3. The Asymmetry: Do Plugins Have the Eager-Loading Problem?

**Short answer: Not for plugin skills, yes for user-level skills.**

**Plugin skills:** ✅ Already implement deferred loading (frontmatter-only at session start)

**User-level skills:** ❌ Currently suffer from eager loading, consuming thousands of tokens unnecessarily (Bug #16616)

**Plugin agents:** ⚠️ **Undocumented.** No official statement on whether agent definitions are eagerly listed or deferred. Inferred: agent names are always listed (required for `/agents` discovery), but system prompts may not be loaded until used.

**Plugin MCP servers:** ✅ Use ToolSearch lazy loading when enabled (applies equally to all MCP sources)

**Plugin hooks:** No context impact (loaded from JSON config, not injected into context)

---

## 4. Per-Project Plugin Enablement

### How `.claude/settings.json` Plugin Control Works

The `enabledPlugins` object in `~/.claude/settings.json` or `.claude/settings.json` (project-level) controls plugin availability:

```json
{
  "enabledPlugins": {
    "my-plugin@marketplace": true,   // Installed and active
    "other-plugin@marketplace": false // Installed but inactive
  }
}
```

### Per-Project Overrides

**Does it work cleanly?** Partially, with caveats:

1. **User-level disables project-level enables**: If you disable a plugin at user level (`~/.claude/settings.json`), disabling it again at project level is redundant but harmless.

2. **Project-level cannot override user-level enabled**: If a plugin is forced-enabled at user/org level (managed settings), project-level `false` cannot disable it. This is by design for security.

3. **Each level overrides the previous**: Plugin marketplace and installed status are global. Enabled/disabled state can differ per project.

4. **No per-component override**: You cannot enable a plugin but disable specific skills or agents within it. It's all-or-nothing.

**Evidence:** Plugins documentation on [settings](/en/settings) and [plugin installation](/en/discover-plugins), though explicit per-project override behavior is not heavily documented.

---

## 5. Roadmap Signals and Future Direction

### Documented Signals

1. **User-skill lazy loading**: No roadmap visibility. [Issue #16616](https://github.com/anthropics/claude-code/issues/16616) suggests awareness but no timeline provided.

2. **Deferred loading for other components**: [Issue #44536](https://github.com/anthropics/claude-code/issues/44536) requests "Lazy context loading: extend the ToolSearch pattern to all context components." **Status:** Open, no closure, suggests this is on the radar but not committed.

3. **Feature requests for task agents and skills**: [Issue #19445](https://github.com/anthropics/claude-code/issues/19445) requests deferred loading for task agents and skills (similar pattern to MCP ToolSearch). **Status:** Open.

### Inference

Anthropic is aware of context bloat from eager loading (MCP ToolSearch was a direct response). The extension to plugins, agents, and user skills is noted in the issue tracker but **not roadmapped publicly**. The vault meta-plugin fills a gap that **the official solution has not yet addressed**.

---

## 6. Conclusion: Is the Vault Idea Viable?

### Problems the Vault Would Solve

1. **User-skill eager loading**: Currently no built-in solution. A vault wrapper could defer loading until per-project activation.
2. **Per-project plugin granularity**: Better than the current all-or-nothing model; allows "install globally, activate selectively per project."
3. **Context window pressure**: Defers non-essential plugin components until needed within a specific project.

### Implementation Considerations

- **Whitelist pattern**: Projects list which installed plugins should be active, overriding global default.
- **Lazy skill pattern**: Skills within vaulted plugins load frontmatter-only until project selection is made, further deferring full content.
- **Hook compatibility**: Hooks are lightweight (JSON config), so vaulting them adds little value unless you also want to disable plugin agents.

### Open Questions

- How much token savings would a vault mechanism yield in practice? (Depends on size/count of user-level skills and unused plugins per project.)
- Would developers prefer a vault over upgrading Claude Code to fix the user-skill bug natively?
- What's the adoption curve for new meta-plugins vs. waiting for official solutions?

---

## References

- [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works.md)
- [Plugins documentation](https://code.claude.com/docs/en/plugins.md)
- [Issue #16616: User skills fully loaded instead of frontmatter-only](https://github.com/anthropics/claude-code/issues/16616)
- [Issue #44536: Lazy context loading for all components](https://github.com/anthropics/claude-code/issues/44536)
- [Issue #19445: Deferred loading for task agents and skills](https://github.com/anthropics/claude-code/issues/19445)
- [MCP Tool Search: Save 95% Context](https://claudefa.st/blog/tools/mcp-extensions/mcp-tool-search)
- [Tool search tool - Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)
- [Claude Skills Solve the Context Window Problem](https://tylerfolkman.substack.com/p/the-complete-guide-to-claude-skills)

