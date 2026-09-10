---
title: "angular-guide: Modern Angular Enforced by Hooks"
date: 2026-09-10
draft: false
author: "Joseph"
tags: ["angular", "typescript", "claude-code", "ai-tools", "hooks", "signals", "productivity"]
categories: ["tools", "ai"]
description: "A Claude Code plugin that enforces Angular v20+ patterns through 30 deterministic hook rules, 9 skills, and 3 review agents. Plus the showcase app I built under it to find out whether any of it works."
---

Released v1.0.0 of [angular-guide](https://github.com/j-morgan6/angular-guide), a Claude Code plugin for modern Angular. Nine skills, thirty rules wired into native hooks, and three read-only review agents. Then I built [a real app under it](https://github.com/j-morgan6/angular-guide-showcase) to see what the rules actually caught.

<!--more-->

## Why Hooks and Not Just Advice

Angular already ships good material for agents. There is the official [angular-developer skill](https://github.com/angular/angular/tree/main/skills/dev-skills/angular-developer), a CLI MCP server (`npx @angular/cli mcp`), and a model-readable reference at [angular.dev/llms.txt](https://angular.dev/llms.txt). This plugin is built to sit alongside all three, not replace them. Its skills deliberately do not restate Angular's reference docs. Each one says so and points back to those sources.

The gap it fills is enforcement. A skill is advisory: the model has to elect to load it, and nothing stops it from writing `*ngIf` anyway when it is halfway through a refactor and reaching for whatever pattern it saw most during training. Worse, subagents inherit no skills at all. A `Task`-dispatched subagent starts with none of the context you carefully set up in the parent session.

angular-guide's rules run on every `Write`, `Edit`, and `Bash` call regardless of which skill the model loaded, and regardless of whether the call came from the main session or a subagent. A `SubagentStart` hook injects the condensed blocking rule set into every subagent's context, and the `PreToolUse` hooks still run underneath it either way.

## The Nine Skills

Each skill loads based on the file being edited.

| Skill | Triggers on | Covers |
|---|---|---|
| `angular-essentials` | any `.ts` / `.html` | Standalone by default, no `@NgModule`, native control flow, `inject()`, signal `input()`/`output()`/`model()`, no `any`, the `host` object |
| `signals-essentials` | any `.ts` | Choosing `computed()` vs `linkedSignal()` vs `effect()`, effects never writing signals, the `resource()` family for async state |
| `rxjs-interop` | any `.ts` | When RxJS is still the right tool, teardown on every manual `.subscribe()`, converting at the boundary with `toSignal()` |
| `component-architecture` | `*.component.ts`, `.html` | Keeping HTTP and validation out of components, presentational vs container, the size ceiling |
| `state-management` | `*.store.ts`, `*.service.ts` | The escalation ladder from component state to a service with signals to `@ngrx/signals`, matched to actual complexity |
| `data-loading` | `*.service.ts`, any `.ts` | `httpResource()` for signal-keyed reads vs `HttpClient` for commands, functional interceptors, error handling at the right boundary |
| `testing-essentials` | `*.spec.ts` | Vitest as the default runner, never Jasmine |
| `performance-and-zoneless` | any `.ts` / `.html` | No explicit `OnPush` on v22+, no `zone.js` in a zoneless workspace, mandatory `@for` `track`, lazy routes, `NgOptimizedImage` |
| `project-structure` | any `.ts`, `angular.json` | Hyphenated intent-named files, feature-first directories |

## The Thirty Rules

The rules are plain bash and python3. No `jq`, no install script, nothing written to `~/.claude/settings.json`. Hooks are delivered through the plugin's own `hooks/hooks.json` using `${CLAUDE_PLUGIN_ROOT}`, so installing and removing the plugin is the entire story.

### BG001 to BG006: the Bash guard

Runs before any Bash command. It catches a forced push that resolves to the default branch (checking the destination side of a `src:dst` refspec, so `main:feature/x` is left alone), the `ng build --prod` flag that was removed in Angular 12, a package manager that disagrees with the workspace's lockfile, an `ng test` invocation with no flag to prevent watch mode, an `rm -rf` that takes a lockfile along with `node_modules`, and `ng update --force`.

### NG001 to NG018: blocking checks

These run on `PreToolUse` for `Write` and `Edit`, against the effective post-edit content. For a `Write` that is the pending content; for an `Edit` it is the on-disk file with `old_string` already swapped for `new_string`. A violation introduced into existing context gets caught before it lands, not after. Comments are stripped first, including inside template literals.

The blocking set covers legacy idioms (`@NgModule`, `*ngIf`/`*ngFor`, constructor injection, decorator `@Input()`/`@Output()`, explicit `OnPush` on v22+), reactivity misuse (`.mutate()`, effects that write signals, a `@for` with no `track`), and security and hygiene (`[innerHTML]`, `bypassSecurityTrust*`, a secret-shaped key assigned a literal in `environment*.ts`, stray `zone.js` in a zoneless workspace, Jasmine in a Vitest workspace).

A few of them do more work than a grep. NG004 only flags constructor-parameter DI inside a file that already carries an Angular class decorator, so a plain value class with `constructor(private amount: number)` is left alone. NG013 extracts the `@for` clause by counting parens rather than matching to the first `)`, so `@for (x of items(); track x.id)` is correctly not flagged. NG012 does the same balanced-paren extraction on an effect body so it can span nested calls.

### NG101 to NG106: advisory checks

These run on `PostToolUse` against the file already on disk. The write has happened; the finding is shown to the model to act on. They cover eager route components that should be `loadComponent`, Reactive Forms where Signal Forms are available, `@Injectable({ providedIn: 'root' })` where `@Service()` is available, un-optimized `<img>` tags, direct `document`/`window` access in a component, and structural complexity in a component file (an `HttpClient` reference, more than ~200 lines, more than ~8 inputs).

## Version Awareness

A `SessionStart` hook walks up for `angular.json`, reads the Angular version from `package.json` or a lockfile, and caches a profile at `.angular-guide-project.json`. Every version-sensitive rule is gated on a field in that profile.

If the version cannot be resolved, `angular_major` is `"unknown"` and every gate derived from it is `false`, so NG001, NG002, NG006, NG013, NG102, and NG103 stay quiet instead of firing on a guess. Same for `zoneless` (NG017) and `test_runner` (NG018). Each rule is gated on the specific field the profile reports.

## Three Review Agents

Hooks see one file at a time, at the moment of writing. That is the wrong vantage point for anything structural, so there are also three read-only agents (`Read`, `Grep`, `Glob`, no write tools) meant to be dispatched after a feature is finished:

- `angular-architecture-review` for component and service boundaries, catching logic that landed in the wrong layer.
- `signals-review` for `signal()`, `computed()`, `linkedSignal()`, `effect()`, and `resource()` usage.
- `angular-project-structure` for file layout and naming against the current style guide.

## Building Something Under It

Writing rules is easy. Finding out whether they hold up on real code is the harder part, so I built [angular-guide-showcase](https://github.com/j-morgan6/angular-guide-showcase): an Angular v22 dashboard that displays the plugin's own rules and skills, pulled live from the plugin's GitHub repo.

Three lazily loaded routes, about 2,500 lines of source. **Rules** parses the plugin's README into structured rule cards. **Skills** renders the nine skill documents from real markdown. **Activity** shows commits and contributors. Everything reads the GitHub REST API from the browser with no token, which caps it at 60 requests per hour, so resources are created once at root scope and reused across navigation, and a rate-limited response renders a distinct error state with the reset time rather than a generic failure.

It runs at [j-morgan6.github.io/angular-guide-showcase](https://j-morgan6.github.io/angular-guide-showcase/). But the actual deliverable is [`docs/plugin-findings.md`](https://github.com/j-morgan6/angular-guide-showcase/blob/master/docs/plugin-findings.md), a log of every rule that fired while the app was being built, every rule that stayed silent, and a verdict on whether the plugin was worth having.

## What It Got Right

**NG103 changed real code with zero argument.** The task brief specified a `GithubApi` service with `providedIn: 'root'`. The hook flagged it and pointed at `@Service()`, which Angular v22 ships as the same root-provided singleton without the nested options object. I checked it against `@angular/core`'s own type definitions rather than taking the hook's word for it, and it was right. One-line swap, one import dropped.

**NG014 was the interesting one.** Blocking `[innerHTML]` and `bypassSecurityTrust*` outright sounds like the kind of rule that is correct in principle and unusable in practice. The showcase renders nine markdown documents, so this was a direct test of that.

The rejected shortcut is about five lines: one `<div [innerHTML]>` bound to `sanitizer.bypassSecurityTrustHtml(marked.parse(md))`. What the ban forced instead was four small components, 282 lines, composing `marked`'s token tree into real DOM with no HTML string built anywhere. Against the nine skill documents that is 982 block tokens with zero unhandled types, covered by 15 tests in 128 lines.

That is a 55x line multiplier, which is a real cost and worth saying plainly. But every one of those lines is a small statically-typed unit, and when the design did produce a bug (a silent drop in nested list items) it was findable and fixable in a few lines precisely because the contract said never drop, always degrade visibly. The verdict in the findings doc is that the blocking rule is workable, and that unsanitized HTML never had a path into this codebase at all.

**Most of the rules never said anything.** Twenty-six of the thirty never fired across nine implementation tasks. Some of that is expected: nobody writing Angular in 2026 is reaching for `@NgModule` or `.mutate()`. But the most template-heavy task in the project, four components and 282 lines landing squarely in NG003, NG005, NG008, NG009, NG013, and NG014 territory, produced zero firings. That silence is a result, not an absence of one. A blocking ruleset that has nothing to say during idiomatic work is doing its job.

## Known Issues, Headed for v1.1

Four rules fired at all, seven times between them, and the log is honest about which of those were the plugin's fault. Three items came out of it:

**`is_spec()` is called by 4 of 24 rules.** The plugin already has a helper that recognizes a `.spec.ts` file, in explicit acknowledgement that test files legitimately contain code-shaped text as data. Only NG010, NG102, NG105, and NG106 call it. So a test fixture quoting a rule's own trigger text as sample data gets blocked by that rule, which happened three times while writing fixtures for the Rules page. The fix is to call the existing helper, or strip string and template-literal contents before matching.

**The Bash guard reads command text with no awareness of quoting.** BG004 blocks `ng test --help`, which prints an option list and exits and cannot hang anything. It also blocks any command whose text merely contains `ng test`, including a git commit message describing the flag. BG001 has the same shape: writing this post through a shell heredoc tripped it, because the paragraph above describing what BG001 catches contained the phrase it looks for. And BG004's suggested remedy recommends `--run`, which the v22 Vitest builder does not accept. Exclude the `--help`, `-h`, and `--version` forms, recommend `--watch=false` unconditionally, and teach both guards where a quoted string starts.

**`testing-essentials` is silent on testing the async primitives the other skills mandate.** This is the one worth fixing first. `performance-and-zoneless` requires `@defer`, `data-loading` recommends `httpResource()`, and nothing in the testing skill covers driving either to completion under TestBed. The skills teach how to write the pattern but not how to prove it works, and that gap cost more real defects on this project than every hook firing combined. It needs a section on `TestBed.tick()` and microtask flushing for resources, and one on `deferBlockBehavior: Manual` and `getDeferBlocks()` for defer blocks.

None of these indict the underlying idea. They are an `is_spec()` call, an argument exclusion, a corrected remedy string, and two missing skill sections.

## Install

```bash {style=github}
/plugin marketplace add j-morgan6/angular-guide
```

Then install `angular-guide` through Claude Code's plugin manager.

Requirements: Angular v20 or later (some rules are gated further to v21/v22), bash 3.2+, and python3. No `jq`. The test suite is 130 checks and includes drift checks that fail the build if a skill cites a rule ID no script implements, or if the README stops documenting a rule that exists.

## Try It

GitHub: [angular-guide](https://github.com/j-morgan6/angular-guide)

Showcase: [angular-guide-showcase](https://github.com/j-morgan6/angular-guide-showcase) | [live](https://j-morgan6.github.io/angular-guide-showcase/) | [findings](https://github.com/j-morgan6/angular-guide-showcase/blob/master/docs/plugin-findings.md)
