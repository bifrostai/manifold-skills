---
name: containerize-policy
description: >
  Package a researcher's policy to run on Manifold's GPUs, in three stages.
  Stage 1 records the project's context. Stage 2 wraps the policy with the
  manifold-sdk and proves the wrap with check_compatibility, verify, and a
  test run. Stage 3 builds the container image, pushes it, registers it, and
  offers a scored test run. Use when the user wants Manifold compute to run
  their policy, or asks to "containerize my policy", "wrap my policy for
  Manifold", "pair <model> with LIBERO / SIMPLER / RoboCasa", "why is
  check_compatibility INCOMPATIBLE", "build the policy image", "push my wrap
  to ghcr", or "register my policy on the platform". To serve a policy from
  the user's own machine, use `/serve-policy` instead.
compatibility: >
  Run from the user's policy project directory. Requires the manifold CLI on
  PATH, authenticated with `manifold auth login`, and a Python project with a
  file that lists its dependencies (pyproject.toml, requirements.txt,
  Pipfile, environment.yml, or setup.py). Stage 3 also requires docker and a
  push credential for the target container registry (ghcr by default).
  Everything this skill writes goes inside `<project>/.manifold/`.
---

## Summary

This skill prepares the user's policy to run on Manifold's GPUs. It
works in three stages:

1. **Set up** (Phases 1 to 3). The skill looks at the project and the
   machine, interviews the user once, and writes
   `.manifold/CONTEXT.md`.
2. **Wrap** (Phases 4 to 6). The skill uses the manifold-sdk to write
   the Python files that let a Manifold benchmark run the policy. Two
   checks and a test run prove those files.
3. **Containerize** (Phases 7 to 10). The skill builds a container
   image, pushes it to the user's registry, registers it on Manifold,
   and offers a scored test run.

The skill pauses after Stages 1 and 2, so the user can review each
result before the next stage starts. Stage 3 spends registry storage
and cloud time, so it needs an explicit yes.

If the user wants to serve the policy from their own machine, stop
and point them at `/serve-policy`.

## Rules

**Do not run this skill without an explicit user invocation.** This
skill writes into the user's project. If the user has not asked for
it by name, stop and wait.

**Confirm before doing anything else.** First thing after this skill
loads, tell the user in your own words what the three stages do.
Stage 1 records the project's context in `.manifold/CONTEXT.md`.
Stage 2 adds `manifold-sdk` to the project's dependencies and writes
the wrap under `.manifold/`. Stage 3 builds a Docker image (10 to 30
GB of local disk), pushes it to their registry, and registers it on
Manifold. Wait for a yes before touching anything. Push, register,
and submit still have their own confirmations later.

**Run in the user's policy directory.** This skill acts on the folder
that is your current working directory. Before anything else, confirm
that folder is the user's policy project: the one that holds the model
code, and where a `pyproject.toml` / `requirements.txt` or the
equivalent sits at the top level. If it looks like the wrong place (no
such file, or a generic directory like the user's home), tell the user
you appear to be in the wrong directory and ask for the correct
project path before continuing.

**Start at the right stage.** The user may have run part of this
flow before. Check for earlier output before Phase 1:

- If `.manifold/CONTEXT.md` is missing, start at Stage 1.
- If `CONTEXT.md` exists, read it and tell the user what it records.
  Ask whether to reuse it or to add a new policy. To reuse it, start
  at Stage 2.
- If `.manifold/<slug>/` also holds wrap files, and `CONTEXT.md` has a
  **manifold-sdk revision** line for the policy, rerun
  `check_compatibility` and `verify`. If both pass, offer to start at
  Stage 3. If either fails, start at Phase 6.

Whenever `CONTEXT.md` exists, read it before asking the user
anything. Ask only about details that it does not cover.

**Speak to the user in their language, not the SDK's.** The user has
not read the SDK docs. They will not recognize class names, config
fields, Docker fields, registry commands, or CLI flag names. This
skill names those identifiers freely because you need them to write
correct code. When you narrate progress or ask a question, translate.

Say things like:
- "How much GPU memory does the model need at inference?" rather
  than "What is your peak VRAM in GB?"
- "I'll write the files that let the benchmark run your policy."
- "The wrap passes the SDK's compatibility check."
- "Some parts of the wrap were not tested by the check."
- "The image built and pushed to your registry."
- "Registered on Manifold as version 0.1.0."

If the user uses one of those terms themselves, follow their lead.
Otherwise, describe what happened and why it matters.

**Plan the entire task in a to-do list before you start, and update it as
you go.** Use whichever planning tool your harness provides:

- **Claude Code:** `TaskCreate` to seed the plan, `TaskUpdate` to move items
  between `pending` / `in_progress` / `completed`, `TaskList` / `TaskGet` to
  read state.
- **Codex:** use `update_plan` to create and maintain an ordered plan, with
  exactly one item `in_progress` at a time. Keep validation and the scored
  test run as explicit items until they pass.
- **Other harnesses:** check the harness for a to-do list or planning tool
  before using the fallback below.
- **No planning tool available:** keep the plan as a plain-text checklist in
  your responses and re-post it (with statuses updated) each time you advance.

The intent is (1) to **structure the work** so nothing gets skipped, and
(2) to **stay accountable and informative** by updating the list as steps
start and finish, so the user can follow along without asking.

**Pause between stages.** At the end of Stages 1 and 2, give a short
handoff and ask whether to continue. Start the next stage only after
a yes. If the user says no, or wants to change something first, stop
and wait.

Keep each handoff under 8 lines. The user watched the stage happen,
so do not restate the design or review the checks. Write in the
user's vocabulary. The user does not know what a "wrap" is.

**Ask, do not invent.** Every user-only field that ends up in
`CONTEXT.md` is the user's answer to a specific question. Do not
fabricate a policy slug, a registry namespace, a peak VRAM number, or
a list of benchmarks the user might want. If the
user does not know something, write "unknown"; that is better than a
made-up value that later stages will trust.

# Stage 1: Set up the project

Phases 1 to 3. The skill looks at the project and the machine,
interviews the user in one round, and writes `.manifold/CONTEXT.md`.
Stage 1 installs nothing.

**Read [`references/setup.md`](references/setup.md) in full before
Phase 1.** Follow it, including its checklist, before you move on.

## End of Stage 1: ask whether to continue

Summarize what was written:

- The file created (`.manifold/CONTEXT.md`).
- A one-line recap of the recorded context: policy name, registry,
  weights location, benchmarks of interest.
- Any prerequisites that Phase 1 found missing. Say so if Docker is
  not installed or not reachable. Give the user two more facts when
  they apply. If this machine cannot run the model, Stage 2 skips its
  test run, and the wrap stays untested until its first run on
  Manifold. If this machine is arm64, the image must be built for
  linux/amd64. That build runs under emulation and can fail on GPU
  wheels. Point the user at how to install or arrange each missing
  piece (for example Docker's install docs).

Then ask something like: **"Ready to prepare your policy for use with
Manifold?"** On a yes, start Stage 2.

---

# Stage 2: Wrap the policy

Phases 4 to 6. The skill adds `manifold-sdk` to the project, writes
the wrap under `.manifold/<slug>/`, and proves it with
`check_compatibility`, `verify`, and a test run.

**Read [`references/wrap.md`](references/wrap.md) in full before
Phase 4.** Follow it, including its checklist, before you move on.
For import paths, read
[`references/sdk-imports.md`](references/sdk-imports.md).

## End of Stage 2: ask whether to continue

Stage 3 costs registry storage and often cloud time, so the user has
to opt in. Use this shape for the handoff:

1. **One line: status.** Example: "Wrap is written and passes the
   checks."
2. **One line: what got made.** The module path and the pairing name.
3. **Caveats, only if any.** One line each. List `verify` entries
   under `not_checked`, a skipped test run, or a credential that the
   user still needs to arrange. Skip this item if there are none.
4. **The question.** For example: "Next, I can build the container
   image, push it to your registry, and register it on Manifold. The
   build uses 10 to 30 GB of disk. Should I continue?"

On a yes, start Stage 3.

---

# Stage 3: Containerize and register

Phases 7 to 10. The skill writes a `Dockerfile` and a launcher,
builds the image, pushes it, registers it on Manifold, and offers a
scored test run.

**Read [`references/containerize.md`](references/containerize.md) in
full before Phase 7.** Follow it, including its checklist.
