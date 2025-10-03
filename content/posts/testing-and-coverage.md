---
title: "Testing and Coverage: A Practical Approach"
date: 2025-10-03T16:00:00-04:00
draft: false
author: "Joseph"
categories: ["Software Development", "Testing"]
tags: ["testing", "coverage", "quality", "best-practices"]
---

## Tests and Coverage

Getting started on my project I wanted to make sure I had a solid foundation in place. One of the first things I wanted to do was setup test coverage to maintain confidence in my code.

## Initial Coverage Report

Using the --cover flag when running tests, I was able to generate a coverage report for my project.

{{< highlight bash "style=monokai" >}}
Finished in 0.3 seconds (0.1s async, 0.1s sync)
5 tests, 0 failures

Generating cover results ...

Percentage | Module
-----------|--------------------------
     0.00% | TraysWeb.PageHTML
    18.42% | TraysWeb.CoreComponents
    25.00% | Trays.DataCase
    50.00% | Trays.Repo
    50.00% | TraysWeb.ErrorHTML
    75.00% | TraysWeb.Layouts
    75.00% | TraysWeb.Router
    80.00% | Trays.Application
    80.00% | TraysWeb.Telemetry
   100.00% | Trays
   100.00% | Trays.Mailer
   100.00% | TraysWeb
   100.00% | TraysWeb.ConnCase
   100.00% | TraysWeb.Endpoint
   100.00% | TraysWeb.ErrorJSON
   100.00% | TraysWeb.Gettext
   100.00% | TraysWeb.PageController
-----------|--------------------------
    37.99% | Total

Coverage test failed, threshold not met:

    Coverage:   37.99%
    Threshold:  90.00%
{{< /highlight >}}

## Ignoring What Doesn't Matter

It was good that I could see my coverage, but a lot of it was noise. I decided to ignore some files that didn't need to be tested since they were either config files or generated code.
{{< highlight elixir "style=monokai" >}}
defp test_coverage do
  [
    ignore_modules: [
      Trays.Application,
      Trays.DataCase,
      Trays.Repo,
      TraysWeb.ConnCase,
      TraysWeb.CoreComponents,
      TraysWeb.ErrorHTML,
      TraysWeb.ErrorJSON,
      TraysWeb.Layouts,
      TraysWeb.PageHTML,
      TraysWeb.Router,
      TraysWeb.Telemetry,
    ]
  ]
end
{{< /highlight >}}

## Improved Coverage Results

After configuring the ignore list to focus on what matters, here's the new coverage report:

{{< highlight bash "style=monokai" >}}
Running ExUnit with seed: 372784, max_cases: 16

.....
Finished in 0.2 seconds (0.09s async, 0.1s sync)
5 tests, 0 failures

Generating cover results ...

Percentage | Module
-----------|--------------------------
   100.00% | Trays
   100.00% | Trays.Mailer
   100.00% | TraysWeb
   100.00% | TraysWeb.Endpoint
   100.00% | TraysWeb.Gettext
   100.00% | TraysWeb.PageController
-----------|--------------------------
   100.00% | Total
{{< /highlight >}}

This is more readable now and doesn't show all the noise that was in the previous report. There is more I could do to improve my coverage, but for now, I'm happy with what I have for the start of my project.
