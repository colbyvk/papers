# Building Software With Agents — a Field Guide

*A working framework for teams that want frontier coding agents to build, test and integrate a real product, written from what one
team learned building a data-engineering product for financial clients.*

---

> **Read this first: what this guide is sized for.**
>
> This framework can take a **smaller project** all the way to completion. It is written for a product small enough that you can
> hold the whole of it in your head: you know every promise it makes, every service it runs, and roughly where each piece of code
> lives. At that size, the board, the info lanes, the architecture pages, the laws and the three skills below are enough.
>
> **A much larger product needs more.** Once no one person can hold the whole of it, you will have to go deeper in several places.
> We had to. A few of the things we took a step further:
>
> - **Slices.** Each road through the product carries its own promises and the checks that prove them, so that "does it work end
>   to end" has an answer that a script computes.
> - **Decisions.** The few binding calls a human makes are kept apart from everything an agent writes, and agents build against them.
> - **More kinds of lanes.** Beyond the product's features, we write promises down for the stand-ins, the infrastructure, the runtime
>   surface and how each part recovers from a fault.
> - **Staged readiness.** Work climbs from "it runs" to "it holds up locally" to "it holds up in the cloud" to production, with a
>   gate at each step.
> - **Sized proof and shared machines.** Rules for which tests run when, and how many sessions can safely share one build machine.
>
> Those pieces are not in this guide. Each one costs real effort to set up, and each one only pays off past a certain size. Start
> here, and add them when the product outgrows what you can hold in your head.

---

## 0 · About this guide

### Where it comes from

This is a field study. It comes from the development my team and I ran while we built a data-engineering product for Wall Street
clients. The author's own usage is the evidence:
- about 600 billion tokens over roughly 115 days;
- of those, about 15% output, 30% input and 55% cache reads;
- development with AI since the summer of 2025.

### What it is for

It is the first framework a team needs before it can work this way. It says what to write down, where to put it, how an agent finds
it, and how a fleet of agent sessions turns it into a product without drowning in its own output.

### What it is not

It is not a prompt library, and it is not tied to one vendor. Every example is invented: the product, the paths and the names. The
shapes are what matter, and you will adapt them.

### The one idea

> **The agent does not remember your product. The repository does.**

Everything below follows from that sentence. A model starts every session with nothing, so what a product promises, how its code is
laid out and what rules it keeps must live in files the agent reads before it works. The files must be:
- **short enough to read in full**;
- **shaped the same way every time**, so that a human can audit them at a glance;
- **routed**, so that an agent reads only the page its task needs.

---

## 1 · The board: one home for promises, patterns and laws

Make one directory that holds everything an agent must know and nothing it can compute. We call it **the board**.

- Working in one repository: put it inside the repository, at `board/`.
- Working across several repositories: put it beside them, as its own repository, at the root of the project folder.

The board holds four families of pages. Each family has one job, and no page belongs to two families.

| Family | What it holds | Who reads it |
|---|---|---|
| **Info lanes** | what the product promises, one subject per lane | every agent that builds or judges a feature |
| **Architecture** | where code goes: patterns, lanes and service patterns, per repository | every agent that adds or moves a file |
| **Laws** | the rules of work that hold everywhere: test recursion, evaluation, runtime, writing | every agent whose task the law's condition names |
| **DevOps** | how the product is built, tested and shipped: the pipeline, and one page per test kind | every agent that adds a job, a test or a suite |

Laws and DevOps get the same treatment. Each family has:
- a `README.md` hub that only lists its pages;
- one page per subject, in one fixed shape (section 5);
- one condition per page in the agent file, so an agent reads only the page its task needs.

The file tree, for a project of three repositories:

```text
project/                              ← the project root; agents start here
├── AGENTS.md                         ← the agent file (CLAUDE.md, AGENTS.md or codex.md): every agent reads it first
├── .agents/                          ← the agent stack: skills, sub-agent definitions, hooks
│   ├── skills/
│   │   ├── product-documentation/SKILL.md
│   │   ├── scaffolding/SKILL.md
│   │   ├── development/SKILL.md
│   │   └── integration/SKILL.md
│   ├── agents/                       ← one file per sub-agent type: its role, its tools, its limits
│   └── hooks/                        ← scripts that refuse what text alone cannot stop
├── board/                            ← the board: documents only, no product code
│   ├── README.md                     ← the hub: headers and tables of links, nothing else
│   ├── products/
│   │   └── payments-ledger/
│   │       ├── README.md             ← the product hub: a table of its pages, each linked once
│   │       └── info-lanes.md         ← every info lane of the product, one page
│   ├── architecture/
│   │   ├── README.md
│   │   ├── ledger-api.md             ← one page per repository
│   │   ├── ledger-engine.md
│   │   ├── web-console.md
│   │   └── services/
│   │       └── telemetry.md          ← one page per cross-repository system
│   ├── laws/                         ← one page per rule of work
│   │   ├── README.md                 ← the hub: a table of the laws, each linked once
│   │   ├── test-suite-recursion.md
│   │   ├── evaluation.md
│   │   ├── runtime.md
│   │   └── writing.md
│   ├── devops/                       ← one page per pipeline and per test kind, the same shape as a law
│   │   ├── README.md                 ← the hub: a table of the devops pages, each linked once
│   │   ├── ci-cd.md
│   │   ├── test-placement.md         ← where every test file goes and how it is named
│   │   ├── unit-tests.md
│   │   ├── integration-tests.md
│   │   ├── regression-tests.md
│   │   ├── smoke-tests.md
│   │   ├── e2e-tests.md
│   │   ├── property-tests.md
│   │   └── environment-variables.md
│   └── schemas/
│       ├── info-lane.md              ← the shape every info lane keeps
│       └── architecture-page.md      ← the shape every architecture page keeps
├── ledger-api/                       ← product repositories, side by side
├── ledger-engine/
└── web-console/
```

Five rules hold the board together:

1. **The code outranks the document.** When a page and the code disagree, the code wins. The disagreement is a finding, so you report
   it and correct the page. You never quietly bend the code to an old sentence.
2. **One home per fact.** A fact lives on one page. Every other page links to it and never copies it. A copy drifts.
3. **A schema per family.** Every page of a family has the same shape. The shape lives in `board/schemas/`, and the skill that writes
   the page reads it.
4. **Nothing the code can compute is typed.** A count, a state ("built", "70% done"), a line number or a date in a lane goes stale the
   day it is written. A script computes those, or nobody writes them.
5. **A hub only points.** A hub page is headers and tables of links. Each page is linked once. No prose explains the links: the
   linked page explains itself.

---

## 2 · Skills: state the task, not the method

A skill is a packaged instruction that an agent loads when the task matches it. You know them already. Here is what makes one good.

### What a good skill does

- **It states the task, not how to do it.** Current frontier models plan the work well on their own. A skill that scripts the steps
  makes the agent follow your steps even when they are wrong for this case. Say what the result is.
- **It says what the agent works with.** Name the pages it must read, the files it may write and the commands that prove the work.
- **It says what not to do.** This is the most valuable part. The "never" list is where you encode the mistakes an agent makes on
  instinct: weakening a test to reach green, editing a generated file, writing a page outside its family.
- **It ends with a done-when that a command can check.** "Looks right" is not a done-when. "`lanecheck.py` exits 0" is.
- **It routes.** If a neighbouring skill owns part of the job, it names that skill instead of doing the job.

### An example skill

```markdown
---
name: product-documentation
description: Write or update a product's info lanes and architecture pages, each on its schema.
  Use when a product's promises or its code layout change.
---

# /product-documentation — the product's promises and patterns, on schema

## Goal
Every subject the product promises has one info lane on `board/products/<product>/info-lanes.md`, and every
repository the product touches has an architecture page that matches its code at HEAD.

## Read first
- `board/schemas/info-lane.md` — the lane's shape. This skill never restates it.
- `board/schemas/architecture-page.md` — the page's shape.
- the code at HEAD of every repository the lane names. The code outranks every page.

## Done when
python3 board/tools/lanecheck.py <product>      → 0 refusals, exit 0
python3 board/tools/linkcheck.py               → no new broken link

## Never
- type a state, a count, a date, a commit sha or a line number into a lane: a script computes them
- copy a fact that lives on another page: link to it
- delete a lane: retire it with its id, so that the id is never reused
- edit a generated region or a rendered page by hand
- write a lane for a promise the code cannot hold: report it as a gap instead

## Routes
- a fact from outside the repository → /research
- a call between two options that no rule settles → return it in chat, with a default
```

---

## 3 · Info lanes: the product's promises

An info lane is one subject the product promises: what it is, how it works, the one rule that decides it. All of a product's lanes
live on **one page**, grouped by theme, each lane on the same schema.

### The schema

```text
THE LANE — a heading, the head lines, one Summary. Nothing after it: no table, no state, no checklist

  ## LANE_<n> — <the subject, a few words>   <id>
  **Repos:**            the repositories it describes
  **Product:**          the product that owns it
  **Group:**            one group the page declares
  **Interworks with:**  the other products or lanes that depend on it, or none
  **Summary:**          one paragraph, 4 to 8 sentences, each with a job, in this order:
                          1   what it is, against the thing beside it
                          2–3 how it works: the two or three facts a reader must hold
                          4   the design choice, and what it buys
                          5   "The one rule:" — the single sentence that decides every edge case
                          6–8 (optional) the limits, the refusals by name, what it never does

THE ID — short and stable, never reused
  A lane keeps its id through every edit, and a test that proves the lane names that id.

THE BAN — what never goes in a lane
  a count the code owns · a state word ("built", "partly done") · a date · a commit sha · a version number · a line number ·
  a plan or ticket number · a table of checks
```

Why so strict? Agents write exaggerated, prose-heavy output, and every lane in a different shape cannot be audited. A fixed schema
fixes three things at once:
- **the length**: an 8-sentence ceiling stops a lane from becoming an essay;
- **the order**: a reviewer always finds "The one rule:" in the same place;
- **the content**: the ban keeps out everything that goes stale.

### Where the checks go

A lane is **intent**. A check is **code**. Do not put a table of acceptance criteria under a lane. We did, with thousands of rows,
and they went stale faster than anything else on the board: no test was bound to them, and nothing failed when they became false.

What a lane promises is proved by a test that can fail, and the test names the lane's id in a comment. Any list of "what proves this
lane" is then **generated** by searching the code for those comments. It is never typed.

```rust
// proves: a7c3e910 — a posting that would unbalance the ledger is refused by name
#[test]
fn an_unbalanced_posting_is_refused() {
    let err = post(&ledger(), entry(debit = 100, credit = 90)).unwrap_err();
    assert_eq!(err.name(), "UnbalancedPosting");
}
```

### A full example page

````markdown
# Payments Ledger — info lanes

## The groups

| Group | What it holds |
|---|---|
| posting | how money enters the ledger and the rules every entry keeps |
| reading | how a client reads balances and history, and what it never sees |

# Group · posting

## LANE_1 — the double-entry posting   a7c3e910

**Repos:** ledger-engine · ledger-api
**Product:** Payments Ledger
**Group:** posting
**Interworks with:** Reconciliation

**Summary:** A posting is the only way money enters the ledger, unlike an adjustment, which corrects an
entry that already exists. Each posting carries a debit leg and a credit leg in one currency, and the
engine commits both legs or neither inside one transaction. The API assigns the posting key before the
engine sees it, so a retried request finds its first result instead of writing twice. Keeping the key
outside the engine means the engine stays a pure function of the entries it is given, and every retry
policy lives in one place. The one rule: the sum of every account's legs is zero after every commit.
A posting that would break it is refused as `UnbalancedPosting`, and a mixed-currency posting is refused
as `CurrencyMismatch`.

## LANE_2 — the posting key   c20f5d71

**Repos:** ledger-api
**Product:** Payments Ledger
**Group:** posting
**Interworks with:** none

**Summary:** The posting key is the client-supplied identity of one intended posting, unlike the internal
entry id, which the engine assigns on commit. The API records the key and the request's hash before any
side effect, then forwards the posting. A second request with the same key and the same hash returns the
first result, and a second request with the same key and a different hash is refused as
`KeyReusedWithDifferentBody`. The key lives for the retention window the client contract names, so a
retry storm after an outage cannot double-post. The one rule: one key, one posting, forever.

# Group · reading

## LANE_3 — the balance read   9b41e6aa

**Repos:** ledger-api · web-console
**Product:** Payments Ledger
**Group:** reading
**Interworks with:** Reconciliation

**Summary:** A balance read returns an account's balance as of a point in time, unlike the history read,
which returns the entries themselves. The API answers from the committed entries at or before the
requested instant, never from a cache that a later posting could have moved. A read names its instant
explicitly, so two readers asking about the same instant always agree. The one rule: a balance is a pure
function of the committed entries and the instant, and nothing else.
````

---

## 4 · Codebase architecture: where code goes

Agents on a large codebase re-create services that already exist, because the place they are working in does not show the existing
one. An architecture page per repository stops that. It answers one question before the first file is written: **where does this
code go?**

Each page holds three kinds of block, all in codeblocks, all on one schema.

### 4.1 · Common patterns: the names every directory keeps

```text
COMMON PATTERNS — the directory names every service keeps, in every repository

  core/            pure logic: no I/O, no clock, no network. Takes values, returns values
  source_routers/  everything that READS from a store or an outside system (a database, a bucket, a vendor feed)
  sink_routers/    everything that WRITES to a store or an outside system
  dmz/             the contracts: the types two sides of a wire share. It imports nothing. A wire's contract is
                   mirrored on both sides in the same commit
  routers/         the doors: request handlers, CLI entry points, job entry points
  hydration/       what shapes data read from a store into what a screen or a caller shows
  middleware/      what wraps every door: authentication, sessions, request ids, rate limits
  runtime/         what a shipped program tells its operator: exit codes, event names, the log line's fields

  THE IMPORT WALL — imports go one way only
    middleware → routers · source_routers · sink_routers · hydration → core → dmz
    A lower layer never imports a higher one. A service never imports a sibling service: it goes through a contract.
```

### 4.2 · Lanes: where a service lives

An architecture lane is the one directory where everything that does one task lives. When work touches that task, it goes to the
lane, never to the nearest convenient folder.

```text
ARCH_LANE · telemetry — the one home of the product's telemetry

  Lives at        ledger-api/src/core/telemetry/
  Owns            the event types, the emitter, the sampling rule, the redaction of secrets
  Does not own    where events are shipped (infrastructure), the dashboards (web-console)
  Add here        a new event type, a new field on every event, a new redaction rule
  Never           a second emitter anywhere else; a log line that bypasses the emitter
```

### 4.3 · Service patterns: a lane's service used elsewhere

A lane holds the main service. Sometimes another part of the product needs that service shaped for its own job. Do not copy the
lane, and do not grow the lane with every caller's special case. Write a **service pattern**: a thin directory inside the caller that
uses the lane's service and adapts it.

```text
SERVICE_PATTERN · telemetry inside the bucket connection

  The lane        ledger-api/src/core/telemetry/              ← the main service, unchanged
  The pattern     ledger-api/src/core/bucket_connection/telemetry/
  Why             while a client connects their own storage bucket, they must read the connection's events and
                  errors live; the main lane ships events to operators, not to clients
  What it adds    a client-visible event filter, and a stream the connection screen reads
  What it reuses  the lane's event types, emitter and redaction, imported, never copied
  Never           a second emitter; an event type defined here instead of in the lane
```

### 4.4 · The placement rules: read before the first file

Open each repository's page with a numbered block that an agent reads first. One line is one decision.

```text
HOW TO PLACE CODE · read this first

1   A new feature in a known service   → that service's lane. Never a new top-level directory for it
2   A read from a store                → <service>/source_routers/. A write → <service>/sink_routers/
3   A type two services share          → dmz/, a directory named for the pair. One service's own type stays in that service
4   Opening a database connection      → only the persistence lane does it; a router asks the persistence lane
5   A new environment variable         → read once at start by the program's one config reader, passed down as a value
6   A rule a client must never bypass  → middleware/, with an adversarial test beside it
7   Code that is only for tests        → tests/, mirroring the path of the code it tests
8   Nothing fits                       → stop and say so; a new pattern is written on this page before the code
```

---

## 5 · DevOps: one page per pipeline and per test kind

DevOps gets the same treatment as the laws (section 7). It is its own directory, `board/devops/`, with a hub that only lists its pages
and one page per subject. Each subject is one pipeline, one test kind, or one rule of the build such as environment variables. The
agent file routes to each page by its own condition, so an agent that adds a unit test reads the unit-test page and nothing else.

### 5.1 · The page shape both families keep

Every devops page and every law page has the same four parts, in this order.

````markdown
# <Subject>

<Two or three sentences: what this page governs, and the failure it exists to prevent.>

## Recursion
<How this subject keeps up as the product grows: what a change must add here in the same commit, what it
must delete when its source leaves, and when an agent re-reads the whole subject to prune it.>

## The rules
```text
<THE SUBJECT IN CAPITALS>
  <rule name>     <one rule per line: the condition, then what the agent does>
```

## Never
- <the mistakes an agent makes here on instinct, one per line>
````

Recursion comes first because it is the part teams leave out. A page that says how to add a test but not when to delete one grows a
suite nobody trusts.

### 5.2 · The hub

````markdown
# DevOps

## The pipeline

| Page | What it governs |
|---|---|
| **[CI/CD](./ci-cd.md)** | every workflow: the doors, the reusable jobs, the gates a merge needs |
| **[Environment Variables](./environment-variables.md)** | how every program reads its settings, and where secrets live |

## The test suites

| Page | What it governs |
|---|---|
| **[Test Placement](./test-placement.md)** | where every test file goes and how it is named |
| **[Unit Tests](./unit-tests.md)** | one function, plain values, no I/O |
| **[Integration Tests](./integration-tests.md)** | one seam, the real dependency or a faithful fake of it |
| **[Regression Tests](./regression-tests.md)** | every defect becomes a test before it becomes a fix |
| **[Smoke Tests](./smoke-tests.md)** | the first question asked of a fresh deployment |
| **[E2E Tests](./e2e-tests.md)** | one user journey through the running product |
| **[Property & Fuzz Tests](./property-tests.md)** | invariants over generated input |
````

### 5.3 · The CI/CD page

````markdown
# CI/CD

Every workflow the product runs, as one tree. It exists so that a gate cannot be skipped and a job cannot be
copied into a second repository and drift.

## Recursion
Every change that adds a deployable, a language, a test kind or a gate adds its job here, in the same pull
request. A job is written once as a reusable workflow and called by every repository; a repository never
copies a job body. When a gate is added, it is required by name, so a merge without it is impossible. When a
deployable is removed, its jobs leave in the same pull request. Every quarter, an agent lists each job's
minutes and its last red, and deletes or merges the jobs that never fail and never gate.

## The rules
```text
THE TREE
  .github/workflows/pr.yml       the pull request door: runs only what the diff can break
  .github/workflows/merge.yml    after the merge: the full suite of each changed repository, once
  .github/workflows/nightly.yml  the long work: property runs, fuzzing, determinism replays
  .github/workflows/deploy.yml   build, deploy, smoke test; a red smoke test rolls back
  reusable/<language>-check.yml  one body per language, called by every repository

A NEW JOB
  Called by      the door that needs it, by name and pinned version
  Selects        only the targets the diff reaches (an "affected" script), never the whole workspace on a PR
  Required       yes, if it guards correctness; it is added to the branch rule in the same change
  Bounded        a time limit; a job over it fails, it does not wait
```

## Never
- a step that always passes, or a job with no target
- a job body copied into a second repository
- a gate added but not required by name
````

### 5.4 · The test placement page, and one page per test kind

The placement page decides where every test lives, so that an agent never invents a folder.

```text
TEST PLACEMENT
  The mirror      every test sits under tests/, at the path of the code it covers
  The name        test_<kind>_<subject>, kind one of: unit · integration · regression · smoke · e2e · property
  One subject     one file per source file (unit) or per seam (integration) or per defect (regression)
  A new kind      is a new devops page first, then its tests
```

Each test kind then has its own page in the shape of 5.1. Here are their rules blocks.

```text
UNIT TESTS
  One file        per source file: tests/<mirror of the source path>/test_unit_<subject>
  Tests           one function, plain values, no I/O, no clock, milliseconds
  Proves          each promise of the function, and its refusals by name
  Mutation        each test is run once against a planted bug in the code it covers; a test that stays green is rewritten
  Recursion       a unit file is deleted only in the commit that deletes its source file
```

```text
INTEGRATION TESTS
  One file        per seam: tests/<mirror>/test_integration_<seam>
  Far side        the real dependency in a container, or a faithful fake of it, reached through the production code path
  Owns            its own state; cleans up on exit; never shares a database with another test
  Never           an in-process mock of the thing the seam exists to talk to
```

```text
REGRESSION TESTS
  Born            before the fix: red at the commit that had the defect, green at the fix
  Named           as the defect's sentence: test_regression_<what_went_wrong>
  Lives           beside the code where the defect lived
  Never           pruned while the behaviour it guards still exists
```

```text
SMOKE TESTS
  Asks            one question of a fresh deployment: did it come up coherent?
  Checks          alive · connected · one seam wired end to end · the build is the one we meant to ship
  Budget          sixty seconds; a red rolls the deployment back; a missing smoke test counts as red
```

```text
E2E TESTS
  A road          one user journey: its actions, its expected outcomes, its named failures
  Runs            against the real app: the frontend served, the backend live, a browser driving it
  Evidence        each road keeps its own evidence (logs, screenshots, responses)
  Recursion       a road whose journey the product no longer offers is deleted with the feature
```

```text
PROPERTY & FUZZ TESTS
  When            parsing, encoding, and arithmetic on money or time
  Shape           an invariant over generated input; a failure is shrunk to its smallest case and saved as a regression
```

```text
ENVIRONMENT VARIABLES
  One reader      each program reads its settings once at start, in one place, and passes values down
  One file        a committed .env per repository with safe local defaults; secrets only in an ignored .env.local
  Deployed        the deployed task definition sets the same names; a secret comes from the secret store, never a file
  Recursion       a removed setting leaves the reader, the .env and the deployment in the same commit
```

---

## 6 · The agent file: how CLAUDE.md, AGENTS.md or codex.md is written

The agent file is the one page every agent and sub-agent reads before it works. Most teams fill it with prose and advice. Ours is
almost all **codeblocks**, and each block is one situation: what the agent must know, or read, when it is doing that thing.

### 6.1 · The rules for the file

1. **Condition first, then the page.** "If you add a test file, read the test architecture page." The agent scans for its own
   situation, and finds it.
2. **One situation per block.** Do not mix the test rules into the deploy block.
3. **Exact paths.** A wrong path is worse than no path. A script checks every path in the file.
4. **The file routes, and the pages hold the law.** The agent file says *which* page to read, and the page says what to do. When the
   law changes, only its page changes.
5. **Write it once, generate the copies.** If several tools read different file names, keep one source on the board and generate the
   others with a script. A hand-kept second copy drifts.
6. **Make the agent prove it read the law.** Ask it to name the page it read and the one rule that page imposed on this change. That
   sentence is cheap, and it shows a session that worked from memory.

### 6.2 · An example agent file

````markdown
# AGENTS.md — read before any work

```text
STEP 0 — YOUR ROLE
  A dispatch that opens with "ROLE: <NAME> SUB-AGENT" makes you that sub-agent: your definition in
  .agents/agents/<name>.md is your law, and you never orchestrate.
  A session the human started with /development or /integration is an ORCHESTRATOR: you write the code
  yourself, and you hand a sub-agent only a task you judged big and separate enough.
  Anything else: you are a builder working one task.
```

```text
THE HARD STOPS — never, whatever the task says
  1  Never merge into main. Merging main is the human's click.
  2  Never apply infrastructure (terraform apply, a deploy). Author and test it only.
  3  Never weaken, skip or delete a test or a lint rule to reach green. Fix the code.
  4  Never `git add -A`, never `git stash`. Commit your own files by path.
  5  A secret in a diff stops everything. It never leaves the machine.
```

```text
ALWAYS — before any work
  The code outranks every page: when they disagree, the code wins and you report the disagreement.
  A path that is missing or moved is flagged, never rebuilt from memory.
  Report bad news first. Every claim says how you know it: ran it · read it · inferred it.
```

```text
ARCHITECTURE — before you add, move or name a file
  Read the repository's page, its "HOW TO PLACE CODE" block first:
    board/architecture/ledger-api.md
    board/architecture/ledger-engine.md
    board/architecture/web-console.md
  If the change touches telemetry in any repository, also read:
    board/architecture/services/telemetry.md
```

```text
PRODUCT — before you build or change a feature
  Read the info lanes the task names, whole: board/products/<product>/info-lanes.md
  A promise that the code cannot keep is reported, never quietly narrowed.
```

```text
LAWS — the condition first, then the page
  If you change any code, read board/laws/test-suite-recursion.md: every change updates its tests in the same commit
  If you change what a shipped program logs, emits or exits with, read board/laws/runtime.md
  If you write a report, a pull request body or a final message, read board/laws/writing.md
```

```text
DEVOPS — the condition first, then the page
  If you add, move or name a test file, read board/devops/test-placement.md
  If you write a unit test, read board/devops/unit-tests.md
  If you write a test across a seam, against a real dependency or a fake of it, read board/devops/integration-tests.md
  If you fix a defect, read board/devops/regression-tests.md
  If you write the probe a fresh deployment runs, read board/devops/smoke-tests.md
  If you write or change a user journey test, read board/devops/e2e-tests.md
  If you change parsing, encoding, or money or time arithmetic, read board/devops/property-tests.md
  If you add or change a pipeline job, read board/devops/ci-cd.md
  If you add, change or remove an environment variable, read board/devops/environment-variables.md
```

```text
DONE — before you say a task is done
  Read board/laws/evaluation.md, and run what it names.
  Your final message lists: the files changed · the tests and their pass counts · the info lanes worked ·
  the architecture blocks used · what you did not do, and why.
```
````

---

## 7 · Laws: one page per rule of work

A law is a rule of work that holds everywhere, beside the architecture and on top of it. Laws get the same treatment as DevOps:
their own directory, `board/laws/`, a hub that only lists them, one page per law in the shape of section 5.1 (purpose, recursion, the
rules, never), and one condition per law in the agent file. Four laws paid for themselves many times over. Below are their rules
blocks.

### 7.1 · Test suite recursion

```text
TEST SUITE RECURSION — every change keeps its suites true, in the same commit

  Read the diff, not your memory     git diff --name-status <base>..HEAD lists what changed; each changed source
                                     file answers: which tests cover it, and are they still right?
  Each changed file gets an answer   WRITE (no test covers the new behaviour) · UPDATE (a test asserts the old
                                     behaviour) · MERGE (two tests now assert one thing) · MOVE (the source moved) ·
                                     DELETE (the source left the product in this same diff) · NONE (stated, with why)
  A deletion needs its cause         a test is deleted only when the code it covered left in the same diff
  A failing test leaves the loop     a red test is never skipped to finish the task; it goes to its root cause
  The rolling sample                 each change also re-reads the ten test files changed least recently, and fixes
                                     or deletes what has rotted
  The ledger                         the final message lists every answer above, one line per file
```

### 7.2 · Evaluation

```text
EVALUATION — how an agent proves the product works, in any repository

  The real thing, not the test      the backend runs, the frontend is served, and a browser tool clicks the button;
                                    the agent reads what happened, not what the code says should happen
  One canonical job                 one realistic job runs end to end, through a throwaway harness that is purged after
  Six questions                     it started (invoke) · it gave the right answer (correctness) · it gives the same
                                    answer twice (reproducibility) · it wrote nothing it should not (integrity) · it
                                    stayed within its limits (envelope) · it left nothing running (teardown)
  Tests that can fail               every new test is run once against a planted bug; a test that stays green proves
                                    nothing and is rewritten
  Never                             a test written to pass (asserting the code's current output instead of the
                                    promise); a mock of the very thing under test; "it should work" without a run
```

### 7.3 · Runtime and observability

This is the law we most wished we had written first. A product nobody can debug in production is not done.

```text
RUNTIME — what every shipped program tells its operator

  The log line        one structured line per event: timestamp · level · service name · request id · event name ·
                      the event's own fields. Never a bare print to stdout or stderr in production code
  The request id      minted where a request or job begins, carried across every hop (a header, or a field in the
                      queued message), written on every line; use the W3C traceparent header if you can
  Named events        each program keeps its list of event names in its own code; a test drives the program and
                      checks that every listed event appears, and no unlisted one does
  Named failures      one exit code or error kind per failure, with the operator's fix written beside it in the code
  Errors carry names  an error or warning line without an event name is refused by a lint: an alarm cannot match it
                      and nobody can join it to a request
  Alerts              every alert names a runbook, and a test proves that every runbook it names exists
  Secrets             never in a log; a test plants a token and checks that it never appears in the output
```

### 7.4 · A law for the writing itself

Agents write fluent, confident prose, and confidence is not evidence. We hold every agent report to one discipline.

```text
CONFIDENCE TRACKS EVIDENCE — every claim says how it was established

  I ran it          "Verified: 32 paths present, 0 missing."
  I read the code   "The code reads X; I did not run it."
  A tool said so    "The test runner reported X; not checked independently."
  I am inferring    "This suggests X; not proven."
  Bad news first    a red result, a skipped step or cut scope is the first sentence of the report
  Exact or absent   a path, a line number or a count is exact, or it is not stated
```

---

## 8 · What we found you were missing

### 8.1 · An agent's calls go to chat, never to a log

An agent constantly meets small choices that no rule decides. We used to log each of those calls on a page, so that later agents
could read them. After a few months the pages held thousands of lines that nothing needed, and agents spent their context reading
old reasoning. We deleted them.

```text
CALLS
  An agent's call  a choice no rule settles: the agent picks the safe, reversible default, ships it, and says so in its
                   final message. It is not logged anywhere
  A cut            a call that would cut or narrow a promise is not the agent's to make: it goes to the human in chat
                   as a proposal, and the non-cutting default ships meanwhile
  An exception     a gap a gate accepts is a field on the thing it excuses (a number beside its test, a limit in its
                   script), overwritten when it changes, never an append-only log
```

### 8.2 · Hooks, for what text cannot stop

Some rules matter too much to trust to reading. A hook is a script the harness runs before a tool call, and it can refuse the call.
Use hooks for the few rules whose break is expensive or silent:
- a hand edit of a generated page;
- a commit with a secret in it;
- a dispatch to a sub-agent that asks it to run a whole test battery;
- a `git add -A`.

Keep the list short. A hook that fires on everything trains people to bypass it.

---

## 9 · The agentic framework: scaffolding, development, integration

With the board in place, three skills run the development itself.

### 9.1 · `/scaffolding` — the structure, once

```text
/scaffolding — read the architecture and the lanes; lay out the repositories

  Reads      every architecture page, the info lanes, the test architecture page, the CI/CD page
  Writes     the directory tree each architecture page names; empty lanes with their contract stubs in dmz/; the
             test mirror under tests/; the CI jobs; one smoke test per deployable
  Runs       once at the start, and again only when the architecture changes or is restructured
  Done when  every directory an architecture page names exists; the linkcheck and the architecture check pass;
             the empty build is green
  Never      product logic; a directory no page names
```

### 9.2 · `/development` — one session per piece of work

Run several development sessions side by side. Each takes one or more info lanes and works in its own worktree.

```text
/development <lanes> — build what the lanes promise, on the architecture, under the laws

  Reads      the named lanes whole · the architecture page of every repository it touches · the laws its task meets
  Works in   its own worktree, on its own branch
  Builds     the code and its tests, each test naming the lane id it proves
  Proves     its own tests pass, and each one fails when a bug is planted in the code it covers
  Opens      a pull request into the integration branch, never into main. Its body lists:
               - the info lanes it worked on, by id
               - the architecture blocks it used or added
               - the tests it wrote, updated or deleted (the recursion ledger)
               - what it did not do, and why
  Never      touch a file another session owns; edit a lane to match what it built; run the full battery
```

### 9.3 · `/integration` — many branches into one product

```text
/integration — take every finished development branch and make one product of them

  Takes      every open development pull request
  Merges     them into one integration branch, one at a time, running every suite the merge touched after each
  Resolves   each conflict by reading the lanes and the architecture each side worked from: the side that keeps the
             promise and the placement rules wins; the integrator rewrites the code, never the lane
  Evaluates  the evaluation law on the merged product: the real app, a browser, one canonical job, the six questions
  Closes     each development branch once its work is in, with a link to the merge
  Opens      ONE clean pull request into main, listing every lane delivered and the tests that prove each. The human merges it
  Never      merge main itself; close a branch whose work it did not land; call a lane delivered that a fresh run has not proved
```

---

## 10 · Orchestration: fewer agents, and the main agent owns the code

The instinct with parallel agents is to start twenty. Do not. Twenty agents on one codebase produce more conflicts than you can
untangle, and the orchestrator ends up reading reports instead of writing code. Put this in the agent file.

```text
ORCHESTRATION — how many agents, and who does what

  The main agent works      the orchestrator writes most of the code itself. It hands a sub-agent only a task it judged
                            big enough and separate enough: its own files, its own worktree, a clear done-when
  One owner per file        two agents never edit one file at the same time; tasks that share a file are one task
  Sub-agents write, they    a sub-agent writes the code and its tests, commits in its worktree, and stops. It never runs
  do not judge              the suites, and never grades its own work
  The orchestrator proves   it takes each sub-agent's work, runs the tests itself, and reads the failures
  The orchestrator fixes    a failure in a sub-agent's work is fixed by the orchestrator, against the lanes and the
                            architecture, never by sending the sub-agent back to fix what it got wrong
  Size by the machine       the count of agents is set by what the machine can run (disk, memory, cores) and by how
                            many files the work can be split into without sharing, never by a fixed number
  Readers are cheap         a read-only agent that answers one question needs no worktree, and can run beside the others
```

---

## 11 · The goal prompt: what to hand a session

Run every development and integration session from one goal prompt. It names the work, the laws, and what done looks like. A
session without a stated "done" stops when it feels done, and that is too early.

```text
/goal

THE TASK
  Build the posting road of the Payments Ledger end to end: a client posts a payment, and the console shows it.

THE LANES (read each whole before the first edit)
  a7c3e910  the double-entry posting
  c20f5d71  the posting key

THE ARCHITECTURE
  board/architecture/ledger-api.md     — HOW TO PLACE CODE, the middleware and sink_routers blocks
  board/architecture/ledger-engine.md  — HOW TO PLACE CODE, the core block

THE LAWS
  board/laws/test-suite-recursion.md · board/laws/evaluation.md · board/laws/runtime.md

DONE MEANS — every line true, each checked by a command
  1  every test that names a7c3e910 or c20f5d71 passes on the merged code, and its planted bug is caught
  2  a browser posts a payment in the running console, and the account history shows both legs
  3  an unbalanced posting, a mixed-currency posting and a reused key are each refused by name, in the UI and the API
  4  the same request sent twice posts once
  5  every new log line carries the request id; the posting events appear in the program's event list
  6  the pull request body lists the lanes, the architecture blocks and the recursion ledger

HOW TO KNOW IT IS NOT DONE
  any red test · any step skipped · a promise narrowed to fit the code · a check that passes with its bug planted

NEVER
  merge main · weaken a test · edit a lane to match the code · hand a sub-agent its own failure to fix
```

---

## 12 · Mistakes we made, so you can skip them

1. **Tables of checks inside the spec.** They went stale, and nothing failed when they did. Keep the intent as prose and the check as
   code.
2. **Logs of agent reasoning.** We kept every agent call on a page, and it became thousands of lines that nobody read but every agent
   loaded. Calls go to chat, and nothing logs them.
3. **Typed state.** "Built", "70%", "18 services": every typed state was wrong within a week. Compute it, or leave it out.
4. **A schema changed after its pages.** We once fixed the pages and left the schema. The next generation run wrote the old shape
   back. Change the schema and its writers first, then the pages.
5. **Tests that cannot fail.** Agents write tests that assert what the code does, not what it should do. A planted bug that the test
   misses exposes it in seconds.
6. **Too many agents.** Merge conflicts and duplicated services grew faster than the work.
7. **Paths from memory.** An agent "remembered" a directory that had moved. Every path in the agent file is checked by a script, and
   a missing path is flagged, never rebuilt.
8. **Shared checkouts.** One session's `git add -A` committed another session's half-finished work. Commit by path, one worktree per
   session.
9. **Proof run everywhere.** Running every suite on every change took hours a day. Prove narrowly per task, and run one wide check
    per merge.
10. **Runtime as an afterthought.** We wrote the logging rules after the alarms, and the alarms named runbooks that did not exist.
    Write the runtime law with the first program.

---

## 13 · Getting started

```text
DAY 1   Make board/ (or the board repository) and the agent file. Write the five hard stops and the "ALWAYS" block.
DAY 1   Write the schemas: board/schemas/info-lane.md and board/schemas/architecture-page.md.
DAY 2   Write the /product-documentation skill. Run it: one info-lane page per product, one architecture page per
        repository, each with its HOW TO PLACE CODE block.
DAY 2   Write board/laws/: its README hub, then test suite recursion and evaluation. Route each from the agent file by
        its condition.
DAY 3   Write board/devops/: its README hub, ci-cd.md, test-placement.md and one page per test kind you run. Route each
        from the agent file by its condition. Add the runtime and writing laws.
DAY 3   Write the scripts that keep it honest: a link check, a lane-shape check, a path check on the agent file.
DAY 4   Run /scaffolding once. Fix the architecture pages where the scaffold disagreed with them.
DAY 5   Run two /development sessions on separate lanes, then /integration.
AFTER   Every finding about the process becomes a line in a law, a skill's "Never" list, or a hook, the same day.
```

The last line matters most. The framework is never finished. Each time an agent makes a mistake you did not foresee, the fix is not a
better prompt next time. It is a sentence on the page that agent will read.

---

## 14 · Where your eyes are most needed

Agents write the code, the tests and most of the pages. Four things still need a human, because an agent cannot judge them against
what you meant. Spend your review time here, in this order.

### 14.1 · The test suites: they must be able to fail

An agent that is asked to make the tests pass will, sooner or later, write tests that pass no matter what. Watch for:
- a test that asserts what the code returns today, instead of what the info lane promises;
- a mock of the very thing the test claims to check;
- a test that was weakened, skipped or deleted in the same change that made the suite green;
- a suite that is all green and has never been red.

The quickest check is to break the code on purpose. Change one line the test should guard, and run the test. If it stays green, the
test proves nothing, and it is rewritten.

### 14.2 · The product itself, the way a user meets it

Green tests are not a working product. Use the product through the surface your users touch:
- **a GUI:** open it, click the buttons, fill the forms, and read what appears;
- **a CLI:** run the commands with real arguments, wrong arguments and no arguments, and read the output and the exit code;
- **an API:** call it with a real client, including the refusals.

What you see there outranks every report an agent writes about it.

### 14.3 · The info lanes: they must say what you mean

The lanes are your intent, written down. An agent builds exactly what they say, so a lane that drifted from what you want builds the
wrong product correctly. Read them often, and check two things:
- **aligned:** each lane says what you want the product to do, and its "one rule" is a rule you would defend;
- **up to date:** when you change your mind, the lane changes first, before any agent builds on the old version.

### 14.4 · The architecture: use the patterns to trace a fault

The common patterns are not only for placing code. They tell you where to look when something breaks, because each kind of fault has
one home.

```text
TRACING A FAULT BY ITS PATTERN

  What you see                                         Where to look first
  data is not being stored, or is stored wrong         the product's sink_routers/ for that service
  data is stored but does not reach the screen         source_routers/ (is it read?) then hydration/ (is it shaped for the view?)
  a request is refused that should pass, or passes     middleware/ (authentication, sessions, limits) and the door in routers/
  that should be refused
  two services disagree about the shape of a message   dmz/ — the contract, and whether both sides were changed together
  the answer itself is wrong                           core/ — the pure logic, with its unit tests
  nobody can tell what happened in production          runtime/ — the events, the request id, the named failures
```

When you hand a fault to an agent, name the pattern. "The balances are not reaching the console; start in `source_routers/` and
`hydration/` of the ledger API" gets you a fix. "The console is broken" gets you a guess.
