# 🐘/🐡 (Elephant/Goldfish): Claude Code plugin

A Claude Code plugin: five-stage workflow for software work, built around the elephant/goldfish pattern from [Dave Rensin's article](https://drensin.medium.com/elephants-goldfish-and-the-new-golden-age-of-software-engineering-c33641a48874).

> The **elephant** is your working session - Claude Code with full context: the conversation, CLAUDE.md, recent file reads, decisions already made. The **goldfish** is a fresh subagent with no prior context that stress-tests a problem doc, a design doc, or a diff. The asymmetry is the test: a goldfish that can't reach the same conclusion from the doc alone tells you the doc is wrong, not the goldfish.

## Install

```
/plugin install elephant-goldfish@claude-plugins-official
```

Five skills become available, namespaced under `/elephant-goldfish:`:

| Skill | When to use |
|---|---|
| `/elephant-goldfish:brainstorm <rough idea>` | Early-stage concept design. Multiple goldfish run in parallel with different lenses (technical / business / UX / contrarian / market research). Output: a concepts brief. |
| `/elephant-goldfish:prd <idea \| feature \| #issue>` | Turn an idea into a Product Requirements Document. Codebase grounding, structured gap-filling, deep research. Output: a PRD with explicit Open Questions. |
| `/elephant-goldfish:fix-bug <description \| #issue \| URL>` | Bug fix flow. Problem doc → goldfish diagnosis check → failing test → fix → precommit review → test gate. |
| `/elephant-goldfish:new-feature <description \| #issue \| URL>` | Feature flow. Scope confirm → design doc → three-goldfish design check (readiness / critic / implementer) → implement → precommit review → test gate. |
| `/elephant-goldfish:precommit-review` | Independent reviewer loop on the pending diff. Lint + typecheck + tests as pre-flight, then a fresh subagent reviews the diff cold. |

Implementation skills (`fix-bug`, `new-feature`) stop short of committing. You authorize the commit explicitly.

Usage examples:

```sh
/elephant-goldfish:fix-bug gh issue 42
/elephant-goldfish:new-feature gh issue 67
/elephant-goldfish:precommit-review
/elephant-goldfish:brainstorm "I have a an idea, but I don't know what to do with it."
/elephant-goldfish:prd "I need to implement X in Y, here is the description."
```

## The pipeline

> `brainstorm` produces a **concept**. `prd` turns a concept into **requirements**. `new-feature` and `fix-bug` produce **code**. `precommit-review` produces **validated code**. Each upstream stage feeds the next.

```mermaid
flowchart LR
    R(["rough idea"]) --> A["brainstorm"]
    A -- concept --> B["prd"]
    B -- PRD --> C["new-feature"]
    BG(["bug, issue, repro"]) --> D["fix-bug"]
    C -- code change --> E["precommit-review"]
    D -- code change --> E
    E -- validated change --> F(["commit"])
```

You don't have to start at the top. Pick the stage that matches what you have:

| You have | Start with | The output |
|---|---|---|
| A half-formed thought, no direction yet | `brainstorm` | A concepts brief; pick a direction. |
| A direction but no requirements | `prd` | A PRD: scope, users, metrics, open questions. |
| A clear feature to build | `new-feature` | Implemented + reviewed code, ready to commit. |
| A bug or a `#<issue>` | `fix-bug` | A failing-test-driven fix, ready to commit. |
| A diff already in hand | `precommit-review` | A reviewer-cleared diff, ready to commit. |

## How each skill uses the pattern

- **`brainstorm`** inverts the pattern. Multiple goldfish run in parallel, each with a different lens, free to web-search. The elephant synthesizes the divergent ideas into a concepts brief. All clarifying questions go through `AskUserQuestion`.
- **`prd`** uses two waves: exploration goldfish ground the request in the existing codebase, then research goldfish run in parallel across distinct lenses (web search, optional Chrome MCP for logged-in sources). The elephant synthesizes a PRD with explicit Open Questions for whatever the user deferred.
- **`new-feature`** uses **three** goldfish per round: comprehension (does the doc read cleanly to a cold reader?), critic (where does the design break?), readiness (could a first-pass implementer ship this without follow-up questions?). A no-code gate holds until critic AND readiness sign off; comprehension is informational. Round 2+ skips comprehension.
- **`fix-bug`** uses one goldfish to diagnose from only the symptom and repro. The elephant's hypothesis stays hidden; convergence buys confidence, divergence is signal. The bug is captured as a failing test before any fix.
- **`precommit-review`** is itself a goldfish. Sees only the diff, not the conversation. Findings triaged round by round with a hard cap and an `AskUserQuestion` escalation if the loop doesn't converge.

## Workflows

Each skill structures a different elephant↔goldfish dance. The diagrams below show the message flow. The **elephant** is your Claude Code session — full context, institutional memory. A **goldfish** is a fresh subagent spawned with no shared context, receiving only what the elephant hands it. The **user** is you, kept in the loop via `AskUserQuestion` at decision points.

### `brainstorm`

**Inverts the pattern.** Multiple goldfish run in **parallel**, each on a different lens (technical, business, UX, contrarian, market research). Their lack of shared context is what makes them generate divergent ideas. The elephant synthesizes the divergent output into a concepts brief and helps the user converge on a direction.

**Output:** a clusters → ranked picks → open questions brief; optional handoff to `/elephant-goldfish:new-feature`.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant E as Elephant
    participant G1 as Goldfish (Technical)
    participant G2 as Goldfish (Business)
    participant G3 as Goldfish (UX)
    participant Gn as Goldfish (Contrarian/Market)
    participant GC as Contrarian sweep

    U->>E: rough idea
    E->>U: Q1 stage / Q2 breadth / Q3 web research
    U->>E: framing answers
    E->>U: SEED (problem statement)
    U->>E: approve / refine / restart

    par Divergent lenses (parallel)
        E->>G1: SEED + Technical lens
        E->>G2: SEED + Business lens
        E->>G3: SEED + UX lens
        E->>Gn: SEED + Contrarian/Market lens
    end
    G1-->>E: concepts (lens complete)
    G2-->>E: concepts (lens complete)
    G3-->>E: concepts (lens complete)
    Gn-->>E: concepts (lens complete)

    opt Breadth = ~10 / ~20
        E->>GC: SEED + dedup'd concepts ("what did they all miss?")
        GC-->>E: 2-3 outsider concepts
    end

    E->>E: cluster, rank, surface convergence/divergence
    E->>U: CONCEPTS BRIEF + ranked picks
    U->>E: pick / re-run / save / drop
    opt Handoff
        E-->>U: "Run /elephant-goldfish:new-feature <concept> when ready"
    end
```

---

### `prd`

**Two waves of goldfish.** Wave 1 grounds the request in the existing codebase (parallel exploration goldfish). Wave 2 — after structured gap-filling Q&A with the user — runs research goldfish in parallel across distinct lenses. The elephant synthesizes a PRD with explicit Open Questions for whatever the user deferred.

**Output:** a PRD (executive summary, scope, requirements, metrics, risks, open questions, sources); optional save to disk and/or handoff to `/elephant-goldfish:new-feature`.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant E as Elephant
    participant GA as Goldfish (Existing surfaces)
    participant GB as Goldfish (Architecture)
    participant R1 as Goldfish (Market/Prior art)
    participant R2 as Goldfish (Technical patterns)
    participant R3 as Goldfish (UX/Compliance/Perf)

    U->>E: idea or #issue
    E->>U: Q1 depth / Q2 research / Q3 output target
    U->>E: framing answers

    par Wave 1 — codebase grounding
        E->>GA: find closest existing surfaces
        E->>GB: read CLAUDE.md, manifests, conventions
    end
    GA-->>E: file:line citations
    GB-->>E: stack, constraints, patterns
    E->>U: CODEBASE BRIEF + numbered gap list (G1..Gn)

    U->>E: which gaps to fill
    loop For each selected gap
        E->>U: structured Q (3-5 plausible answers + Defer/Other)
        U->>E: answer or defer
    end

    par Wave 2 — research lenses (parallel)
        E->>R1: SEED + brief + answered gaps (Market lens)
        E->>R2: SEED + brief + answered gaps (Technical lens)
        E->>R3: SEED + brief + answered gaps (UX/Compliance lens)
    end
    R1-->>E: findings + sources (lens complete)
    R2-->>E: findings + sources (lens complete)
    R3-->>E: findings + sources (lens complete)

    E->>E: synthesize PRD (deferred gaps → Open Questions)
    E->>U: full PRD
    U->>E: approve / refine sections / restart
    opt Output
        E->>E: write to disk / memory
        E-->>U: "Run /elephant-goldfish:new-feature <summary> when ready"
    end
```

---

### `new-feature`

**Three goldfish per round** stress-test the design doc the elephant drafted. Comprehension (does the doc read cleanly to a cold reader?), Critic (what gaps?), Readiness (could a first-pass implementer ship this without asking any questions?). A **no-code gate** holds until BOTH Critic and Readiness sign off (`design ready` + `implementation ready`). Round 2+ skips Comprehension. Implementation only starts after the gate closes; then the diff goes through `/elephant-goldfish:precommit-review`.

**Output:** implemented + reviewed code, ready for the user to commit.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant E as Elephant
    participant PA as Goldfish A (Comprehension)
    participant PB as Goldfish B (Critic)
    participant PC as Goldfish C (Readiness)
    participant PR as /elephant-goldfish:precommit-review

    U->>E: feature description or #issue
    E->>U: scope confirmation (1-2 sentences)
    U->>E: confirm / correct
    E->>U: DESIGN DOC (no code yet — gate is closed)

    rect rgb(245,245,245)
    Note over E,PC: Round 1 — all three passes
    par Three-goldfish design check
        E->>PA: design doc only (cold reader paraphrase)
        E->>PB: design doc only (find gaps)
        E->>PC: design doc only (executable in one pass?)
    end
    PA-->>E: "comprehension passed/unclear"
    PB-->>E: gaps + "design ready" or "design needs revision"
    PC-->>E: open questions + "implementation ready" or "not ready"
    end

    alt Both Critic & Readiness sign off
        Note over E: Gate opens — implementation allowed
    else Gaps remain
        loop Up to 3 revisions (skip Pass A)
            E->>E: revise doc, address every gap or rebut verbatim
            E->>U: revised doc
            par
                E->>PB: revised doc
                E->>PC: revised doc
            end
            PB-->>E: gaps / design ready
            PC-->>E: questions / implementation ready
        end
        opt Still not converging
            E->>U: stop — feature under-specified, need direction
        end
    end

    E->>E: implementation plan (layer-ordered)
    E->>E: implement layer by layer with per-layer verification
    E->>PR: hand off diff for independent review
    PR-->>E: rounds, fixes, rebuttals
    E->>E: run test gate + UI walkthrough (golden + edge case)
    E->>U: final report (STOP — no commit)
```

---

### `fix-bug`

**One goldfish diagnoses the bug** from only the symptom + repro. The elephant's hypothesis stays hidden until after the goldfish reports — convergence buys confidence; divergence is signal worth investigating. The bug gets captured as a **failing test before any fix is written**. Then the same diff goes through `/elephant-goldfish:precommit-review`.

**Output:** failing-test-driven fix, ready for the user to commit.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant E as Elephant
    participant GD as Goldfish (Diagnosis)
    participant PR as /elephant-goldfish:precommit-review

    U->>E: bug description / #issue / URL
    opt Triviality gate (typo, formatter, version bump)
        E->>PR: skip ceremony, go straight to review
    end

    E->>E: PROBLEM DOC (symptom, repro, hidden hypothesis, "fixed means")
    opt No repro provided
        E->>U: ask for repro path (URL, steps, log line)
        U->>E: repro details
    end

    Note over E,GD: Asymmetry: goldfish gets symptom + repro only,<br/>NOT the elephant's hypothesised root cause
    E->>GD: investigate, rank candidate root causes (no fix)
    GD-->>E: top 1-3 candidates with file:line + falsifying evidence

    alt Convergence — goldfish matches elephant hypothesis
        Note over E: Proceed with confidence
    else Divergence
        E->>E: re-investigate, update problem doc if goldfish is right
        E->>U: surface — goldfish flagged a different root cause
    end

    E->>E: write failing test capturing the bug
    E->>E: run test — must fail for the right reason
    E->>E: smallest fix that turns it green (no adjacent refactors)
    E->>E: re-run test — must go green

    E->>PR: hand off diff
    PR-->>E: rounds, fixes, rebuttals
    E->>E: test gate + re-verify original repro
    E->>U: final report (root cause, fix, test, goldfish agreement) — STOP
```

---

### `precommit-review`

**The reviewer is itself a goldfish.** It sees only the diff, not the conversation, not the implementation intent, not what the elephant was trying to do. Findings are triaged round by round: **fix or rebut verbatim** (no silent dismissals). The loop runs until `no findings` AND every prior-round finding is settled, with a **hard cap of 5 rounds** and structured user escalation if it doesn't converge.

**Output:** a reviewer-cleared diff with every rebuttal surfaced verbatim to the user.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant E as Elephant
    participant GR as Goldfish (Reviewer)

    Note over E: Pre-flight: lint, typecheck, unit, e2e, codegen<br/>(sequential, not chained — new errors only are blockers)

    rect rgb(245,245,245)
    Note over E,GR: Loop — hard cap 5 rounds
    loop Round N
        E->>GR: EXACT template (no intent leakage) + focus area if any
        Note over GR: Reads git status / diff / diff --cached /<br/>diff main...HEAD / log, reads touched files in full
        GR-->>E: numbered findings (file:line + why + fix) OR "no findings"
        E->>E: triage each finding — fix or rebut verbatim
        E->>U: ledger — open findings going into round N+1
        opt All settled AND "no findings"
            Note over E: Exit loop
        end
    end
    end

    alt Hit 5-round cap or repeat finding at same file:method
        E->>U: AskUserQuestion — accept / keep working / abandon
        U->>E: choice
    end

    E->>U: final report (rounds, fixes, rebuttals VERBATIM) — STOP, no commit
```

## Stack support

The plugin is **stack-agnostic**. On every invocation the skill reads your repo's manifests (`package.json`, `Gemfile`, `pubspec.yaml`, `pyproject.toml`, `go.mod`, etc.), version managers (`mise.toml`, `.tool-versions`, `.nvmrc`), CI config (`.github/workflows/`), and `CLAUDE.md` itself, then picks the right lint / typecheck / test / e2e commands for that repo. No install-time configuration.

Tested patterns include Rails (with mise + Brakeman + MiniTest), Flutter (with build_runner + Drift), Node + Vite + Cloudflare Workers, Python (Django / FastAPI), and Go.

## Project-specific commands

The skills include a routing hint in `new-feature`: if your repo has its own stack-specific commands (e.g. `/new-module`, `/new-migration`, `/new-worker`), the skill suggests them instead of running its generic feature flow. Project-specific commands stay in your `.claude/commands/` and don't need to be part of this plugin.

## Local development

```sh
git clone https://github.com/<your-fork>/elephant-goldfish-plugin
cd /path/to/your-test-repo
claude --plugin-dir /path/to/elephant-goldfish-plugin
```

Inside the session:

```
/elephant-goldfish:precommit-review
```

The skill should detect your test repo's stack and run the appropriate lint / test sequence.

## License

[MIT](LICENSE).
