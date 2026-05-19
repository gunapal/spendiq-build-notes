---
title: Building SpendIQ with a Multi-Agent Claude Code Workflow
description: How I architected and shipped a privacy-first Android finance app as one engineer plus six specialised AI agents, in 12 weeks.
---

# Building SpendIQ with a Multi-Agent Claude Code Workflow

*Notes from shipping a privacy-first Android finance app as a solo engineer, using Claude Code with six specialised subagents.*

—

I've spent the last three months building **[SpendIQ](https://gunapal.github.io/spendiq-legal/)** — an Android app that auto-tracks personal spending by parsing Indian bank and UPI transaction SMS entirely on-device. ~12,000 lines of TypeScript, 11 git commits, zero collaborators, beta launching this week. The interesting part of the story isn't the app itself — it's the *workflow* I used to build it. This post is a candid write-up of that workflow, what worked, and what didn't.

The TL;DR: I designed the architecture, owned every product and technical trade-off, and used Claude Code with **six custom subagents** as a force-multiplier for implementation, review, QA, and evaluation. I was not "vibe coding." I was running a pipeline.

---

## Why this matters

The conversation about AI in software development has mostly polarised into two camps:

- **Vibe coders** — prompt-and-pray, AI writes the code, you ship whatever comes out.
- **Skeptics** — AI is a stochastic parrot, no senior engineer should use it.

Both miss what I think is the actually-interesting middle ground: **using AI to extend the scope of what a single architect-level engineer can deliver, while keeping the architectural decisions, taste calls, and reviewability fully human.**

That's what I want to share. Not "look how cool AI is", but a concrete pattern for how a single engineer can run a multi-role software pipeline (PM → dev → reviewer → QA → release) by designing the agents carefully, scoping their work, and acting as the architect-in-the-loop.

---

## The setup

The project is a React Native 0.84 app on Android, TypeScript strict mode, local SQLite via `op-sqlite`, no backend. The technical surface includes:

- A SMS BroadcastReceiver pipeline parsing 15+ Indian bank and wallet sender formats
- A custom priority-rule engine (`PriorityRuleEngine`) for transaction categorisation with four rule types (Exact, Keyword, Regex, AmountRange)
- Budgets, savings goals, monthly Report Card with charts
- An optional, opt-in AI insights feature that sends a privacy-filtered monthly summary to an LLM (currently Anthropic Claude Haiku) — bring-your-own API key, hard-rate-limited, sensitive categories scrubbed pre-send

The tooling that made this tractable:

1. **Claude Code** as the main developer agent (I drive it in the terminal).
2. **Six custom subagents** scoped to specific roles, defined as Markdown files in `.claude/agents/`.
3. **Two project-rule files** in `.claude/rules/` (`code-style.md`, `testing.md`) that every agent reads and enforces.
4. **A `CLAUDE.md`** style file at the repo root that orients any new agent to the project's conventions, file structure, and "where things live."

Every agent has its own system prompt, tool allowlist, and trigger conditions. They're not chat sessions — they're roles in a pipeline.

---

## The agent roster

| Agent | Role | When it runs |
|---|---|---|
| `feature-brainstorm` | Explores the codebase, generates structured, opinionated feature ideas across UX/data/technical dimensions | Manually, when I start a new feature |
| `product-manager` | Reads the brainstorm doc, applies PM discipline (user stories, acceptance criteria, success metrics), produces a PRD | After `feature-brainstorm` |
| Main developer (me + Claude) | Implements the PRD, writing code that follows `code-style.md` and adds tests per `testing.md` | Continuous |
| `git-commit-reviewer` | Reviews every git commit for quality, standards compliance, missing tests, regressions | **Automatically after every commit** (via a hook) |
| `rn-qa-e2e-android` | Boots an Android emulator, installs the latest APK, runs all documented user flows, captures screenshots, files bugs | Automatically after commit, or manually before release |
| `sms-evaluator` | Runs the SMS parser against real and sample SMS corpora, scores categorisation accuracy, flags regressions | Manually before each release, or after adding a new bank |
| `user-flow-documenter` | Maintains a living document of every user flow + edge case, consumed by the QA agent as its test plan | Triggered after significant feature work |

Each agent has a one-line description and an example block in its frontmatter, so the main thread can auto-route work to the right subagent without me typing "use the X agent" every time. For example, here's the top of `sms-evaluator.md`:

```yaml
---
name: "sms-evaluator"
description: "Use this agent to evaluate SMS parsing quality and transaction
  categorisation accuracy against real or sample SMS data. Trigger manually
  before a release, after adding new bank senders, or when the user reports
  miscategorised transactions."
tools: Bash, Glob, Grep, Read, Write
model: sonnet
color: purple
memory: project
---
```

Tool allowlists matter. `sms-evaluator` cannot write code into `src/`. `git-commit-reviewer` cannot push to remote. `rn-qa-e2e-android` has ADB access. Each agent's blast radius is constrained by what it actually needs to do its job.

---

## A concrete walkthrough: shipping the "Spend Analysis & Goals" feature

I want to make this less abstract. Here's how the spend analysis + goals feature went from idea to merged.

1. **Brainstorm.** I typed *"brainstorm a spend analysis and goals feature"* into the main thread. It auto-routed to `feature-brainstorm`, which spent 90 seconds reading `src/`, then wrote a 30-KB markdown brainstorm doc covering UX flows, data shapes, technical risks, and three alternative approaches with trade-offs. I read it, killed two ideas, expanded one.

2. **PRD.** I asked the `product-manager` agent to turn the brainstorm into a PRD. It produced a 38-KB doc with prioritised user stories, acceptance criteria per story, and success metrics. I edited four things and approved.

3. **Implementation.** Back to the main thread. I broke the PRD into commits and worked through them. For each commit I wrote the high-level approach, Claude implemented, I reviewed the diff and tweaked. Tests were added inline because `testing.md` requires it for any change to `src/services/`.

4. **Commit review.** Every commit fires the `git-commit-reviewer` agent on the diff. It produces a 200-word report: lint compliance, test coverage delta, suspicious patterns, missing JSDoc. Two of those reviews flagged real bugs that would have made it to production.

5. **QA.** Before pushing the branch, I ran the `rn-qa-e2e-android` agent. It boots the emulator, installs the APK, navigates through documented flows from `user-flow-documenter`'s output, captures screenshots at each step, and updates a living bug report. Twice it found regressions in screens I wasn't actively touching — that's the value of having an automated regression pass without writing a single Detox test.

6. **Release-readiness.** Before tagging the AAB, `sms-evaluator` runs against ~600 real SMS messages I'd collected over months. It scores: capture rate, categorisation accuracy per category, and "high-confidence miss" cases that warrant new rules. If the score regresses by >2% vs the last release, I block the build.

That entire pipeline runs on my laptop. No CI, no Slack bots, no orchestration platform. It's just Markdown agent definitions, a hook in `settings.json`, and Claude Code.

---

## What the human (me) actually did

This is the part hiring managers care about, so I'll be specific. The agents do not own the product. I did all of the following:

- **Architecture.** Chose React Native vs native, op-sqlite vs Realm, the rule-engine pattern for categorisation, the SMS receiver vs polling, the BYO-API-key model for AI insights.
- **The privacy model.** Decided that no raw SMS leaves the device. Decided that sensitive categories (medical, personal care) get scrubbed from any AI request. Decided that AI is opt-in by default. Wrote the principles into the privacy policy myself.
- **Technical trade-offs.** Chose to keep `applicationId` and `namespace` decoupled when I had to rename packages, so I wouldn't have to move Kotlin files. Chose to enroll in Play App Signing so a lost keystore wouldn't kill the app forever. Chose to use the user's own LLM API key so I'd never operate a server.
- **Scope.** Said no to a lot of agent suggestions (cloud sync, "social comparison" features, gamification). Cut features the PM agent argued for. Killed the brainstormed "investment tracking" feature on day one.
- **Taste.** Reviewed every diff. Rewrote agent system prompts when their output was sloppy or evasive. Pushed back when agents tried to add abstractions for hypothetical future needs. Kept comments out of the codebase per project style.
- **Decisions about decisions.** Where to host the privacy policy, what to put in the Play Store screenshots, how to structure the testing tracks, whether to bundle vs split commits — all me.

The agents are very good implementers and very good reviewers. They are mediocre architects, weak taste-makers, and they will gladly build the wrong thing efficiently if you let them.

---

## What worked

1. **Specialisation beats generality.** A "do everything" agent is meaningfully worse than five focused agents with tight system prompts. The QA agent doesn't know how to write code, and that's a feature. The dev agent doesn't try to run E2E tests, because it lacks ADB.

2. **Rules files as the project conscience.** Every agent reads `code-style.md` and `testing.md` before doing work. This eliminates 90% of "please don't use any" / "please add a test" review comments. It's documentation that also functions as a linter.

3. **Auto-running review on every commit.** This is the single highest-ROI thing I did. The `git-commit-reviewer` agent runs in seconds on every commit, catches issues while context is fresh, and the friction is zero because it happens automatically. Two of its reviews caught real bugs.

4. **Evaluating the eval.** The `sms-evaluator` agent doesn't just run the parser — it produces a structured accuracy report scored against a real corpus. That makes parser improvements verifiable and parser regressions impossible to ship by accident. This is the kind of testing infrastructure I'd usually never bother to build for a side project.

5. **Brainstorm → PRD → code as separate steps.** Forcing each phase through a different agent with a different system prompt produced strictly better artifacts than one long thread. The brainstorm doc is opinionated and creative. The PRD is structured and disciplined. The code is conventional and well-tested. Same root prompt would have produced a soggy middle.

---

## What didn't

Honesty matters here.

1. **Agents will silently broaden scope** if you let them. The PM agent loved expanding feature surface. The brainstormer wanted to invent a "social spending circle" feature. I had to learn to read the output as a starting point, not a target.

2. **First-time builds fail nondeterministically.** Cleaning the gradle cache after an `applicationId` change triggered a flaky native-build failure that resolved on retry. Agents are not great at diagnosing "just retry" — they tend to spelunk into the cause. Manual override is sometimes faster.

3. **UI/UX is the weakest agent capability.** Layout, spacing, and visual hierarchy are still things I had to drive entirely myself. The agents would produce technically-correct screens that *looked* off.

4. **Multi-agent pipelines accumulate latency.** A brainstorm → PRD round trip is ~3 minutes of agent time. That's a lot if you change your mind midway. I learned to batch: hold ambiguity in my head longer before invoking the next agent.

5. **The model is non-deterministic.** Same prompt, different day, different output. I solved this with high-temperature creative work (brainstorming) and low-temperature structural work (code, reviews), but it's a real limitation.

---

## Lessons that generalise

If you're a senior engineer thinking about this kind of workflow:

- **Subagents are project artifacts, not personal tools.** Commit them to the repo. Version them. Treat them like CI configuration.
- **Constrain the tool allowlist per agent.** This is your blast radius. Agents without `Write` access can't damage code; agents without `Bash` can't run installs.
- **Make rules machine-readable.** A `code-style.md` that says "use absolute imports, no `any`, no inline styles" is enforced for free by every agent that reads it. Write the rules once, get compliance everywhere.
- **The architect-in-the-loop is irreplaceable.** AI doesn't have taste, doesn't know what to leave out, and doesn't push back on bad product decisions. You still need someone who does.
- **Build the eval, not just the feature.** The `sms-evaluator` agent transformed how I think about shipping. Every meaningful pipeline should have a scorer.

---

## What's next

Beta opens this week with ~10 testers via Play Store Internal Testing. Once feedback stabilises, I'll roll out to Closed Testing (~100 users) and submit the SMS Permissions Declaration. The next major feature on deck is offline LLM inference (running a small finance-tuned model on-device) so the AI insights feature stops costing money. Stay tuned.

—

*Built with [Claude Code](https://claude.com/claude-code) by [Gunapal Sripal](mailto:gunapal.spendiq@gmail.com). Source for the legal docs is at [gunapal/spendiq-legal](https://github.com/gunapal/spendiq-legal). SpendIQ source remains private through beta.*
