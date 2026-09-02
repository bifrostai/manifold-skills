---
name: setup-manifold
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
suggested next skill is either `/wrap-policy` or `/wrap-remote-policy`,
depending on where the model runs (see below).

The output is one file: **`.manifold/CONTEXT.md`** at the project
root. Plain markdown with everything the next two skills need to know:
policy slug, model runtime, registry, benchmarks, the detected package
manager, plus a set of fields that changes with the model runtime.

The interview asks first where the model runs, then branches to the
follow-up questions that fit that answer. Two possible answers:

- **Manifold loads and runs the model.** The user gives Manifold
  the checkpoint (weights on disk, or a hub id). Each run loads
  the checkpoint into GPU memory on Manifold's machines.
  Next skill: `/wrap-policy`, then `/containerize-wrap`.
- **The model already runs on the user's own server; Manifold
  calls it.** The user keeps an inference server up somewhere
  (Modal endpoint, private HTTPS box). Manifold sends observations
  to it over the network and reads back the actions. No GPU is
  needed on Manifold's side.
  Next skill: `/wrap-remote-policy`, then `/containerize-remote-wrap`.

The follow-up questions and the sections written into `CONTEXT.md`
depend on that answer.

setup-manifold does not touch the project's dependencies and does not
scaffold any policy folder. Installing dependencies and creating
`.manifold/<slug>/` are both the wrap skill's job.

Once `CONTEXT.md` is written, hand back to the user. The user invokes
the next skill themselves.

## Rules

**Run in the user's policy directory.** This skill acts on the folder
that is your current working directory. Before anything else, confirm
that folder is the user's policy project: the one that holds the model
code, and where a `pyproject.toml` / `requirements.txt` or the
equivalent sits at the top level. If it looks like the wrong place (no
such file, or a generic directory like the user's home), tell the user
you appear to be in the wrong directory and ask for the correct
project path before continuing.

**Use the agent's structured question tool for the interview.** In
Claude Code that is `AskUserQuestion`. Call it with the interview's
questions, providing options where the answer is a small set. In other
harnesses use the equivalent structured-question tool if one exists.
Fall back to plain-chat questions (numbered list) only if no structured
tool is available.

**Use judgment. Ask about what is hard to know; look up what is easy.**
Do the cheap, reliable lookups yourself. For example, `nvidia-smi` for
a GPU and its VRAM, `which docker` for Docker, a lockfile like
`uv.lock` for the package manager, grepping the project's deps for the
ML framework. Ask the user for what only they can answer or what would
take a lot of exploration to determine.

**Do the interview upfront, in two rounds at most.** A round is one
use of the structured question tool: one prompt to the user that
gathers several answers together. The first round asks the branch
question (where the model runs) along with the questions that apply
to both paths. The second round asks the follow-ups that apply to
the chosen path. Do not drip further questions across the skill.

**Ask, do not invent.** Every user-only field that ends up in
`CONTEXT.md` is the user's answer to a specific question. Do not
fabricate a policy slug, a registry namespace, a peak VRAM number, an
endpoint URL, or a list of benchmarks the user might want. If the
user does not know something, write "unknown"; that is better than a
made-up value that later skills will trust.

**Respect the project. Zero impact on how it already works.** Figure
out which package manager the project uses and record it in
`CONTEXT.md`. If it's ambiguous, ask the user. Do not install anything
yourself, and do not migrate the project to a different manager.

**Do not run this skill without an explicit user invocation.**
setup-manifold writes into the user's project. If the user has not
asked for it by name, stop and wait.

## What CONTEXT.md is

`.manifold/CONTEXT.md` is the persistent context for every Manifold
skill that runs on this project. Plain markdown, sections by topic
(Project, Registry, Runtime, Policies). setup-manifold writes the
first version; later skills read from it before asking any question
of their own, and can append to it as they learn more.

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
  framework is in the project's deps. These only matter for the
  in-container-model branch; look them up anyway because they are
  cheap.
- **Weight files.** Look for likely checkpoint folders in the project
  so you can propose them to the user rather than asking them to type
  a path from memory. Only relevant for the in-container-model branch.
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

Two rounds of questions.

### Round 1: branch question and common answers

Ask these together, in one prompt to the user. They apply regardless
of which branch is chosen.

- **Where does the model run when Manifold uses it?** Options:
  - **Manifold loads and runs it.** The user gives Manifold the
    checkpoint (weights on disk, or a hub id); Manifold loads it
    into GPU memory on their machines each run.
  - **The user already runs it on their own server; Manifold calls
    it.** The user has an inference server up somewhere (a Modal
    endpoint, a private HTTPS box). Manifold sends observations to
    it over HTTPS and reads back the actions.

  Ask the question in whatever plain-language wording fits the
  conversation; the exact phrasing above is not required. Avoid the
  word "container" unless the user has already used it. The answer
  picks the branch for Round 2 and the recommended next skill in
  the handoff.
- **Policy name.** What to call this policy on Manifold. Becomes the
  `<slug>` in `manifold policy init <slug>`. The user names it; do not
  suggest one.
- **Container registry destination.** URL plus namespace (for example
  `ghcr.io/<org>`). Include a default of `ghcr.io` if the user has no
  preference stated.
- **Benchmarks of interest.** Present the list from Phase 1
  (`manifold benchmark list`) as options for a multi-select: slug
  plus one-line description each. **Group benchmarks that belong to
  the same family** (shared slug prefix and/or a shared word in the
  description usually gives it away) and present the family as a
  single grouped choice (with the individual suites as multi-select
  items inside), not as several unrelated one-of-N options. A
  researcher shipping a policy for a benchmark family usually wants
  all of its suites. If none of the listed benchmarks fit, let the
  user name a different one, but flag that it must exist on the
  platform for a run to succeed.

Skip these unless the user brings them up: display name (defaults to
the slug), visibility (defaults to `org`).

### Round 2, branch A: in-container model

Ask these only if the user picked "Manifold loads and runs it":

- **Peak VRAM at inference, in GB.** The user knows this from their
  own runs; the agent cannot measure it without running the model.
- **Where the weights live.** If Phase 1 found candidate folders,
  present them as options plus "elsewhere / hub / cloud storage." The
  user picks or fills in the actual path or ref.
- **Where the user typically deploys this container.** Options: local
  box, Modal (as compute), other cloud, none-yet.
  This does not change what the container looks like; it is context
  for later skills.

### Round 2, branch B: hosted endpoint

Ask these only if the user picked "the user's own server":

- **Endpoint URL.** The base URL the container will dial (for example
  `https://<user>--<app>.modal.run`). The user has this from wherever
  they deployed the server.
- **Auth situation.** How the endpoint is secured. Options:
  - **Open.** No credential required. Anyone with the URL can hit it.
  - **Network restricted.** The endpoint is behind an IP allowlist,
    VPC, or similar. The user will need to add the Manifold runner's
    egress addresses.
  - **Requires a header token.** A bearer token or API key must ride
    each request. Flag this to the user: the Manifold platform does
    not currently give the container a secret store, so a header
    token cannot be delivered safely at run time. The workaround is
    to make the endpoint network-restricted for now, and drop the
    token, until a runner-side secret channel exists.
- **Wire contract summary.** One or two sentences from the user or
  their handoff: what the description route is called (for example
  `GET /config`), what the inference route is called (for example
  `POST /infer`), and what the wire encoding is (JSON, msgpack, a
  custom variant). The full contract is Phase 1 work for the wrap
  skill; this is just enough to record which server the wrap will
  target.

> **Phase 2 checkpoint (both branches):**
> ```
> policy_slug         = ?
> model_runtime       = in_container | hosted_endpoint
> registry_url        = ?
> registry_namespace  = ?
> benchmarks          = [list]
> display_name        = ? | default (slug)
> visibility          = ? | default (org)
> ```
>
> **Additional (branch A, in_container):**
> ```
> peak_vram_gb        = ?
> weights_location    = ?
> deployment_style    = ?
> ```
>
> **Additional (branch B, hosted_endpoint):**
> ```
> endpoint_url        = ?
> auth_situation      = open | network_restricted | header_token
> wire_contract       = ? (one or two sentences)
> ```

---

## Phase 3: Write CONTEXT.md

Create the file at `<project>/.manifold/CONTEXT.md`. Prose sections
organized by topic, not a config schema. The sections below cover the
common case: use them as a starting point, add new ones when the
project has facts that don't fit, and drop any you genuinely have
nothing to say about.

Common sections (both branches):

- `# Manifold context for this project`
- `## Project`. Package manager, the dependency file it uses, Python
  version, source folders, ML framework.
- `## Registry`. URL, namespace.

Runtime and Policies sections depend on the branch.

### Branch A: in-container model

- `## Runtime`. GPU / CUDA / Docker facts you detected, plus how the
  user typically deploys this container.
- `## Policies`. One `### <slug>` per policy, with display name,
  visibility, peak VRAM, benchmarks paired with, and a **Weights**
  paragraph (location and any auth notes).

### Branch B: hosted endpoint

- `## Runtime`. State that the built container does not load the
  model; it dials the user's inference server. Docker facts you
  detected still go here (the container is still built and pushed).
  Note that no GPU is needed on the build machine or on the runner.
- `## Policies`. One `### <slug>` per policy, with display name,
  visibility, benchmarks paired with, and an **Endpoint** paragraph.
  The Endpoint paragraph records the URL, the auth situation, and the
  wire contract summary. No Weights paragraph in this branch.

Add new sections when something is worth recording that doesn't fit
above. For example, a `## Cloud storage` section if the weights live
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

setup-manifold's own work is done. Do not chain into another skill
without asking the user first. Ask before continuing, and only proceed
if the user says yes.

Each step in the Manifold flow is a separate skill by design, so the
user gets to review the previous step's output before agreeing to the
next.

Hand back to the user in their vocabulary, not the skill's. The user
does not know what a "wrap" is or that `/wrap-policy` is the next
skill's name.

Summarize what was written:

- The file created (`.manifold/CONTEXT.md`).
- A one-line recap of the recorded context. For branch A: policy name,
  registry, weights location, benchmarks of interest. For branch B:
  policy name, registry, endpoint URL, benchmarks of interest.
- Any prerequisites Phase 1 found missing that a later skill will
  need. For branch A: Docker not installed or not reachable, or no
  GPU on a machine where the deployment style needs one. For branch
  B: Docker not installed or not reachable. Point the user at how to
  install or arrange them (for example Docker's install docs) so they
  can fix it before the next skill runs.

Then ask something like: **"Ready to prepare your policy for use with
Manifold?"** Do not name the next skill.

Pick the next skill from the branch:

- Branch A (in-container model): the next skill is `/wrap-policy`.
- Branch B (hosted endpoint): the next skill is `/wrap-remote-policy`.

If the user says yes, invoke that skill. If the user says no, or
wants to change something in `CONTEXT.md` first, stop and wait.

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
- [ ] User interview happened upfront, in at most two rounds, via
      the structured question tool (or fallback). Round 1 asked the
      branch question and the common answers; Round 2 asked the
      branch-specific follow-ups.
- [ ] `model_runtime` is recorded in `CONTEXT.md` as either
      `in_container` or `hosted_endpoint`
- [ ] Nothing invented; unknowns recorded as "unknown"
- [ ] `.manifold/CONTEXT.md` written in prose, sectioned by topic,
      including the detected package manager and the branch-specific
      Runtime and Policies sections
- [ ] Nothing else touched. No `.manifold/<slug>/` folder created, no
      project dependency added. Both are the wrap skill's job.
- [ ] Handed back to the user with a summary; asked whether to
      continue before invoking the next skill (`/wrap-policy` for
      in_container, `/wrap-remote-policy` for hosted_endpoint)
