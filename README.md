# An issue-driven workflow for AI-written software

This is a development method for software whose code is written almost
entirely by AI agents and directed by a human owner. It exists because
"vibe coding" has earned its bad reputation: code nobody reviewed, changes
nobody can trace, tests nobody ran. The claim here is that the problem is
missing process, and that an ordinary, strict engineering process fixes it.

Nothing in it is new. It is issues, branches, pull requests, tests and
written rules, applied without exceptions to contributors that happen to be
language models.

## The short version

1. **Every change starts as an issue.** No issue, no work.
2. **One branch and one pull request per issue**, cut from a clean main branch.
3. **A local test gate blocks the push** when anything is red.
4. **The pull request quotes the test result**, as numbers, in its body.
5. **The owner decides** what is merged and what is deployed. Agents propose.
6. **Every issue and pull request names the model that wrote it.**
7. **The rules live in the repo** (`AGENTS.md`) and change only through this
   same flow, with the owner's approval.

## Where it comes from

The method is in daily use on three unrelated projects with different stacks:
a browser application with a small Node service, a Python backend with no
third-party dependencies, and a Node monorepo deployed on several servers.
Together they account for more than a thousand issues and more than a
thousand merged pull requests. This document describes only the method; the
examples have every project detail removed.

## The roles

- **The owner** is a person. They set the goals, answer the questions only
  they can answer, approve merges, and run deploys.
- **Agents** are language models working through a coding tool. They file
  issues, write the code and the tests, open pull requests, and review each
  other's work.

The split matters more than anything else in this document. An agent that can
merge and deploy its own work has no one checking it.

## The four files every repo carries

| File | Holds | Example of a line in it |
|---|---|---|
| `AGENTS.md` | The process: how work is done here. | "Red means fix forward. Never merge red." |
| `GOALS.md` | The product law, with numbers. | "The whole page weighs at most 100 KiB compressed." |
| `HANDOFF.md` | What the next agent must know: current state, past incidents. | "The generated copy must be committed or the deploy leaves the tree dirty." |
| `docs/decisions.md` | Decisions the owner already made, dated. | "2026-09-29: origin tags may be visible on the public repo." |

Agents have no memory between sessions. These files are the memory. A
decision that is not written down gets asked again, or worse, gets guessed.

## The cycle

### 1. Issue

An issue says what was observed, what it costs, and which direction the fix
should take. Before filing, the agent checks the report against the current
main branch, because many reported bugs are already fixed.

Issues carry labels for severity (`critical`, `high`, `medium`, `low`) and
difficulty (`hard`, `medium`, `easy`). Work is picked by severity first, then
easy before hard.

References to code use the name of the function, never a line number. Line
numbers are wrong after the next merge.

### 2. Branch

One branch per issue, from a clean and current main branch. Each agent works
in its own fresh clone and never shares a working folder with another agent.

The largest of the three projects later moved to one branch per area of the
code, with one commit per issue inside it. With hundreds of small fixes, a
stream of tiny merges made every review re-learn the same files. Each commit still closes its own issue,
so every fix can be reviewed and reverted alone.

### 3. The gate

A `pre-push` hook runs the test suite on the developer's machine and refuses
the push when it fails. There is no hosted CI bill and no way to "push now and
fix later".

The gate is tiered so that it stays fast enough to be obeyed:

| Tier | When | What runs |
|---|---|---|
| Push | Every push | Syntax checks, builds, the full test suite in parallel, and a check that generated files match their sources |
| Merge | Before a merge | The push tier plus the slow suites: browser tests, or one serial network-isolated run |
| Timer | Hourly or daily | The push tier in a fresh clone of the main branch, with the result sent as a phone notification |

Speed is a feature of the gate. The largest suite here has about 3,600 tests
and its parallel push tier takes about 8 seconds; the serial run takes under a
minute. A gate that takes ten minutes gets bypassed.

The tests may not reach the network. The runner blocks every connection that
is not to the local machine, so a test cannot pass by accident because a
real service answered.

The escape hatch (`git push --no-verify`) is allowed for production
emergencies only, and using it creates a follow-up issue.

### 4. Pull request

The body says what changed and why, links the issue with `Closes #N`, and
quotes the gate result: test counts and exit code. "Tests pass" without the
numbers is not accepted.

A change in behaviour also records three review passes:

- **Correctness**: the suite, plus reasoning over the diff.
- **Deploy and operations**: does this need a restart, a config change, a
  migration?
- **Documentation**: are the user guide and the format documents still true?

A fix needs a regression test for the kind of defect it fixes. A refactor
must keep the suite green with the tests untouched; if a test had to change,
it was not a refactor.

### 5. Merge and deploy

The owner approves merges. Deploying is a separate, deliberate step run by the
owner through one script, which is the only supported way to change the live
system. Merged does not mean live.

The project with the most users adds two things: work lands on a
`development` branch and reaches `main` only as a release, and every release
is tried on a staging server first.

## Origin tags

Every issue and pull request shows which model wrote it, in three places:

- the title starts with the tag: `[opus] Fix the row grouping`
- the label is the same tag
- the last line of the body is `*Generated by <model id> — flag <tag>*`

An agent never uses another agent's tag and never removes one.

More than a dozen different models have contributed to these projects. The tag
answers questions that otherwise have no answer: which model introduced this
bug, which model's pull requests get rejected most, which one can be trusted
with the hard issues. It also keeps the history honest about how the software
was made.

## Rules that keep agents on track

**Zero deviations from the issue.** The issue is the plan. When the work
reveals a choice the issue did not make, the agent does not improvise. A small
factual question goes to the owner. A design question means: build what the
issue says, file the open choice as a new issue, and mention it in the pull
request.

**Ask with options.** A question to the owner comes with the situation in
plain words and two or three concrete options, one marked as recommended. A
bare "what do you want?" pushes the agent's work onto the owner.

**Size budgets.** A new source file prompts a split at a soft limit and may
not pass a hard limit (400 and 800 lines in the Python and Node projects, 600
and 1,200 in the browser project). Files already over the limit may only
shrink. Very large files are where agents make their worst mistakes, because
they can no longer hold the whole file in view.

**Landmine files.** Each repo lists the files that fail in subtle ways: input
validation, signing, anything that deletes user data. A change to one of them
gets a second pass over the diff before the push.

**Debt must be visible.** Known shortcuts live as issues labelled `DEBT`, each
with one sentence on what it costs to leave. Defined triggers turn a debt
issue into scheduled work: a production incident, or two pull requests that
both need to touch the same indebted code.

**Shadow before apply.** A change that can delete, rename or fetch files
runs first in a mode where it only reports what it would do.
The owner switches it on after reading the reports.

**Formats carry a version.** Stored data and file formats name their version,
readers refuse versions newer than they know, and a field is never renamed in
place.

**No personal information.** Commits use a no-reply identity, and the push
gate rejects any other. A script checks the public code for names, addresses,
paths and keys before it is published.

**Unreadable output is wasted work.** Long commands write to a log file, and
the agent reads the file. Re-running a ten-minute command because its output
scrolled away is the most common way agents burn time.

## The rules change, in the open

`AGENTS.md` is amended through an issue and a pull request like any code, and
only with the owner's approval. No agent loosens or tightens a rule alone.

Certain events force a review of the rules: an incident reached the live
system, a rule was followed and caused harm, a mistake happened because a rule
was missing, or a sibling project turned out to have a practice this one
lacks. Every rule states its reason. A rule whose reason has gone is removed,
never quietly ignored.

Rules travel between the projects this way, and each imported rule says where
it came from.

## What went wrong, and the rule it produced

Most of the rules above exist because of a specific failure.

| What happened | What changed |
|---|---|
| The gate piped the test output through `tail`, so the hook saw `tail`'s success and a failing suite could be pushed. | Tests run without a pipe, and the hook reads the real exit code. |
| An agent asked to test notifications decided to test every type, and sent 25 samples to the owner's phone. | The zero-deviations rule. |
| An agent stalled on a documentation task and the whole queue waited for it overnight. | Every dispatched agent gets a deadline. A stalled one is stopped and worked around. |
| GitHub's merge button wrote a personal address into the history, twice. | The push hook rejects any commit whose author is not the no-reply identity. |
| A pull request in the private repo used a closing keyword for an issue in the public repo, which closed the issue before the fix was published. | Cross-repo references use `Refs`, and the public issue is closed by hand after publishing. |
| Two files grew past 2,500 lines each. Making them safe to change again took a series of extraction pull requests. | Size budgets, and "shrink, not grow" for the big files. |
| The rules quoted a file's line count as its current size. Weeks later the file had grown again and the number was wrong. | Rules never quote a measurement as current. They say how to measure. |
| A file generated at build time was not committed, so each deploy rebuilt it and left the tree dirty, which then blocked publishing. | The gate checks that every generated file matches its source. |
| An agent widened a feature beyond what the issue asked for, and the run had to be stopped halfway. | The same zero-deviations rule, now with a second example. |

## Where the three projects differ

The method is shared; the settings are not.

| Setting | Variants in use |
|---|---|
| Branching | One branch per issue; one branch per code area with one commit per issue; one branch per issue into a `development` branch |
| Who merges | The owner; or an agent, only on the owner's explicit word |
| Gate | Local hook plus a timer; or hosted CI plus a staging server |
| Emergency lane | `--no-verify` and a follow-up issue; or a direct commit to `main` with a retroactive issue and the full suite within the hour |
| Size budget | 400 / 800 lines; or 600 / 1,200 lines |

## What this does not solve

- **The owner is the bottleneck.** That is the design, and it is also the
  cost. Work waits for a person.
- **Tests only catch what they describe.** The gate has let through bugs that
  only showed up with real users' files. Each one became a regression test,
  after the fact.
- **Agents still make mistakes inside the rules**: a wrong number in a pull
  request body, a test written to match the code instead of the requirement.
  Review by a second agent and by the owner catches some of it.
- **Rules rot.** They need the same upkeep as code, and this takes real time.

## Use it

The [`templates/`](templates/) folder has a starting point:

- [`AGENTS.md`](templates/AGENTS.md): a process contract to adapt
- [`PULL_REQUEST_TEMPLATE.md`](templates/PULL_REQUEST_TEMPLATE.md)
- [`pre-push`](templates/pre-push): a gate hook that reads the real exit code

Start with three things: issues first, a gate that blocks red pushes, and the
owner holding the merge. Add the other rules when a failure shows you need
them, and write down which failure it was.
