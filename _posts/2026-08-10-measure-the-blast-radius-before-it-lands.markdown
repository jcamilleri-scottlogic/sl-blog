---
title: Measure the blast radius before it lands
date: 2026-08-10 09:00:00 Z
categories:
- Artificial Intelligence
tags:
- AI
- Artificial Intelligence
- Agentic AI
summary: A practical guide to rebalancing code review by building your own toolchain that measures blast radius, classifies risk, and focuses human attention on the changes that matter most.
author: jcamilleri
image: "/uploads/Rapid-web-app-development-with-Devin---A-Developer%E2%80%99s-Perspective-.jpg"
---
# Measure the blast radius before it lands

*A blueprint for a code-review toolchain in the agentic era*

Over the past year I've been working with agentic coding tools such as Devin and Claude Code, both in my own projects and as part of Scott Logic's AI-assisted development programme. In [a previous post](https://blog.scottlogic.com/2025/10/20/rapid-development-with-devin.html) I described building a production-ready application with Devin in seven days, and noted that the hardest part wasn't generating the code, it was validating it. Given the volume these tools produce, I simply could not read every line.

This has been observed by many and is a concern I hear from my clients. In appears that the bottleneck of software delivery is increasingly moving from writing code to reviewing it. Agent-authored changes have three properties that make traditional review a poor fit:

- **They're wide.** A single agent instruction can edit code across more modules than any one person hold in their head at once.
- **They're fast.** Changes land quicker than a reviewer can follow the chain of consequences by hand, so the velocity outruns comprehension.
- **They're unfamiliar.** You're reviewing code you never watched being written, with no built-up mental model of the author's intent. There was no handover of context and it is hard to go back to the LLM to find out the reasoning for a decision.

Traditional review works best when changes are relatively small, understandable and accompanied by shared context. So rather than throttling the coding agent or asking reviewers to work harder, I think we need to rethink where the work of review actually happens.

One option is to get another LLM to code review the PR before a human does. Tools like Code-Rabbit are built for this approach and certainly work up to a point, but don't catch everything and so do not replace human revue. Instead, they supplement the review by catching things early so that a cleaner PR goes to the reviewer. However, most of the problems, such as volume of change and lack of context, remain.

The building blocks for the solution are all readily available. You'll need a source control system (e.g. git), a language parser, and some tightly scoped LLM calls. In this post I'll walk through how I would approach building a review toolchain.

## The purpose of code review

Before building anything, it's worth having a look at what code review does. Typically, I find it used for:

- verifying correctness against requirements
- sharing knowledge across the team
- catching defects before production
- maintaining quality and consistency
- improving maintainability and readability
- ensuring adequate testing and documentation
- catching security concerns

Traditionally we perform all seven at a single gate, when the pull request is raised. But do they all still belong there? In my experience, they don't. Before writing any tooling, redistribute them across three stages:

- the agentic workflow itself
- a code walkthrough
- an automated review process 

By splitting the concerns across different phases, where different tools can be applied, what finally reaches a human is a more focused review of the changes. The toolchain you build only needs to cover the third stage; the first two are process changes that can be implemented into your agentic workflow.

## Move correctness into the workflow

Rather than leaving correctness checks until review time, verification against requirements belongs inside the workflow that produces the code. The request to the LLM should have correctness and verification built in from the start. This means clear acceptance criteria, a test-first approach, and dedicated unit, integration and UI testing agents validating the work as it happens. Regardless of code review, this is a necessary part of creating effective agentic loops, so we should lean into it.

Meanwhile, the build pipeline runs linters, so style and convention are settled before anyone opens the pull request. If a human reviewer is still commenting on formatting, something upstream needs fixing.

None of this removes human responsibility for correctness. The engineer orchestrating the agents remains responsible for ensuring that the implementation satisfies the requirements, through precise task specification, effective agentic workflows, and both automated and manual testing. The details are beyond the scope of this post, but keeping a human engineer in the loop to direct and verify coding agents is critical for real-world systems.

## Keep the walkthrough

Of the seven aims, knowledge sharing is the one most at risk of quietly disappearing. The fix is a deliberate code walkthrough.

There are two ways to achieve this. The first is to simply get an LLM to reverse-engineer the reasoning behind why a change was made. The problem with this approach is that the LLM could be very confidently wrong. But it may be better than nothing if there is no alternative. Just take the answer with a pinch of salt.

A better approach is to get the agent to record a structured log of decisions it has made. This could take the following form:
- requirements addressed
- significant implementation decisions
- alternatives considered
- assumptions
- files/components affected
- tests performed
- known limitations
- unresolved questions

This can be achieved with a skill or hook (or equivalent for your agentic toolchain) to keep a summary of decisions and write it to a log folder in the project. This artefact should be source controlled. Customise the log entry template as required.

Next:

1. Prompt the coding tool to walk you through the changes it has made, using the decision log.
2. Gain an understanding of what it implemented, why, and how it has changed the codebase.
3. Understand how it has verified correctness.

This step deliberately rebuilds the mental model you would normally have acquired by writing the code yourself or pairing on it. It turns reviewing a stranger's work back into reviewing a colleague's.

## Design your toolchain around two analysis layers

So what should the tooling itself do? The remaining review tasks split cleanly into two analysis layers. Deterministic analysis uses static analysis tools to discover issues. AI reasoning uses LLM analysis in a fresh context to uncover more nuanced issues. Both layers serve to focus human review time on the most important parts of a PR.

| | **Deterministic analysis** | **AI reasoning** |
| --- | --- | --- |
| **Nature** | Questions with precise answers | Judgement calls over established facts |
| **Implementation** | Static tooling, graph analysis | Tightly scoped model calls |
| **Tasks** | Diff changed files; build and invert the import graph; blast-radius analysis; classify changed symbols as internal or external; map interface changes to callers; static code analysis; churn scoring; security signals; dependency integrity | Summary and decisions; anomaly detection; test-gap analysis; security scoring; regression analysis |

The principle to follow is that if a question has an exact answer, compute it. Save the model calls for questions that genuinely require reasoning, and feed them the computed facts as context. This also keeps your costs down and your outputs reproducible. This makes it more likely the tool will be used and trusted.

In terms of inputs, you need surprisingly little: a path to the repository and two commits to compare.

## Start with the blast radius

The starting point of the deterministic layer is blast-radius analysis. For each file a change touched, who depends on it? The changes with the furthest-reaching impact are the ones that deserve attention first. The blast radius is one input to risk classification, not a risk score itself.

It works in three steps:

1. **Walk the reverse graph.** Parse the import or dependency statements in your codebase, build the import graph, and invert it. Then run a breadth-first search outward from the changed files, recording each dependent's distance in import hops. A file one hop away is a direct consumer; three hops away, it's downstream ripple. You may want to cap the search depth at around three if you find that everything depends on everything and the signal fades.
2. **Enrich the direct hits.** For distance-1 dependents, extract the exact symbols they import, so that a reviewer sees which symbols are at risk, not just which files.
3. **Nearest first.** Order the results by distance, then path, with the changed files themselves excluded. The output is a prioritised map of impact, closest and most concrete at the top.

This measures a static dependency blast radius, not complete runtime impact. You will need to customise this approach depending on your codebase. For example, the use of dependency injection, reflection, events or other patterns and practices may require more thought.

You don't need perfect import resolution for this to be useful. Most mainstream languages have accessible parsing tools such as Python's `ast` module, the TypeScript compiler API, or tree-sitter if you want to cover several languages with one approach. An approximate graph that is right most of the time is still useful compared to a reviewer scrolling a forty-file diff wondering where to start. However, the output should communicate confidence/limitations of the analysis.

Blast radius is a starting point, but we need more to get a feel for the risk of the change.

## Classify what changed

Blast radius tells you *reach*; the next layer tells you *kind*. Tag each changed symbol by impact type. Parse the before and after versions of each file and compare their public signatures, falling back to regex where parsing fails. Then use the tag to guide the next step:

- **`contract_removed`**: A public contract item is gone. Existing callers will break, so enumerate them.
- **`contract_changed`**: A public contract item has moved or requires different parameters. Cross-reference the reverse graph to list the callers now at risk.
- **`contract_added`**: New public surface. No existing in-repository caller is expected to break.
- **`internal`**: A body or logic change on a private surface. Route this to file-level ripple analysis plus the reasoning layer.

The internal tag catches pure body edits. Map the changed line numbers from the diff onto symbol ranges, so a function whose name never appears anywhere in the diff is still flagged as modified. Internal changes flow through to AI diff review. Contract changes then produce a concrete caller list which answers the question "who breaks?". 

## Scope the reasoning calls tightly

When the reasoning layer does run, resist the temptation to make it one enormous "please review this PR" prompt with the whole repository stuffed into context. Instead, make a small set of tightly scoped model calls, each given only the material it needs:

- **Summary, decisions, and assumptions**: fed the diffs plus the blast-radius signatures.
- **Anomaly detection**: diffs plus before/neighbour signatures and file history, looking for changes that don't fit their surroundings.
- **Test-gap analysis**: diffs plus the discovered test files.
- **Security-signal scoring**: pattern signals, diffs, and file context.
- **Regression analysis**: diffs plus before/after signatures, checking whether a refactor actually preserved behaviour.

Make the last two conditional so that they fire only when the deterministic layer has raised a flag worth investigating, such as a suspicious pattern in the diff or a change to a dependency manifest. This keeps cost proportionate to risk: routine changes get a summary and a sanity check, whilst suspicious ones get the full treatment. Dependency changes in particular reward a deterministic first pass. This checks new packages against a vulnerability database such as OSV, and looks for near-miss names and loosened version pins before any model gets involved.

## Fit it into your pipeline

A tool nobody runs reviews nothing, so think about where yours will live. The aim isn't just to generate another report; it's to put the right findings in front of the right people at the right point in the workflow.

- Run it in CI on every pull request and post the report as a PR comment. Produce Markdown for humans and JSON for anything downstream that wants to consume the results. The report should make the highest-risk changes obvious rather than asking reviewers to work through another wall of output.

- Separate findings from gates. Deterministic findings with clear answers are good candidates for failing the build: a broken contract, a missing dependency, or a known security issue can be treated as a hard failure. AI-generated findings are better treated as advisory initially. If a team later finds that a particular signal is consistently accurate, it can choose to promote it to a gate. This keeps an uncertain model judgement from unexpectedly blocking delivery.

- Make it configurable per repository. Let teams mark high-sensitivity modules, such as authentication or payments, for extra scrutiny, and suppress known-noisy signals with a recorded reason. Noise can kill adoption; a finding the team has already accepted should not reappear on every pull request.

- Keep a little history. Churn scoring needs to know how often an area has changed before, so retain a record of previous runs. You don't need a large data platform for this: a small database containing change history, findings, and their outcomes is enough. Over time, that history can also show which signals are useful and which are generating noise.

- Measure whether it helps. The goal is not to maximise the number of findings. Track things such as review time, accepted versus dismissed findings, escaped defects, and the proportion of high-risk changes receiving human attention. The toolchain should demonstrate that it is improving the allocation of review effort, not simply producing more analysis.

The tool should pull together a standardised report that pulls together the two layers of analysis for easy consumption by the reviewer. Put critical stats like the blast radius and churn at the top followed by a brief summary of the change. The rest of the report can then be navigated by the reviewer depending on their concerns.

## What lands on the reviewer

By the time an engineer opens the pull request, your toolchain should have given them:

- Some confidence in the correctness of the code.
- Style and conventions settled by linting and static analysis.
- An overall feel for the quality of test and security coverage.
- An idea of how far-reaching the changes are and whether anything is likely to break.
- A summary of what has changed and why.

The engineer can now perform a more focused review of the parts of the PR that matter most, while still taking advantage of the speed of agentic code generation. This allows human judgement to be applied where it adds the most value.

## Conclusion

I think the goal isn't to remove humans from code review. It's to stop spending human attention on questions machines can answer deterministically, and spend it where judgement actually matters.

Strengthen your agentic coding process so correctness is tested at the point of creation, and have the agent walk engineers through its changes to preserve knowledge sharing. Then build tooling for what remains: deterministic analysis for complexity, churn, and blast radius, and tightly scoped AI analysis for test gaps, security signals, and anomalies, so human attention lands on the highest-risk, highest-value changes.

None of this requires a big platform investment. The pieces are well within reach of a motivated team. Agentic tools can genuinely accelerate delivery, but only if review keeps pace with generation. Measuring the blast radius before it lands is, I think, how we get there.