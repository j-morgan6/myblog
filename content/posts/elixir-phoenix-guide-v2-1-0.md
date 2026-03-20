---
title: "Elixir Phoenix Guide v2.1.0: Full Lifecycle Coverage"
date: 2026-03-20
draft: false
tags: ["elixir", "phoenix", "claude-code", "ai-tools", "authentication", "ecto", "productivity"]
categories: ["tools", "ai"]
description: "Version 2.1.0 adds six new skills and a migration-safety hook. The plugin now covers authentication, authorization, changesets, PubSub, nested associations, and safe migrations across 14 skills and 15 hooks."
---

## What Changed

Released v2.1.0 of my [Elixir Phoenix Guide plugin](https://github.com/j-morgan6/elixir-phoenix-guide). Six new skills bring the total to 14. One new hook brings it to 15. The plugin now covers the full Phoenix development lifecycle, from authentication through data patterns to deployment safety.

<!--more-->

## From Detection to Coverage

v2.0.0 introduced automated code quality detection. That was about depth: analyzing what you just wrote and surfacing problems. v2.1.0 is about breadth. The plugin can now guide Claude Code through authentication flows, authorization logic, complex changeset patterns, real-time PubSub, nested associations, and safe migrations.

Each of these skills came from a real debugging session. The kind where you spend 30 to 90 minutes tracking down why `cast_assoc` keeps returning "can't be blank" on a foreign key that should be set automatically, or why `@current_scope` crashes in a template when the user isn't logged in. These are patterns that look straightforward in the docs but break in specific, frustrating ways in practice.

## What's New in v2.1.0

### 6 New Skills

| Skill | Rules | Covers |
|-------|-------|--------|
| **phoenix-liveview-auth** | 7 | `on_mount` callbacks, `current_scope`, import conflicts, session handling |
| **ecto-changeset-patterns** | 7 | Separate changesets per operation, `cast_assoc` pitfalls, composition |
| **phoenix-auth-customization** | 6 | Extending `phx.gen.auth` with custom fields, fixtures, confirmation |
| **phoenix-pubsub-patterns** | 6 | Subscriptions, broadcasting from contexts, topic naming |
| **phoenix-authorization-patterns** | 6 | Server-side authz, ownership, policy modules, scoped queries |
| **ecto-nested-associations** | 6 | `cast_assoc`, `Ecto.Multi`, cascades, FK indexes |

### phoenix-liveview-auth

LiveView authentication has a few patterns that look simple until you hit the edge cases. The skill enforces `on_mount` callbacks over inline auth checks in `mount/3`, and catches the bracket-access issue that bites most people at least once.

The standard pattern:

```elixir {style=github}
# Define once in UserAuth
def on_mount(:require_authenticated_user, _params, session, socket) do
  socket = mount_current_scope(socket, session)

  if socket.assigns.current_scope && socket.assigns.current_scope.user do
    {:cont, socket}
  else
    socket =
      socket
      |> put_flash(:error, "You must log in to access this page.")
      |> redirect(to: ~p"/users/log_in")

    {:halt, socket}
  end
end
```

One rule that saves a lot of confusion: use `assigns[:current_scope]` in templates, not `@current_scope`. Dot access crashes on nil when the user isn't authenticated. Bracket access returns nil. Small difference, but it matters in layouts rendered for both authenticated and unauthenticated users.

Another: `Phoenix.Controller` and `Phoenix.LiveView` both export `redirect/2` and `put_flash/3`. The skill enforces explicit `except:` imports to avoid ambiguity.

### ecto-changeset-patterns

The biggest rule here is separate changesets per operation. Registration needs email, username, and password validation. Email changes need only email validation with reconfirmation. Password changes need only password validation. Overloading a single `changeset/2` with conditional logic leads to bugs.

```elixir {style=github}
# Registration — all fields, password hashing
def registration_changeset(user, attrs, opts \\ []) do
  user
  |> cast(attrs, [:email, :username, :password])
  |> validate_email(opts)
  |> validate_username()
  |> validate_password(opts)
end

# Email change — only email
def email_changeset(user, attrs, opts \\ []) do
  user
  |> cast(attrs, [:email])
  |> validate_email(opts)
end

# Profile update — non-sensitive fields only
def profile_changeset(user, attrs) do
  user
  |> cast(attrs, [:username, :bio])
  |> validate_username()
end
```

The `cast_assoc` rule is the one that saves the most time. When you use `cast_assoc`, the parent sets the foreign key automatically. If your child changeset requires that foreign key in `cast`, you get "can't be blank" errors that make no sense until you realize the key hasn't been set yet. The skill catches this before you waste 30 minutes on it.

### migration-safety Hook

The 15th hook in the plugin. It runs before any migration file is written and checks for four common deployment issues:

- **Missing FK indexes** on `references()` columns
- **Missing `on_delete` strategy** (should you cascade, nilify, or do nothing?)
- **Unsafe column removals** without a safety comment explaining the two-step deploy process
- **`NOT NULL` without a default** (locks the table on large datasets)

```text {style=github}
⚠️  Migration Safety Check:
   - Missing index on foreign key column(s). Add: create index(:table, [:foreign_key_id])
   - Missing on_delete strategy on references(). Specify on_delete: :delete_all, :nothing, or :nilify_all
```

These are the kind of issues that pass locally, pass in CI, and break in production on a table with a few million rows. The hook catches them before the migration is committed.

### SubagentStart Hook Update

Every time Claude Code spawns a subagent, it now injects condensed rules from all 14 skills. This means code written by subagents follows the same patterns as the main conversation. The auth rules, changeset patterns, and authorization checks all carry through.

## The Full Picture

| Component | v1.4.0 | v2.0.0 | v2.1.0 |
|-----------|--------|--------|--------|
| Skills | 7 | 8 | 14 |
| Hooks | 13 | 14 | 15 |
| Analysis Scripts | 0 | 3 | 3 |
| Agent Docs | 4 | 4 | 4 |

The plugin started with Elixir fundamentals, LiveView, and Ecto. v1.4.0 added OTP and Oban. v2.0.0 added automated code quality detection. v2.1.0 fills the remaining gaps: authentication, authorization, advanced data patterns, and migration safety. If you're building a Phoenix app with Claude Code, the plugin now has guidance for every layer of the stack.

## Update

```bash {style=github}
/plugin
# Select "Marketplaces" → "elixir-phoenix-guide" → "Update"
```

## Try It

GitHub: [elixir-phoenix-guide](https://github.com/j-morgan6/elixir-phoenix-guide)

Requirements: Elixir 1.15+, Phoenix 1.7+
