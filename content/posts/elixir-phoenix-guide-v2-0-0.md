---
title: "Elixir Phoenix Guide v2.0.0: Automated Code Quality Detection"
date: 2026-03-19
draft: false
tags: ["elixir", "phoenix", "claude-code", "ai-tools", "code-quality", "productivity"]
categories: ["tools", "ai"]
description: "Version 2.0.0 adds automated code quality analysis: AST-based duplication detection, ABC complexity scoring, unused function detection, and template duplication checks that run after every file write."
---

## What Changed

Released v2.0.0 of my [Elixir Phoenix Guide plugin](https://github.com/j-morgan6/elixir-phoenix-guide). This is a major version bump. The plugin now detects code quality issues automatically, not just through skills and hooks telling you what to do.

<!--more-->

## From Guidance to Detection

Through v1.4.0 the plugin had seven skills and thirteen hooks. Skills tell Claude Code how to write idiomatic Elixir. Hooks catch anti-patterns as you write. That foundation works well for prevention.

v2.0.0 adds the next layer: detection after the code is written. The plugin can now look at a file you just saved and spot duplicated functions across modules, complexity that's crept up in a callback, `defp` functions left behind after a refactor, or two LiveView templates sharing most of their markup. It surfaces these findings automatically so you can act on them while the context is still fresh.

## What's New in v2.0.0

### PostToolUse Code Quality Hook

This is the core of the release. A new hook runs automatically after every file write (Write or Edit). It analyzes what you just wrote and reports what it finds.

For `.ex` and `.exs` files, it runs three checks:
- **Code duplication detection** across sibling modules
- **ABC complexity analysis** on every function
- **Unused private function detection** within the module

For `.heex` files, it checks for template duplication against other templates in the same directory.

The hook doesn't block anything. It reports what it finds and lets you decide what to do about it. Sometimes you're in the middle of a refactor and duplication is temporary. The hook keeps you informed so you can address it when you're ready.

### code-quality Skill

The 8th skill in the plugin. 7 rules covering:

```text {style=github}
Rules:
1. Extract duplicated functions into a shared module
2. Break functions exceeding ABC complexity of 30
3. Remove unused private functions after refactoring
4. Extract shared HEEx markup into function components
5. Run full project scan before opening a PR
6. Fix all critical findings before committing
7. Use on-demand scripts for CI/CD integration
```

The skill gives Claude Code the vocabulary to act on what the hook finds. The hook says "these two functions are 85% similar." The skill says "extract them into a shared module and call it from both places."

### Analysis Scripts

Three scripts power the detection:

**code_quality.exs** is the main engine. It uses Elixir's AST to parse function bodies, converts them to normalized strings with `Macro.to_string/1`, and compares them using trigram similarity. Two functions with more than 70% overlap get flagged. It also walks the AST counting assignments, branches, and conditions to produce an ABC complexity score. Anything over 30 gets flagged.

```bash {style=github}
# Analyze a single file
elixir ~/.claude/scripts/elixir-phoenix-guide/code_quality.exs all lib/my_module.ex

# Scan an entire project
elixir ~/.claude/scripts/elixir-phoenix-guide/code_quality.exs scan lib/
```

**detect_template_duplication.sh** compares HEEx templates in the same directory. Templates sharing more than 40% identical markup get flagged.

**run_analysis.sh** runs everything together. Point it at your project and get a full report. This one is built for CI/CD. Run it in your pipeline before merge and catch quality issues before they land.

```bash {style=github}
# Full project analysis
bash ~/.claude/scripts/elixir-phoenix-guide/run_analysis.sh
```

### How It Works in Practice

You ask Claude Code to write a context module. It writes the file. The PostToolUse hook fires, parses the AST, and finds that `create_item/1` in the new module is 82% similar to `create_record/1` in a sibling module. It also flags a `defp format_params/1` that nothing calls.

Claude Code sees the findings, references the code-quality skill, and extracts the shared logic into a helper module. It removes the unused function. You didn't have to ask for any of this. The hook surfaced the findings, the skill provided the playbook.

## Update

```bash {style=github}
/plugin
# Select "Marketplaces" → "elixir-phoenix-guide" → "Update"
```

To verify the new analysis scripts are installed:

```bash {style=github}
ls ~/.claude/scripts/elixir-phoenix-guide/
```

You should see three files:
- code_quality.exs
- detect_template_duplication.sh
- run_analysis.sh

## Try It

GitHub: [elixir-phoenix-guide](https://github.com/j-morgan6/elixir-phoenix-guide)

Requirements: Elixir 1.15+, Phoenix 1.7+
