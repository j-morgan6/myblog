---
title: "My AI Tool Stack (January 2026)"
date: 2026-01-11
draft: false
tags: ["ai-tools", "productivity", "elixir", "phoenix", "development-workflow"]
categories: ["tools", "ai"]
description: "The AI tools I actually use daily for Elixir and Phoenix LiveView development - Windsurf, Claude Code, and TideWave."
---

## The Stack

Here's what I'm actually using day-to-day for Elixir/Phoenix LiveView development. Not what I tried once and forgot about.

### Code Editing: Windsurf

**What I use it for:**
- Primary IDE for Elixir/Phoenix development
- In-editor AI assistance without context switching
- Refactoring suggestions that understand Phoenix patterns

**Why it stuck:** Feels native. The AI suggestions understand Elixir idioms and Phoenix conventions. When I'm working on a LiveView component, it actually suggests LiveView-appropriate patterns, not React patterns.

### Command Line AI: Claude Code

**What I use it for:**
- Quick terminal assistance without leaving the command line
- File operations and git workflows
- Research and documentation lookup

**Why it stuck:** Lives where I already work. When I'm debugging an Elixir error, I can pipe it straight to Claude without opening a browser.

### AI Agent Development: TideWave

**What it is:** Framework for building custom AI agents

**What I use it for:**
- Building specialized automation for my workflow
- Custom agents for Elixir-specific tasks
- Integrating AI into deployment pipelines

**Why it stuck:** Lets me build exactly what I need instead of hoping a generic tool adds the feature. For Elixir work, that matters - the generic tools often miss Elixir-specific patterns.

## What I'm NOT Using (And Why)

### GitHub Copilot
Tried it, didn't stick. Suggestions felt generic and often suggested JavaScript/Python patterns when I needed Elixir patterns. Windsurf does this better for my stack.

### ChatGPT Web Interface
Too much context switching. Claude Code in the terminal is faster for development questions.

## The Workflow

Here's how they work together:

**Feature Development:**
1. Code in Windsurf (AI assists with implementation)
2. Debug with Claude Code (pipe errors directly to AI)
3. Deploy with TideWave agents (automated AI-assisted deployment)

**Quick Fixes:**
1. Claude Code in terminal (fast, no context switch)
2. Edit in Windsurf if it's more complex

## The Elixir/Phoenix Angle

Why does this matter for Elixir specifically?

**Pattern Recognition:** Generic AI tools often suggest JavaScript/Python patterns. These tools (especially Windsurf) understand functional programming patterns and Phoenix conventions.

**Community Size:** Elixir's smaller community means fewer AI training examples. Having tools that *deeply* understand the ecosystem matters more than with mainstream languages.

**LiveView Complexity:** Phoenix LiveView is different enough from React/Vue that generic suggestions fall flat. Tools that understand server-rendered, stateful components make a huge difference.

## Would I Recommend This Stack?

**If you're doing Elixir/Phoenix:** Yes, especially Windsurf and Claude Code.

**If you're doing other languages:** Maybe. The workflow works, but Windsurf's Elixir understanding is a big reason it stuck for me.

**If you're on a team:** The workflow is focused on individual productivity. Larger teams may need additional collaboration tools.

**If you're cost-conscious:** Start with Claude Code (pay-as-you-go). Add the others if you find yourself wishing for their specific features.

## Try It Yourself

This is what works for *me* in *January 2026* for *Elixir/Phoenix*. Your mileage will vary.

The best stack is the one you actually use. Try these, see what sticks, and drop what doesn't.

---