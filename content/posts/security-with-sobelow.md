---
title: "Basic Security Measures with Sobelow and Mix Audit"
date: 2025-10-03T18:00:00-04:00
draft: false
author: "Joseph"
categories: ["Elixir", "Security"]
tags: ["sobelow", "security", "mix-audit", "best-practices"]
---

## Security First

Security should be a priority from the start of any project. In the Elixir ecosystem, we have excellent tools to help catch security vulnerabilities early in development. Two essential tools are Sobelow for static security analysis and Mix Audit for dependency vulnerability checking.

<!--more-->

## Adding Security Tools to Your Project

Add both dependencies to your `mix.exs` file:

{{< highlight elixir "style=monokai" >}}
{:mix_audit, "~> 2.1", only: [:dev, :test], runtime: false},
{:sobelow, "~> 0.13", only: [:dev, :test], runtime: false}
{{< /highlight >}}

Then run `mix deps.get` to install them.

## Configuring Sobelow

You can configure Sobelow to suit your project's needs by creating a `.sobelow-conf` file in your project root this is mine:

{{< highlight elixir "style=monokai" >}}
[
  verbose: false,
  private: false,
  skip: false,
  router: nil,
  exit: false,
  format: "txt",
  out: nil,
  threshold: :low,
  ignore: ["Config.HTTPS"],
  ignore_files: [],
  version: false
]
{{< /highlight >}}

### Fixing Security Issues

When I ran Sobelow, it immediately pointed out an issue in my router file. The `put_secure_browser_headers` plug needed additional security headers.

The original plug in `lib/trays_web/router.ex`:

{{< highlight elixir "style=monokai" >}}
pipeline :browser do
  plug :accepts, ["html"]
  plug :fetch_session
  plug :fetch_live_flash
  plug :put_root_layout, html: {TraysWeb.Layouts, :root}
  plug :protect_from_forgery
  plug :put_secure_browser_headers
end
{{< /highlight >}}

I fixed this by updating the `put_secure_browser_headers` plug with custom security headers:

{{< highlight elixir "style=monokai" >}}
plug :put_secure_browser_headers, %{
  "content-security-policy" => "default-src 'self'",
  "x-frame-options" => "DENY",
  "x-content-type-options" => "nosniff"
}
{{< /highlight >}}

## Running the Security Checks

Now that everything is configured, I can run these three commands to check my project's security:

- `mix deps.audit` - Checks for vulnerable dependencies
- `mix hex.audit` - Checks for retired Hex packages
- `mix sobelow --config` - Scans code for security issues
