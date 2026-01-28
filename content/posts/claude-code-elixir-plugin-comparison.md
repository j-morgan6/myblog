---
title: "Building Better Elixir: A Claude Code Plugin Experiment"
date: 2026-01-27
draft: false
tags: ["elixir", "phoenix", "claude-code", "ai-development", "developer-tools"]
description: "Can custom skills and hooks improve AI-assisted Elixir development? I built the same app twice to find out."
---

## The Experiment

I've been using Claude Code for Elixir development and wondered: **Does customizing the AI with skills, hooks, and agent documentation actually make a difference?**

To find out, I built the same Phoenix LiveView image gallery application twice:

- **Build 1 (Baseline)**: Minimal configuration - just project guidelines and specs
- **Build 2 (Plugin)**: Custom skills, hooks, agent docs, and comprehensive project documentation

Both builds used identical project specifications and the same base agent guidelines. The only difference was the additional plugin configuration in Build 2.

## What I Built

A Phoenix LiveView image gallery with:
- Image uploads with progress tracking
- Folder organization (create, delete, move images between folders)
- Tab-based navigation
- Responsive TailwindCSS UI
- Local file storage with PostgreSQL metadata

Standard Phoenix/Ecto/LiveView stack

## The Plugin Configuration

I created a comprehensive configuration package:

**4 Skills:**
- `elixir-patterns.md` - Pattern matching, pipe operators, with statements
- `phoenix-liveview.md` - LiveView lifecycle, uploads, streams
- `ecto-database.md` - Schemas, changesets, queries
- `error-handling.md` - Tagged tuples, error handling patterns

**5 Hooks:**
- Warn on nested if/else (prefer pattern matching)
- Block hardcoded config values (require `Application.get_env/2`)
- Warn on `Enum.map |> Enum.filter` chains
- Block missing `@impl true` on callbacks
- Warn on string concatenation in loops

**4 Agent Documentation Files:**
- Project structure and context boundaries
- LiveView implementation checklist
- Ecto conventions and best practices
- Testing guide

**Plus:** `CLAUDE.md` with project overview, commands, and style preferences.

This configuration is a **one-time investment** that's reusable across all your Elixir projects.

## The Results: An Honest Look

### Code Quality & Testing

| Metric | Build 1 (Baseline) | Build 2 (Plugin) |
|--------|-------------------|-----------------|
| **Lines of Code** | ~550 | ~2,000+ (with tests) |
| **Automated Tests** | 0 | 46 tests |
| **Test Coverage** | Manual only | 28/28 context tests passing |
| **Credo Score** | Not run | Excellent (7 minor issues) |
| **Convention Adherence** | Good (2 mistakes) | Excellent |

**The plugin produced a more complete, production-ready codebase** with comprehensive test coverage and better adherence to Elixir conventions.

**The 7 minor Credo issues found:**
- 4 readability suggestions (formatting, test numbers)
- 3 design suggestions (nested module aliasing)

These are all minor quality improvements - the kinds of things that don't break functionality but make code more idiomatic and maintainable.

### Manual Testing: Still Required

**Both builds required manual testing to catch functional issues.**

Build 1 had issues with static file configuration and form handling that were caught during user testing. Build 2 encountered a broken upload feature that compiled fine but didn't work correctly until manually tested.

The plugin helps write better code with proper patterns and conventions, but it **cannot replace human validation**. Skills and hooks focus on code structure and idioms - they don't test whether your features actually work as intended. This is an important limitation to understand.

## Where the Plugin Shined

### 1. **Comprehensive Testing**

Build 2 included 46 automated tests covering all context operations, edge cases, and error handling. Build 1 had zero automated tests.

```elixir
# Example from Build 2's Gallery context tests
test "list_images_by_folder/1 returns only images in specified folder" do
  folder = folder_fixture()
  image1 = image_fixture(%{folder_id: folder.id})
  image2 = image_fixture()  # No folder

  images = Gallery.list_images_by_folder(folder.id)

  assert length(images) == 1
  assert Enum.any?(images, fn i -> i.id == image1.id end)
  refute Enum.any?(images, fn i -> i.id == image2.id end)
end
```

The agent documentation guided comprehensive test creation from the start, not as an afterthought.

### 2. **Better Code Quality**

Build 2 ran Credo analysis showing excellent adherence to Elixir conventions:
- Consistent pattern matching throughout
- Proper pipe operator usage
- Tagged tuples for error handling
- `@impl true` on all callbacks
- No critical or warning-level issues

Build 1 worked fine but didn't have this quality validation.

### 3. **Better Documentation & Structure**

Build 2 included:
- Clear context boundaries
- Comprehensive function documentation
- Project structure guides
- Quick reference for common patterns

This makes the codebase more maintainable for teams.

### 4. **Idiomatic Elixir from the Start**

The skills and agent docs encouraged idiomatic patterns:

```elixir
# Pattern matching over if/else
def process({:ok, result}), do: transform(result)
def process({:error, reason}), do: handle_error(reason)

# with statements for multi-step operations
def create_image_with_folder(attrs, folder_id) do
  with {:ok, folder} <- get_folder(folder_id),
       {:ok, image} <- create_image(attrs) do
    {:ok, %{image: image, folder: folder}}
  end
end
```

Build 1 also used good patterns (likely from the base agent guidelines), but Build 2's code was more consistently idiomatic throughout.

## Key Takeaways

### What the Plugin Does Well

- **Encourages comprehensive testing** - Tests are written during development, not as an afterthought
- **Enforces idiomatic Elixir** - Pattern matching, pipe operators, proper error handling
- **Improves code quality metrics** - Credo scores, convention adherence
- **Creates better documentation** - Clear structure, maintainable codebases
- **Reduces decision fatigue** - Agent docs guide implementation choices

### What the Plugin Doesn't Do

- **Doesn't prevent functional bugs** - Manual testing still essential for validating features work correctly
- **Doesn't guarantee faster development** - Focus is on quality over speed
- **Doesn't catch integration issues** - Focus is on code patterns, not runtime behavior

### The Honest Verdict

The plugin doesn't magically make development faster or bug-free. **But it produces higher-quality, more maintainable code with better test coverage.**

If you're building Elixir applications professionally, the investment is worth it. If you're prototyping or building one-off projects, the baseline approach might be sufficient.

## What's Next

This is version 1 of the plugin, and I plan to improve it based on these findings:

**Potential improvements:**
- Add skills for functional testing patterns
- Create hooks that catch common LiveView lifecycle issues
- Include integration testing guidance
- Add checklists for manual feature validation
- Expand error handling patterns

## Try It Yourself

I'm continuing to refine this plugin for the Elixir community. The configuration is available on GitHub:

**[Try it for yourself](https://github.com/j-morgan6/elixir-claude-optimization)**

The configuration package includes:

- Skills for Elixir patterns, LiveView, Ecto, and error handling
- Hooks that enforce conventions and warn about anti-patterns
- Agent documentation for project structure and testing
- Sample CLAUDE.md for project setup
---

*Tags: #elixir #phoenix #claude-code #ai-development #developer-tools*
