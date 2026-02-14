---
title: "Elixir Phoenix Guide v1.2.0: From 8 Skills to 4 Essentials"
date: 2026-02-14
draft: false
tags: ["elixir", "phoenix", "claude-code", "ai-tools", "productivity"]
categories: ["tools", "ai"]
description: "Version 1.2.0 of the Elixir Phoenix Guide plugin consolidates eight skills into four essentials with non-negotiable RULES sections and real-time anti-pattern detection."
---

## What Changed

Released v1.2.0 of my [Elixir Phoenix Guide plugin](https://github.com/j-morgan6/elixir-phoenix-guide). This is a major restructuring: eight separate skills consolidated into four essential modules, each with enforced rules and real-time validation.

This isn't a minor update. It's a complete rethinking of how Claude Code should guide Elixir/Phoenix development.

## The Problem with v1.1.x

In v1.1.x, I had eight skills covering different aspects of Elixir and Phoenix development. The structure worked, but created problems:

**Skill fragmentation:** Logic was scattered across multiple files. Want to understand LiveView patterns? You'd need to check 3-4 different skills.

**Inconsistent enforcement:** Some skills suggested best practices. Others were more directive. The inconsistency meant Claude Code would sometimes follow patterns and sometimes skip them.

**Overlap and gaps:** Related concerns lived in separate files. This created both duplication and blind spots where patterns fell through the cracks.

The result: The plugin had good information but inconsistent application.

## What Changed in v1.2.0

### Four Essential Skills

Consolidated eight skills into four focused modules:

- **elixir-essentials** - Core Elixir patterns (pipe operators, with statements, pattern matching)
- **phoenix-liveview-essentials** - LiveView lifecycle, rendering, events, navigation
- **ecto-essentials** - Database schemas, queries, changesets, migrations
- **phoenix-uploads** - File upload patterns and static file serving

Each skill is comprehensive. Everything related to a domain lives in one place.

### RULES Sections: Non-Negotiable Patterns

Every skill now includes a RULES section with 7-8 patterns that **must** be followed:

**Example from ecto-essentials:**
```text {style=github}
RULES:
1. NEVER use Ecto.Query.from/2 without select - always explicit
2. ALWAYS use Repo.transaction/1 for multi-step operations
3. NEVER modify associations without preloading first
4. ALWAYS validate foreign keys in changesets
5. NEVER use string-based queries - use query syntax
6. ALWAYS handle Repo errors with pattern matching
7. NEVER bypass changesets for data modification
```

These aren't suggestions. They're requirements. Claude Code must follow these patterns before writing code.

**Why this matters:** Previous versions suggested best practices. v1.2.0 enforces them. The difference between "you should" and "you must."

### Real-Time Anti-Pattern Detection

Enhanced hooks now catch violations as they happen:

- Writing a LiveView without proper mount? Blocked.
- Modifying Ecto associations without preloading? Caught.
- Using string-based queries instead of query syntax? Prevented.

The hooks work with the RULES sections to enforce patterns at write-time, not after-the-fact during code review.

### File Pattern Detection

Skills now activate automatically based on file patterns:

- Open a file with `Ecto.Schema` → ecto-essentials activates
- Working on a `.heex` template → phoenix-liveview-essentials activates
- Implementing `handle_event` → LiveView lifecycle rules apply

No need to manually invoke skills. The plugin detects context and applies the right patterns.

## Breaking Changes

**This is a breaking release.** The restructuring means:

- Skill names changed (old skills are gone)
- Hook configurations updated
- Installation structure modified

**You cannot update in place.** You must uninstall v1.1.x and reinstall v1.2.0.

## Migration Steps

### Uninstall v1.1.x

```bash {style=github}
claude plugin uninstall elixir-optimization
# or remove manually
rm -rf ~/.claude/skills/elixir-*
rm -rf ~/.claude/hooks/elixir-*
```

### Install v1.2.0

```bash {style=github}
git clone https://github.com/j-morgan6/elixir-phoenix-guide
cd elixir-phoenix-guide
./install.sh
```

Or via Claude Code plugin system:

```bash {style=github}
claude plugin install j-morgan6/elixir-phoenix-guide
```

### Verify Installation

Check that you have exactly four skills:

```bash {style=github}
ls ~/.claude/skills/ | grep elixir-phoenix-guide
```

You should see:
- elixir-essentials.md
- phoenix-liveview-essentials.md
- ecto-essentials.md
- phoenix-uploads.md

## What You Get

### Consistent Enforcement

Every skill has the same structure:
1. INVOKE BEFORE directive (when to use the skill)
2. RULES section (non-negotiable patterns)
3. Detailed guidance (how to implement patterns)
4. Common mistakes (what to avoid)

Claude Code now consistently applies Elixir/Phoenix best practices across all code it writes.

### Comprehensive Coverage

Instead of wondering "which skill covers this?", each domain is complete:

**Need LiveView help?** Everything is in phoenix-liveview-essentials:
- Mount lifecycle
- Static vs connected rendering
- Event handling
- Navigation and URLs
- PubSub patterns
- Streams
- File uploads
- Testing

**Working with Ecto?** All database patterns in ecto-essentials:
- Schema design
- Changesets and validation
- Query composition
- Associations and preloading
- Transactions
- Migrations
- Testing strategies

No gaps, no hunting across multiple files.

### Fewer False Positives

With clearer rules and better detection, Claude Code makes fewer mistakes:

- Won't suggest anti-patterns that "mostly work"
- Catches errors earlier in the development cycle
- Enforces patterns consistently, not sporadically

## From Guidance to Guardrails

The shift from v1.1.x to v1.2.0 changes the plugin's role:

**v1.1.x:** "Here are best practices you should follow"
**v1.2.0:** "Here are rules you must follow, and I'll enforce them"

The plugin moved from advisory to prescriptive. It doesn't just suggest patterns—it ensures they're followed.

## Why Consolidation Matters

Eight skills felt comprehensive but created cognitive load:
- "Which skill covers this scenario?"
- "Is this pattern in elixir-patterns or elixir-conventions?"
- "Did I invoke all the relevant skills?"

Four essentials eliminate the guesswork. If you're writing Elixir code, invoke elixir-essentials. Working with LiveView? phoenix-liveview-essentials. It's that simple.

Fewer, more comprehensive skills means:
- Less decision fatigue
- More consistent application
- Clearer mental model of what the plugin covers

## Try It

GitHub: [elixir-phoenix-guide](https://github.com/j-morgan6/elixir-phoenix-guide)

Requirements: Elixir 1.15+, Phoenix 1.7+ (for LiveView features)

**Note:** If you're on v1.1.x, you must uninstall first. This is not an in-place upgrade.

