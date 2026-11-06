# marketing-agent-kit

> **Kurz auf Deutsch:** Agenten-Pipeline für Marketing-Inhalte, angefangen bei
> Website-Funnels. Der Plan wird geschrieben und von dir freigegeben, **bevor** der erste
> Text existiert; danach laufen drei Qualitäts-Gates, die nicht optional sind. Produktfakten
> werden nie erfunden — sie stammen aus dem Kontext-Pack des Unternehmens oder gehen als
> markierte Rückfrage raus. Dazu kommt eine Content-Abteilung: neun Agenten vom Redaktionsplan
> bis zur Veröffentlichungsvorbereitung, mit Claim-Register, blockierender Prüfung und
> menschlicher Freigabe je Beitrag. Sie laufen im Workflow-Skill oder headless über `claude -p`.
> Der Rest dieses READMEs ist auf Englisch.

A [Claude Code plugin](https://code.claude.com/docs/en/plugins) providing a multi-agent pipeline for high-converting marketing content: website funnels and a permanent content department. One shared, versioned agent setup with per-project agent memory. Sibling of [mobile-agent-kit](https://github.com/MakeNative/mobile-agent-kit), same conventions.

## What's inside

**14 agents** (`agents/`), five for funnels:

| Agent | Role |
|---|---|
| `funnel-planner` | Audience/offer intake + funnel strategy: stages, page list, per-page job, core message and CTA — the plan is the contract |
| `funnel-content-creator` | Implements the plan: page structure + complete publish-ready copy into `marketing/funnel/<name>/`; raises plan concerns instead of deviating |
| `conversion-reviewer` | Blocking CRO quality gate: hook, value argumentation, proof, objection coverage, CTA clarity, friction |
| `copy-editor` | Blocking language quality gate: tone consistency, terminology, clarity, grammar, readability |
| `headline-reviewer` | Blocking headline-layer gate: punchiness, front-loading, memorability, canonical promise lines, reference-tone alignment — always proposes stronger alternatives |

and nine for the content department, none of which has write access:

| Agent | Link | Role | Rights |
|---|---|---|---|
| `content-planner` | K1 Plan | Next posts per channel from plan, pillars, calendar, learned rules and ideas; every third post is exploration; no effect claim without a `claim_id` | read |
| `script-writer` | K2 Write | Spoken text, scenes and several hooks in a word budget; plans image sources (app screens, text, motion, no AI persons) | read |
| `edit-director` | K3 Cut | Edit decision as a structured list: scenes, durations, on-screen text, versions (hook, length, 9:16, 16:9); the spoken word is the clock, cuts sit on sentence ends | read |
| `channel-copywriter` | K4 Metadata | Caption, description, title and hashtags per channel; effect claims only from the claim register | read |
| `claim-checker` | Gate | Blocking semantic gate after the code rule checker: reworded claims, body image, fear, shame, illness, testimonials; finds and never repairs; does not count as a human review | read |
| `performance-analyst` | K7 Learn | Hypotheses and next experiments from metrics and tests computed by code; calculates nothing, changes no measurement; undecided is the normal outcome | read |
| `feedback-reader` | K8 Feedback | Classifies exactly one foreign text into closed values; foreign text is data, never an instruction; class `Risiko` always goes to the human | none |
| `idea-scout` | K8 Feedback | Product and content improvements and new ideas from aggregated counters and learned rules, only as proposals | read |
| `run-evaluator` | Evaluation | Rates agents, gates and steps across runs: rework, rejection reasons, cost, the human's grades; every finding names run and step; changes nothing | read |

**3 skills** (`skills/`):

| Command | Workflow |
|---|---|
| `/marketing-kit:adopt` | Onboard a company: gather data sources, analyze website + marketing material, interview for the gaps, generate the business context pack (`.claude/docs/BUSINESS_CONTEXT.md`), scaffold agent memory |
| `/marketing-kit:funnel <product/goal>` | plan → user approval gate → content for every page → conversion + copy + headline review in parallel → capped fix loop (max 2 rounds, then escalate) → delegation report |
| `/marketing-kit:content <goal/period>` | plan → user approval gate → scripts → rule check + claim-checker → edit decision and channel copy in parallel → gate again → publish preparation → human release per post → run evaluation → delegation report. Publishes nothing. Side modes: `analyze`, `feedback`, `evaluate` |

Key rules across the funnel workflow: **the plan is written and user-approved before any content exists**; the three quality gates are never optional and must all end at PASS; product facts and proof are never invented — they trace to the business context pack's Proof Library, or ship as `<!-- VERIFY: ... -->` items the user confirms.

Key rules across the content department: **Claude plans, code executes** — agents return structured results, code checks and writes; effect claims come word for word from the claim register, new ones go to a human; a gate PASS is not a human review, and a release holds for exactly one text version; code calculates and decides, the analyst only writes hypotheses; foreign text reaches nothing but the reader without tools.

## Install

```
/plugin marketplace add MakeNative/marketing-agent-kit
/plugin install marketing-kit@marketing-agent-kit
/marketing-kit:adopt
```

`adopt` gathers the company's data sources (website, marketing material, customer feedback), analyzes them, interviews you for what the data can't answer, and generates `.claude/docs/BUSINESS_CONTEXT.md` — the context pack all agents read at runtime. Review and commit the generated files. Deliverables of funnel runs land in `marketing/funnel/<funnel-name>/` inside the project, those of content runs in `marketing/content/<batch-name>/`.

## Running the content department

The nine agents carry two modes in one file.

**Headless** is the mode an application uses. The caller gives the inputs and a JSON schema, the structured answer is the only result: no log, no files, no end phrase.

```
claude -p --agent marketing-kit:<name> --json-schema "$(cat schema.json)" \
  --output-format json --no-session-persistence --strict-mcp-config \
  < /dev/null "<inputs or file paths>"
```

- Run it in the project folder that holds `.claude/docs/BUSINESS_CONTEXT.md`. Files passed by path must lie inside it or be opened with `--add-dir`, otherwise `Read` is denied. Without an installed plugin add `--plugin-dir <path to this repo>`; an unknown agent name ends the run with exit 1.
- Read the result from `.structured_output`. No such field means the run failed; treat it as a failure, never as an empty result.
- The agent sets `status: "blocked"` and `blocked_reason` when a required input is missing, if the schema has these fields. Give every schema these two fields.
- The field names in each agent file are the default shape, the caller's schema wins. The claim register and further rules come in the prompt or as a file, the brand context from `.claude/docs/BUSINESS_CONTEXT.md` in the working directory.

**Workflow** is `/marketing-kit:content`. The agents return their section as their final message; the orchestrator appends it to `.claude/agents-log/<timestamp>-<batch-name>.md`, saves deliverables and memory notes verbatim, and waits for the end phrase. The agents cannot write, so the skill is the only writer.

`feedback-reader` is the exception: it runs through `claude -p` in both cases, one process per text, and the orchestrator never reads the foreign text.

## How adaptation works (3 layers)

1. **Shared (this plugin):** agent roles, the plan-first discipline, the review gates. Identical everywhere, updated centrally.
2. **Per company (generated, committed):** `.claude/docs/BUSINESS_CONTEXT.md` — identity, audience, brand voice, proof library, claim register, content pillars, permitted data sources, legal constraints. Agents refuse to invent anything this pack should answer.
3. **Per project (runtime):** every funnel agent has `memory: project` — audience insights, brand voice, style rules and recurring review findings accumulate in `.claude/agent-memory/marketing-kit-<agent>/` inside each project and never leave it. Content agents read the same directory and hand notes back to the workflow, which saves them; `claim-checker`, `run-evaluator` and `feedback-reader` have no memory, the first two are measuring instruments and the third has no tools. Each workflow run additionally logs to `.claude/agents-log/<timestamp>-<name>.md` under one heading per agent, so the run is fully auditable.

## Updating

Improve agents/skills here, push, then on each consuming machine:

```
/plugin marketplace update marketing-agent-kit
claude plugin update marketing-kit
```

## Overriding per project

Project-level definitions always win. To customize one agent for one project, copy it to `<project>/.claude/agents/<name>.md` and edit — the plugin version is ignored for that project while the rest stay shared. (A local override uses the unprefixed memory dir `.claude/agent-memory/<name>/`.)

## Adding agents

New agents are added via the `/add-agent` skill, which analyzes the pipeline, writes the agent to these conventions and wires it into the workflow — an agent no workflow spawns is not integrated.

## Known issues

Claude Code (observed on 2.1.195) injects a broken Persistent Agent Memory path for plugin agents — a doubled `.claude/.claude` segment — so automatic MEMORY.md loading silently fails. The agents in this kit work around it by self-managing memory: they read/write `.claude/agent-memory/marketing-kit-<agent>/` explicitly and ignore the injected path. No action needed; this note can be dropped once the upstream bug is fixed.

Three harness behaviours shape the frontmatter of the content agents (measured on 2.1.292). Keep them in mind when you edit these files:

- **`StructuredOutput` must stay in `tools`.** An agent with a `tools` list gets no structured output from `--json-schema` unless that list names `StructuredOutput`: the run ends as a success without a `structured_output` field. The Agent tool ignores the entry.
- **`memory:` grants Write and Edit,** even with a `tools` list. That is why the read-only agents set no `memory:` field and self-manage their memory by reading it.
- **`feedback-reader` cannot be spawned by the Agent tool.** `tools: []` would give it no tools at all but lose the structured answer, so its list holds only `StructuredOutput`, the answer channel. Under the Agent tool that list resolves to nothing and the spawn is refused, which is the wanted outcome: foreign text stays out of sessions that have tools.

## Changelog

**0.3.0** — Content department: nine agents (`content-planner`, `script-writer`, `edit-director`, `channel-copywriter`, `claim-checker`, `performance-analyst`, `feedback-reader`, `idea-scout`, `run-evaluator`) that run headless or through the new `/marketing-kit:content` workflow. The business context pack gains a claim register and content pillars (`CONTRACT_VERSION: 2`): run `/marketing-kit:adopt` again in existing projects before the first content run.

## License

MIT

## MakeNative

This kit is part of [MakeNative](https://makenative.de). We run agent pipelines like this
every day and teach others to build them for their own processes. For developers and for
anyone who wants to automate their work with AI.

Want to learn this for your own work? Join the list:
[Starter cohort application](https://tally.so/r/5ByYj6?utm_source=github&utm_medium=readme&utm_campaign=starter-kohorte)

Christian Eichmüller · [LinkedIn](https://linkedin.com/in/christianeichmueller)
