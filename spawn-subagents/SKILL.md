---
name: spawn-subagents
description: Teaches how to split work across Claude Code subagents and picks the right model for each one: Haiku 5.5 for speed and efficiency, Sonnet 5.5 for well-scoped work, Opus 5.5 for everyday intelligence. When invoked, it breaks the user's task into pieces, assigns each piece a model, and spawns the subagents (in parallel where independent) with the Agent tool. Use when the user says "spawn subagents", "break this down", "parallelize this", "use agents for this", "which model should I use", "fan out", or hands over a task big enough to split.
---

# Spawn Subagents

Two jobs: **teach** the model-routing rules, and **do** the work by spawning subagents accordingly.

When this skill is invoked on a real task, do not just explain. Run the loop below and spawn the agents.

## The three models

| Model | `model` param | Role | Use it for |
|---|---|---|---|
| **Haiku 5.5** | `haiku` | Speed and efficiency | Lookups, grep/glob sweeps, file inventories, summarizing a doc, format conversion, bulk renames, checklist verification, anything where the answer is "find it and report" |
| **Sonnet 5.5** | `sonnet` | Well-scoped work | A clearly specified implementation: "write this function to this spec", "refactor these 3 files this way", "draft this section from these notes", tests for a defined module |
| **Opus 5.5** | `opus` | Everyday intelligence | Ambiguous problems, design decisions, debugging with unknown root cause, multi-file reasoning, review/critique, synthesis across sources, anything where judgment matters |

The orchestrator (you, in the main session) is the planner and the integrator. Subagents execute.

### Routing test (ask in this order)

1. Is the output just facts found and reported? → **Haiku**.
2. Could I hand over a spec with inputs, outputs, and done-criteria, and the agent needs no judgment calls? → **Sonnet**.
3. Does it need judgment, trade-offs, or figuring out what the spec should be? → **Opus**.
4. Unsure between two? Pick the cheaper one for read-only work, the stronger one for anything that writes or decides.

Never put Haiku on writes that are hard to undo. Never burn Opus on a grep.

## The loop

### 1. Decompose

Break the task into units. A good unit has:
- one clear deliverable
- inputs the agent can find itself or that you hand it
- a done-criterion you can check
- no dependency on a sibling's unfinished output

Write the plan as a short table before spawning:

| # | Unit | Model | Depends on | Deliverable |
|---|---|---|---|---|

### 2. Order into waves

- **Wave 1**: independent read-only work (research, inventory, search). Mostly Haiku.
- **Wave 2**: implementation or drafting that uses wave 1 output. Mostly Sonnet.
- **Wave 3**: review, synthesis, hard calls. Opus.

Units in the same wave run in parallel. A later wave waits for the earlier one.

### 3. Spawn

Use the `Agent` tool. Set `model` explicitly on every call; never rely on inheritance.

**Send all independent agents in one message** so they run concurrently. That is the whole speedup.

Pick `subagent_type` by job:
- `Explore`: read-only search across many files (pair with `haiku`)
- `Plan`: design an approach before building (pair with `opus`)
- `general-purpose`: anything that writes or runs commands (pair with `sonnet` or `opus`)

Example of a parallel wave 1:

```
Agent(description="Inventory skills", subagent_type="Explore", model="haiku", prompt="...")
Agent(description="Find X post drafts", subagent_type="Explore", model="haiku", prompt="...")
Agent(description="Scan pipeline board", subagent_type="Explore", model="haiku", prompt="...")
```

### 4. Write each prompt like a brief for a stranger

The subagent has none of your context. Every prompt includes:

1. **Goal**: one sentence, plus why it matters.
2. **Inputs**: exact file paths, links, or data it needs.
3. **Scope fence**: what it may touch and what it must not.
4. **Output format**: what to return, and how short ("under 200 words", "table of path | finding").
5. **Done when**: the check that proves it finished.

Weak: "look into the auth stuff."
Strong: "Find every place `verifyToken` is called under `src/`. Return a table of `path:line | caller | what it does with the result`. Read-only. Under 150 words of prose."

Delegate the work, not the understanding. Don't write "based on your findings, fix it". State what to fix.

### 5. Integrate and verify

- Read every result. A subagent report describes what it intended, not always what it did.
- Spot-check writes: open the changed files, run the test, check the output exists.
- If a Haiku result is thin or wrong, rerun that unit on Sonnet. If Sonnet stalls on ambiguity, escalate to Opus.
- Merge the pieces yourself. Do not spawn another agent just to concatenate.
- Report to the user: what was split, which model took which unit, what came back, what's verified, what's open.

## Cost and speed rules

- Parallelize when units are independent. Serialize only on real dependencies.
- Don't spawn for a task you can finish in one or two tool calls. Agents have overhead.
- Don't duplicate: if you delegated a search, don't also run it yourself.
- Cap a wave at about 5 to 8 agents unless the user asks for more.
- Background by default; only block on an agent when your very next step needs its result.

## Worked example

**Task**: "Audit the docs folder, fix inconsistent formatting, and tell me which pages are ready to publish."

| # | Unit | Model | Wave |
|---|---|---|---|
| 1 | List every file in `docs/` with its status line | Haiku | 1 |
| 2 | Pull the open items from the project tracker | Haiku | 1 |
| 3 | Fix formatting in the pages to a given spec (heading, link style, dates) | Sonnet | 2 |
| 4 | Judge which pages clear the publish bar and why | Opus | 3 |

Wave 1 runs 1 and 2 in parallel. Wave 2 takes their output and runs 3. Wave 3 runs 4 on the cleaned pages. Then you integrate and report.

## When invoked with no task

If the user just types the skill name with no task, give a 5-line summary of the model table and routing test, then ask what they want to split up.
