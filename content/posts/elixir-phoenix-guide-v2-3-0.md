---
title: "Elixir Phoenix Guide v2.3.0: Expanded Domains"
date: 2026-03-26
draft: false
tags: ["elixir", "phoenix", "claude-code", "ai-tools", "hooks", "security", "productivity"]
categories: ["tools", "ai"]
description: "Version 2.3.0 adds five new skills covering security, deployment, channels, telemetry, and JSON APIs. Six new security hooks catch vulnerabilities as you write code."
---

## What Changed

Released v2.3.0 of my [Elixir Phoenix Guide plugin](https://github.com/j-morgan6/elixir-phoenix-guide). This release pushes beyond core LiveView and Ecto into five additional Phoenix development domains: security, deployment, channels, telemetry, and JSON APIs. Six new hooks enforce security rules as you write, blocking dangerous patterns before they reach a commit.

<!--more-->

## Five New Skills

Previous versions focused heavily on LiveView, Ecto, and OTP. That coverage was solid, but left gaps when you moved into areas like deploying a release, setting up channels, or building an API endpoint. v2.3.0 fills those gaps.

### security-essentials

Seven rules covering the security issues that show up most often in Phoenix projects: atom table exhaustion from user input, SQL injection through unparameterized fragments, XSS via `raw/1`, open redirects, sensitive data in logs, timing attacks on token comparison, and dependency auditing. Each rule includes the vulnerable pattern and the safe alternative.

### deployment-gotchas

Seven rules for the things that break when you move from `mix phx.server` to a production release. Runtime config with `runtime.exs`, running migrations inside a release, setting `PHX_HOST` and `PHX_SERVER`, deploying assets correctly, managing secrets, health check endpoints, and log level configuration. Most of these only bite you once, but that one time is usually at 2 AM during a deploy.

### phoenix-channels-essentials

Six rules for real-time features: socket authentication, topic-level authorization, the `handle_in`/`push`/`broadcast` patterns, Presence tracking, and testing channel code. If you've ever had a channel silently drop messages because the topic authorization was wrong, this one helps.

### telemetry-essentials

Six rules for observability: structured logging with metadata, attaching telemetry handlers correctly, using Ecto's built-in telemetry events, LiveDashboard integration, and consistent metadata tagging across your application.

### phoenix-json-api

Seven rules for building JSON APIs in Phoenix: the `:api` pipeline setup, FallbackController for error handling, cursor and offset pagination, URL versioning, Bearer token authentication, and using `json/2` for responses.

## Six New Security Hooks

The new hooks work alongside the security-essentials skill. The skill teaches the patterns. The hooks enforce them.

### Blocking Hooks

These three stop Claude from writing code that contains a known vulnerability.

**atom-from-user-input** catches `String.to_atom/1` called on user-controlled data. Atoms are not garbage collected. An attacker who can create arbitrary atoms will eventually crash your VM by exhausting the atom table. The fix is always `String.to_existing_atom/1` or a explicit allowlist.

**unparameterized-sql-fragment** catches string interpolation inside Ecto's `fragment`. This is the Elixir equivalent of building SQL strings by hand. The parameterized form (`fragment("... ? ...", ^value)`) is just as readable and eliminates the injection vector.

**unsafe-redirect** catches redirects where the target URL comes from user input. Without validation, an attacker can craft a link to your site that redirects to a phishing page after login. The hook enforces that redirects use known paths or a validated allowlist.

### Warning Hooks

These three flag patterns that are often (but not always) wrong. They warn instead of blocking because there are legitimate uses.

**raw-html-warning** flags `raw/1` usage in templates. Most of the time you want the HTML escaping that Phoenix provides by default. When you do need `raw/1`, it should be on content you've sanitized yourself.

**sensitive-logging** flags Logger calls that include words like "password", "token", or "secret". Credentials in logs are a common audit finding. Sometimes you're logging a token type or a secret name rather than the value itself, which is why this warns rather than blocks.

**timing-unsafe-compare** flags `==` comparisons involving tokens or secrets. String equality in Erlang short-circuits on the first differing byte, which leaks information about how much of the token matched. `Plug.Crypto.secure_compare/2` runs in constant time.

## The Full Picture

| Component | v2.0.0 | v2.1.0 | v2.2.0 | v2.3.0 |
|-----------|--------|--------|--------|--------|
| Skills | 8 | 14 | 14 | 19 |
| Hooks | 14 | 15 | 21 | 27 |
| Analysis Scripts | 3 | 3 | 4 | 4 |
| Agent Docs | 4 | 4 | 4 | 4 |

The skill count went from 14 to 19. The hook count went from 21 to 27, with the new additions split evenly between blocking (3) and warning (3). The SubagentStart rules were also expanded so that agents dispatched for security, channel, telemetry, deployment, or JSON API work get the right rule sets loaded automatically.

## Update

```bash {style=github}
/plugin
# Select "Marketplaces" → "elixir-phoenix-guide" → "Update"
```

## Try It

GitHub: [elixir-phoenix-guide](https://github.com/j-morgan6/elixir-phoenix-guide)

Requirements: Elixir 1.15+, Phoenix 1.7+
