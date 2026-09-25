# Stage 1: Set up the project

Stage 1 writes one file: **`.manifold/CONTEXT.md`** at the project
root. It is plain markdown, with everything that Stages 2 and 3 need
to know: policy slug, registry, benchmarks, the detected package
manager, GPU memory, and the weights location.

In this flow, Manifold loads and runs the model. The user gives
Manifold the model weights (as a file on disk, or a reference to a
hosted model). Each run loads the weights into GPU memory on
Manifold's machines.

Stage 1 does not touch the project's dependencies and does not create
a policy folder. Stage 2 does both.

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

**Do the interview upfront, in one round.** Ask every question in a
single call to the structured question tool. Do not drip further
questions across the skill.

**Respect the project. Zero impact on how it already works.** Figure
out which package manager the project uses and record it in
`CONTEXT.md`. If it's ambiguous, ask the user. Stage 1 installs
nothing. Stage 2 adds `manifold-sdk` and nothing else. Do not migrate
the project to a different manager.

## What CONTEXT.md is

`.manifold/CONTEXT.md` holds the context for this project. It is
plain markdown, with sections by topic (Project, Registry, Runtime,
Policies). Stage 1 writes the first version. Stages 2 and 3 read it
before they ask the user anything, and append to it as they learn
more.

The user answers each question once. Later stages do not ask again
for anything that `CONTEXT.md` records.

---

## Phase 1: Look at the project

Do this before asking anything. The interview in Phase 2 skips every
question you can answer here.

- **manifold CLI.** Confirm it is installed and the user is
  authenticated. If not, stop and tell the user to install it and run
  `manifold auth login`.
- **The project.** Figure out which package manager it uses, which
  folders hold the Python source the wrap will import from, and
  whether `.manifold/` already exists (see "Start at the right
  stage").
- **The machine.** Look up five things. Run `uname -s` for the OS and
  `uname -m` for the CPU architecture. Run `nvidia-smi` to see
  whether an NVIDIA GPU is present, and read its CUDA version and
  total VRAM if there is one. Run `which docker`, then `docker info`,
  to see whether Docker is reachable. Run `df -h` on the project's
  filesystem for free disk, because you will build an image of 10 to
  30 GB in Stage 3. Read the project's dependency file for the
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
  a path from memory.
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

## Tell the user what this machine can do

Stage 2 tests the wrap by loading the model on this machine.
The user cannot see from the interview whether this machine can do
that, so state the machine facts in the interview:

- If this machine cannot run the model, ask before continuing. Give
  one sentence of why, then the question: the wrap cannot be tested
  on this machine because it has no GPU, so does the user want to
  proceed on this machine anyway? If the user says yes, record
  `local_test_run_possible = no` in the Runtime section of
  `CONTEXT.md` and continue. If the user says no, stop here. The user
  can move to a Linux machine with an NVIDIA GPU and run this skill
  again there. Stage 2 reads that field back, skips its test run, and
  repeats the warning in its handoff.
- If this machine cannot build the image, flag that too. Stage 3
  needs Docker and 10 to 30 GB of free disk. On an
  arm64 machine the CUDA image builds under emulation. That build is
  slow and can fail on GPU wheels, so tell the user that this build
  works dependably on a Linux x86_64 machine.

---

## Phase 2: Interview the user

Ask these together, in one prompt to the user.

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
  for later stages.

Skip these unless the user brings them up: display name (defaults to
the slug), visibility (defaults to `org`).

> **Phase 2 checkpoint:**
> ```
> policy_slug         = ?
> registry_url        = ?
> registry_namespace  = ?
> benchmarks          = [list]
> image_preprocessing = none | flip_vertical | flip_horizontal | rotate_180 | unknown  (source: file:line | user)
> display_name        = ? | default (slug)
> visibility          = ? | default (org)
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

Sections:

- `# Manifold context for this project`
- `## Project`. Package manager, the dependency file it uses, Python
  version, source folders, ML framework.
- `## Registry`. The service URL and the account or organization
  the image goes under.

- `## Runtime`. The OS, CPU architecture, GPU, CUDA, and Docker
  facts you detected, plus how the user typically deploys this
  container. State whether this machine can load and run the model,
  because Stage 2 reads that back before it starts. If this machine
  cannot run the model, write `local_test_run_possible = no` here, so
  Stage 2 knows to skip its test run.
- `## Policies`. One `### <policy-name>` per policy, with display
  name, visibility, GPU memory needed, benchmarks paired with, an
  **Image preprocessing** line (the value and where it came from:
  a file and line, or the user's answer), and a **Weights**
  paragraph (location and any auth notes).

Add new sections when something is worth recording that doesn't fit
above. For example, a `## Cloud storage` section if the weights live
behind an S3 bucket with quirks, a `## Notes` section for anything a
later stage should be aware of, or a `## Known issues` section for
constraints the user flagged during the interview.

Write in plain English, without jargon or metaphors. Reach for
technical language only when it helps a reader understand the project
better. This file is read by both agents and humans.

> **Phase 3 checkpoint:**
> ```
> context_md_written        = yes
> ```

---

## Stage 1 checklist

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
      result; told the user what this machine can do during the
      interview
- [ ] If this machine cannot run the model, asked whether to proceed
      without a test run; recorded `local_test_run_possible = no`
      only after a yes
- [ ] `image_preprocessing` recorded with its source (file and line,
      or the user's answer); `unknown` only if the user said "not
      sure"
- [ ] User interview happened upfront, in one round, via the
      structured question tool (or fallback)
- [ ] Nothing invented; unknowns recorded as "unknown"
- [ ] `.manifold/CONTEXT.md` written in prose, sectioned by topic,
      including the detected package manager and the Runtime and
      Policies sections
- [ ] Nothing else touched in Stage 1. No `.manifold/<slug>/` folder
      created, no project dependency added. Both happen in Stage 2.
- [ ] Handed back to the user with a summary; asked whether to
      continue before starting Stage 2
