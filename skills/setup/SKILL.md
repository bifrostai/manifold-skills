---
name: setup
description: >
  Use this skill before any other Manifold skill to gather context about the
  user's project. Ask the user for context about their project, policy model,
  codebase, inference requirements, and container registry so that you can
  build and serve their policy correctly. This skill will guide you to create
  a working .manifold/ folder at the root of the project directory.
compatibility: >
  Requires the manifold CLI on PATH (install first if it is missing;
  authenticate with `manifold auth login`). Requires a Python project at
  the root of the working directory, with a file that lists its
  dependencies (pyproject.toml, requirements.txt, Pipfile,
  environment.yml, or setup.py). The skill figures out which package
  manager the project uses so later skills can respect it; it does not
  modify the project's dependencies itself.
---

## Summary

Set up a project for the Manifold platform. This runs first; the
suggested next skill is `/wrap-policy`.

The output is one file: **`.manifold/CONTEXT.md`** at the project
root. Plain markdown with everything the next two skills need to know:
policy slug, model source folders, weights location, registry,
inference requirements, deployment style, benchmarks, and the detected
package manager.

Setup does not touch the project's dependencies and does not scaffold
any policy folder. Installing `manifold-sdk` and creating
`.manifold/<slug>/` are both `/wrap-policy`'s job — that skill runs
when there is actually wrap code to write.

Once `CONTEXT.md` is written, hand back to the user. The suggested
next step is `/wrap-policy`; the user invokes it themselves.

## Rules

**Run in the user's policy directory.** This skill acts on the folder
that is your current working directory. Before anything else, confirm
that folder is the user's policy project — the one that holds the
model code, and where a `pyproject.toml` / `requirements.txt` or the
equivalent sits at the top level. If it looks like the wrong place
(no such file, or a generic directory like the user's home), tell the
user you appear to be in the wrong directory and ask for the correct
project path before continuing.

**Use the agent's structured question tool for the interview.** In
Claude Code that is `AskUserQuestion` — call it with the interview's
questions, providing options where the answer is a small set. In other
harnesses use the equivalent structured-question tool if one exists.
Fall back to plain-chat questions (numbered list) only if no structured
tool is available.

**Use judgment. Ask about what is hard to know; look up what is easy.**
Do the cheap, reliable lookups yourself — for example, `nvidia-smi` for
a GPU and its VRAM, `which docker` for Docker, a lockfile like
`uv.lock` for the package manager, grepping the project's deps for the
ML framework. Ask the user for what only they can answer or what would
take a lot of exploration to determine.

**Do the interview once, upfront.** Gather every user-only answer in a
single interview call before writing anything. Do not drip questions
across the skill.

**Ask, do not invent.** Every user-only field that ends up in
`CONTEXT.md` is the user's answer to a specific question. Do not
fabricate a policy slug, a registry namespace, a peak VRAM number, or
a list of benchmarks the user might want. If the user does not know
something, write "unknown" — better than a made-up value that later
skills will trust.

**Respect the project. Zero impact on how it already works.** Figure
out which package manager the project uses and record it in
`CONTEXT.md`. If it's ambiguous, ask the user. Do not install anything
yourself, and do not migrate the project to a different manager.

**Do not run this skill without an explicit user invocation.** Setup
writes into the user's project. If the user has not asked for setup by
name, stop and wait.

## What CONTEXT.md is

`.manifold/CONTEXT.md` is the persistent context for every Manifold
skill that runs on this project. Plain markdown, sections by topic
(Project, Registry, Deployment, Policies). Setup writes the first
version; wrap-policy and containerize-wrap read from it before asking
any question of their own, and can append to it as they learn more.

The point is that the user answers each question once. Later skills
never re-ask what CONTEXT.md already knows.

---

## Phase 1: Look at the project

Do this before asking anything. The interview in Phase 2 skips every
question you can answer here.

- **manifold CLI.** Confirm it is installed and the user is
  authenticated. If not, stop and tell the user to install it and run
  `manifold auth login`.
- **The project.** Figure out which package manager it uses, which
  folders hold the Python source the wrap will import from, and
  whether `.manifold/` already exists (if so, read `CONTEXT.md`, tell
  the user what is already recorded, and ask whether they want to add
  a new policy or re-verify).
- **The machine.** Check whether there is a GPU (CUDA version and
  total VRAM if so), whether Docker is reachable, and which ML
  framework is in the project's deps.
- **Weight files.** Look for likely checkpoint folders in the project
  so you can propose them to the user rather than asking them to type
  a path from memory.
- **Available benchmarks.** Run `manifold benchmark list` and note
  the slugs and one-line descriptions so you can present them as
  options in the interview.

> **Phase 1 checkpoint:**
> ```
> manifold_cli_ready      = yes | no  (if no: stop)
> package_manager         = ?
> manifold_folder_exists  = yes | no  (if yes: what does CONTEXT.md say?)
> source_folders_seen     = [list]
> gpu_present             = yes | no
> cuda_version            = ? | unknown
> docker_present          = yes | no
> ml_framework            = ? | unknown
> weight_dirs_seen        = [list, or empty]
> available_benchmarks    = [list of {slug, description}]
> ```

---

## Phase 2: Interview the user

One upfront call to the structured question tool. Only ask what Phase 1
could not answer:

- **Policy name.** What to call this policy on Manifold — becomes the
  `<slug>` in `manifold policy init <slug>`. The user names it; do not
  suggest one.
- **Peak VRAM at inference, in GB.** The user knows this from their own
  runs; the agent cannot measure it without running the model.
- **Where the weights live.** If Phase 1 found candidate folders,
  present them as options plus "elsewhere / hub / cloud storage." The
  user picks or fills in the actual path or ref.
- **How the user typically serves this policy.** Options: local box,
  Modal, RunPod, other cloud, none-yet. This changes downstream
  decisions about weights staging and build environment.
- **Container registry destination.** URL + namespace (e.g.
  `ghcr.io/<org>`). Include a default of `ghcr.io` if the user has no
  preference stated.
- **Benchmarks of interest.** Present the list from Phase 1
  (`manifold benchmark list`) as options for a multi-select — slug
  plus one-line description each. **Group benchmarks that belong to
  the same family** — shared slug prefix and/or a shared word in the
  description usually gives it away — and present the family as a
  single grouped choice (with the individual suites as multi-select
  items inside), not as several unrelated one-of-N options. A
  researcher shipping a policy for a benchmark family usually wants
  all of its suites. If none of the listed benchmarks fit, let the
  user name a different one, but flag that it must exist on the
  platform for a run to succeed.

Skip these unless the user brings them up: display name (defaults to
the slug), visibility (defaults to `org`).

> **Phase 2 checkpoint:**
> ```
> policy_slug         = ?
> peak_vram_gb        = ?
> weights_location    = ?
> deployment_style    = ?
> registry_url        = ?
> registry_namespace  = ?
> benchmarks          = [list]
> display_name        = ? | default (slug)
> visibility          = ? | default (org)
> ```

---

## Phase 3: Write CONTEXT.md

Create the file at `<project>/.manifold/CONTEXT.md`. Prose sections
organized by topic, not a config schema. The sections below cover the
common case — use them as a starting point, add new ones when the
project has facts that don't fit, and drop any you genuinely have
nothing to say about.

Common sections:

- `# Manifold context for this project`
- `## Project` — package manager, the dependency file it uses,
  Python version, source folders, ML framework.
- `## Registry` — url, namespace.
- `## Runtime` — GPU / CUDA / Docker facts you detected, plus how the
  user typically deploys this policy.
- `## Policies` — one `### <slug>` per policy, with display name,
  visibility, peak VRAM, benchmarks paired with, and a **Weights**
  paragraph (location and any auth notes).

Add new sections when something is worth recording that doesn't fit
above — for example, a `## Cloud storage` section if the weights live
behind an S3 bucket with quirks, a `## Notes` section for anything a
future skill should be aware of, or a `## Known issues` section for
constraints the user flagged during the interview.

Write in plain English, without jargon or metaphors. Reach for
technical language only when it helps a reader understand the project
better. This file is read by both agents and humans.

> **Phase 3 checkpoint:**
> ```
> context_md_written        = yes
> ```

---

## Ask the user whether to continue

Setup's own work is done. Do not chain into another skill without
asking the user first. Ask before continuing, and only proceed if the
user says yes.

Each step in the Manifold flow is a separate skill by design, so the
user gets to review the previous step's output before agreeing to the
next.

Hand back to the user in their vocabulary, not the skill's. The user
does not know what a "wrap" is or that `/wrap-policy` is the next
skill's name.

Summarize what was written:

- The file created (`.manifold/CONTEXT.md`).
- A one-line recap of the recorded context: policy name, registry,
  weights location, benchmarks of interest.
- Any prerequisites Phase 1 found missing that a later skill will
  need — for example, Docker not installed or not reachable, or no
  GPU on a machine where the deployment style needs one. Point the
  user at how to install / arrange them (e.g. Docker's install docs)
  so they can fix it before the next skill runs.

Then ask something like: **"Ready to prepare your policy for use with
Manifold?"** Do not name the next skill.

- If the user says yes, invoke `/wrap-policy` and continue with it.
- If the user says no, or wants to change something in `CONTEXT.md`
  first, stop and wait.

---

## Final checklist

- [ ] Current working directory is the user's policy project (if not,
      stopped and asked the user for the correct path)
- [ ] `manifold` CLI on PATH and authenticated (if not, stopped and
      told the user to install and `manifold auth login`)
- [ ] Project has a file listing its dependencies (if not, stopped
      and told the user)
- [ ] Cheap lookups (package manager, GPU, Docker, source folders,
      framework, `manifold benchmark list`) done without asking; user
      asked only for what is hard or user-only
- [ ] User interview happened once, upfront, via the structured
      question tool (or fallback)
- [ ] Nothing invented; unknowns recorded as "unknown"
- [ ] `.manifold/CONTEXT.md` written in prose, sectioned by topic,
      including the detected package manager
- [ ] Nothing else touched — no `.manifold/<slug>/` folder created, no
      project dependency added (both are wrap-policy's job)
- [ ] Handed back to the user with a summary; asked whether to
      continue before invoking `/wrap-policy`
