---
title: "Running the Security Review Plugin on Trays Social"
date: 2026-05-12
draft: false
author: "Joseph"
tags: ["security", "claude-code", "elixir", "phoenix", "ios"]
categories: ["security", "tools"]
description: "Scanning Trays Social end-to-end with the security-review Claude Code plugin. 39 findings, zero critical, and a useful punch list."
---

I pointed the [security-review](https://github.com/cheezy/security-review) Claude Code plugin at [Trays Social](https://github.com/j-morgan6/trays_social) and let it walk the whole repo. 289 files across the Phoenix backend, the iOS client, the Dockerfile, the Fly workflow, and Ecto migrations. It batched the scan into 29 chunks and merged the results.

<!--more-->

## The Numbers

39 findings total.

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 0 |
| Medium | 15 |
| Low | 23 |
| Info | 1 |

No criticals or highs is a relief, but the medium list has real work in it. The plugin is not just pattern-matching for `eval` and `system`. It reads the changeset, follows the controller to the schema, and tells you when a `cast` includes `:user_id` that the update path should never accept.

## The Findings That Mattered

A few stood out as worth fixing before anything else.

**Unhashed API tokens.** `user_token.ex` stores mobile API tokens verbatim. The web session token path already hashes with SHA-256, so the fix is mirroring that for the API branch. One leaked backup or one read replica compromise and every mobile session is a permanent bearer token.

**No rate limiting on the credential endpoints.** Login, magic link, registration, and the mobile auth controller are all unbounded. Hammer is already wired up in the codebase for the confirmation-resend flow. It just needs to be applied to the other four.

**Delete account with no re-auth.** The settings LiveView lets you destroy the account on a `data-confirm` prompt alone. The sibling change-password and change-email actions both require `User.valid_password?`. The delete path skipped it.

**Apple Sign In trusts the wrong email.** The mobile auth controller does `params["email"] || claims["email"]`. The claim is the verified one. A malicious client with its own valid Apple token could bind to a victim's email by passing it in the body.

**Device token IDOR.** The unregister endpoint takes a token from the request and deletes the matching row with no ownership check. Any authenticated user who learns another user's APNs token can silently unregister it.

**Seeds with no environment guard.** `priv/repo/seeds.exs` unconditionally `delete_all`s users and posts and inserts demo accounts with a hardcoded password. No `Mix.env()` check. One accidental `mix run priv/repo/seeds.exs` against prod and the database is gone.

## What I Liked About the Output

The findings cite file and line, the CWE, the OWASP category, and a confidence score. The fix paragraph is concrete: it names the function to mirror, the helper that already exists in the codebase, the exact validator to add. Not "consider adding input validation" but "split into `insert_changeset` and `update_changeset`, set `user_id` once from the session at insert."

The supply-chain and config findings were the easiest wins. Pinning the Fly workflow's actions to commit SHAs, adding `permissions: { contents: read }`, pinning the Dockerfile base images by digest. None of them needed application code changes.

## The Punch List

In rough order:

1. Hash the API tokens, add expiry.
2. Rate-limit the four credential endpoints.
3. Require re-auth on account delete.
4. Trust `claims["email"]` from Apple, verify a nonce.
5. Scope the device-token unregister to the current user.
6. Wrap the seeds script in a `Mix.env() in [:dev, :test]` guard.

The plugin will not open edits without consent, which is the right default for a security tool. It hands you the list and you decide what to act on.

GitHub: [security-review](https://github.com/cheezy/security-review)
