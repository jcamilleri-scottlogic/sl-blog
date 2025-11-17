---
title: 'Building JARVIS Properly - Phase 5: From Ultron''s Ruins to JARVIS''s Foundation'
date: 2025-11-17 12:28:00 Z
categories:
- AI
tags:
- AI
- product owner
summary: 'Phase 5 marks the moment JARVIS stopped being an experiment and started
  demanding discipline. When Mini-Me collapsed, the temptation to immediately add
  RAG, FAISS, and knowledge retrieval was overwhelming: every instinct screamed to
  make it "intelligent" right away. Instead, I did something harder: I said "not yet"
  and built multi-agent orchestration and governance first. JARVIS can now critique
  itself, challenge other agents, synthesize consensus across conversation histories,
  and most critically, every thread carries a blast_radius marker - governance baked
  in from day one, not bolted on as an afterthought. This is delayed gratification
  as product strategy: building the decision-making loop before giving the system
  real power. The temptation to rush to memory was real. The decision to build orchestration
  first was right. Because just as Vision proved worthy of wielding Thor''s hammer
  through restraint rather than raw power, JARVIS earns its capabilities by demonstrating
  it can be trusted with them. The transformation from JARVIS to Vision begins next,
  and the foundations are finally sound enough to support it.'
author: rgriffiths
---

## Act 1: Picking up the pieces

When Mini-Me collapsed under its own weight, it was terribly tempting to declare the entire experiment a failure and simply walk away. It had been, initially, a rather promising AI-agnostic logger, but governance slipped, scope crept, and before long it was a monolith wobbling on foundations that could never hold.

The ruins, though, were instructive. Out of that wreckage came a clear view of what truly mattered: modularity, clean interfaces, and, most importantly, a product owner's steady hand. Phase 5 is therefore not a patch job. It is a rebirth. This is where JARVIS takes its true shape, leaner, more sustainable, and truer to its vision.

If Mini-Me was Ultron, powerful but unstable, overreaching until it self-destructed, then what emerges from Phase 5 is JARVIS itself: purposeful, disciplined, and built on foundations solid enough to support the transformation ahead.

## Act 2: The temptation to rush ahead

The original plan was straightforward: implement a retrieval layer immediately. RAG, FAISS, vector stores, the whole knowledge-awareness infrastructure. Let JARVIS draw on external memory, make it truly intelligent, and watch it soar.

It was seductive. It was also precisely the wrong move.

This is where most AI projects go astray. The temptation to add capabilities is overwhelming. Every demo reveals a new possibility, every conversation with a stakeholder surfaces another "wouldn't it be amazing if" feature. The velocity of modern AI tooling makes it trivially easy to bolt on new functions at lightning speed.

But velocity without discipline is just flailing.

## Act 3: The disciplined pivot

In [Phase 4, I argued that AI tools can generate features at light speed, yet without product ownership discipline, scope creep turns your elegant vision into feature bloat](https://blog.scottlogic.com/2025/09/08/building-an-ai-agnostic-conversation-logger-phase-4-mini-me.html). The moment had arrived for me to heed my own advice!

The strategic pivot was clear: the immediate problem was not **memory**, but **trust and control**. The long-term advantage lay not in *what* JARVIS knew, but in *how* it arrived at that knowledge.

So I delayed RAG. I delayed FAISS. I delayed the entire knowledge layer.

Instead, I invested fully in multi-agent orchestration and governance. This meant building the decision-making loop first, ensuring it was robust, auditable, and controllable. Only then would I be ready to give JARVIS real power.

**This is delayed gratification as product strategy.**

It is contrary to how most AI projects evolve. It requires saying "not yet" when every instinct screams "now". But it is the only path to building something that endures rather than something that impresses for a fortnight before collapsing.

## Act 4: What actually got built

### A clean-slate architecture

The architecture is modular from the ground up. No more dumping everything into a single script and hoping it plays nicely. Instead:

* **Agents** (`jarvis/agents`) contain backend-specific adapters. Whether it's OpenAI, Claude, or the late-arriving (to my implementation!) Gemini, each conforms to the same interface.
* **Services** (`jarvis/services`) handle cross-cutting concerns like logging, search (for a future version), and orchestration logic.
* **Data** (`jarvis/data`) holds threads and metadata. Each conversation is its own object, with clean methods for adding messages, following existing sessions, or starting anew.
* **Resources** (`jarvis/resources/prompts`) defines the special instructions for critique (including self-critique and cross-critique), consensus (including the consensus_last_n special option), and also both the compare & contrast modes.

The CLI (`jarvis/cli/main.py`) stitches these parts together. Its job is orchestration, not heavy lifting.

The difference is subtle but profound. JARVIS is no longer "code that works for now" but a system that can grow without becoming incomprehensible.[^1]

[^1]: For those interested in implementation details: JARVIS now supports full vendor agnosticism with `--agent` (persistent switching) and `--using` (temporary override) flags. Persona management via `--as` allows loading context-specific instruction sets. The three supported backends (OpenAI, Anthropic, Gemini) all implement the same core interface, making vendor lock-in a relic of the past.

### Multi-agent orchestration: The real innovation

With foundations in place, JARVIS could take on features that make it a genuine companion rather than a brittle prototype. The orchestration capabilities are where this phase truly shines:

**Self-critique and cross-critique modes** introduce checks and balances that most conversational AI systems simply lack. Trusting a single response blindly is risky. JARVIS can now:

* Ask one agent to review another's work (`--critique`)
* Make an agent generate and then review its own response (`--self-critique`)
* Set multiple agents against each other for mutual review (`--cross-critique`)
* Run agents in parallel and synthesise consensus or highlight differences (`--compare` and `--contrast`)

But the most powerful feature is **consensus from history**. The `--consensus-last` mode can synthesise a fresh response by analysing the final agent messages from the last *N* conversation threads. This means JARVIS doesn't just learn within a conversation. It learns *across* conversations, building institutional memory without yet needing a full retrieval layer.

Imagine running the same complex query against three different agents across five separate sessions, then asking JARVIS to analyse those fifteen responses and provide a meta-synthesis. That's not a chatbot. That's a reasoning platform.

### The governance pattern: blast_radius

Every new conversation thread now begins with an explicit governance marker: `"blast_radius": "low"`. This is not documentation. This is not a comment. It is a first-class field in the data model, present from the very first message.

Here's why this matters: as JARVIS gains capabilities, particularly when we introduce tool use in the next phase, the potential for unintended consequences grows. A model that can read files might accidentally expose sensitive data. A model that can execute commands might, well, actually execute commands.

The `blast_radius` marker is a constraint that travels with every thread. It signals to future orchestration logic what level of action is permissible. A thread marked `"low"` might only answer questions. A thread marked `"medium"` might read files. A thread marked `"high"` might write to disk or call external APIs.

This isn't theoretical. When Phase 6 introduces tool use, the orchestration layer will check this marker before granting any permissions. The governance isn't bolted on afterwards. It's baked in from the start.

This is what mature AI engineering looks like: Built-in not Bolt-on.

## Act 5: The foundation is laid

There's a quiet satisfaction in seeing JARVIS operate: not perfect, not finished, but coherent. The scaffolding is sound, the architecture modular, and even small markers like `blast_radius` signal a new level of discipline.

JARVIS is no longer just an experiment. It is a platform with foundations solid enough to support the transformation ahead. From Ultron's chaos, I've built something purposeful and restrained. JARVIS is ready to evolve.

## The roadmap: An evolutionary arc

The path forward follows the natural progression of Tony Stark's own AI evolution, and it's a deliberate sequence built on delayed gratification:

**Phase 6: Vision Awakens** will mark JARVIS's transformation into something greater. Like Falcon's wings extending capability through disciplined tool use, JARVIS will gain the ability to interact with the world: reading files, listing directories, working with local codebases. But these capabilities will be strictly governed by the `blast_radius` marker already in place. This is where JARVIS becomes Vision, *worthy* of wielding ~~Thor's hammer~~ power because restraint is built into its very nature.

**Phase 7: Friday's Library** continues the evolution. With orchestration proven and tool use safely implemented, the system gains the retrieval layer (the placeholder for which exists already) that transforms context-awareness into knowledge-awareness. Like Friday accessing all of Stark's historical data and institutional knowledge, JARVIS will finally have RAG, FAISS, and the memory infrastructure originally envisioned. Memory arrives, but only after we've proven we can control what the system does with that memory.

**Phase 8: House Party Protocol** explores genuine autonomy. Remember Iron Man 3's climactic battle, when Tony summoned the entire Iron Legion? Marks 1 through 42, each with different specialisations, operating autonomously but coordinated towards a single goal. That's the vision for Phase 8 and beyond: multiple agents working in concert, chaining actions, making decisions, operating with minimal human intervention. But by that point, every layer beneath will be solid, auditable, and safe.

This is not the roadmap of a project chasing shiny objects. This is architecture with intent. From JARVIS to Vision to Friday to the Iron Legion, each phase builds on the last. This is delayed gratification as competitive advantage.

## Closing: The foundations are sound

Phase 5 marks the point where this project stopped being a tinkering experiment and started demanding discipline. JARVIS has a body worth protecting and a mind worth nurturing.

The temptation to rush to memory was real. The decision to build orchestration first was right.

The transformation from JARVIS to Vision begins next. The journey continues, with discipline as my guide.