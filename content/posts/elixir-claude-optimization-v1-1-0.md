---
title: "Elixir Claude Optimization v1.1.2: Better Skill Discovery"
date: 2026-02-09
draft: True
tags: ["elixir", "claude-code", "ai-tools", "productivity"]
categories: ["tools", "ai"]
description: "Version 1.1.2 of the Elixir Claude optimization plugin adds systematic skill discovery and clearer invocation patterns."
---

## What's New

Released v1.1.2 of my [Elixir Claude optimization plugin](https://github.com/j-morgan6/elixir-claude-optimization). The focus: making skills easier to discover and use.

## The Problem

In v1.0.0, Claude Code would often miss applicable skills. The "Use when" language wasn't directive enough, and there was no systematic way to identify which skills applied to a task.

Result: Skills existed but weren't being invoked consistently.

## What Changed

### New Skill Discovery System
Added a meta-skill called `skill-discovery` that provides a systematic checklist based on file types and task requirements. Think of it as a flowchart for skill selection.

### Clearer Invocation Language
Replaced passive "Use when" phrasing with mandatory "INVOKE BEFORE" directives across all skills.

**Before (v1.0.0):**
```text {style=github}
Use when working with Ecto schemas or queries
```

**After (v1.1.2):**
```text {style=github}
INVOKE BEFORE modifying any Ecto schema, query, or migration
```

The difference: Command, not suggestion.

### File Pattern Detection
The skill-discovery system now recognizes file patterns:
- `.ex` files with `Ecto.Schema` → invoke ecto-database
- LiveView modules → invoke phoenix-liveview
- Pattern matching logic → invoke elixir-patterns

## Expected Impact

Early testing shows a 50%+ increase in skill utilization. Claude Code now consistently identifies applicable skills before starting work.

## Migration

If you installed v1.0.0:

```bash {style=github}
claude plugin update elixir-optimization
```

Or re-run the installer:

```bash {style=github}
./install.sh
```

No manual configuration changes needed. The update handles everything automatically.

## Check Your Version

Look at any skill file in `~/.claude/skills/`. If you see "INVOKE BEFORE" language, you're on v1.1.2+. If you see "Use when", you're still on v1.0.0.

## Why This Matters

Skills are only valuable if they're actually invoked. v1.0.0 had the right skills but inconsistent activation. v1.1.2 fixes the activation problem.

The plugin now does what it was supposed to do from the start: ensure Claude Code follows Elixir/Phoenix best practices *before* writing code.

## Try It

GitHub: [elixir-claude-optimization](https://github.com/j-morgan6/elixir-claude-optimization)

Requirements: Elixir 1.15+, Phoenix 1.7+ (for LiveView features)
