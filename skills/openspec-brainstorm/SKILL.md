---
name: openspec-brainstorm
description: Brainstorm an idea into an approved design before any OpenSpec artifact is written — classify the request as a spike, bounded, or architectural, explore the codebase, ask one question at a time, propose approaches with a recommendation, present the design in sections for approval, then hand the agreed brief to OpenSpec to create or update the change artifacts. Use when the user wants to brainstorm, explore, think through, or design something before or during an OpenSpec change, or before writing a proposal.
---

# OpenSpec Brainstorm

Turn an idea into a design the user has recognised and approved, then hand it to OpenSpec.

The dialogue is borrowed from superpowers' brainstorming: classify the request, understand the intent, propose approaches, present the design in sections, get approval. The output is not. There is no design document under `docs/`, no implementation plan, no spec file in this skill's own format. The approved design lives in the conversation until OpenSpec's artifact flow captures it — proposal, specs, design, tasks, or whatever the active schema asks for — in the shape `openspec instructions` dictates.

One exception: **an approved visual is a file, never a memory.** A mockup, screen sketch, or layout the user approved cannot be captured by prose, and a conversation is not read by whoever implements the change. Every approved visual is saved verbatim into the change and referenced from its design artifact — see [Visual references](#visual-references). A UI change handed off without them is a lost design.

Brainstorming is for thinking, not implementing. Read files, search code, run `openspec` read commands. Never write product code. Never write or edit an OpenSpec artifact before the design that feeds it is approved.

**Store selection:** If the user names a store (a standalone OpenSpec repo registered on this machine) or the work lives in one, run `openspec store list --json` to discover store ids, then pass `--store <id>` on the commands that read or write specs and changes (`new change`, `status`, `instructions`, `list`, `show`, `validate`, `context`). Hints printed by commands already carry the flag; keep it on follow-ups.

## Establish shared understanding

The outcome of brainstorming is an understanding the user can recognise and correct, grounded in what they want to accomplish.

1. **Discover intent.** From the request and the context available, identify the intended outcome, who it is for, and what success looks like. When that is missing, ask one focused question about purpose or intended use before proposing features or an approach. Knowing what kind of system it is does not tell you why the user wants the change.
2. **Write back your understanding.** Summarise the intended outcome, the constraints, and the success criteria in a short note the user can assess. Separate what they said from what you assumed. Incorporate their correction before treating the note as the brief.
3. **Carry intent into the design.** Every approach, feature, and technical choice gets checked against that understanding. The handoff brief restates it.

When the request already supplies purpose and constraints, reflect them back instead of asking the same questions again.

<HARD-GATE>
Before writing or editing any OpenSpec artifact, invoking a skill that creates artifacts (`openspec-propose`, `openspec-new-change`, `openspec-continue-change`, `openspec-ff-change`), writing product code, scaffolding, or installing dependencies, complete the selected path's prerequisites:

- Spike: the user approves the question and the probe.
- Bounded: the user approves the short in-chat design.
- Architectural: the user approves each design section, then the assembled brief.

A reply approves the stage actually presented. Approval of an idea or a scope does not approve a design that has not been shown yet. Resume at the earliest incomplete stage; never turn one approval into permission to skip the rest. Read-only exploration of the project and the OpenSpec change is allowed throughout.
</HARD-GATE>

## Three paths

Before the first question, classify the request and say the classification out loud — "this looks bounded, so I'll present a short design here and then hand it to OpenSpec" — so the user can override it.

| Path | Signal | What you present | Terminal state |
| --- | --- | --- | --- |
| **Spike** | A feasibility question — "can we", "is it possible", "quick and dirty is fine" — whose output is an answer, not code you keep | The question and what you will try, in 2–3 sentences | A reported recommendation. No change is created. Anything built is labelled throwaway. If a change is active, offer to record the finding in its design artifact. |
| **Bounded** | A well-scoped change to a flow that already exists in this repo: a new flag, a small endpoint, a one-file fix. Bounded measures the repo, not your familiarity — if there is no existing flow to read, it is not bounded | The clarifying questions that matter, then a short design in chat: approach, behaviour that changes, files touched, how to verify | Handoff to OpenSpec |
| **Architectural** | New projects, new subsystems, changes that restructure how components fit together or alter interfaces others depend on | The full process below: questions, 2–3 approaches, sectioned design, assembled brief | Handoff to OpenSpec |

In OpenSpec both bounded and architectural end in a change with artifacts. The active schema decides which artifacts exist, not the path. The path decides how much brainstorming happens before the handoff.

When in doubt between two paths, take the heavier one. The ratchet is one-way: hidden complexity discovered mid-brainstorm upgrades the path — stop, say so, and step up. Nothing downgrades mid-task.

## OpenSpec context

At the start, check what exists:

```bash
openspec list --json
```

**No relevant change.** Think freely. The handoff creates one.

**A change is named, or one is clearly relevant.** Resolve and read it before asking anything:

```bash
openspec status --change "<name>" --json
```

Read every file under `artifactPaths.<artifact>.existingOutputPaths`. Reference the artifacts naturally — "the proposal scopes this to premium users, but you are now describing everyone" — and treat them as the current contract. The handoff updates them.

A change that is partly implemented deserves one extra check: a design that changes behaviour a spec already states, or that a ticked task already built, affects finished work. Name that in the brief.

## The process

The subsections below serve the bounded and architectural paths; a spike stops at "present the probe, get a nod". From **Exploring approaches** onward is architectural depth — for bounded work, context plus a few questions plus a short in-chat design is the whole process.

**Understanding the idea**

- Check the project state first: files, docs, recent commits, agent instruction files.
- Before detailed questions, assess scope. If the request describes several independent subsystems — "a platform with chat, file storage, billing, and analytics" — flag it immediately. Do not spend questions refining details of something that needs decomposing first.
- Too large for one change: help decompose. What are the independent pieces, how do they relate, what order should they be built in? Each piece becomes its own OpenSpec change. Brainstorm the first one through the normal flow.
- For appropriately scoped work, ask questions one at a time. Prefer multiple choice — **AskUserQuestion** when the harness has it — but open-ended is fine. One question per message; a topic that needs more becomes several messages.
- Focus on purpose, constraints, and success criteria.

**Exploring approaches**

- Propose 2–3 approaches with trade-offs. Lead with your recommendation and why.
- YAGNI ruthlessly. Remove unnecessary features from every approach.
- Where existing code has problems that affect the work — a file that has grown too large, tangled responsibilities — include targeted improvements as part of the design, the way a good developer improves code they work in. Do not propose unrelated refactoring.

**Presenting the design**

- Present it in sections scaled to their complexity: a few sentences if straightforward, up to 200–300 words if nuanced.
- Ask after each section whether it looks right so far. Go back and clarify when something does not land.
- Cover: the behaviour that changes, architecture and components, data flow, error handling, how it will be verified.
- Design for isolation. Each unit has one purpose, a well-defined interface, and can be understood and tested on its own. For each unit you should be able to say what it does, how it is used, and what it depends on.

**Shaping the brief for the artifacts**

OpenSpec artifacts are requirement- and scenario-shaped, so the brainstorm must surface the material they need. Without naming artifacts in the conversation, make sure the design answers:

| Question the design answers | Where OpenSpec will want it (spec-driven schema) |
| --- | --- |
| Why now, and what is wrong today | proposal — Why |
| What changes, and what explicitly does not | proposal — What Changes, scope |
| Which capabilities are added or modified | proposal — Capabilities; one spec per capability |
| Observable behaviour, as "when X, the system does Y" | specs — requirements and scenarios |
| Technical decisions and alternatives rejected | design |
| The order work has to happen in, and what verifies each step | tasks |

Other schemas ask different questions; `openspec instructions <artifact> --change "<name>" --json` is the authority at handoff time.

**Visualise**

Use ASCII diagrams when a shape is clearer drawn than described: system sketches, state machines, data flows, dependency graphs, option comparison tables. A good diagram replaces paragraphs. Draw in chat; this skill has no browser companion.

**A mockup is optional; once approved, it is binding.** When a UI change is clearer shown than described, present a mockup with whatever the harness renders (an inline widget, an artifact, or ASCII when nothing renders). Ask for approval as its own question and record the exact answer, amendments included ("yes, plus the last-sync summary"). From then on it is part of the design and must be saved — see below.

## Visual references

Everything the user approved visually must reach the implementer exactly as approved. Inline widgets and chat drawings vanish with the conversation; skills downstream (`openspec-apply-change`, `openspec-orchestrate-change`, reviewers) read only the change directory.

1. **Keep the source.** When you present a visual, keep its exact source (widget HTML/SVG, artifact file, ASCII block). Do not redraw it later from memory.
2. **Save it at handoff**, once the change exists, verbatim, under `<changeDir>/mockups/`: `<slug>.html` or `.svg` for rendered mockups (wrap a bare widget fragment in a minimal `<!doctype html>` page so it opens in a browser), `<slug>.md` for ASCII. One file per approved visual.
3. **Write `<changeDir>/mockups/README.md`**: for each file, the screens and states it shows, the user's approval wording, every amendment made with the approval, and which details are binding (layout, hierarchy, labels, colours) versus illustrative (sample data).
4. **Reference them from the design artifact.** The design gets a `Visual references` section naming each file, the screens/states it binds, and the rule: the implemented UI matches these mockups; any deviation needs the user's approval. Specs stay behavioural — they describe what happens, the mockups describe how it looks.
5. **Verify after artifact creation.** When another skill wrote the artifacts (`openspec-propose`, `openspec-ff-change`, …), pass it the mockup paths in its arguments, then open the design artifact and confirm the `Visual references` section is there and names every file. Fix it before you report the handoff done.

With **Stop here**, nothing is written, so the approved visuals are not saved: say so explicitly, and offer to save them to a path the user picks.

## Handoff to OpenSpec

When the design is approved, write back the **brief** in one message. It restates the agreed understanding and is the input to every artifact:

```
## Design brief: <working title>

**Intent** — the outcome, who it is for, what success looks like
**What changes** — and what is explicitly out of scope
**Capabilities and behaviour** — each capability, with its observable behaviour
**Approach** — the chosen approach, decisions made, alternatives rejected and why
**Visual references** — each approved mockup (what it shows, the approval wording, amendments), or "none"
**Affects existing work** — artifacts, specs, or ticked tasks this contradicts (if a change exists)
**Open questions** — none, or each one explicitly deferred with who decides
**Verification** — how we will know it works
```

Then **AskUserQuestion** with the options that apply:

| Option | When | What happens |
| --- | --- | --- |
| **Create the change and generate all artifacts** (recommended for bounded and small architectural work) | No change exists | `openspec new change "<name>"`, then every artifact in dependency order until `applyRequires` is satisfied — the flow of `openspec-propose` or `openspec-ff-change`. Invoke that skill when available; otherwise run the CLI flow inline. |
| **Create the change and step through artifacts one at a time** | No change exists; the user wants to review each artifact | `openspec new change "<name>"`, then one artifact per turn — the flow of `openspec-new-change` and `openspec-continue-change`. |
| **Update the existing change** | A change exists | Edit the artifacts the brief touches, using the table below. Re-read `openspec instructions <artifact> --change "<name>" --json` for each, so the edit follows the artifact's template and rules. |
| **Stop here** | Always | The brief in chat is the record. Nothing is written. |

For an existing change, route each insight to its artifact:

| Insight | Artifact (spec-driven) |
| --- | --- |
| New requirement discovered | `specs/<capability>/spec.md` |
| Requirement changed | `specs/<capability>/spec.md` |
| Design decision made | `design.md` |
| Scope changed | `proposal.md` |
| New work identified | `tasks.md` |
| Assumption invalidated | whichever artifact held it |

Rules for the handoff, whichever option is chosen:

- **The brief is content; the schema is form.** Every artifact takes its structure from the `template`, `rules`, and `instruction` fields of `openspec instructions`, and is written to `resolvedOutputPath`. Never paste the brief verbatim as a proposal, and never copy `context` or `rules` blocks into an artifact.
- **Nothing in the artifacts that was not in the approved brief.** A detail the artifacts need but the brainstorm did not settle is a question back to the user, not an invention.
- **Every approved visual is saved and referenced** per [Visual references](#visual-references): files under `<changeDir>/mockups/`, a `mockups/README.md`, and a `Visual references` section in the design artifact. The handoff is not done until you have opened the design and seen it.
- **Verify before you report.** Every artifact written or edited goes through [Verify the artifacts](#verify-the-artifacts) — including artifacts another skill wrote.
- **The brainstorm ends at the verified artifacts.** Do not apply, implement, or archive. Say what comes next — `openspec-apply-change`, `openspec-apply-change-parallel`, or `openspec-orchestrate-change` — and stop.

## Verify the artifacts

Whenever the handoff wrote or edited artifacts, verify them before reporting the handoff done. Artifacts written by `openspec-propose`, `openspec-ff-change`, or `openspec-continue-change` are unchecked until you read them. With **Stop here** there is nothing to verify.

1. **Validate.** Run `openspec validate "<name>" --type change --strict` (with `--store <id>` when the change lives in a store). Fix each reported issue in the artifact that owns it and re-run until it passes. When a fix needs a decision the brief does not contain, ask — never invent content to satisfy the validator.
2. **Check against the brief.** Read every artifact the handoff wrote or touched (`openspec status --change "<name>" --json` → `existingOutputPaths`) with the brief beside it:
   - **Coverage** — every capability, behaviour, decision, and verification step in the brief appears in the artifact that owns it.
   - **No additions** — no requirement, scenario, decision, or task the brief did not settle. Remove it, or ask.
   - **Consistency** — the artifacts agree with each other: capabilities in the proposal match the spec directories, every requirement has tasks that build and verify it, the design contradicts no scenario.
   - **Precision** — no placeholders (`TBD`, `TODO`, unfilled template text), no scenario a tester could not check, no requirement with two readings.
   - **Existing work** — for an update, everything the brief named under "Affects existing work" is stated in the artifacts.
3. **Check visual references** — step 5 of [Visual references](#visual-references).
4. **Report.** In a few lines: what validation and the read-through found, and what you fixed. A gap that needs the user's decision is reported as an open question, never left silently in the artifacts.

## Red flags

| Thought | Reality |
| --- | --- |
| "This is too simple to need a design" | A bounded change gets two sentences in chat and a yes. That is the design. |
| "I'll call it bounded and skip the approaches" | Reaching for a label to skip work is the doubt. Take the heavier path. |
| "The design is obvious, I'll create the change while they read it" | The gate is the approval, not the design's length. Present, then stop. |
| "I know this kind of system, so it's bounded" | Bounded measures the repo. No existing flow to change means architectural. |
| "I'll write the proposal now and refine it as we talk" | Artifacts capture an approved design. Drafting them to think with is writing the spec with superpowers, not with OpenSpec. |
| "The spike works, so I'll keep the code" | A spike's output is an answer. Keeping the code is a new request — classify it. |
| "They approved the brief, so I can start implementing" | The brief approves artifact creation. Implementation is `openspec-apply-change`'s job and the user's call. |
| "The schema wants a field the brief lacks, I'll fill it sensibly" | Ask. The artifacts must not say more than the user agreed to. |
| "It grew, but we're nearly done — no need to re-classify" | Hidden complexity upgrades the path. Stop and say so. |
| "The user approved the mockup; the brief describes it in words" | A description is not the design. Implementers built a different UI from exactly such a description. Save the file. |
| "The mockup was shown inline; it's in the conversation" | Nobody downstream reads the conversation. If it is not under `mockups/` and named in the design, it is lost. |
| "`openspec-propose` wrote the artifacts, so they match the brief" | Another skill's output is unchecked. Validate it and read it against the brief. |
| "Validation passed, so the artifacts are right" | The validator checks structure, not that the artifacts say what the user approved. Do the read-through too. |

## Checklist

Classify first, announce the path, then work the items in order.

**Spike**

1. Explore project context — enough to frame the probe
2. Present question and probe plan — 2–3 sentences
3. Get approval — a nod is enough
4. Investigate — as cheaply as correctness allows
5. Report a recommendation — label anything built as throwaway; offer to record the finding in an active change's design artifact

**Bounded**

1. Explore project context — files, docs, recent commits; read the active change's artifacts if one is relevant
2. Ask the clarifying questions that matter — one at a time
3. Present a short design in chat — behaviour that changes, approach, files touched, verification; a mockup where the UI is clearer shown
4. Get approval — stop and wait for an explicit yes
5. Write the brief and hand off — create the change, or update the existing one; save every approved visual under `mockups/`
6. Verify the artifacts — `openspec validate --strict`, read them against the brief, confirm the design references every mockup; report what you found and fixed

**Architectural**

1. Explore project context — files, docs, recent commits; read the active change's artifacts if one is relevant
2. Check scope — decompose into separate changes if several independent subsystems are described
3. Ask clarifying questions — one at a time: purpose, constraints, success criteria
4. Propose 2–3 approaches — trade-offs and a recommendation
5. Present the design in sections — approval after each; a mockup the user approves becomes part of the design
6. Write the brief — assembled from the approved sections, with its Visual references; confirm it
7. Hand off — create the change and its artifacts, or update the existing one; save every approved visual under `mockups/`
8. Verify the artifacts — `openspec validate --strict`, read them against the brief, confirm the design references every mockup; report what you found and fixed; then stop before implementation
