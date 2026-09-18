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

- **The model already runs on the user's own server. Manifold
  calls it.** The user keeps an inference server up somewhere
  (Modal endpoint, private HTTPS box). Manifold sends observations
  to it over the network and reads back the actions. No GPU is
  needed on Manifold's side.
  Next skill: `/wrap-remote-policy`, then `/containerize-remote-wrap`.
- **Manifold loads and runs the model.** The user gives Manifold
  the model weights (as a file on disk, or a reference to a hosted
  model). Each run loads the weights into GPU memory on Manifold's
  machines.
  Next skill: `/wrap-policy`, then `/containerize-wrap`.

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

**Ask in the user's language.** The user has not read the SDK docs
and will not recognize its terms. Present each concept by the
outcome it describes, not by its acronym or config field name. Ask
"how much GPU memory does the model need at inference?" rather
than "what is your peak VRAM in GB?". Ask "where can Manifold
reach the model?" rather than "what is the deployment style?". If
the user uses a term themselves, follow their lead. Otherwise stay
with plain language.

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
- **The machine.** Look up five things. Run `uname -s` for the OS and
  `uname -m` for the CPU architecture. Run `nvidia-smi` to see
  whether an NVIDIA GPU is present, and read its CUDA version and
  total VRAM if there is one. Run `which docker`, then `docker info`,
  to see whether Docker is reachable. Run `df -h` on the project's
  filesystem for free disk, because you will build an image of 10 to
  30 GB in a later skill. Read the project's dependency file for the
  name of the ML framework.

  Then decide one thing from those results: **can this machine load
  and run the model?** A Linux machine with an NVIDIA GPU can run it.
  macOS cannot, because CUDA does not run on macOS. A machine with no
  NVIDIA GPU cannot run it either. Windows with an NVIDIA GPU is
  uncertain: CUDA works there, but many robotics packages publish
  Linux-only wheels, so ask the user whether their model stack runs
  on Windows, and mention WSL2 as the usual route. For anything
  else, ask the user instead of guessing. Record the answer as `can_run_model_locally`.
  When the answer is no, follow the section after this phase.
- **Weight files.** Look for likely checkpoint folders in the project
  so you can propose them to the user rather than asking them to type
  a path from memory. Only relevant for the in-container-model branch.
- **Image preprocessing.** Search the project source for a flip or
  rotation applied to camera frames before inference. Patterns to
  grep for: `[::-1]` on an image array, `np.flip`, `np.flipud`,
  `np.fliplr`, `np.rot90`, `rotate(180)`, `transpose` on an image,
  or a config flag such as `flip_images`. If you find one, note the
  file and line and which orientation change it makes: flipped top
  to bottom, flipped left to right, or rotated 180 degrees
  (`img[::-1, ::-1]` is a 180-degree rotation). Skip the interview
  question about this when the search finds something.
- **Available benchmarks.** Run `manifold benchmark list` and note
  the slugs and one-line descriptions so you can present them as
  options in the interview.

> **Phase 1 checkpoint:**
> ```
> manifold_cli_ready      = yes | no  (if no: stop)
> package_manager         = ?
> manifold_folder_exists  = yes | no  (if yes: what does CONTEXT.md say?)
> source_folders_seen     = [list]
> os                      = linux | macos | windows | ?
> arch                    = x86_64 | arm64 | ?
> gpu_present             = yes | no
> cuda_version            = ? | unknown
> can_run_model_locally   = yes | no  (yes only on linux with an NVIDIA GPU)
> docker_present          = yes | no
> free_disk_gb            = ?
> ml_framework            = ? | unknown
> weight_dirs_seen        = [list, or empty]
> image_preprocessing_found = none | flip_vertical | flip_horizontal | rotate_180  (file:line) | not_found
> available_benchmarks    = [list of {slug, description}]
> ```

---

## Tell the user what each path needs before they choose

The branch question below asks where the model runs. The answer also
decides what this machine needs to be able to do, and the user
cannot see that from the question. State the machine facts while
asking, in one or two plain sentences, whether
`can_run_model_locally` is yes or no. For example, on a machine with
no GPU:

> There are two ways to connect your policy. The first is to keep
> running it on your own server and let Manifold call it. That path
> works fully from the machine you are on. The second is to give
> Manifold your weights and let Manifold run the model. You can
> prepare that from here too, but the test at the end of the next
> step has to load your model on this machine, and this machine
> cannot load it. The first full test would then happen on
> Manifold's machines instead.

Both answers stay open on any machine. The machine facts decide what
you flag, not what the user may pick:

- If the user picks in-container and the machine cannot run the
  model, ask before continuing. One sentence of why, then the
  question: the wrap cannot be tested on this machine because it has
  no GPU, so do they want to proceed on this machine anyway? On a
  yes, record `local_test_run_possible = no` in the Runtime section
  of `CONTEXT.md` and continue. On a no, offer the two ways out.
  First, switch to the hosted endpoint path: continue this same
  interview on branch A, and the user deploys their model to their
  own server before the next skill. Second, move to a Linux machine
  with an NVIDIA GPU: stop here, and the user runs this skill again
  on that machine. `/wrap-policy` reads that field back, skips its
  test run, and repeats the warning in its handoff.
- If the user picks in-container and the machine cannot build the
  image, flag that too. The containerize step needs Docker and 10 to
  30 GB of free disk. On an arm64 machine the CUDA image builds
  under emulation, which is slow and can fail on GPU wheels, so tell
  the user that this build works dependably on a Linux x86_64
  machine.
- If the user picks the hosted endpoint, the model never runs on
  this machine, and no GPU is needed at any step. The machine still
  needs Docker, because the containerize step builds a small image
  that forwards requests to their server.

---

## Phase 2: Interview the user

Two rounds of questions.

### Round 1: branch question and common answers

Ask these together, in one prompt to the user. They apply regardless
of which branch is chosen.

- **Where does the model run when Manifold uses it?** Options:
  - **The user already runs it on their own server. Manifold calls
    it.** The user has an inference server up somewhere (a Modal
    endpoint, a private HTTPS box). Manifold sends observations to
    it over HTTPS and reads back the actions.
  - **Manifold loads and runs it.** The user gives Manifold the
    model weights (as a file on disk, or a reference to a hosted
    model). Manifold loads them into GPU memory on their machines
    each run.

  Ask the question in whatever plain-language wording fits the
  conversation; the exact phrasing above is not required. Avoid the
  word "container" unless the user has already used it. The answer
  picks the branch for Round 2 and the recommended next skill in
  the handoff.
- **Policy name.** What to call this policy on Manifold. The name
  becomes the identifier used in the register command later. The
  user names it. Do not suggest one.
- **Where to push the container image.** Manifold packages the
  policy into a container image (a self-contained bundle that runs
  the same way on any machine) and pushes it to a storage service
  like ghcr.io or Docker Hub. Ask for the service URL and the
  account or organization the image goes under. For example
  `ghcr.io/<org>`. Default to `ghcr.io` if the user has no
  preference.
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
- **Image preprocessing.** Ask only when the Phase 1 search found
  nothing. "Before your model sees a camera image, is the raw image
  flipped or rotated?" Options: no change; flipped top to bottom;
  flipped left to right; rotated 180 degrees; not sure. If the user
  says the cameras differ, ask once for each camera. Explain in one
  sentence why it matters: the benchmark sends images in a fixed
  orientation, and the wrap has to convert them to match the training
  images. The local checks do not test image orientation. If the
  answer is wrong, the run scores near zero. If the user picks "not
  sure," record `unknown`.

Skip these unless the user brings them up: display name (defaults to
the slug), visibility (defaults to `org`).

### Round 2, branch A: hosted endpoint

Ask these only if the user picked "the user's own server":

- **Is the inference server already deployed and reachable?** If
  not, record `endpoint_status = not_yet_deployed` in `CONTEXT.md`.
  Tell the user in the handoff that the Manifold skills do not write
  or deploy the server. They need to deploy it before running the
  next skill.
- **Endpoint URL.** The base URL the container will dial (for example
  `https://<user>--<app>.modal.run`). The user has this from wherever
  they deployed the server.
- **How is the endpoint secured?** Options:
  - **No security.** Anyone with the URL can call it.
  - **Restricted by network.** Only certain machines can reach the
    URL, controlled by an allowlist of IP addresses or a private
    network. The user will need to allow the addresses Manifold's
    runners use.
  - **Requires an API key or token in each request.** The token goes
    into the registered version's config as an environment variable
    (`manifold policy init --env`). The platform stores it in
    plaintext. Tell the user this. Network restriction is the safer
    option when the user can arrange it.
- **Request and response format.** A one or two sentence summary
  from the user or their setup notes. What HTTP route describes
  the loaded checkpoint (for example `GET /config`), what HTTP
  route accepts an observation (for example `POST /infer`), and
  what data format the request and response bodies use (JSON,
  msgpack, a custom variant). The full details are Phase 1 work
  for the wrap skill. This is just enough to record which server
  the wrap will target.
- **How many requests can the server handle at once?** One, if the
  server runs one model on one GPU with no replicas. More, if it
  runs several replicas. If the user is not sure, record `1`. The
  wrap uses this number to cap how many benchmark runners call the
  server at the same time.

### Round 2, branch B: in-container model

Ask these only if the user picked "Manifold loads and runs it":

- **How much GPU memory does the model use at inference, in GB?**
  The user knows this from their own runs. The agent cannot
  measure it without running the model.
- **Where the model weights live.** If Phase 1 found candidate
  folders, present them as options plus "elsewhere" (a hosted
  model reference, cloud storage, or a path the user will type in).
  The user picks or fills in the actual location.
- **Where the user typically deploys this container.** Options: local
  box, Modal (as compute), other cloud, none-yet.
  This does not change what the container looks like; it is context
  for later skills.

> **Phase 2 checkpoint (both branches):**
> ```
> policy_slug         = ?
> model_runtime       = in_container | hosted_endpoint
> registry_url        = ?
> registry_namespace  = ?
> benchmarks          = [list]
> image_preprocessing = none | flip_vertical | flip_horizontal | rotate_180 | unknown  (source: file:line | user)
> display_name        = ? | default (slug)
> visibility          = ? | default (org)
> ```
>
> **Additional (branch A, hosted_endpoint):**
> ```
> endpoint_status     = deployed | not_yet_deployed
> endpoint_url        = ?
> auth_situation      = open | network_restricted | header_token
> wire_contract       = ? (one or two sentences)
> concurrent_requests = ? (1 if unsure)
> ```
>
> **Additional (branch B, in_container):**
> ```
> peak_vram_gb        = ?
> weights_location    = ?
> deployment_style    = ?
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
- `## Registry`. The service URL and the account or organization
  the image goes under.

Runtime and Policies sections depend on the branch.

### Branch A: hosted endpoint

- `## Runtime`. State that the built container does not load the
  model. It calls the user's inference server. Docker facts you
  detected still go here (the container is still built and pushed),
  along with the CPU architecture, because the image has to be built
  for linux/amd64. Note that no GPU is needed on this machine or on
  the runner.
- `## Policies`. One `### <policy-name>` per policy, with display
  name, visibility, benchmarks paired with, an **Image
  preprocessing** line (the value and where it came from), and an
  **Endpoint** paragraph. In the Endpoint paragraph, write whether
  the server is deployed yet, the URL, how the endpoint is secured,
  the one-line summary of the request and response format, and how
  many requests the server handles at once. No Weights paragraph in
  this branch.

### Branch B: in-container model

- `## Runtime`. The OS, CPU architecture, GPU, CUDA, and Docker
  facts you detected, plus how the user typically deploys this
  container. State whether this machine can load and run the model,
  because `/wrap-policy` reads that back before it starts. If this
  machine cannot run the model, write `local_test_run_possible = no`
  here, so `/wrap-policy` knows to skip its test run.
- `## Policies`. One `### <policy-name>` per policy, with display
  name, visibility, GPU memory needed, benchmarks paired with, an
  **Image preprocessing** line (the value and where it came from:
  a file and line, or the user's answer), and a **Weights**
  paragraph (location and any auth notes).

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
  registry, endpoint URL, benchmarks of interest. For branch B:
  policy name, registry, weights location, benchmarks of interest.
- Any prerequisites Phase 1 found missing that a later skill will
  need. On both branches, say so if Docker is not installed or not
  reachable. On branch B, give the user two more facts when they
  apply. If this machine cannot run the model, `/wrap-policy` will
  skip its test run, and the wrap stays untested until its first run
  on Manifold. If this machine is arm64, the image must be built for
  linux/amd64, which runs under emulation and can fail on GPU
  wheels. Point the user at how to install or arrange each missing
  piece (for example Docker's install docs) so they can fix it
  before the next skill runs.

Then ask something like: **"Ready to prepare your policy for use with
Manifold?"** Do not name the next skill.

Pick the next skill from the branch:

- Branch A (hosted endpoint): the next skill is `/wrap-remote-policy`.
- Branch B (in-container model): the next skill is `/wrap-policy`.

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
- [ ] Cheap lookups (package manager, OS, CPU architecture, GPU,
      Docker, source folders, framework, image preprocessing,
      `manifold benchmark list`) done without asking; user asked
      only for what is hard or user-only
- [ ] `can_run_model_locally` decided from the OS and the GPU
      result; told the user what hardware each answer needs while
      asking the branch question
- [ ] If this machine cannot run the model and the user picked the
      in-container path, asked whether to proceed without a test
      run; recorded `local_test_run_possible = no` only after a yes
- [ ] `image_preprocessing` recorded with its source (file and line,
      or the user's answer); `unknown` only if the user said "not
      sure"
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
