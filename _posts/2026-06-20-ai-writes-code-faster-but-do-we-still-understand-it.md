# AI Writes Code Faster, But Do We Still Understand It?

There's a particular kind of unease that settles in when you're staring at a 400-line pull request that an AI agent generated in twelve minutes. The tests pass. The linter is happy. The feature works. And yet you can't shake the feeling that you're about to approve something you don't fully understand.

Addy Osmani recently gave this feeling a name: [Comprehension Debt](https://addyosmani.com/blog/comprehension-debt/). He defines it as "the growing gap between how much code exists in your system and how much of it any human being genuinely understands." Unlike technical debt, which screams at you through build failures and slow queries, comprehension debt accumulates silently. Everything looks green. Everything works. Until it doesn't, and nobody knows why.

I've been thinking about this a lot while building [Jonggrang](https://github.com/porcupine-md/jonggrang), a platform that orchestrates AI coding agents through structured development pipelines. The irony isn't lost on me: I'm building tooling to manage the very problem that keeps me up at night. But working on it has given me a front-row seat to how deep this problem actually goes.

---

## From Writer to Auditor

The job description has shifted under our feet.

Five years ago, being a good developer meant you could architect a system, write clean code, and debug it when things went sideways. Those skills still matter, but they've been reweighted. The bottleneck is no longer writing code. AI handles that part faster than most of us ever could. The bottleneck is *understanding* what got written.

When I delegate a feature to a coding agent, I'm not thinking about syntax or boilerplate. I'm thinking: Did it choose the right data structure? Are there race conditions hiding in that async flow? Will this query still perform when the table hits a million rows? My role has quietly shifted from author to auditor. And auditing code you didn't write, from a mind that doesn't think like yours, is a fundamentally different skill than writing it yourself.

This is why Jonggrang enforces [role-based boundaries](https://jonggrang.dev/deck): Lead agents design but cannot edit code. Developer agents implement but cannot alter architecture. Reviewer agents audit but cannot merge. The separation exists because we learned the hard way that when an AI wears all hats simultaneously, nobody is truly reviewing anything. The system just validates itself.

The uncomfortable truth is that most of us were never trained to be auditors. We were trained to be writers. And the industry hasn't caught up to the fact that these require entirely different muscles.

---

## The Speed Trap

Here's what nobody warns you about: AI makes the *wrong* architecture feel exactly like the right one.

You can spin up a prototype in hours now. Server wrapper, database schema, API integration, deployment config, all of it. And it works. It passes tests. The demo goes great. The stakeholders are impressed. Ship it.

Then six months later, you're debugging a memory leak at 2 AM and you realize the AI chose an in-memory caching strategy that made sense for the prototype's dataset but collapses under production load. Or it picked a database schema that's beautifully normalized but requires six joins for your most common query. Or it wrote an event-driven architecture when a simple queue would have been fine.

The problem isn't that AI makes bad choices. Often its choices are perfectly reasonable for the information it had. The problem is speed. When you write code manually, the slowness is a feature. You feel the friction of a bad abstraction. You notice the awkward data flow because you had to type it out. The act of writing *is* the act of thinking.

When an AI generates a full system in one session, that thinking never happens. You get the output without the process. And the process was where the understanding lived.

This is the exact trap that led us to build Jonggrang's [16-phase pipeline](https://jonggrang.dev/deck). It might seem like overkill, plan, implement, simplify, test, review, each phase with its own agent and its own quality gate. But the phases exist to reintroduce friction where AI removed it. The simplification phase, for example, revisits every changed file specifically to ask: "Is this complexity actually necessary, or did the AI just pick the first pattern that worked?"

It slows things down on purpose. That's the point.

---

## The Missing Mental Map

This is the emotional core of comprehension debt, and it's the part that's hardest to talk about without sounding like a luddite.

When you write code by hand, you're not just producing text. You're building a mental model. Every variable name you choose, every function you extract, every conditional you wrestle with, they all become part of a map in your head. You know where the data flows. You know why that edge case handler exists. You know which module is fragile and which one you can trust.

When AI writes the code, you get the artifact but not the map. And the map is what you need at 3 AM when the system is down and customers are waiting.

An [Anthropic study](https://www.anthropic.com/research/AI-assistance-coding-skills) tracked 52 engineers learning a new Python library and found that developers who relied on AI assistance scored 17% lower on comprehension assessments, averaging 50% compared to 67% for the hand-coding group. The steepest decline? Debugging skills. The exact skill you need most when AI-generated code breaks in production.

The [METR study](https://metr.org) took it further: in a randomized controlled trial with experienced open-source developers, AI tools made them 19% *slower* at completing tasks. But here's the kicker: those same developers *believed* they were 20% faster. The perception gap is the scary part. We're not just losing understanding, we're losing awareness that we've lost it.

I see this pattern constantly. A team ships fast for three months, then hits their first real incident. The mean time to recovery (MTTR) balloons because nobody has the mental map anymore. They're not debugging their own system. They're reverse-engineering a stranger's code under pressure. And that stranger is an AI that can't explain its reasoning.

---

## The Productivity Illusion

Let's talk about metrics, because the numbers look great on paper.

Pull requests are up. Deployment frequency has doubled. Test coverage is at 95%. Features ship in days instead of weeks. By every traditional measure, the team is crushing it. Engineering leadership puts together a slide deck. Everyone gets a pat on the back.

But what are we actually measuring?

[GitClear's research](https://www.gitclear.com/ai_assistant_code_quality_2025_research) analyzed 211 million lines of code changes and found that code churn, the rate at which newly written code gets revised or deleted, doubled from 3.3% to 7.1% between pre-AI baselines and 2025. Code duplication increased eightfold. Refactoring activity dropped from 25% of changed lines to less than 10%. We're writing more code, but we're also throwing more of it away and copying more of it wholesale.

The [DORA metrics](https://dora.dev), the industry standard for measuring engineering performance, are [increasingly misleading in the AI era](https://larridin.com/developer-productivity-hub/why-dora-metrics-break-ai-era). A team can improve deployment frequency because AI generates code faster while simultaneously increasing change failure rate because that code is harder to review and maintain. The metrics capture output volume but not output comprehension.

[Bain's 2025 research](https://www.bain.com/insights/from-pilots-to-payoff-generative-ai-in-software-development-technology-report-2025/) found that AI productivity gains stall at 10-15% because traditional metrics hide AI-specific problems: rework cycles, hidden technical debt, and the compounding cost of code that nobody truly owns mentally.

So I'll ask the uncomfortable question: Are we actually more productive, or are we just building bigger piles of code that we don't understand? Velocity is not the same as progress. You can run very fast in the wrong direction.

---

## Paying Down the Debt

Identifying the problem is the easy part. Here's what I think we can actually do about it.

### Enforce "Explain-It-To-Me" Reviews

Before any AI-generated code gets merged, the developer who requested it should be able to walk through it line by line and explain the reasoning. Not "what does this code do" but "why does it do it this way instead of another way." If you can't explain the trade-offs, you haven't reviewed it. You've just rubber-stamped it.

In Jonggrang, we enforce this through [deterministic review gates](https://github.com/porcupine-md/jonggrang): seven hooks that block agent exits until quality criteria are met. A "dirty bit" system tracks which code domains have been modified, and the pipeline won't proceed until each modified domain has passed both review and testing. The reviewer agent literally has to produce an explanation of the architectural decisions before the code can move forward.

### Write Architecture Documentation by Hand

Let AI write the code. But make humans write the documentation that explains *why* the architecture looks the way it does. Not auto-generated API docs, those are fine for AI. I mean the kind of document that answers: "Why did we choose this database? What are the scaling limits? What happens when this service goes down? What did we consider and reject?"

If you can't write that document, you don't understand your system well enough to operate it safely. The act of writing it forces the same kind of mental model building that coding by hand used to provide.

### Keep AI-Generated PRs Small

This one is simple but hard to enforce: don't let AI generate 500 lines of core business logic in a single PR. Break it into pieces that a human can actually digest. A 50-line PR that someone truly understands is worth more than a 500-line PR that passed CI but nobody really read.

Jonggrang handles this through [task decomposition](https://jonggrang.dev/deck): features are broken into atomic tasks, and each task runs with a fresh agent instance. Fresh context per task prevents the agent from accumulating stale assumptions, but it also keeps each unit of work small enough for humans to meaningfully review.

### Measure Comprehension, Not Just Output

Start tracking metrics that actually reflect understanding: MTTR during incidents (how fast can the team actually fix things, not just ship things), the ratio of churned code to stable code, how often recently shipped code needs hotfixes. These won't look as good on a slide deck, but they'll tell you whether your team is building on solid ground or quicksand.

### Reintroduce Friction Intentionally

This is the counterintuitive one. In a world where AI removes all friction from code generation, selectively reintroducing friction is a feature, not a bug. Code review cycles, architecture discussions, "why did we do it this way" sessions, these all slow you down. That's the point. The friction is where the understanding lives.

---

## The Paradox of Building Jonggrang

I'll be honest about the tension here. Jonggrang is a tool that helps AI agents write code faster and more reliably. Its entire reason for existing is to enable teams to ship AI-generated code at scale. And yet, a significant chunk of its architecture, the review gates, the simplification phase, the role boundaries, the fresh-context isolation, exists specifically to slow things down.

That's not a contradiction. That's the whole thesis.

The answer to comprehension debt isn't to stop using AI. That ship has sailed, and frankly, the productivity gains for well-understood tasks are real. The answer is to build systems that force understanding to keep pace with generation. Make AI write the code. Make humans understand it. And make sure your process doesn't let anyone skip that second part.

Because Addy Osmani is right: making code generation cheap doesn't eliminate the need to genuinely understand what you're shipping. It just makes it easier to pretend you do.

---

*References:*

- *Osmani, A. (2026). [Comprehension Debt](https://addyosmani.com/blog/comprehension-debt/)*
- *Anthropic. (2026). [How AI Assistance Impacts the Formation of Coding Skills](https://www.anthropic.com/research/AI-assistance-coding-skills)*
- *METR. (2025). [Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity](https://metr.org)*
- *GitClear. (2025). [AI Copilot Code Quality: 2025 Data Suggests 4x Growth in Code Clones](https://www.gitclear.com/ai_assistant_code_quality_2025_research)*
- *Larridin. (2026). [Why DORA Metrics Break in the AI Era](https://larridin.com/developer-productivity-hub/why-dora-metrics-break-ai-era)*
- *Bain & Company. (2025). [From Pilots to Payoff: Generative AI in Software Development](https://www.bain.com/insights/from-pilots-to-payoff-generative-ai-in-software-development-technology-report-2025/)*
- *Jonggrang. [AI Development Workflow Orchestrator](https://jonggrang.dev/deck) | [GitHub](https://github.com/porcupine-md/jonggrang)*
