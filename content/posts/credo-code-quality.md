---
title: "Using Credo to Ensure Code Quality in Elixir"
date: 2025-10-03T17:00:00-04:00
draft: false
author: "Joseph"
categories: ["Elixir", "Code Quality"]
tags: ["credo", "elixir", "linting", "best-practices"]
---

## Why Code Quality Matters

Maintaining code quality is essential for any project that aims to be maintainable and scalable. In the Elixir ecosystem, Credo is the go-to static code analysis tool that helps enforce consistency and catch potential issues before they become problems.

Credo analyzes your code for readability, refactoring opportunities, software design suggestions, and common mistakes. It's like having an experienced Elixir developer review your code automatically.

<!--more-->

## Adding Credo to Your Project

First, add Credo as a dependency in your `mix.exs` file:

{{< highlight elixir "style=monokai" >}}
{:credo, "~> 1.7", only: [:dev, :test], runtime: false}
{{< /highlight >}}

Then run `mix deps.get` to install it.

## Configuring Credo

Credo is highly configurable. Here's my `.credo.exs` configuration file:

{{< highlight elixir "style=monokai" >}}
%{
  configs: [
    %{
      name: "default",
      strict: true,
      checks: [
        # Additional and reconfigured checks
        {Credo.Check.Design.AliasUsage,
          if_nested_deeper_than: 3,
          if_called_more_often_than: 1,
          files: %{excluded: ["test/support/data_case.ex"]}},
        {Credo.Check.Readability.AliasAs, []},
        {Credo.Check.Readability.MultiAlias, []},
        {Credo.Check.Readability.NestedFunctionCalls, []},
        {Credo.Check.Readability.SeparateAliasRequire, []},
        {Credo.Check.Readability.StrictModuleLayout, []},
        {Credo.Check.Readability.WithCustomTaggedTuple, []},
        {Credo.Check.Refactor.ABCSize, []},
        {Credo.Check.Warning.UnsafeToAtom, []},

        # Disabled checks
        {Credo.Check.Design.TagFIXME, false},
        {Credo.Check.Design.TagTODO, false},
        {Credo.Check.Readability.ModuleDoc, false},
        {Credo.Check.Refactor.LongQuoteBlocks, false},
        {Credo.Check.Refactor.Nesting, false}
      ]
    }
  ]
}
{{< /highlight >}}

## Running Credo

Once configured, running Credo is simple. I run it in strict mode to ensure the highest quality standards:

```bash
$ mix credo --strict
```

Here's the output from running it on my project:

{{< highlight bash "style=monokai" >}}
Checking 21 source files ...

Please report incorrect results: https://github.com/rrrene/credo/issues

Analysis took 0.7 seconds (0.05s to load, 0.7s running 70 checks on 21 files)
60 mods/funs, found no issues.
{{< /highlight >}}
