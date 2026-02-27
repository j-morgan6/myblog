---
title: "Elixir Phoenix Guide v1.3.0: Testing Essentials"
date: 2026-02-27
draft: false
tags: ["elixir", "phoenix", "claude-code", "ai-tools", "testing", "productivity"]
categories: ["tools", "ai"]
description: "Version 1.3.0 adds a dedicated testing-essentials skill with 8 rules, TDD guidance, and pre-built test templates to make testing a consistent part of the workflow."
---

## What Changed

Released v1.3.0 of my [Elixir Phoenix Guide plugin](https://github.com/j-morgan6/elixir-phoenix-guide). This one is all about testing.

v1.2.0 built out four skills that cover Elixir patterns, LiveView, Ecto, and file uploads. What I didn't have was anything for tests. You could open a `_test.exs` file and the plugin had nothing to say. That felt like a hole worth fixing.

## The Problem with v1.2.0

The four skills worked well for production code but testing was just not covered. Claude Code would write a context module following strict patterns, then write the tests with no real guidance. The result was inconsistent test setup, missing edge cases, and assertion styles that made failures hard to read.

Testing deserves the same treatment: clear rules, templates ready to go, and automatic activation when you open a test file.

## What's New in v1.3.0

### testing-essentials Skill

The new `testing-essentials` skill kicks in automatically on any `_test.exs` file. It comes with 8 rules:

```text {style=github}
RULES:
1. ALWAYS use DataCase for database tests, ConnCase for HTTP tests
2. NEVER test implementation details - test behavior and outcomes
3. ALWAYS write the failing test first before implementation
4. NEVER use raw Repo calls in tests - use context functions
5. ALWAYS use ExUnit.Case async: true for unit tests
6. NEVER assert on full structs - assert specific fields
7. ALWAYS test both happy path and error cases
8. NEVER share mutable state between test cases
```

It also nudges you to write the failing test before writing any implementation. Not a strict TDD process, just enough of a reminder to keep the habit.

The skill includes ready-to-use templates for:

- DataCase and ConnCase setup
- Factory and fixture patterns
- LiveView testing with `live/2` and `assert_patch`
- Context operation tests
- Changeset validation tests

### Testing wired into the other skills

All four existing skills now point to `testing-essentials` when you are working on test files. So whether you are in elixir-essentials, phoenix-liveview-essentials, ecto-essentials, or phoenix-uploads, opening a test file will bring in the testing guidance automatically.

### testing-guide.md cleanup

The DataCase and ConnCase setup templates that lived in `testing-guide.md` moved into the skill itself so they are available immediately. The guide still has all the detailed examples but now it is more of a reference you go to when you want to dig deeper, not the first thing you read.

## No Breaking Changes

Nothing changed in the four existing skills except for the added references to `testing-guide.md`. Your current setup keeps working.

## Update

```bash {style=github}
cd elixir-phoenix-guide
git pull
./install.sh
```

Or reinstall via Claude Code:

```bash {style=github}
claude plugin install j-morgan6/elixir-phoenix-guide
```

To verify the new skill is there:

```bash {style=github}
ls ~/.claude/skills/ | grep elixir-phoenix-guide
```

You should see five files now:
- elixir-essentials.md
- phoenix-liveview-essentials.md
- ecto-essentials.md
- phoenix-uploads.md
- testing-essentials.md

## Why Testing Needed Its Own Skill

Tests are easy to skip when you are moving fast. When a skill activates automatically and tells you exactly what structure to use, there is less friction. The plugin now covers writing the code and writing the tests for it, which is where I wanted it to be.

## Try It

GitHub: [elixir-phoenix-guide](https://github.com/j-morgan6/elixir-phoenix-guide)

Requirements: Elixir 1.15+, Phoenix 1.7+
