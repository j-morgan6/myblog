---
title: "Hacking My Rancilio Silvia: Using Stride to Manage a Complex Sensor Integration"
date: 2026-01-09
draft: false
tags: ["project-management", "ai-tools", "stride", "productivity", "embedded-systems", "rancilio-silvia", "espresso"]
categories: ["tools", "productivity"]
description: "How I used Stride to manage implementing a temperature sensor for my Rancilio Silvia espresso machine mod - from protocol research through hardware testing to pivoting when things didn't work out."
---

## The Setup: Modding a Rancilio Silvia

I'm hacking my Rancilio Silvia espresso machine with a Raspberry Pi Zero W running Elixir/Nerves. The goal? PID temperature control for better espresso. When I decided to add a TSIC306 temperature sensor, I knew it'd be tricky, but I didn't know *how* tricky.

Here's the thing about embedded systems work: you can't plan what you don't understand yet. And I needed some way to stay organized without drowning in project management overhead.

That's when I decided to try [Stride](https://www.stridelikeaboss.com). Full disclosure: I know the creator and was curious about his project. Stride bills itself as "task management designed for human-AI collaboration"—basically a kanban board that AI agents can actually work with. Sounded interesting, so I figured I'd use it for this sensor implementation and see how it went.

## How Stride Actually Works

So what is Stride exactly? It's a kanban-style task board with a twist: it's designed from the ground up for human-AI collaboration. The creator [built it using AI-assisted development](https://www.stridelikeaboss.com/about) (Claude Code and TideWave), documenting the whole process across 16 blog posts. Pretty meta.

The cool part is that AI agents can claim and complete tasks via the API, with capability matching so they only see tasks that match their skills. There's also human approval workflows if you want them, and client-side hooks for custom automation.

## Breaking Down the Unknown

I started simple: "I need to implement TSIC306 temperature sensor support for my Raspberry Pi Zero W project using Elixir."

Stride broke it down into concrete tasks:

```
W14: Research ZACWire protocol specification
W15: Implement bit-level GPIO reading logic
W16: Add parity bit validation
W17: Write temperature conversion and unit tests
W18: Implement comprehensive error handling
W19: Add timeout protection for sensor reads
W20: Integration testing with full system
W21: Hardware testing on actual Raspberry Pi
W22: Document implementation and findings
```

Each task had:
- **Clear scope**: What specifically needs to be done
- **Acceptance criteria**: How to know when it's complete
- **Dependencies**: Which tasks must come first
- **Context**: Why this task matters

## The Development Flow: W14-W20

Tasks W14-W20 went surprisingly smoothly. Here's what that looked like:

### Task W14: Protocol Research

I just said "Do W14" and Stride went to work gathering ZACWire protocol specs, explaining duty-cycle encoding (25% = 1, 50% = 0, 75% = start bit), and flagging the microsecond timing requirements that would later bite me.

The nice part? Stride didn't just say "go research the protocol." It actually helped find resources, summarized the gnarly technical details, and pointed out the stuff that mattered.

### Tasks W15-W17: Core Implementation

Here's where things got interesting. The AI agent actually wrote the protocol implementation in Elixir. My role? Review the code it generated. Here's a snippet of what it produced:

```elixir
defmodule Silvia.TSIC306 do
  use GenServer
  alias Circuits.GPIO

  defp read_single_bit do
    high_start = System.monotonic_time(:microsecond)
    wait_for_edge(:falling)
    high_duration = System.monotonic_time(:microsecond) - high_start

    low_start = System.monotonic_time(:microsecond)
    wait_for_edge(:rising)
    low_duration = System.monotonic_time(:microsecond) - low_start

    total = high_duration + low_duration
    duty_cycle = high_duration / total

    cond do
      duty_cycle < 0.35 -> {:ok, 1}
      duty_cycle < 0.65 -> {:ok, 0}
      true -> {:ok, :start_bit}
    end
  end
end
```

**This is the real power of Stride's AI collaboration:** The agent understood the task breakdown, wrote the implementation, and I reviewed and approved the code. I wasn't context-switching between writing code and updating tickets—I was reviewing code the agent wrote and making sure it was correct.

### Tasks W18-W20: Robustness and Testing

Error handling, timeouts, and integration testing. By the time I reached W20, I had about 300 lines of carefully crafted protocol implementation. Everything worked beautifully... in unit tests.

## The Pivot Point: W21 Hardware Testing

**Me:** "Do W21"

Time for the moment of truth. I deployed the firmware to my actual Rancilio Silvia setup, connected the sensor to GPIO 4, and started collecting readings:

```
[debug] Temperature reading: {:error, :parity_error}
[debug] Temperature reading: {:ok, -47.3}
[debug] Temperature reading: {:error, :parity_error}
[debug] Temperature reading: {:ok, 149.2}
[debug] Temperature reading: {:error, :parity_error}
[debug] Temperature reading: {:ok, 24.65}
```

**The results:**
- 75% parity error rate
- Most "successful" reads were garbage data
- Occasional valid readings (~5%)

This is where Stride's value really showed. Instead of just marking W21 as "blocked" or "failed," here's what happened:

### Stride's Response to Unexpected Results

**Stride:** "The high error rate suggests timing issues. Raspberry Pi Zero uses a 1GHz ARMv6 CPU, which is significantly slower than Pi 3/4 models. Linux's non-real-time scheduler can cause timing jitter in microsecond-precision GPIO operations. Let me research solutions..."

Within moments, Stride had:

1. **Diagnosed the root cause:** Linux scheduler preemption + slow CPU + microsecond timing requirements = incompatible combination

2. **Researched existing solutions:** Found that others use `pigpio` daemon (runs with elevated priority) for better timing

3. **Identified the tradeoff:** pigpio helps but doesn't eliminate the fundamental mismatch between ZACWire's requirements and Linux's capabilities

4. **Proposed alternatives:** Should we try pigpio, or evaluate different sensors?

This is where Stride felt less like a task tracker and more like a senior developer looking over my shoulder.

## The Decision: Evaluate Alternatives

**Me:** "What are better sensor options for Raspberry Pi + Elixir?"

**Stride:** "Let me research temperature sensors with hardware-supported protocols..."

Stride produced a comparison matrix:

| Sensor | Protocol | Kernel Support | Accuracy | Code Complexity | Cost | Notes |
|--------|----------|----------------|----------|-----------------|------|-------|
| TSIC306 | ZACWire | ❌ No | ±0.5°C | 300 lines | $5-8 | Works on microcontrollers |
| MAX31865+PT100 | SPI | ✅ Yes | ±0.5°C | ~20 lines | ~$15 | Industry standard for espresso |
| DS18B20 | 1-Wire | ✅ Yes | ±0.5°C | ~5 lines | $3-5 | Easiest implementation |
| MCP9808 | I2C | ✅ Yes | ±0.25°C | ~15 lines | ~$5 | Best accuracy |

**Stride's recommendation:** "For your espresso machine project, MAX31865 with PT100 RTD is the best choice. PT100 is what commercial espresso machines use, SPI has full kernel support, and the 20-line implementation vs. 300-line ZACWire implementation means less maintenance burden."

**The decision:** Switch to MAX31865 + PT100 RTD.

## What Made This Work: Key Stride Features

### 1. Context Awareness

Stride remembered details across conversations:
- "You're using Raspberry Pi Zero W" (not Pi 3/4)
- "Your project is in Elixir using Circuits.GPIO"
- "This is for an espresso machine (90-95°C operating range)"

This meant I didn't have to re-explain my project every time I asked a question.

### 2. Adaptive Planning

When W21 revealed fundamental issues, Stride didn't:
- Force me to complete a plan that wasn't working
- Just mark tasks as "blocked" without helping
- Make me feel like I'd "failed"

Instead, it:
- Helped diagnose the problem
- Researched solutions and alternatives
- Adapted the plan based on new information
- Documented learnings for future decisions

### 3. Technical Research Integration

Stride could:
- Search for protocol specifications
- Find existing implementations (python-tsic library)
- Compare technical approaches
- Explain tradeoffs in context of my project

This felt like having a senior developer who's already researched similar problems.

### 4. Learning Capture

Every task completion captured:
- What was done
- Why decisions were made
- What was learned
- What to do differently next time

W21 wasn't a "failure"—it was a successful discovery that the approach needed to change.

## The Final Outcome

**Tasks W14-W20:** ✅ Complete
- Successfully implemented full ZACWire protocol in Elixir
- Demonstrated understanding of timing-sensitive protocols
- Built comprehensive error handling and validation

**Task W21:** ✅ Complete (with pivot)
- Tested on actual hardware
- Discovered fundamental timing limitations
- Documented root cause analysis
- Evaluated alternatives
- Made informed decision: MAX31865 + PT100 RTD

**Task W22:** 🔄 In Progress
- Document findings (this blog post!)
- Share learnings with community

**Next Steps:**
- Order MAX31865 module (~$15)
- Implement SPI-based sensor reading (~20 lines)
- Continue with espresso PID controller

## Lessons About Project Management

### 1. Good PM Adapts to Learning

The best project management doesn't enforce an original plan—it helps you navigate as you learn. Tasks W14-W22 were a roadmap, not a rigid schedule. When reality diverged from assumptions, the plan evolved.

### 2. "Failure" Is Just Discovery

W21 "failed" in that the sensor didn't work reliably. But W21 "succeeded" in:
- Identifying the problem
- Understanding why it happened
- Documenting the finding
- Evaluating alternatives
- Making an informed decision

Stride recognized both aspects. Traditional tools would just show a red "Failed" status.

### 3. Solo Developers Need Different Tools

Team-based PM tools optimize for:
- Coordination between people
- Visibility for managers
- Formal tracking and reporting

Solo developers need tools that optimize for:
- Breaking down unfamiliar work
- Capturing context and decisions
- Reducing cognitive load
- Supporting technical research

Stride feels purpose-built for the solo developer experience.

### 4. Documentation Happens Naturally

Because Stride captured learnings at each task, I didn't need a separate documentation phase. The project history *is* the documentation. When writing this blog post, I just reviewed the task history—all the technical details and decision-making were already there.

## When Stride Works Best

Based on my experience, Stride excels when:

- **Working with unfamiliar technology:** Breaking down what you don't fully understand
- **Discovery-driven development:** Plans need to adapt as you learn
- **Solo or small teams:** Don't need heavyweight team coordination
- **Technical decision support:** Want AI help researching options
- **Building a knowledge base:** Documenting why, not just what

## The Bottom Line

Stride turned what could have been a frustrating "failed sensor implementation" into a well-documented learning experience with clear next steps. The tool adapted to reality instead of forcing me to follow a plan that wasn't working.

For solo developers tackling complex technical challenges, having an AI assistant that does both project management and technical research feels like having a senior developer looking over your shoulder—in the best way possible.

**The technical outcome:** I learned ZACWire protocol, implemented it successfully, and discovered its limitations on my platform.

**The project management outcome:** I stayed organized, captured learnings, adapted to new information, and made informed decisions about next steps.

**The human outcome:** I didn't feel lost, frustrated, or overwhelmed. I felt supported.

That's what good project management should feel like.

---

## Key Takeaways

1. **Break down complex work iteratively:** You don't need to know everything upfront
2. **Capture context as you go:** Future you will thank present you
3. **Treat pivots as learning, not failure:** Discovery is progress
4. **Use AI for research and decision support:** Don't just track work, get help with work
5. **Choose tools that match your team size:** Solo developers need different tools than teams

---

**Project Stats:**
- Tasks completed: W14-W21 (8 tasks)
- Lines of protocol code: ~300
- Hardware test failure rate: 75%
- Alternative sensors evaluated: 3
- Final decision: MAX31865 + PT100 RTD
- Time saved on project management overhead: Immeasurable

---

*Want to try Stride for your own projects? Check out [stridelikeaboss.com](https://www.stridelikeaboss.com).*