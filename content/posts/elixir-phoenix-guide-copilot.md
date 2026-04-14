---
title: "Elixir Phoenix Guide: Now on GitHub Copilot"
date: 2026-04-14
draft: false
tags: ["elixir", "phoenix", "github-copilot", "ai-tools", "hooks", "productivity"]
categories: ["tools", "ai"]
description: "The Elixir Phoenix Guide plugin is now available for GitHub Copilot. Same 19 instructions, 27 hooks, and 4 analysis scripts, built for Copilot's native instruction system."
---

The [Elixir Phoenix Guide](https://github.com/j-morgan6/elixir-phoenix-guide) is now available for [GitHub Copilot](https://github.com/j-morgan6/elixir-phoenix-guide-copilot). Everything from the Claude Code version, ported into Copilot's native instruction and hook system. Nothing was cut.

<!--more-->

## What the Plugin Does

The plugin gives your AI assistant opinionated, production-grade Elixir and Phoenix knowledge. It covers 19 domains across the framework and enforces 27 rules automatically as you write code.

### 19 Context-Aware Instructions

Each instruction targets a specific area of Phoenix development and loads automatically based on the file you're editing. Open a LiveView module and the LiveView rules are there. Open a test file and the testing guidance activates. No manual invocation needed.

The full set:

| Domain | Instructions |
|--------|-------------|
| Elixir Core | elixir-essentials |
| Ecto | ecto-essentials, ecto-changeset-patterns, ecto-nested-associations |
| LiveView | phoenix-liveview-essentials, phoenix-liveview-auth, phoenix-uploads |
| Auth | phoenix-auth-customization, phoenix-authorization-patterns |
| Real-Time | phoenix-channels-essentials, phoenix-pubsub-patterns |
| APIs | phoenix-json-api |
| Infrastructure | otp-essentials, oban-essentials, deployment-gotchas, telemetry-essentials |
| Quality | testing-essentials, code-quality, security-essentials |

Each instruction contains concrete rules with code examples showing both the wrong pattern and the correct one. They're not vague best-practice lists. They tell the model exactly what to write and what to avoid.

### 27 Automated Hooks

Hooks run on every tool call and catch problems before they make it into your codebase. 21 run before code is written. 6 run after, validating what was produced.

**Blocking hooks** stop known-dangerous patterns outright:

- `String.to_atom/1` on user input (atom table exhaustion)
- String interpolation inside Ecto `fragment` (SQL injection)
- Redirects built from user input without validation (open redirect)
- `mix ecto.reset` and force push (data loss)
- Deprecated APIs like `form_for`, `live_redirect`, and `.flash_group`

**Warning hooks** flag patterns that are usually wrong but sometimes intentional:

- `raw/1` in templates (XSS risk)
- Passwords, tokens, or secrets in Logger calls
- `==` comparison on tokens instead of `Plug.Crypto.secure_compare/2`
- Association access without a visible `preload` (N+1 queries)
- `with` blocks missing an `else` clause
- `Repo` calls directly in LiveView modules
- Nested conditionals, string concatenation in loops, hardcoded upload sizes

Every warning includes a copy-pasteable fix so you don't have to stop and look up the correct pattern.

### 4 Analysis Scripts

These run outside the instruction system and analyze your project directly:

- **code_quality.exs** detects code duplication, high complexity functions, and unused functions using AST analysis
- **detect_template_duplication.sh** identifies repeated HEEx blocks that should be extracted into components
- **detect_project.sh** maps your project's stack (Phoenix version, LiveView presence, Ecto adapter, Oban) so hooks can adapt their behavior
- **run_analysis.sh** orchestrates all of the above in one pass

### Project Detection

The plugin detects your project's stack at session start and adjusts accordingly. Building a JSON API without LiveView? The LiveView hooks stay silent. On Phoenix 1.7 instead of 1.8? You won't see warnings about `@current_scope`. The rules fit the project.

## Installation

Clone the repo and copy four things into your project root:

```bash {style=github}
git clone https://github.com/j-morgan6/elixir-phoenix-guide-copilot.git

cp -r elixir-phoenix-guide-copilot/.github your-project/.github
cp -r elixir-phoenix-guide-copilot/hooks your-project/hooks
cp -r elixir-phoenix-guide-copilot/scripts your-project/scripts
cp elixir-phoenix-guide-copilot/AGENTS.md your-project/AGENTS.md
```

Copilot picks up the instructions automatically. No configuration, no restart.

## Try It

GitHub: [elixir-phoenix-guide-copilot](https://github.com/j-morgan6/elixir-phoenix-guide-copilot)

Works with any Elixir/Phoenix project. Elixir only required locally if you want to run `code_quality.exs`.
