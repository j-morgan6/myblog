---
title: "Deploying to Fly.io for Development"
date: 2025-10-22T10:00:00-04:00
draft: false
author: "Joseph"
categories: ["Software Development", "DevOps"]
tags: ["fly.io", "deployment", "infrastructure", "elixir"]
---

## Getting Your App Live with Fly.io

When it comes to deploying Elixir applications for development, Fly.io stands out as a developer friendly platform that makes the deployment process straightforward. In this post, I'll walk through deploying my project to Fly.io and share some key learnings along the way.

<!--more-->

## Why Fly.io?

Fly.io offers several advantages for development deployments:

- **Global Edge Network**: Deploy your app close to your users
- **Elixir Friendly**: Built in support for Phoenix and Elixir applications
- **Docker Based**: Uses Dockerfiles for flexible deployments
- **Built in Postgres**: Easy database provisioning

## Getting Started

First, install the Fly.io CLI (flyctl):

{{< highlight bash "style=monokai" >}}
# macOS
brew install flyctl

# Or use the install script
curl -L https://fly.io/install.sh | sh
{{< /highlight >}}

## My Deployment Experience

The deployment process was surprisingly straightforward. I only ran two commands:

{{< highlight bash "style=monokai" >}}
fly launch
fly auth token
{{< /highlight >}}

The `fly auth token` command prompted me to paste an API token, which I generated from my Fly.io dashboard at https://fly.io/user/personal_access_tokens.

From there, Fly.io handled everything automatically. The `fly launch` command:
1. Detected my Elixir/Phoenix application
2. Generated a `fly.toml` configuration file
3. Created the necessary `Dockerfile` and `.dockerignore` files
4. Set up the app on Fly.io's infrastructure
5. Configured the database connection automatically

It really was that simple. The platform took care of all the heavy lifting.

## Automating Deployments with GitHub Actions

To make deployments even smoother, I set up continuous deployment using GitHub Actions. This automatically deploys my app whenever I push to the main branch.

I added this job to my `.github/workflows/elixir.yml` file:

{{< highlight yaml "style=monokai" >}}
deploy-review:
  name: Deploy to Review App
  runs-on: ubuntu-latest
  needs: test
  if: github.ref == 'refs/heads/main'
  concurrency: deploy-group
  steps:
    - uses: actions/checkout@v4
    - uses: superfly/flyctl-actions/setup-flyctl@master
    - run: flyctl deploy --remote-only --config ./fly.review.toml --dockerfile ./Dockerfile.review
      env:
        FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
{{< /highlight >}}

This workflow:
- Only runs on pushes to the main branch
- Waits for tests to pass first (via `needs: test`)
- Uses Fly.io's official GitHub Action to set up flyctl
- Deploys using a custom config file and Dockerfile for the review environment
- Uses `--remote-only` to build on Fly.io's servers instead of locally
- Uses the `FLY_API_TOKEN` secret for authentication

To set this up, you'll need to:
1. Generate an API token from https://fly.io/user/personal_access_tokens
2. Add it as a secret named `FLY_API_TOKEN` in your GitHub repository settings
3. Create your custom `fly.review.toml` and `Dockerfile.review` if needed

Now every push to main automatically deploys to Fly.io after the tests pass!

## Running Seeds

After deploying, I needed to populate the database with initial data using my seeds file.

Here's the process I followed:

{{< highlight bash "style=monokai" >}}
# 1. Deploy first
fly deploy -a trays-review

# 2. Connect to the remote console
fly ssh console -a trays-review -C "/app/bin/trays remote"

# 3. Run seeds from within the remote console
Code.eval_file(Path.join([:code.priv_dir(:trays), "repo", "seeds.exs"]))
{{< /highlight >}}

This workflow is useful for any time you need to update your seeds file. The remote console gives you full access to your application's environment, allowing you to run Elixir code directly.

## Monitoring Your App

Fly.io provides several monitoring tools:

{{< highlight bash "style=monokai" >}}
# View logs
flyctl logs

# Check app status
flyctl status

# Monitor in real-time
flyctl logs -a your-app-name --follow
{{< /highlight >}}

You can also access the Fly.io dashboard at https://fly.io/dashboard to see metrics, logs, and machine status.

## Useful Commands

Here are some commands I find myself using frequently:

{{< highlight bash "style=monokai" >}}
# Open your app in the browser
flyctl open

# SSH into your running app
flyctl ssh console

# Scale your app
flyctl scale count 2

# Check app info
flyctl info

# Restart your app
flyctl apps restart
{{< /highlight >}}

## Conclusion

Fly.io has made deploying my development environment incredibly smooth. The platform's focus on developer experience shows, from automatic Elixir detection to built in Postgres support.