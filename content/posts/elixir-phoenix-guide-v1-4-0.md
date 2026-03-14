---
title: "Elixir Phoenix Guide v1.4.0: OTP, Oban, and Subagent Enforcement"
date: 2026-03-14
draft: false
tags: ["elixir", "phoenix", "claude-code", "ai-tools", "otp", "oban", "productivity"]
categories: ["tools", "ai"]
description: "Version 1.4.0 adds otp-essentials and oban-essentials skills, three new safety hooks, and subagent enforcement so spawned agents follow the same rules."
---

## What Changed

Released v1.4.0 of my [Elixir Phoenix Guide plugin](https://github.com/j-morgan6/elixir-phoenix-guide). Two new skills, three new hooks, and a fix for something that's been bugging me: subagents ignoring the rules.

## The Problem with v1.3.x

The plugin had five skills covering Elixir core, LiveView, Ecto, uploads, and testing. That's solid for web layer code. But two big gaps remained.

**No OTP guidance.** GenServer, Supervisor, Task, Agent, ETS. These are core to any real Elixir application, and the plugin had nothing to say about them. Claude Code would write GenServers with blocking `handle_call` callbacks, skip supervision trees, or reach for Agent when a simple GenServer would do.

**No Oban guidance.** Background jobs are everywhere in production Phoenix apps. Without rules, Claude Code would write workers without idempotency, skip unique job constraints, or test against the real queue instead of using `Oban.Testing`.

And then there was the subagent problem. You could have all five skills loaded in your main conversation, but the moment Claude Code spawned a subagent to write code in parallel, that subagent had none of the context. It would happily write code that violated every rule the plugin defines.

## What's New in v1.4.0

### otp-essentials Skill

7 rules for OTP patterns:

```text {style=github}
Covers:
- GenServer (init, handle_call, handle_cast, handle_info)
- Supervisor and DynamicSupervisor
- Task and Task.Supervisor
- Agent
- Registry
- ETS
- Common anti-patterns
```

The big ones: always supervise your GenServers, use `handle_continue` for expensive init work instead of blocking in `init/1`, and don't use Agent as a poor man's GenServer when you need more than get/update.

### oban-essentials Skill

7 rules for Oban background jobs:

```text {style=github}
Covers:
- Worker definition and configuration
- Queue setup and tuning
- Idempotent job design
- Unique job constraints
- Cron scheduling with Oban.Cron
- Testing with Oban.Testing
- Error handling and retries
```

The rule I'm happiest about: every worker must be idempotent. If a job can run twice and produce a different result, that's a bug waiting to happen. The skill enforces this pattern from the start.

### Three New Hooks

**dangerous-operations-blocker** blocks destructive commands before they run:

```text {style=github}
Blocked commands:
- mix ecto.reset (use mix ecto.rollback instead)
- git push --force (use --force-with-lease)
- MIX_ENV=prod in development
```

This is the first Bash-level hook in the plugin. It exits with code 2, which means the command is stopped entirely, not just warned about.

**debug-statement-detector** catches leftover debug statements when writing or editing files:

```text {style=github}
Warns on:
- IO.inspect (outside test files)
- dbg()
- IO.puts (outside test files)
```

It exits with code 1, so you get a warning but the write still goes through. Sometimes you want `IO.inspect` while you're working. The reminder is there so you don't forget to remove it.

**security-audit-reminder** nudges you to run security audits when `mix.exs` changes:

```text {style=github}
Suggests running:
- mix deps.audit
- mix hex.audit
- mix sobelow (if installed)
```

This is a soft nudge (exit 0). Changing dependencies is a good time to check for known vulnerabilities.

### Subagent Enforcement

This is the one that fixes the subagent blind spot. A new `SubagentStart` hook injects condensed rules from all seven skills into every spawned subagent. When Claude Code kicks off a parallel agent to write a module, that agent now starts with the same rules loaded.

Before this, a subagent writing a GenServer had zero guidance. Now it gets the OTP rules, the Elixir essentials rules, and whatever else applies. Same standards everywhere.

## By the Numbers

| | v1.3.x | v1.4.0 |
|---|---|---|
| Skills | 5 | 7 |
| Hooks | 10 | 13 |
| Subagent enforcement | No | Yes |

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

To verify the new skills are there:

```bash {style=github}
ls ~/.claude/skills/ | grep elixir-phoenix-guide
```

You should see seven files now:
- elixir-essentials.md
- phoenix-liveview-essentials.md
- ecto-essentials.md
- phoenix-uploads.md
- testing-essentials.md
- otp-essentials.md
- oban-essentials.md

## Try It

GitHub: [elixir-phoenix-guide](https://github.com/j-morgan6/elixir-phoenix-guide)

Requirements: Elixir 1.15+, Phoenix 1.7+
