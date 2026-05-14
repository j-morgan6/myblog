---
title: "Same Code, Three Runs: What --rci 2 Did to the Findings"
date: 2026-05-13
draft: false
author: "Joseph"
tags: ["security", "claude-code", "elixir", "phoenix", "ios", "swift"]
categories: ["security", "tools"]
description: "Two single-pass runs of stride-security-review on the same code disagreed about whether a Critical even existed. A third run with --rci 2 added two critique passes and locked in a stable headline set."
---

A plugin update for [stride-security-review](https://github.com/cheezy/stride-security-review) shipped today, so I ran it against [Trays Social](https://github.com/j-morgan6/trays_social) without changing a line of application code. Then I ran it again. Same plugin version, same files, different counts and, more importantly, different reads on whether there was a Critical at all.

- Run 1 (single-pass): 44 findings, 0 Critical, 2 High.
- Run 2 (single-pass): 40 findings, 1 Critical, 2 High.
- Run 3 (`--rci 2`): 33 findings, 1 Critical, 3 High.

`--rci 2` adds two extra critique passes (the flag is clamped to a max of 3). After that the picture settled.

<!--more-->

## Why The Two Single-Pass Runs Disagreed

The variance is not magic. There are concrete reasons.

**LLM non-determinism.** Each batch is a sub-agent invocation with sampling enabled. Same prompt, similar but not identical output. There is no knob to pin this without changing the harness.

**Batch isolation.** Each agent sees only ten files at a time. The same vulnerability can surface in two different files with two different severities. The Apple-email override is a good example: it lives in `auth_controller.ex:66` (where the param is accepted) and in `accounts.ex:138` (where `maybe_grant_admin` trusts the resulting email). Run 1 flagged it on the controller side as Critical. Run 2 flagged it on the context side as High. Same bug, different file, different severity. Cross-batch dedup keys on `(file, line, vulnerability_class)` and cannot catch this.

**Severity is judgment, not a lookup.** Whether the Apple admin path is Critical or High depends on how the agent weights "attacker has a valid Apple token" as a precondition. Both reads are defensible.

**The realism filter is per-agent.** Each batch independently decides what is too noisy to flag. Run 1 caught the `APIClient` cert-pinning gap, the anonymous resend bucket, and the hardcoded `signing_salt`. Run 2 caught the remember-me cookie missing `Secure`/`HttpOnly`, the Google Fonts SRI gap, the registration rate-limit, the multipart CRLF in `APIClient`, and the iCloud Keychain sync on the bearer token. Each run skipped real findings the other one caught.

**File count drift.** Run 1 reported 289 files, Run 2 reported 290. The difference is `Preview Content/Preview Assets.xcassets/Contents.json`, which has a space in the path. The binary-filter pipeline classified it as binary in one run and text in the other depending on how `xargs` chunked the list. Cosmetic but visible.

The practical takeaway is that a single run is a sample of findings, not a complete audit. Findings that appear in both runs are high-signal. Findings that appear in only one run are worth a manual second look but may be sampling noise.

## What --rci 2 Changed

`--rci 2` runs the analysis once and then critiques the result twice. Each critique pass asks the model to re-read its own findings, drop the ones that do not survive a careful read, and add anything it missed. It costs roughly 2-4x the dispatches of a single-pass run. There are cheaper alternatives: running twice and taking the union (what I did manually for the first two runs) is less convergent but has lower cost, and `--baseline .security-review-baseline.json` freezes a known set so subsequent runs only surface new findings, which is useful for CI gating but not for a first-time audit.

With `--rci 2` the output went from 40 single-pass findings down to 33, and severity calibration tightened up across the document.

Things that got dropped because they were not actually exploitable:

- A `post_controller` mass-assignment finding. The controller does `Map.put(params, "user_id", user.id)` before the changeset, which overrides any client value. The changeset's cast list is irrelevant.
- An `<img src>` `javascript:` XSS. Browsers do not execute `javascript:` URLs from `<img src>`. The pattern looks bad but the sink does not exist.
- A hardcoded `signing_salt`. The salt is paired with `secret_key_base` from env. Salt alone is useless.
- An iOS username path-encoding finding. The username is regex-validated to `[a-zA-Z0-9_]` at the source, so encoding is unnecessary.
- An Apple JWK no-caching finding. DoS only, on the suppression list.

Things that got added that the single-pass run missed or under-counted:

- A multipart CRLF injection in the iOS `APIClient`.
- The iCloud Keychain sync risk on the bearer token, sharper severity.
- `maybe_grant_admin` flagged as a standalone design defect, not just as the sink the Apple bug walks into.
- Two of the three auth-path rate limits (magic-link, registration) that the single-pass run did not call out separately from login.

And severity got resolved more consistently across the document.

## The Critical: Apple Email Override Into Admin Grant

The Apple Sign In handler in `auth_controller.ex:66` resolves the user's email like this:

```elixir
email = params["email"] || claims["email"]
```

The client-supplied request body wins over the Apple-verified JWT claim. That alone is a bug, but the chain is what makes it Critical. The chosen email flows into `find_or_create_apple_user`, which calls `maybe_grant_admin/1` in `accounts.ex:138`. That helper auto-flips `is_admin = true` when the new user's email matches the configured `admin_emails` allowlist.

So an attacker with any free Apple ID and a valid identity token POSTs to `/api/v1/auth/apple` with `email=<admin-allowlist-address>`. A new user is created with the spoofed email and `is_admin=true`. Unauthenticated remote admin takeover with one request.

Two fixes, both needed. First, derive the email exclusively from `claims["email"]` and never let the request body override a verified claim. Second, remove the email-allowlist auto-grant. Bootstrap the first admin from a mix task or `iex -S mix` calling `Accounts.set_admin(user, true)`. Email is not an authentication factor and treating it like one is the actual root cause.

## The Other Highs

**`maybe_grant_admin` as a standalone design defect.** Even after the Apple controller is patched, the helper still treats email as authorization. Any future code path that lets a user influence the email field (a custom signup form, a different OAuth provider, an email-change flow that forgets to re-check) becomes a privilege-escalation vector. The fix is removing the auto-grant entirely, not narrowing the conditions under which it fires.

**`setup-flyctl@master` unpinned in CI.** The Fly deploy workflow references `superfly/flyctl-actions/setup-flyctl@master` on lines 27 and 40. Both jobs run with `FLY_API_TOKEN` for review and production environments. A force-push to that branch or a maintainer-account compromise runs attacker code inside a runner holding the production deploy token. Fix is a 40-character commit SHA pin and Dependabot for the `github-actions` ecosystem.

**API tokens stored raw with no expiry.** `user_token.ex:59` writes mobile API tokens verbatim to `users_tokens.token` (no SHA-256 hashing) and `verify_api_token_query/1` has no `where: token.inserted_at > ago(...)` clause. The same module hashes email tokens correctly, so the safer pattern is right there. Any DB read (backup, replica, an errant `Repo.all` in a debug shell) yields directly usable bearer tokens for every active mobile session, valid until manual logout. Fix is to hash on store, decode-then-hash on verify, and add a 90-day expiry.

## What Got Reclassified

A few findings shifted severity between the single-pass and `--rci 2` runs:

- **`delete_account` with no password re-auth** sat at Medium in both runs. Still worth fixing the same way (current-password input, `User.valid_password?` guard), but the realism filter weights it as a follow-on attack rather than a primary one.
- **Remember-me cookie missing `secure: true`** stayed Low. `config/prod.exs` sets `force_ssl: [rewrite_on: [:x_forwarded_proto]]`, which causes `Plug.SSL` to rewrite `secure` onto `Set-Cookie` headers automatically. The risk is contingent on a future config regression, not on current state.
- **`register_device` hijack** stayed Medium because APNS tokens are 64 hex characters and not guessable. The IDOR is real, but the precondition narrows realistic impact.
- **Dockerfile pinned by tag instead of digest** moved up to Medium under `--rci 2` because the file is production-bound (`MIX_ENV=prod`, `mix release`, the release `CMD`). Supply-chain rules deserve more weight when the artifact lands in prod.

I would still ship fixes for all of these. The reclassifications are about how to prioritize, not whether to act.

## The iOS Side

Three iOS findings worth pulling out, since the original scans had not focused there.

**Bearer token syncs to iCloud Keychain.** `KeychainService.swift:23` saves the API bearer token with `kSecAttrAccessibleAfterFirstUnlock`. The accessibility class is missing the `ThisDeviceOnly` qualifier, so the entry is included in encrypted iTunes/Finder backups and Quick Start device migration. A token issued on iPhone can be restored onto iPad. The biometric credential storage on line 71 in the same file correctly uses `WhenUnlockedThisDeviceOnly`, so this is an internal inconsistency more than a missed concept. One constant change: `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`.

**Raw password under the biometric gate.** `KeychainService.swift:66` keeps the user's actual password in the keychain behind a biometric ACL. The gate is fine, but storing the raw password is the wrong primitive. If the entry is ever extracted, the attacker recovers a reusable credential. Migrate to a server-issued refresh token bound to the device, stored under the same ACL, refreshed on biometric unlock.

**CRLF risk in the multipart upload.** `APIClient.swift:75` interpolates a caller-supplied filename into the `Content-Disposition` header without sanitizing CR or LF. Current callers pass internally generated filenames, so this is not exploitable today. It is on the list because the helper is the kind of utility that gets reused, and the next caller might pass a `PHPickerResult` original name or clipboard paste. Strip CR, LF, and embedded quotes, or generate a fixed safe local name.

## What I Trust Now

The high-signal set is the union of findings that appeared in both single-pass runs, regardless of whether each run agreed on severity. That set includes the Apple email-override chain (whether it was flagged on the controller or the context), `delete_account` with no re-auth, `setup-flyctl@master` unpinned, the missing Apple nonce, API tokens stored unhashed with no expiry, the device-controller unregister IDOR, `Post.changeset` mass-assignment of `user_id`, the auth-plug Base64 handling, the missing login rate limit, `health_controller` leaking `inspect(reason)`, photo upload extension-only validation, the seed script's hardcoded password, the CSP `unsafe-inline` style and missing directives, and the session cookie not explicitly `secure`/`http_only`.

`--rci 2` then promoted the Apple controller-side finding to the canonical Critical, added `maybe_grant_admin` as a standalone High, kept the `setup-flyctl` and API-token Highs, and dropped a stack of single-pass items that did not survive a careful re-read.

Top of the fix list:

1. Apple email override into admin grant (the chained Critical)
2. `maybe_grant_admin` allowlist auto-grant
3. API tokens raw with no expiry
4. `setup-flyctl@master` unpinned

After that, the auth-path rate limits, the iCloud Keychain accessibility class, and the `delete_account` re-auth are next.

`--rci 2` is not magic. Run it twice and you will still get some variance. But the convergence is real, and the items that survive two extra critique passes are the items I am willing to spend engineering time on.

GitHub: [stride-security-review](https://github.com/cheezy/stride-security-review)
