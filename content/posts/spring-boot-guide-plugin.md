---
title: "spring-boot-guide: Spring Boot Enforced by Hooks"
date: 2026-09-25
draft: false
author: "Joseph"
tags: ["spring-boot", "java", "jpa", "claude-code", "ai-tools", "hooks", "productivity"]
categories: ["tools", "ai"]
description: "A Claude Code plugin that enforces modern Spring Boot through 39 deterministic hook rules, 10 skills, 3 review agents, and a mutation gate. Plus the backend I built under it, which found a hole in the enforcement surface of both this plugin and its Angular sibling."
---

Released v1.0.0 of [spring-boot-guide](https://github.com/j-morgan6/spring-boot-guide), a Claude Code plugin for modern Spring Boot. Ten skills, thirty-nine rules wired into native hooks, three read-only review agents, and a mutation gate that fails the build when a rule stops enforcing anything. Then I built a real Spring Boot 4.1 backend under it, inside [the same showcase repo](https://github.com/j-morgan6/angular-guide-showcase) that validated the Angular plugin, to see what the rules caught.

<!--more-->

## Why Spring Needs This More Than Angular Did

When I wrote [angular-guide](/posts/angular-guide-plugin/), the argument for it was that Angular already ships good agent material and what was missing was enforcement. Spring does not have that problem, because Spring ships nothing. Verified on 2026-09-17: `docs.spring.io/llms.txt` returns 404, and so does `docs.spring.io/spring-boot/llms.txt`. There is no official Spring agent skill.

There is Spring AI material titled "Agent Skills", which is easy to mistake for the real thing. It is about building skills into a Spring application, the reverse direction. Nothing official covers an agent writing Spring.

So this plugin's skills do what angular-guide's deliberately avoided: they are the reference, not a pointer to one. And the enforcement argument still holds on top of that. A skill is advisory, the model has to elect to load it, and a `Task`-dispatched subagent inherits no skills at all. The hooks here run on every `Write`, `Edit`, and `Bash` call regardless of which skill loaded, and a `SubagentStart` hook injects the condensed blocking set into every subagent.

## The Ten Skills

| Skill | Covers |
|---|---|
| `spring-boot-essentials` | The reasoning behind every rule, since there is no upstream reference to defer to |
| `transactions` | How the proxy actually works, why a non-public `@Transactional` silently does nothing, where the boundary belongs |
| `web-layer` | Composed mapping annotations, DTOs at the boundary, controller slices |
| `data-jpa` | `equals`/`hashCode` on the business key, lazy by default, fetching with `@EntityGraph` |
| `configuration-and-profiles` | `@ConfigurationProperties` records over scattered `@Value`, profile layering, externalized secrets |
| `security-essentials` | The lambda DSL `SecurityFilterChain`, password encoders, matcher ordering |
| `testing-essentials` | JUnit 5, slices over `@SpringBootTest`, Awaitility over `Thread.sleep` |
| `http-clients` | `RestClient` and `@HttpExchange` interface clients, `RestTemplate` as maintenance-mode |
| `project-structure` | Package by feature, not by layer |
| `upgrading-to-boot-4` | The Boot 3.5 to 4.0 to 4.1 span, and which gated rules fire on which generation |

## The Thirty-Nine Rules

Plain bash and python3, no `jq`, no install script, nothing written to `~/.claude/settings.json`. Hooks come through the plugin's own `hooks/hooks.json` using `${CLAUDE_PLUGIN_ROOT}`.

### BG001 to BG006: the Bash guard

A forced push whose destination resolves to the default branch, a bare `mvn` or `gradle` when the workspace has a wrapper, a Gradle command in a Maven workspace or the reverse, a run command that never exits and hangs the session, an `rm -r` that takes the build wrapper along with it, and a destructive `ddl-auto` value passed on the command line.

### SB001 to SB020: blocking checks

These run on `PreToolUse` against the effective post-edit content, so for an `Edit` the rule sees the on-disk file with `old_string` already swapped for `new_string`. A violation introduced into existing context gets caught before it lands. Exit 2 blocks the write.

The set covers dependency injection (field `@Autowired`, a redundant `@Autowired` on a sole constructor, injected fields that are not `final`), transactions (`@Transactional` on a non-public method, `@Transactional` in a controller), JPA (`FetchType.EAGER` inside a relationship annotation, Lombok `@Data` on an entity, a destructive `ddl-auto` in main config, a `@Query` string built by concatenation), security (`NoOpPasswordEncoder`, a permit-all `anyRequest()`, a secret-shaped config key assigned a literal, a wildcard actuator exposure), platform migration (`javax.` imports of Jakarta-moved packages, `com.fasterxml.jackson.` under Jackson 3), and hygiene (`System.out` logging, an empty catch block, JUnit 4 imports, `Thread.sleep` in tests).

The interesting part is not the list, it is how much narrowing each rule got. Fourteen-plus rules were tightened during implementation review specifically to stop them firing on correct code:

- **SB005** only flags `FetchType.EAGER` inside the argument list of a relationship annotation, not a bare `FetchType.EAGER` token elsewhere in the file, because an ArchUnit governance test asserting "no entity uses eager fetch" legitimately contains that token.
- **SB003, SB004, and SB006** refuse to evaluate any file declaring more than one type. Flat parsing cannot attribute an annotation to the right owner, and a missed finding is the safer failure mode.
- **SB014** excludes `javax.annotation.processing`, the JDK compiler API that never moved to Jakarta.
- **SB020** excludes `com.fasterxml.jackson.annotation`, which Jackson 3 deliberately keeps so the same annotations work against 2.x and 3.x, and stands down entirely when `spring.jackson.use-jackson2-defaults=true` is present.
- **SB015** flags console forms only. A stack trace rendered into a `StringWriter` is not console output and does not fire.

### SB101 to SB113: advisory checks

These run on `PostToolUse` against the file on disk. The write has happened, the finding is shown to the model to act on. They cover `@Transactional` self-invocation, `open-in-view`, `RestTemplate` usage, disabled CSRF, wildcard CORS origins, non-final injected fields, more than five `@Value` annotations in a file, a hand-built `ObjectMapper` outside a `@Bean` method, `@SpringBootTest` in a controller test, an entity crossing the web boundary, an N+1 shape, bare to-one associations with no explicit fetch, and structural complexity.

The reason the highest-value rule in the set is advisory rather than blocking is worth stating: SB101 is the only rule needing real structural analysis, so it is the likeliest to misfire, and blocking correct code is what gets an enforcement plugin uninstalled.

## Version Awareness

`detect_project.sh` runs on `SessionStart` and caches a profile at `.spring-boot-guide-project.json`, walking up to two directory levels to find a nested manifest like `backend/pom.xml` in a monorepo. The profile carries `boot_major`, `boot_minor`, `jackson_major`, `build_tool`, `wrapper`, `starters`, `lombok`, `db`, and `migrations`.

Gating works two ways. A firing gate means a rule simply does not run when the profile does not satisfy it, like SB006 requiring Lombok. A severity gate keys the severity rather than the firing condition, and SB019 is the only one: `@MockBean` is blocking on Boot 4 where it was removed, advisory on Boot 3 where it is merely deprecated.

The floor under both is a never-guess rule. When `boot_version` cannot be resolved, from an unresolvable Maven property or a version range or a snapshot, every gated rule is skipped rather than assumed. And a profile that cannot be found at all silences every rule including the ungated ones, because the plugin should be invisible in a repository it has no business linting.

## The Mutation Gate

This is the piece angular-guide does not have, and it exists because of a specific angular-guide failure. That plugin shipped four rules that installed cleanly, passed their fixtures, and enforced nothing. A dead rule is invisible to a green suite, because all of its negative fixtures pass trivially when the detector is a no-op.

So for each of the 39 rules here, the gate neutralises that rule's detector by inserting `check_sbNNN() { return 0; }` at a sentinel in the script, then asserts the rule's own fixtures stop passing. A fixture that still passes with its detector disabled was never testing the detector, and the build fails.

It caught two rules before release. SB105 was matching against the wrong lexer view and could never see the string literal it needed. SB113 piped its input into `python3 -` alongside a heredoc, both targeting stdin, so the source it was supposed to analyze silently vanished into the program text. Both would have shipped green.

Every rule list the suite consumes is derived from the scripts by `grep` rather than hardcoded, so a rule added without a fixture or without documentation fails the build instead of being quietly skipped. The suite also runs under both the BSD `mktemp` macOS ships and a GNU-mimicking stub, because that difference has silently disabled an entire sibling plugin on Linux before.

## Three Review Agents

Hooks see one file at a time at the moment of writing, which is the wrong vantage point for anything structural. So there are three read-only agents (`Read`, `Grep`, `Glob`, no write tools) meant to be dispatched after a feature is finished:

- `spring-architecture-review` for layer boundaries.
- `jpa-review` for entity mappings, fetch strategy, and N+1 shapes across files.
- `spring-project-structure` for layout and package organisation.

## Building Something Under It

The plugin's own README shipped with a limitation that said the quiet part out loud: it had never been run against a real Spring Boot codebase, every assertion ran against synthetic fixtures, and false positives on real code remained the failure mode most likely to be hiding.

So the showcase app grew a backend. [angular-guide-showcase](https://github.com/j-morgan6/angular-guide-showcase) is now two halves: the Angular v22 dashboard that validated angular-guide, and a Spring Boot 4.1 service behind it that syncs both plugin repositories from the GitHub REST API into Postgres and serves them through a read-only API. The front end stopped calling GitHub directly and reads only from the backend, which also removed the browser rate limit the Angular version worked around. A route-driven switcher surfaces both plugins in the UI.

Thirty-six backend tests across ten test classes, using Testcontainers against a real `postgres:16-alpine`, plus a Flyway baseline migration. Fifty-two front-end tests across eight files and a clean production bundle at 297 kB initial with every route still in its own lazy chunk. The repo is deployable but not deployed, because the built front end expects `/api` on its own origin and nothing here is hosted.

The actual deliverable is [`docs/spring-plugin-findings.md`](https://github.com/j-morgan6/angular-guide-showcase/blob/master/docs/spring-plugin-findings.md), one entry per hook firing with the code, verbatim output, a verdict, and an action.

## The Headline Finding Was Not a Rule

Bash writes bypass every hook in both plugins.

Both plugins scope every content rule to the `Write|Edit` tool matcher. `Bash` is matched only by `bash-guard.sh`, which inspects the shell command text and has no access to what a command writes to disk. A file created or overwritten through a heredoc, `sed`, `cp`, or a script is therefore never linted, in either direction, no matter what it contains.

That is reachable by default, not by evasion. This harness's own auto-mode guidance instructs agents to prefer Bash heredocs over the `Write` and `Edit` tools for file changes. Following that generic advice in a Spring Boot or Angular workspace makes the bypass the default code path.

The blast radius was measured rather than asserted. Tasks 2 through 4 authored every backend file through Bash heredocs, which means all 33 `Write|Edit`-scoped rules were inert for three whole tasks. A re-verification sweep rewrote 15 files through the `Write` tool with byte-identical content, and that sweep was the first time any hook had evaluated them. No blocking rule fired and no advisory appeared, `git status` was empty afterward, and the test suite stayed green. So nothing was actually violating a rule. But the earlier "no hooks fired" entries in the log meant never evaluated, not evaluated and clean, and the log had to be corrected to say so.

A `PreToolUse` hook cannot fix this, because it runs before the command and cannot know what an arbitrary shell command will write. The fix is a `PostToolUse` hook matched on `Bash` that lints whatever the command touched. Until that exists, `Write` and `Edit` are required rather than preferred for any file a rule can apply to. That is why this post was written through the `Write` tool.

## What the Rules Got Right and Wrong

Seven firings across the build: four true positives, three false positives, and zero noise.

**SB101 earned its keep in the one circumstance a static rule can.** A `@Transactional` self-invocation was planted in `SyncService.syncAll()`, calling `this.syncRepo(...)` so the call never passes through the proxy and the transaction never starts. All four tests passed with the defect live. Nothing else in the project would have found it.

**SB113 flagged an eight-parameter constructor**, and splitting the bean to resolve SB101 fixed it as a side effect. Worth noting that the new bean landed at six of seven parameters, so the two rules interact.

**Zero noise is a result, not a blank.** Nothing fired that was technically correct but not worth the interruption. Every firing was either a genuine catch or a genuine defect in the rule. This rule set does not chatter.

**All three false positives were one bug, and it is the worst kind.** SB106 checks for injected fields that are not `final`, using a regex that matches the field name followed by `;` or `=` on a line. A `package com.jmorgan.showcase.github;` line matches when the field is named `github`. It fired three times, on `github` in package `github`, `catalog` in `catalog`, and `activity` in `activity`. Two negative controls confirmed the mechanism: fields named `plugins` and `rules` in the `catalog` package did not fire, because the name has to exactly equal the package's last segment.

The sharp part is what that collides with. Feature-first packaging is what the plugin's own `project-structure` skill mandates, and a field named after the feature it belongs to is exactly the naming that skill pushes you toward. Follow the plugin's structural advice and you get false blocks. SB110 has the same shape in reverse: it is keyed to a layered `.entity.` package layout that the same skill steers you away from, so following the advice turns the rule off. Two of the three most serious findings are the plugin disagreeing with itself, and neither would ever surface in a fixture suite, because fixtures are written rule by rule and this is an interaction between a rule and a skill.

**Two rules should have fired and did not, and an agent caught both.** SB111 missed a deliberately planted double N+1 in `FindingService.toDto`, for two independent reasons: there is no `for` loop in the file, it is a `.stream().map()` pipeline, and the terminal call is a scalar getter on a to-one proxy rather than a collection walk. `jpa-review`, dispatched against the identical unmodified tree, caught it, identified it as a double N+1 spanning three files, and located the unbounded endpoint that made it matter.

SB110 missed a `SyncRun` entity crossing into `SyncController` as a method parameter type, also for two reasons, and only one of them was documented. The package is `github`, not `entity`, so the naming heuristic misses it. But the entity also sits in the same package as the controller, so there is no import line for the regex to find at all, which defeats the rule regardless of naming convention. `spring-architecture-review` found it and named the rule's blind spot unprompted.

**The literal-stripping architecture prevented a whole class of false positive.** The lexer blanks string-literal content by default and a rule opts in when it genuinely needs to see inside the quotes, which only SB105 does. Two linted test files contain `"FetchType.EAGER"` and a `@RequestMapping(method = ...)` string as literal test data, and neither SB005 nor SB009 fired. The sibling angular-guide plugin, in the same workspace, produced three false positives on exactly that shape because it does not strip literals. Same hazard, two architectures, measurably different outcomes.

## Where the Config Rules Stop Looking

SB007, SB012, and SB013 all die at a script-level gate that exits the entire hook process for any `.properties` or `.yml` file whose basename does not start with `application`. So `src/main/resources/db-secrets.yml` containing a plaintext password gets zero coverage from all three, despite sitting in Spring's own config root. The blind spot is naming-based, not location-based, which means adding directories to the path check does not help, because that check is never reached.

SB012 also has a documented trade-off worth repeating: it matches only the final dot-segment of a key, exactly rather than as a substring, so `app.jwt-secret` and `app.auth-token` are not flagged. Key name alone cannot separate a real `auth-token` from a pagination `next-token`.

## Running Both Plugins Together

Coexistence was safe, with one real collision and one cosmetic one.

Both `SessionStart` detectors wrote their own dotfile at the same repo root, side by side and mutually invisible. Both linters gate on file extension within ten lines of reading the path, and the sets are disjoint, so no NG rule ever fired on a `.java` or `.yml` file and no SB rule ever fired on a `.ts` or `.html` file. Both resolve their profile by walking up from the edited file rather than from `$PWD`, so editing a backend file from a frontend directory resolves correctly.

The Bash guards are the collision. Neither gates on file type, because there is no file to gate on, so both inspect every Bash command in the repository whatever you are working on. angular-guide's BG004 blocked a command issued while working on the Spring backend. And both plugins number their guards `BG001` through `BG006` with entirely different meanings, so a reader of a transcript cannot tell which plugin spoke without reading the message body. That caused a real attribution problem while writing the findings doc.

## The Caveat That Limits All of It

The code was written by implementers following a plan I authored with the rule set open in front of me, and briefs handed implementers verbatim Java. So the code largely pre-complies by construction, and a rule staying silent on it is weak evidence that it would stay silent on code written without the rules in view. That structurally limits false-positive discovery, which is the exact failure mode the plugin's README named as most likely to be hiding. "Thirty-one of thirty-nine rules could not fire because the pattern never occurred" is partly a fact about the plan, not only about the codebase.

Three things did generate code the rules had not been consulted about, and every genuine result came from one of them: two deliberately planted defects, eight Boot 4.1 API relocations the plan got wrong and implementers had to work around, and emergent structure nobody designed. All three false positives landed on field names the plan never chose deliberately.

The one class of result this method is good at is rules that should have fired and did not. A miss does not care whether the code was pre-complied. The defect was there, the rule ran, and it said nothing. That is also precisely what the mutation gate structurally cannot produce, because a mutation test asks whether disabling a detector breaks its own fixture, and a fixture is by definition a case the author already thought of.

## Hooks or Agents

The two planted defects gave opposite answers. One was caught by a hook, one was missed by a hook and caught by an agent. That reads like an argument for agents, and there is a counterweight that cuts the other way.

The best single finding of the final review round was that every entity's `equals` reads `that.field` directly instead of `that.getField()`, which silently returns false against an uninitialized lazy proxy, in a codebase where every association is lazy. That defect was written in Task 4, and Task 4's own `jpa-review` run looked at those exact files and reported the equality implementations as well-formed. I read the report and agreed with it. Two independent readers, agent and human, ratified a real defect as clean, and it took a third look eight tasks later to catch it.

So the conclusion is not that agents beat hooks. Hooks give you a fast, deterministic, always-on floor with known blind spots. Agents give you cross-file reasoning with non-deterministic recall. Each covers the other's failure mode and neither covers its own. Ship both, and do not let a clean agent report retire a rule.

## Headed for v1.1

Three items at P0: a `PostToolUse` hook on `Bash` so files written through the shell get linted, the SB106 regex fix so it stops reading a `package` line as a field declaration, and SB110 detecting a type annotated `@Entity` semantically rather than a type imported from a package named `entity`.

At P1: widen the config-rule basename gate, change SB111's message to say what it cannot see and name `jpa-review` as the required backstop rather than reading as full N+1 coverage, and ship the three agents as a required part of the workflow rather than an adjunct.

Four rules get explicitly no change, and that is a result too. SB019, SB020, SB103, and SB102 all ran with their gates satisfied against code that genuinely contained their trigger surface, and all four correctly stayed silent. Those are the rules the whole Boot 4 technology choice was made to exercise, and they behaved.

The README limitation saying the plugin has never been run against a real codebase can also retire now. It gets replaced by what this run actually found, including the methodology caveat, so the next reader does not overread a low false-positive count.

## Install

```bash {style=github}
/plugin marketplace add j-morgan6/spring-boot-guide
```

Then install `spring-boot-guide` through Claude Code's plugin manager.

Requirements: Spring Boot 3.5 or later, Java only, bash 3.2+, and python3. No `jq`. Kotlin is unsupported and `.kt` files are not read at all, which is deliberate: partial support would be worse than none.

## Try It

GitHub: [spring-boot-guide](https://github.com/j-morgan6/spring-boot-guide)

Showcase: [angular-guide-showcase](https://github.com/j-morgan6/angular-guide-showcase) | [spring findings](https://github.com/j-morgan6/angular-guide-showcase/blob/master/docs/spring-plugin-findings.md) | [angular findings](https://github.com/j-morgan6/angular-guide-showcase/blob/master/docs/plugin-findings.md)
