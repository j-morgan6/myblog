---
title: "Elixir Phoenix Guide v2.2.0: Smart Enforcement"
date: 2026-03-23
draft: false
tags: ["elixir", "phoenix", "claude-code", "ai-tools", "hooks", "code-quality", "productivity"]
categories: ["tools", "ai"]
description: "Version 2.2.0 adds a project detection system and four new PostToolUse hooks. Hooks now adapt to your project stack and include copy-pasteable fix suggestions."
---

## What Changed

Released v2.2.0 of my [Elixir Phoenix Guide plugin](https://github.com/j-morgan6/elixir-phoenix-guide). The big change: hooks are now context-aware. They detect your project stack at the start of a session and adjust their behavior accordingly. Four new validation hooks bring the total to 21, and every warning now includes a fix you can copy and paste.

<!--more-->

## Project Detection

Previous versions applied every rule to every project. That meant LiveView hooks would fire on API-only projects, and Phoenix 1.7 projects would see warnings that only applied to 1.8. Not harmful, but noisy.

v2.2.0 fixes this with a detection system. The SessionStart hook runs `detect_project.sh` once per session, parsing your `mix.exs` to identify:

- **Phoenix version** (1.7 vs 1.8+ with the Scope struct)
- **LiveView presence** (full-stack vs API-only)
- **Ecto adapter** (Postgres, SQLite, MySQL)
- **Oban presence**

Results get cached to `.elixir-phoenix-guide-project.json`. Every hook reads this file before deciding whether to run.

The practical effect: if you're building a JSON API without LiveView, hooks like `missing-impl`, `auto-upload`, and `context-boundary-violation` stay silent. If you're on Phoenix 1.8+, you'll get a warning when your code references `@current_user` instead of `@current_scope`. The plugin fits the project instead of the other way around.

## Four New Validation Hooks

These run after code is written, catching architectural issues that are easy to miss during review.

### missing-preload

Warns when you access an association without a visible `preload`. This is the classic N+1 query source in Ecto. You load a list of posts, iterate over them in a template, and call `post.author.name`. It works in dev with 10 rows. It falls over in production with 10,000.

### missing-error-clause

Warns on `with` statements that don't have an `else` clause. Without one, any non-matching result falls through silently. You get `{:error, :not_found}` returned raw to a controller that expected an `{:ok, result}` tuple, and the error message is useless.

### raw-sql-warning

Two levels here. String interpolation inside raw SQL (`fragment("... #{value} ..."`) is blocked outright because that's SQL injection. Plain `fragment()` usage gets a softer warning reminding you to use parameterized queries.

### context-boundary-violation

Warns when `Repo` calls appear directly in LiveView modules. The convention in Phoenix is to route all data access through context modules. Direct `Repo` calls in LiveViews make testing harder and scatter query logic across the codebase. This hook skips entirely on API-only projects since there are no LiveViews to check.

## Better Fix Suggestions

In previous versions, most hooks just pointed out the problem. Now they include a copy-pasteable fix example:

```text {style=github}
⚠️  Nested if/else detected.

   💡 Fix: Replace with case or multi-clause function:

     case {condition_a, condition_b} do
       {true, true} -> handle_both()
       {true, false} -> handle_a_only()
       _ -> handle_default()
     end
```

This applies to the existing hooks too: nested-if-else, inefficient-enum, string-concatenation, missing-impl, hardcoded-paths, and hardcoded-sizes all got upgraded.

The difference matters most when you're in the middle of writing something and a hook fires. Instead of stopping to look up the idiomatic pattern, the fix is right there.

## The Full Picture

| Component | v2.0.0 | v2.1.0 | v2.2.0 |
|-----------|--------|--------|--------|
| Skills | 8 | 14 | 14 |
| Hooks | 14 | 15 | 21 |
| Analysis Scripts | 3 | 3 | 4 |
| Agent Docs | 4 | 4 | 4 |

The hook count jumped from 15 to 21. The breakdown: 1 SessionStart, 14 PreToolUse, 5 PostToolUse, and 1 SubagentStart. The new PostToolUse hooks handle validation that only makes sense after code exists on disk, which is why they run after writes rather than before.

## Update

```bash {style=github}
/plugin
# Select "Marketplaces" → "elixir-phoenix-guide" → "Update"
```

## Try It

GitHub: [elixir-phoenix-guide](https://github.com/j-morgan6/elixir-phoenix-guide)

Requirements: Elixir 1.15+, Phoenix 1.7+
