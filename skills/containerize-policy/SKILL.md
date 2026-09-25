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

Use the manifold-sdk to write the files that let a Manifold benchmark
run the user's policy. The output is a folder of Python files. Three
kinds of file, each with its own role. The skill uses these three
words as labels for the files throughout:

- The **driver** loads the model and runs it.
- The **profile** describes what the model expects as input and returns
  as output.
- The **pairing** ties one driver-plus-profile to one benchmark. There
  is one pairing file per benchmark the user wants to run against.

Together these files are called the **wrap**.

All conversions and transformations happen in the wrap. The benchmark
side is fixed.

Stage 2 is complete when both `check_compatibility` and `verify` pass AND
the wrap runs cleanly under `evaluate`. The two checks pass specs and dummy
data through the PIPELINE only. Any wrong gripper polarity, action width, or
proprioception can pass them but crash (or fail silently) at runtime, which
is why the live `evaluate` run is mandatory. One exception: on a machine
that cannot run the model, Stage 2 is complete when both checks pass, the
`evaluate` run is skipped, and the handoff says so (see "Run the wrap").

**Check whether this machine can run the model, before Phase 4.**
Phase 6 finishes with a test run that loads the user's real
checkpoint, and that run needs Linux and an NVIDIA GPU. No earlier
step needs a GPU. `CONTEXT.md` records the answer as
`can_run_model_locally`, and records `os`, `arch`, and `gpu_present`
next to it. If the file predates those fields, run the lookups
yourself: `uname -s`, `uname -m`, `nvidia-smi`.

A machine that cannot run the model does not block the wrap, but the
user decides. If `CONTEXT.md` already records
`local_test_run_possible = no`, the user said yes to this during
Stage 1. Restate it in one sentence and continue. If the
field is absent, ask now: the wrap cannot be tested on this machine
because it has no GPU, so do they want to proceed anyway? On a yes,
record `local_test_run_possible = no` in `CONTEXT.md`, write the
wrap, run both checks, and skip the test run as described under
"Run the wrap". On a no, stop, and help them move to a machine with
a GPU.

**Prepare the project for the wrap.** Stage 1 wrote `CONTEXT.md` and
nothing else. Do these before writing any wrap code:

- Add `manifold-sdk` to the project's dependency file **and** install
  it into the project's environment. Start with the most recent commit
  on the `manifold-sdk` GitHub repository's default branch, then pin
  the dependency to that commit's full SHA. With uv:

  ```
  uv add "manifold-sdk @ git+https://github.com/bifrostai/manifold-sdk.git@<commit>"
  ```

  With plain `requirements.txt`, add the same pinned Git URL, then run
  `pip install -r requirements.txt`. Add a **manifold-sdk revision**
  line with the full SHA to the policy's `### <policy-name>` subsection
  in `.manifold/CONTEXT.md`. This records the SDK revision used for the
  wrap's checks. Use the same revision when building the container.
  Recording without installing (or the reverse) leaves the project
  half-set-up. If the install fails on a dependency conflict, stop and
  hand the error to the user.
- Create the folder `<project>/.manifold/<slug>/`. The slug is in
  `CONTEXT.md`. All wrap files below live inside it.

## Folder structure

All wrap files live under `<project>/.manifold/<slug>/`, where
`<slug>` is the policy name recorded in `CONTEXT.md`:

| File | Description | Imports the model? |
|---|---|---|
| `.manifold/<slug>/driver.py` | endpoint + session | yes |
| `.manifold/<slug>/profile.py` | frozen dataclass: signature, weights, chunk, exec_steps, layouts, `load()` | no |
| `.manifold/<slug>/<benchmark>.py` | pairing file, exports `PROFILE` / `BENCHMARK` / `PIPELINE` (one file per benchmark) | no |

- `driver.py` loads the model onto the GPU and runs it.
- `profile.py` is a lightweight spec describing what the model expects (image sizes, state shape, chunk length, weights path).
- `<benchmark>.py` is the pairing file. It builds the signature and layouts, instantiates the profile, and declares `PROFILE`, `BENCHMARK`, and `PIPELINE`.

`PROFILE` points at the model spec (from profile.py)
`BENCHMARK` points at the test to run
`PIPELINE` is the list of adapters that translate between the benchmark's data and what the model wants.

---

## Phase 4: Understand the policy and target benchmark

Do this before writing any Manifold code. You need a full picture of both sides
to design anything.

### Policy side

**Find the real policy class**, the main policy model code the project runs to
ingest observations and get actions.

**Read any inference or evaluation code** in files and folders like `eval*`,
`infer*`, `scripts*`, `benchmark*`, `env*`, `run*`, `*server*`, `rollout*`. If
there are existing benchmarks or simulation code in the user's policy repository,
note that these are likely NOT Manifold benchmarks. Still, it is necessary to
read them to understand if any conversions and transformations need to be
implemented in the policy wrap, such as image resize, gripper sign and threshold,
and normalization.

**Identify the chunk numbers:**
- *chunk* = actions predicted per forward pass
- *exec_steps* = actions executed in simulation before running the next forward pass

Find the chunk-returning method (e.g. `predict_action_chunk` is common), and
call that from your driver. Let manifold-sdk's queue handle open-loop dispensing.

Manifold's serving contract is strictly open-loop (predict a chunk, execute the
chunk, predict next chunk).

**Write down the action width** from a model source. Check by hand that it
equals `SIGNATURE.action_space.expected_length()`. No SDK assertion exists.

**If the checkpoint is LeRobot-shaped**, run
`manifold.recipes.from_lerobot_checkpoint(path)` first. It returns a
`SignatureSuggestion` with cameras, action dim, and an explicit undetermined
list. Caveats: camera names are LeRobot feature names (not benchmark sensor
names), and `draft_policy_spec` always builds `JointActionSpace` (wrong for EE
policies).

**Check for normalization.** Normalizer artifacts in the checkpoint directory
mean the model does not normalize itself. The method tutorials call may return
normalized values.

**Find the checkpoint** from the project itself. Do not proceed against a known
embodiment mismatch.

**Note observation preprocessing:** resizes, flips, channel swaps, frame
stacking, proprio re-encoding, normalization.

**Declare camera orientation relative to the real scene. Do not declare
it relative to another step in the pipeline.** Ask: if a person stood in
the scene, how would this image look to them? `UPRIGHT` means the image
looks correct. `FLIPPED_VERTICAL` means the image is upside down, because
the rows are in reverse order. If you declare an orientation using the
wrong reference, the checks will still pass, and the policy will score
near zero.

**If the target benchmark is based on MuJoCo or LIBERO, confirm the
orientation of the training data before you design the signature.** Do not
trust code comments. Comments about LIBERO orientation contradict each
other across the ecosystem. The common LIBERO training sets (OpenVLA RLDS,
HuggingFaceVLA/libero) were rotated 180 degrees during conversion, so their
images are horizontally mirrored relative to the real scene. Confirm the
orientation in one of two ways:

- Trace the training data. Find the dataset that was used to train the
  checkpoint, open a few sample frames, and look for printed text or an
  object with a known left and right side. If text in the frames is
  mirrored, the images are mirrored. Do not judge by gravity alone. A
  mirrored scene still looks normal.
- Ask the user. If the user does not know and you cannot reach the
  dataset, stop. Guessing the orientation is the known cause of LIBERO
  scores near zero.

Record the answer, and the evidence for it, in the Phase 4 checkpoint.

### Benchmark side

**Find the target Manifold benchmark** in `manifold.benchmarks` (e.g. LIBERO,
SIMPLER, ROBOCASA). Use the manifold-sdk's `Benchmark`, not the user's policy
project's vendored copy.

**Read the `Benchmark`:** `embodiment.action`, `embodiment.proprioception`,
`sensors` (name, resolution, mount), `instruction`.

**If the benchmark is not available**, stop and notify the user. The benchmark
must be authored first.

> **Phase 4 checkpoint.** Record before designing anything:
> ```
> Policy side:
>   action_width      = ?  (source: file:line)
>   action_space      = EE|Joint|Unified
>   rotation          = ?  (source: file:line)
>   gripper           = ?  (source: file:line)
>   delta             = ?  (source: file:line)
>   frame             = ?  (source: file:line)
>   chunk             = ?  (source: file:line)
>   exec_steps        = ?  (source: file:line)
>   normalization     = yes|no, location: ?
>   checkpoint        = ?
>   cameras           = [{name, shape, orientation}]
>   orientation_evidence = dataset sample | user answer  (REQUIRED for MuJoCo/LIBERO targets; unknown -> stop)
>   instruction       = true|false
>   proprioception    = ee_pose|joint_pos|none
>
> Benchmark side:
>   target_benchmark  = LIBERO|SIMPLER|ROBOCASA|...
>   embodiment_action = {type, rotation, gripper, delta, frame}
>   cameras_published = [{name, shape, orientation}]
>   instruction       = true|false
>
> Feasible: true|false  (if false: why, and stop)
> ```

---

## Phase 5: Design the signature, layouts, and profile

Decide what your wrap will look like before writing any code. Everything here
is a decision, not implementation.

**Which file holds which thing:**

| Thing | File |
|---|---|
| `Profile` class (the dataclass template) | `profile.py` |
| `PolicySignature(...)` instance | pairing file |
| Two `NativeLayout(...)` instances (input + output) | pairing file |
| Profile instance (`POLICYNAME_BENCHMARKNAME = MyProfile(...)`) | pairing file |
| `PROFILE`, `BENCHMARK`, `PIPELINE` module-level names | pairing file |

`profile.py` defines the Profile class and its fields (weights, chunk size,
layouts, and so on). The pairing file creates one and fills those fields with
real values.

### The signature: what the model emits and consumes

The pairing file declares a signature. This is a description of the
action space the model emits and the observation it consumes. In
code it is a `PolicySignature`.

**Stop if the action is not end-effector, joint, or a unified
variant** (in code: `EEActionSpace`, `JointActionSpace`, or
`UnifiedActionSpace(payload=...)`). Same if the state is not
end-effector pose or joint position (`ee_pose` or `joint_pos`).
Anything else is benchmark work, not a wrap.

- Report real-world units. Meters, radians, gripper state. If your model outputs
  normalized values, convert them in the driver before returning.
- The checks won't catch lies about your model. They compare your signature to the
  benchmark, not to what your model actually does. Wrong shapes pass the checks and
  crash at runtime.
- Describe cameras as what the model actually sees, after your pipeline reshapes them.
  Include extra dimensions from frame stacking.
- No proprioception? Say so with an empty `Proprioception()`. Don't invent fake data
  to fill the gap.
- No instruction? Just set `instruction=False`.
- Don't conflate the two things called "chunk":
  - One action holding N steps (almost always 1)
  - Actions predicted per forward pass (often 10)
- The chunk queue needs both `pack` and `unpack`. Skip either and the server crashes.

Spell out every convention field (`rotation=`, `gripper=`, `delta=`, `frame=`)
from the project's eval. Do not copy the benchmark's action object.

### The layout: benchmark observation to model input, model output to action

The benchmark hands the wrap an observation object. The model
expects a dictionary. A layout describes how each dictionary key
gets filled from the observation, and how the model's output is
sliced back into an action. In code these are two `NativeLayout`
instances.

The input layout maps observation channels to dictionary keys. The
output layout maps raw model output back to an action. Key renames
are entries with no ops.

- **Plain `(chunk, dim)` output**: use
  `LayoutEntry(key=..., source=SourceKind.STATE, source_name=None,
  ops=(Slice(start=0, stop=dim),))`. `Slice` takes keyword args only.
  `Slice(0, dim)` raises `TypeError`.
- **`state["ee_pose"]` layout**: `[pos3, rotation, gripper_qpos]`. Widths from
  `ee_step_layout` / `expected_length()`. Be careful, wrong slices don't get
  caught by pre-flight checks.
- **Instruction entry**: use `SourceKind.INSTRUCTION`. Every step, the current
  task's instruction is sent to the policy. Do not cache it on the policy.

### The profile: the wrap's spec

The profile is a small object that carries the signature, the input
and output layouts, the weights location, and two chunk-related
numbers. It also has a `load()` method that opens the model. In
code it is a frozen dataclass satisfying `recipes.PolicyProfile`,
with fields `signature`, `default_weights`, `input_layout`,
`output_layout`, `chunk`, `exec_steps`, and a
`load(weights, device) -> PolicyEndpoint` method.

Defer the driver import into `load()` so `profile.py` imports without
the model stack. Take `device` as a `load()` parameter. Checkpoints
bake in the training device, and the caller passes the runtime one at
load time.

- **`chunk`/`exec_steps` have no check.** Missing `exec_steps` by exact name
  fails at the first step in `advance`. Missing `.profile` on the endpoint
  kills the connection pre-READY: `PairingRejected("policy rejected the
  pairing (no READY)")`.

> **Phase 5 checkpoint:**
> ```
> PolicySignature:
>   action_space        = {type}(rotation=?, gripper=?, delta=?, frame=?, chunk_size=1)
>   action_width_check  = expected_length() == model source width? yes|no
>   proprioception      = {ee_pose: ..., joint_pos: ...}
>   cameras             = [{name, shape, dtype}]
>   instruction         = true|false
>
> NativeLayout input keys:  [list with source_kind and source_name]
> NativeLayout output keys: [list with ops]
>
> Profile: chunk=?, exec_steps=?, default_weights=?
> ```

---

## Phase 6: Implement the wrap

Write the driver, assemble the pipeline, pass both checks, then prove it live.

### File skeletons

Write these three files. Each snippet below is the minimum shape. Fill in the
model-specific logic, then flesh out with the rules that follow.

**`profile.py`**. Frozen dataclass, no model-stack imports at top level:

```python
from dataclasses import dataclass
from manifold.core.native_layout import NativeLayout
from manifold.core.policy import PolicySignature

@dataclass(frozen=True)
class MyProfile:
    signature: PolicySignature
    default_weights: str
    input_layout: NativeLayout
    output_layout: NativeLayout
    chunk: int
    exec_steps: int

    def load(self, weights: str, device: str | None):
        from mywrap.driver import MyEndpoint  # deferred: keeps profile.py light
        return MyEndpoint(self, weights, device)
```

**`driver.py`**. Endpoint (loads the model once) + session (one per runner):

```python
import threading
from manifold.recipes import OpenLoopChunkQueue

class MyEndpoint:
    def __init__(self, profile, weights, device=None):
        self._profile = profile
        self.signature = profile.signature       # expose by identity, not copy
        self._model = load_the_model(weights, device)
        self._lock = threading.Lock()

    @property
    def profile(self):
        return self._profile

    def session(self):
        return MySession(self)

    def forward(self, native):
        with self._lock:
            return self._model.predict_action_chunk(native)   # or whatever

class MySession(OpenLoopChunkQueue):
    def _forward(self, native):
        return self._endpoint.forward(native)
```

**`<policyname>_<benchmarkname>.py`**. The pairing file. Builds the signature
and two layouts, instantiates the profile, then declares `PROFILE`,
`BENCHMARK`, `PIPELINE`:

```python
from manifold.core.pipeline import Pipeline
from manifold.adapters import PackToNativeLayout, UnpackFromNativeLayout
from mywrap.profile import MyProfile

SIGNATURE = PolicySignature(...)
INPUT_LAYOUT = NativeLayout(entries=(...))
OUTPUT_LAYOUT = NativeLayout(entries=(...))

MYPOLICY_MYBENCH = MyProfile(
    signature=SIGNATURE,
    default_weights="...",
    input_layout=INPUT_LAYOUT,
    output_layout=OUTPUT_LAYOUT,
    chunk=...,
    exec_steps=...,
)

PROFILE = MYPOLICY_MYBENCH
BENCHMARK = ...    # the manifold.benchmarks entry
PIPELINE = Pipeline(
    observation=[...],   # benchmark form -> signature's consumed form
    action=[...],        # signature's emitted form -> benchmark's
    pack=PackToNativeLayout(PROFILE.input_layout),
    unpack=UnpackFromNativeLayout(PROFILE.output_layout, SIGNATURE.action_space),
)
```

The rest of this phase is the rules for filling those `...` in correctly.

### Where to put each translation

The benchmark's data won't match what your model expects. Something has to
convert between them: resize a camera, change a rotation format, normalize a
state vector, and so on.

The SDK gives you three places to put those conversions. They trade
off how much the SDK can check for how much freedom you have. Higher
on the list below, the SDK can check what you did. Lower on the
list, you can do anything but the SDK cannot check it.

- **Pipeline adapters**. For reusable typed conversions. Built-in adapters
  cover rotation format, gripper polarity, camera resize/flip/channel-order,
  frame rebase, and frame history. `check_compatibility` inspects the chain
  and validates it.
- **NativeLayout entries**. For building the input dictionary the
  model reads: renaming keys, casting to `float32`, slicing arrays,
  splitting one vector into two. Not typed, so `check_compatibility`
  cannot reason about them, but `verify` pushes data through and
  confirms the shapes come out right.
- **Driver session code**. For per-model idiosyncrasies the SDK has no
  adapter for: normalization stats baked into a specific checkpoint,
  transposing image axes to `(C, H, W)` for torchvision, computing a gripper
  "openness" from raw joint positions. Neither check sees any of this.

Prefer the highest place that fits. Every session-side operation is invisible
to the checks, so name each one in the pairing docstring.

### Driver

`driver.py` defines two classes: an **endpoint** (loads the model once, shared
by all runners) and a **session** (one per runner, holds the chunk buffer).

**Expose the profile's `SIGNATURE` object as `endpoint.signature`**, not a
copy. The live check reads it by identity.

**Take a `threading.Lock` around the model in `forward`.**

**Keep nothing mutable on the endpoint.** Per-episode state goes in the
session or a stateful adapter.

**Chunked policy**: subclass `OpenLoopChunkQueue`, implement only `_forward`,
return the full chunk. Do not build your own chunk buffer.

**Instruction**: `native[key]` arrives as `("text",)`. Unwrap with `str(x[0])`.

**Axis transpose**: no permute op exists; `SwapChannelOrder` is RGB↔BGR only.
Torchvision models need `(C, H, W)`. Transpose in the session.

**Override `reset()`** (calling `super().reset()`) only if the model holds
per-episode state.

### Pipeline

`recipes.resolve(policy, benchmark, adapters)` finds a chain that bridges the
pairing. Use it as a head start when filling in the pipeline's observation and
action adapter lists. It cannot discover stateful adapters (filters them out).

**Convert encodings, never spaces.** Rotation re-encode, gripper polarity,
resize: encodings. Joint↔EE, position↔velocity: different controllers. Wrong
checkpoint, stop.

**No loose convention math in the driver.** Use catalog adapters or write a
custom adapter (see the `manifold.core.adapter` protocol).

**Replicate every step of the project's action postprocessing.** Read
every line. Helpers routinely scale, then negate, then clip.

**Prove each conversion is needed.** Sample the model's output at low,
middle, and high inputs and compare to what the benchmark expects.
If they already match, add no adapter.

**Wire every input the signature consumes.** Zeros for a trained state input
scores zero silently.

### Traps

Three edge cases the checks won't catch.

**`StackFrameHistory` stacks only cameras.** The adapter stacks camera frames
over time, but leaves proprioception alone. If your model wants stacked state
too, either buffer state history yourself in the session, or write a custom
adapter.

**Action adapters that read a value written by observation adapters fail
`verify`.** Adapters can pass data across the two halves through a shared
Python dictionary (`state_key`). `verify` tests the action chain with an
empty dictionary and never runs the observation chain to populate it, so any
such value is missing, so the action adapter either crashes (`verify` fails) or
fakes a default (`verify` passes, but the first action of every real episode
is silently wrong). Workaround: do the conversion in the driver session;
checks don't inspect session code. Real fix: `verify` should run the
observation chain first. File as an SDK bug.

**No absolute↔delta adapter ships.** The SDK has no action adapter that
converts between absolute pose and delta pose. If your model wants deltas but
the benchmark gives absolutes (or vice versa), write the pair yourself: the
observation adapter writes the current `ee_pose` to the shared dictionary,
the action adapter subtracts it. For chunked models (`exec_steps > 1`), only
the first predicted action has a real reference pose to subtract, since the rest
would need faked references. Flag it and stop; don't fake it.

### Check the wrap

Two checks confirm the wrap is well-formed before you try to run it.

- **`check_compatibility`** compares your signature to the benchmark, walking
  through the pipeline's adapter chain. It returns one of three verdicts:
  `COMPATIBLE`, `COMPATIBLE_VIA_PIPELINE` (works after the adapters run), or
  `INCOMPATIBLE`.
- **`verify`** pushes fake data through the pipeline and reports which pieces
  it managed to exercise. Anything under `not_checked` is a piece only a live
  run can prove.

```python
from manifold.core.check import check_compatibility
from manifold.core.verify import verify

report = check_compatibility(SIGNATURE, BENCHMARK, PIPELINE)
verified = verify(SIGNATURE, BENCHMARK, PIPELINE)
```

Fix your code until both pass. **Never change the signature to shut a check
up.** A signature that doesn't actually match the model will pass the checks
but crash the moment a real observation arrives.

Once both pass, confirm the module itself is well-formed:

```sh
python -c "
import importlib
from manifold.recipes import read_pairing
print(read_pairing(importlib.import_module('<your wrap module>')))
"
```

If `verify` reports `not_checked` entries, list them in the handoff. Do not
claim the wrap is "verified" if pieces went untested.

**When `check_compatibility` returns INCOMPATIBLE**, dump both specs
(`model_dump(mode="json")`) and diff key by key. The table below lists the fix
for each kind of mismatch:

| Mismatch | Fix |
|---|---|
| action-space class (Joint vs EE vs Unified) | STOP: different controller |
| `rotation` (action) | `RotationFormatAdapter` |
| `rotation` (proprio) | `ProprioRotationAdapter` |
| `gripper` polarity (action) | `GripperPolarityAdapter`; `GripperThresholdAdapter` for continuous |
| gripper on Unified action | `UnifiedGripperThresholdAdapter` or `DiscreteBinarize` |
| gripper encoding (observed) | `ObservedGripperAdapter` |
| `frame` (proprio) | `FrameRebaseAdapter` / `DynamicFrameRebaseAdapter` |
| camera shape | `ResizeCameras` |
| camera orientation/channel | `Rotate180Cameras` / `FlipVerticalCameras` / `SwapChannelOrder` |
| camera name mismatch | no rename adapter, custom adapter needed |
| camera rank (clip vs frame) | `StackFrameHistory` + clip shape in signature |
| action width (EE → padded) | `BasePinWiden` (EE→Unified); `UnifiedSliceAdapter` (Unified→EE) |
| `chunk_size` | set to 1, the chunk lives in raw output |
| `delta` (single-step) | custom stateful adapter pair (see Traps above) |
| `delta` (chunked) | open problem, flag it |
| `frame` (action) | no adapter, needs kinematics, STOP |
| joint ↔ EE | STOP: different controller |

Both orientation values describe the image relative to the real scene. The
adapter converts from the benchmark's declared orientation to the policy's
declared orientation. `verify` cannot test orientation with synthetic data,
so the policy's declared orientation must come from the evidence in the
Phase 4 checkpoint. Do not guess it.

### Run the wrap (MANDATORY)

The two checks only test the pipeline. Your driver code doesn't run until a
benchmark connects. That's the last thing to prove.

This step is mandatory on any machine that can run the model. Skip it
only when the machine cannot (`local_test_run_possible = no` in
`CONTEXT.md`, or `can_run_model_locally = no`). When you skip it, say
three things in the handoff: the wrap passed both checks, the driver
has not run yet, and the first full test happens when the user
submits a run in Stage 3. Do not describe the wrap as
verified.

Use `evaluate` to run the wrap in a single Python process with no server. Any
exception comes back as a traceback:

```python
from manifold.recipes import evaluate
result = evaluate(
    endpoint, BENCHMARK, reset, step,
    pipeline=PIPELINE, episodes=2, max_steps=12,
)
```

You supply `reset` and `step`: `reset()` starts a new episode and returns the
first observation; `step(action)` applies an action and returns
`(next_observation, reward, done, info)`. Together they are a minimal
stand-in for the benchmark's real physics. Use the catalog `Benchmark` for
shapes and write the simplest possible step logic (integrating action deltas
into a pose, rendering a frame that changes each step). The run scores
nothing. It exists to prove the driver runs a full episode without
raising an exception.

**How to describe this run to the user.** Call it "a test run with
dummy inputs to check that the driver runs without errors." Do NOT
say you are "faking the physics" or "using fake data." Those
phrases are correct SDK jargon but read as untrustworthy to a
non-engineer.

**Do not treat episode-to-episode differences as a wrap bug.** With
hand-written `reset` and `step`, the two episodes will not exactly
reproduce each other unless the stand-in is deterministic. That is
expected. The live run only tests one thing. Did the driver run
without raising an exception? If both episodes finish
without an exception, proceed to the handoff. Do not go back and
edit the wrap to make the episodes match.

> **Phase 6 checkpoint:**
> ```
> Pipeline observation adapters: [list, in order]
> Pipeline action adapters:      [list, in order]
> Session-side operations:       [list]
>
> Trap decisions:
>   StackFrameHistory proprio    = handled|n/a
>   Cross-side stateful adapter  = identity-fallback|session-side|n/a
>   Absolute↔delta pair          = written|n/a
>
> Conversions NOT added: [list, with three-extremes justification]
>
> Checks:
>   check_compatibility          = COMPATIBLE|COMPATIBLE_VIA_PIPELINE|INCOMPATIBLE
>     lossless                   = true|false
>     reasons                    = [if any]
>   verify                       = N checked, M failed, K not checked
>     failed                     = [list]
>     not_checked                = [list, quote in handoff]
>   read_pairing                 = success|failure
>
> Live run:
>   episodes                     = ?
>   forward_count                = ? (expected: ceil(steps / exec_steps) * episodes)
>   ran without raising          = yes | no | skipped (machine cannot run the model)
> ```

---

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

Stage 3 takes the wrap from Stage 2 (a Python module that exports
`PROFILE`, `BENCHMARK`, and `PIPELINE`) and gets it running on the
Manifold platform.

The output is four things:

1. **A `Dockerfile`** written to `<project>/.manifold/<slug>/Dockerfile`.
   It builds the policy image with the project root as the build
   context.
2. **A launcher script** written to `<project>/.manifold/<slug>/serve.py`.
   It imports the pairing module from `.manifold/<slug>/` and calls
   `launch_server`. Phase 9 shows the exact code to paste in.
3. **A container image** pushed to the registry at the agreed name and
   tag (e.g. `ghcr.io/<org>/policy-<slug>:0.1.0`).
4. **A registered policy version** on the Manifold platform. This is a
   catalog entry created by `manifold policy init` that points at the
   image.

Files 1 and 2 are new files on disk. The other two live on the registry
and on the platform.

Once all four exist, Stage 3's main work is done. Then ask the user
whether to submit a scored test run against a benchmark of their
choice (Phase 10 walks through it). Do not submit a run on your own.

Stage 3 does not fix wrap bugs. If the wrap does not pass both
`check_compatibility` and `verify`, go back to Phase 6 first.

**Write into `.manifold/<slug>/`.** The `Dockerfile` and `serve.py`
that Stage 3 produces both live under
`<project>/.manifold/<slug>/`, next to the wrap files. `docker build`
runs with the project root as the build context so the whole project
is available to `COPY`.

**Ask the user before every step that costs money or writes to a shared
system.** Each of the choices below costs the user time, cloud credits, or
registry storage. Do not pick any of them yourself:

- The target benchmark for the scored test run. Never pick "a reasonable
  benchmark" for the user.
- Pushing the image to the registry.
- Registering the policy on the platform.
- Submitting a run.

At each of those points: state what you are about to do, wait for a
confirmation, and only then proceed. Do not chain "wrap built successfully,
now submitting a run against <a benchmark>" into one action. **If the user
declines any of these, stop the skill there; do not skip to the next
step.**

The **policy name** (the thing that becomes `<slug>` in
`manifold policy init <slug>`) is **not** a choice that this stage
makes. Stage 1 records it in `CONTEXT.md`. Do not rename the policy on the user's
behalf. If the slug is missing, ask the user once for it.

The container image name (registry, namespace, tag) is a
container-registry concept, not a user-facing choice. This stage
constructs it from the policy slug plus the org's registry and
namespace.

## How the platform runs a policy

The platform runs policies as registered container images. When a run is
dispatched, the runner pulls the image, starts it, waits for the server
inside to accept connections on port 8000, and then drives it through the
benchmark.

Three consequences follow:

1. **One image per wrap.** The `CMD` in the Dockerfile picks which wrap the
   container serves. Registration cannot override it.
2. **Tags become versions.** `manifold policy init` reads the image tag and
   uses it as the version string. Never reuse a tag. Re-running
   `manifold policy init` on an existing tag with new settings fails.
3. **Nothing local proves the image works in production conditions.**
   A clean local `docker run` only shows that the server starts.
   Whether the platform can pull, schedule, drive, and score the image
   is only settled by an actual scored run, which is up to the user to
   submit.

---

## Phase 7: Understand the inputs

Do this before writing a Dockerfile. You need the wrap module path, the
policy name, and the manifold-sdk revision pinned down first.

**The wrap.** Confirm the wrap passes both `check_compatibility` and
`verify` from Phase 6. Record the module path that exports
`PROFILE`, `BENCHMARK`, and `PIPELINE`. The Dockerfile's `CMD` will name
it.

**The policy name.** This is the slug used in `manifold policy init
<slug>`. Stage 1 records it in `CONTEXT.md`. If it
is missing, ask the user once for it. Do not invent one.

**The image name.** Built from the policy slug plus the org's registry
and namespace, in the shape:
`<registry>/<namespace>/policy-<slug>:<tag>`. For example
`ghcr.io/<your-org>/policy-mypolicy-mybench:0.1.0`. The tag is a version
string; bump it for every change to the wrap or the Dockerfile. This
skill constructs the image name from those parts; do not ask the user
to type it out.

Examples below use `ghcr.io/<your-org>`. Substitute the actual registry
and namespace throughout.

**The manifold-sdk Git revision.** Read the **manifold-sdk revision**
line from the policy's subsection in `.manifold/CONTEXT.md`. It is the
full Git commit SHA installed when the wrap passed `check_compatibility`
and `verify`. Install `manifold-sdk` from that revision instead of
resolving the repository again. If the line is missing, return to
"Prepare the project for the wrap" in Stage 2, then rerun both checks.

> **Phase 7 checkpoint.** Record before designing:
> ```
> wrap_module      = ?  (e.g. mywrap.mypolicy_mybench)
> checks_pass      = yes (from Phase 6)
> policy_slug      = ?  (source: CONTEXT.md | user)
> image_name       = <registry>/<namespace>/policy-<slug>:<tag>
>                    (constructed by this stage from the slug)
> sdk_revision     = ?  (from the policy's manifold-sdk revision line)
> ```

---

## Phase 8: Design the Dockerfile

Decide what the image will contain before writing any of it. Everything in
this phase is a decision, not implementation.

### What goes in the image

Assume the user has a project folder with the model's code and
dependencies. Whether it's a cloned git repo, a fork, a local working
directory, or a messy research folder on a shared workstation. Treat
it the same way.

The image contains, in this order:

1. **A base image** with Python and CUDA matching what the model was
   trained on. Pin it by digest, not by a floating tag like `latest`,
   so rebuilds are reproducible. If unsure, an official CUDA runtime
   image on Ubuntu for the trained CUDA version is a safe default.
2. **The model project's dependencies**, installed with `uv`. In most
   cases: `uv sync` if the project has a `pyproject.toml` and
   `uv.lock`, or `uv pip install -r requirements.txt` if it only ships
   a requirements file. Fall back to whatever the project actually
   supports (conda env, raw `pip`) only if `uv` cannot handle it.
3. **manifold-sdk**, installed from the full Git commit SHA on the
   policy's **manifold-sdk revision** line:
   `uv pip install "manifold-sdk @ git+https://github.com/bifrostai/manifold-sdk.git@<commit>"`.
4. **The parts of the project folder that the wrap imports from.**
   `driver.py` names what inference needs. Trace its imports (and
   their transitive imports) back to the folders they live in, and
   `COPY` only those. Skip everything else: training scripts, datasets,
   logs (`wandb/`, `tb_logs/`), notebooks, alternate checkpoints,
   `.git`. Use a `.dockerignore` or an explicit list of `COPY` paths.
   If the container crashes at startup on a missing import, add that
   path. Set `ENV PYTHONPATH=/app` (or wherever you copied to) so the
   imports resolve.
5. **The wrap module and the launcher** (`profile.py`, `driver.py`,
   the pairing file, and `serve.py`), copied into the `WORKDIR`.

### Pin the project source

For rebuilds to match, the project folder's state has to be pinned.
If the folder is in git, pin the commit with a build `ARG` and check
it out during build. If it isn't, snapshot the folder (a tarball
saved somewhere durable) and record which snapshot the image was
built from. Bumping that pin is a separate decision from bumping the
manifold-sdk revision.

### Weights: fetched or baked?

Two ways to get the model checkpoint into the running container:

- **Fetched at first load.** The profile's `default_weights` contains a
  hub id or path. Point the model library's checkpoint cache at
  `/opt/manifold/cache` (whatever env var it uses: `HF_HOME` for
  Hugging Face, `TORCH_HOME` for torch hub, etc.). The platform mounts
  a persistent host directory at that path, so the download happens
  once per box and is reused across runs.

  The container has a 10-minute window to become ready (bind port 8000
  and accept a TCP connection). That window covers both the download
  and the load into GPU memory. A large checkpoint may not finish in
  time on a cold box; for those, prefer baking.

  **If the checkpoint needs a credential** (gated Hugging Face repo,
  private hub repo, private S3 bucket), you cannot use "fetched at
  first load" today; the platform does not currently give you a way
  to hand a run-time credential to your container. Bake the checkpoint
  in instead. Do **not** try to work around this by putting the
  credential in the image (`ENV HF_TOKEN=...`, copied
  `~/.huggingface/token`, etc.); anyone who pulls the image gets it.
- **Baked into the image.** Copy the checkpoint files in at build
  time. Two common cases:
  - **You already have the weights on disk** (a checkpoint from your
    training run, a folder you downloaded earlier). Just `COPY` the
    files into the image. No auth, no download, done.
  - **You need to fetch them from a private source at build time.** Do
    it on a machine where you already have the credential, so `pip`,
    `huggingface-cli`, or `aws` can resolve the download using your
    local auth. The credential never enters the image, only the
    resolved files do.

  Only copy what inference needs; hub snapshots often include
  optimizer states, training configs, and alternate checkpoints that
  add size without being used. Baking trades a larger image and
  slower pushes for not having to fetch at run time.

### How the wrap gets served

The container's `CMD` runs a small launcher script that imports the wrap
module and hands its `PROFILE` / `BENCHMARK` / `PIPELINE` to
`launch_server`. Phase 9 shows the launcher and the `CMD` line. Nothing
else in the image needs to know about the wrap.

> **Phase 8 checkpoint:**
> ```
> base_image         = ?  (pinned by digest)
> project_pin        = ?  (git commit ARG | tarball snapshot path)
> weights_approach   = fetched at first load | baked
> ```

---

## Phase 9: Build and push the image

Write the Dockerfile and build it. On a GPU box, run it locally to prove
the server starts; on a CPU-only box, skip the local run. Then push it
to the registry.

### Dockerfile rules

These apply to every image:

**Port 8000.** The container must listen on port 8000.
`DEFAULT_CONTAINER_PORT` is set to 8000 in the runner code and registration
cannot change it. Bind `0.0.0.0:8000` in the `CMD`.

**Bash is required.** The readiness probe runs
`bash -c 'exec 3<>/dev/tcp/127.0.0.1/8000'` inside the container. Install
bash if the base image doesn't have it.

**`PYTHONUNBUFFERED=1`.** Set this in the Dockerfile. Without it, a
crashing container's last output lines stay in a write buffer and are lost.
That output is how you diagnose failures.

**Layer order.** Put the model's large dependencies (torch, jax, CUDA
wheels) in early layers. Put manifold-sdk and your wrap code in later
layers. That way, editing the wrap rebuilds only the cheap layers at
the end.

**Pin the model's dependencies.** manifold-sdk declares dependency
floors with no ceilings. A fresh resolution inside a rebuild can
upgrade a package the model cannot use. Pin the model's `numpy`,
`scipy`, and similar packages in the image to prevent that.

**No network fetches at model construction.** If the loader downloads
anything at construction time (pretrained weights, tokenizer files),
disable it or copy the files into the image at build time. A container in
an offline environment will fail at load otherwise.

**GPU memory pre-allocation.** Some frameworks claim all GPU memory at
startup; JAX does this by default. Disable that behavior in the
Dockerfile with the framework's environment variable. Otherwise, on a
shared box, all GPU memory is consumed before the benchmark renderer
starts.

### Launcher and CMD

Write a small launcher script that imports the wrap module by name and
starts the server:

```python
"""Serve a wrapped policy over TCP."""
import argparse, importlib
from manifold.recipes import launch_server, read_pairing

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--port", type=int, required=True)
    parser.add_argument("--host", default="0.0.0.0")
    parser.add_argument("--pairing", required=True)
    args = parser.parse_args()
    pairing = read_pairing(importlib.import_module(args.pairing))
    launch_server([pairing], port=args.port, host=args.host)

if __name__ == "__main__":
    main()
```

`COPY` the wrap module (the file exporting `PROFILE`, `BENCHMARK`,
`PIPELINE`, plus `profile.py` and `driver.py`) and the launcher into the
`WORKDIR`, then:

```dockerfile
CMD ["python", "serve.py", "--pairing", "my_wrap.mypolicy_mybench", \
     "--host", "0.0.0.0", "--port", "8000"]
```

### Build

Manifold's runners are x86_64 Linux machines, so the image has to be
built for `linux/amd64`. On an Apple Silicon Mac, or any other arm64
machine, `docker build` produces an arm64 image by default. The push
to the registry succeeds. Registration succeeds too. The failure
comes later, when the runner tries to start the container. Name the
platform explicitly:

```sh
docker buildx build --platform linux/amd64 \
    -f <path-to-Dockerfile> \
    -t <registry>/<namespace>/policy-<slug>:<tag> .
```

On an x86_64 Linux machine, plain `docker build` with the same
`-f` and `-t` arguments does the same thing.

Building a CUDA image for amd64 on an arm64 machine runs the build
steps under emulation. That is slow, and some GPU wheels fail to
install under it. Build this image on a Linux x86_64 machine when the
user has one.

A clean `docker build` means the image assembled without errors, nothing
more. The server has not run yet.

### Run it locally (only if the local box has a GPU)

Skip this step on a CPU-only machine. A robotics policy needs a GPU to
load and serve, so a CPU-only local run only tells you whether Python
could import, not whether the container actually works. On CPU-only,
proceed straight to Push; the container will first be exercised for
real when the user submits a run in Phase 10.

On a GPU box, start the container and confirm the server prints its
listening line:

```sh
docker run --gpus all <image>:<tag>
```

Look for:
```
listening on 0.0.0.0:8000 (TCP), up to 8 concurrent shard(s)
```

Dependency conflicts, import errors, and model loading failures surface
here, not during build. If the listening line does not appear (crash,
hang, silent exit), **do not push**. Fix the Dockerfile or the wrap
and rebuild first.

### Push

**Save the Dockerfile and launcher before you push.** If they get lost
or overwritten later, no one can rebuild the image or see what code
went into it.

**Ask the user before pushing.** Show them the exact image name and tag,
and wait for confirmation.

```sh
docker login ghcr.io   # or the registry the user chose
docker push <registry>/<namespace>/policy-<slug>:<tag>
```

**Make sure the platform can pull the image.** Two paths:

- **Public registry** (a public ghcr package, a public Docker Hub repo).
  Pull just works, no credentials involved. A new ghcr package starts
  private. Flip it to public at
  `https://github.com/orgs/<org>/packages/container/<package>/settings`
  before any run.
- **Private registry** (private ghcr package, private Docker Hub repo,
  ECR, Artifact Registry). The platform needs a pull credential
  configured on its side. Confirm with the Bifrost team before
  submitting a run.

Either way, a pull failure at run time surfaces as a scheduling or pull
error, not "permission denied", so if the platform can't reach the
image, the run just looks broken.

> **Phase 9 checkpoint:**
> ```
> build_exit_code        = 0
> listening_line         = yes | no | skipped (CPU-only local box)
> source_saved           = yes (where: git commit hash | folder | ...)
> push_confirmed_by_user = yes | no
> push_exit_code         = 0
> visibility            = public | private (action needed)
> ```

---

## Phase 10: Register, then offer a test run

Register the image. Then ask the user whether to submit a scored test run.
Do not submit one on your own.

If `CONTEXT.md` records `local_test_run_possible = no`, add this when
offering the run: there was no test run on the user's machine, so
this run executes the wrap code for the first time. If the run
crashes, check the wrap code before the image.

### Register

**Ask the user to confirm the slug and version before running the CLI.**
Registration creates a catalog entry for their organization, and the tag
becomes an immutable version string.

```sh
manifold policy init <slug> \
  --image <registry>/<namespace>/policy-<slug>:<tag> \
  --minimum-gpu-memory-gb <N>
```

The image tag becomes the version. So
`--image ghcr.io/<your-org>/policy-mypolicy-mybench:0.1.0` registers version
`0.1.0`.

**`--minimum-gpu-memory-gb` does two things.** It filters which runners
can accept this image (the scheduler compares against reported GPU
memory), and it decides whether the container gets GPU access (`--gpus
all` is only added to `docker run` when `minimum_gpu_memory_gb > 0`). A
GPU model registered with 0 starts without GPU access. It will run very
slowly on CPU or crash at model load. Set `<N>` to the model's actual GPU
memory footprint, rounded up.

**Visibility.** By default a new policy is visible only to its own
organization. Add `--visibility public` only if the user asks for it.

### Offer a scored test run

Registration is done. The image is live on the platform, but you have
not yet seen the platform pull it, schedule it, drive it, and score it.
Only a scored run does that.

**Ask the user whether to submit one.** Something like: "The image is
registered as version `X`. Do you want to submit a scored test run? If
so, which registered benchmark should I pair it against? A run costs
cloud time. If the benchmark has a debug variant, I will run that
first."

Do not pick a benchmark. Do not submit on your own. If the user says no,
stop here; the skill is done.

**Check for a debug variant before submitting.** When the user names
a benchmark, run `manifold benchmark list` and look for a debug
variant of the same family: an entry named `debug-` plus the family
name. A debug run costs much less and still makes the platform pull
the image, start it, pair it, and drive episodes. Submit the first
run against the debug variant if one is listed, otherwise against
the named benchmark.

```sh
manifold run submit <policy-slug> <debug-or-benchmark-slug>
manifold run watch <run-id>
```

If the pairing is rejected on the debug variant, submit against the
named benchmark and report the mismatch to the Bifrost team. After a
clean debug run, ask the user whether to submit the full benchmark.

Before submitting:

- The named benchmark must be registered and must have a working image.
  If the run fails immediately with an error naming the benchmark image
  (missing image, benchmark container crash on start), that is a
  benchmark-side problem, not a wrap bug; flag it to the Bifrost team.
- Runs are visible to everyone in the organization; use a clear
  `--name` if the user wants the run labeled.
- There is no hardware check before scheduling. A benchmark that needs
  a specific GPU will be scheduled onto a box without one and fail at
  runtime.

### Validate the results

A completed run does not mean the wrap is correct. Check the actual
output.

**If the run used a debug variant, skip the score comparison.** A
debug suite is too small for a meaningful score. Check episode
completion, first frames, and action magnitudes below. Compare scores
only on the full run.

**Episode completion.** All episodes should reach `completed`:

```sh
manifold run get <run-id> --episodes
```

**First frames.** Dump the first observation frame from each episode. A
run once scored 0.4 in a scene with no background. A mostly black first
frame means the scene did not load.

**Action magnitudes.** Actions should be in the physical range for the
embodiment (millimeters, radians). Actions stuck at the edges (all 1.0 or
all -1.0) suggest missing denormalization.

**Score vs reference.** Compare `mean_success` against the checkpoint's
published or previously measured score on this suite.

**This is the only point in the whole flow where convention bugs become
visible.** A wrong gripper sign, a wrong component order, a wrong rotation
encoding, or a wrong proprioception mapping all pass `check_compatibility`
and `verify`. They only show up here as a score near zero.

A near-zero score on a suite the checkpoint is known to handle means a
wrap bug until proven otherwise. Go back to Phase 6, fix the pipeline
or session, bump the tag, and continue from Phase 9.

### If the container ran out of GPU memory

An OOM at runtime means the VRAM floor registered with the version was
too low for the model. The wrap code and the image itself are fine.
Signals:

- `manifold run get <run-id>` shows the run as failed with an
  out-of-memory reason.
- The container's stderr (from the platform's logs, or from a local
  test run) says "CUDA out of memory" (torch),
  "RESOURCE_EXHAUSTED" (jax), or the container was `OOMKilled`
  (visible via `docker inspect` on a local run).

**Ask the user before bumping.** A re-register plus a new run is
another spend cycle. State the current floor, the higher floor you
want (round the observed peak up, add ~20% headroom), and the fact
that the image contents do not change. If the user declines, stop.

If the user says yes, the fix is a re-tag, push, and re-register. No
rebuild needed since the image is unchanged:

```sh
# Same image contents, new tag.
docker tag <registry>/<namespace>/policy-<slug>:<old-tag> \
           <registry>/<namespace>/policy-<slug>:<new-tag>
docker push <registry>/<namespace>/policy-<slug>:<new-tag>

# Re-register the new tag with the higher floor.
manifold policy init <slug> \
  --image <registry>/<namespace>/policy-<slug>:<new-tag> \
  --minimum-gpu-memory-gb <new-floor>
```

Then offer another scored test run against the same benchmark. If it
OOMs again, repeat with a higher floor. Each retry needs a fresh
user yes. Do not silently keep bumping.

Also update `CONTEXT.md`'s peak-VRAM entry for this policy to the
value that actually worked. Later runs of this skill then start from
that value.

> **Phase 10 checkpoint:**
> ```
> slug                          = ?
> version                       = ? (from image tag)
> minimum_gpu_memory_gb         = ? (> 0 for GPU models)
> register_confirmed_by_user    = yes | no
> test_run_offered_to_user      = yes  (must be yes, offering is required)
> user_asked_for_test_run       = yes | no  (if no: stop, skill is done)
>
> (fill in below only if the user asked for a test run)
> benchmark_slug                = ? (chosen by user)
> debug_variant                 = <name> | none  (from `manifold benchmark list`)
> debug_run_id                  = ?  (first run, when a debug variant exists)
> debug_episodes_completed      = ? / ?
> submit_confirmed_by_user      = yes | no
> run_id                        = ?
> episodes_completed            = ? / ?
> mean_success                  = ?
> reference_score               = ? (source)
> score_within_noise            = yes | no (if no: investigate)
> first_frames_checked          = yes | no
> action_range_checked          = yes | no
> container_oom                 = yes | no
> (if yes:)
> vram_bump_confirmed_by_user   = yes | no
> new_tag                       = ?
> new_gpu_memory_floor_gb       = ?
> context_md_vram_updated       = yes | no
> ```

---

# Final checklist

## Stage 1

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

## Stage 2

- [ ] The policy's **manifold-sdk revision** line in
      `.manifold/CONTEXT.md` records the full SDK commit SHA used for
      the checks
- [ ] `read_pairing` accepts the module without the model stack
- [ ] Action width: `expected_length()` == model source width
- [ ] `chunk` and `exec_steps` spelled exactly so on the profile; endpoint
      exposes `.profile`
- [ ] Every convention traces to a line in the project's eval (noted in
      docstring)
- [ ] Signature states what the session returns/consumes, in physical units
- [ ] Split clean: conventions as adapters, structural maps as layout, only
      normalization/axis-order/derived tensors in the session
- [ ] Chunked: `OpenLoopChunkQueue` subclass, only `_forward`, both layouts,
      `chunk_size=1`
- [ ] Three-extremes test for each conversion written and NOT written
- [ ] Nothing mutable on endpoint; `endpoint.signature` is the profile's
      object; instruction unwrapped as `str(x[0])`; `ee_pose` sliced against
      embodiment spec
- [ ] Checks pass; `verify` zero failed (or documented exception);
      `not_checked` quoted in handoff
- [ ] **`evaluate` ran** at least 2 episodes without raising; forward
      cadence counted. Or, on a machine that cannot run the model:
      skipped, `local_test_run_possible = no` recorded, and the
      handoff says the driver has not run
- [ ] Handoff states what was and was not proven
- [ ] Asked whether to continue before starting Stage 3

## Stage 3

### Inputs

- [ ] Wrap passes `check_compatibility` and `verify` from Phase 6
- [ ] Policy slug read from `CONTEXT.md` (not invented); image
      name constructed from it
- [ ] manifold-sdk in the image is installed from the policy's
      **manifold-sdk revision** in `.manifold/CONTEXT.md`

### Image contents

- [ ] Base image pinned by digest, not by a floating tag like `latest`
- [ ] Project source pinned (git commit `ARG` or snapshot tarball) so
      rebuilds match
- [ ] Only the folders `driver.py` imports from are `COPY`ed;
      `PYTHONPATH` set so those imports resolve
- [ ] Model's Python dependencies pinned (`numpy`, `scipy`, etc.) so
      a rebuild can't silently upgrade them
- [ ] Weights approach decided: fetched at first load *or* baked in
- [ ] No credentials baked into the image (no `ENV HF_TOKEN=...`, no
      copied `~/.aws/credentials`, no copied `~/.huggingface/token`)

### Dockerfile rules

- [ ] Dockerfile builds without errors
- [ ] Image built for `linux/amd64` (`docker buildx build --platform
      linux/amd64` on any arm64 machine)
- [ ] On a GPU box, container starts and prints its listening line;
      on a CPU-only box, skipped (rely on Phase 10)
- [ ] Port 8000, bound on `0.0.0.0`
- [ ] `PYTHONUNBUFFERED=1` set
- [ ] Bash present in the image
- [ ] No network fetch at model construction (weights are already local
      by the time the model loads)
- [ ] GPU pre-allocation disabled where the framework does it by
      default (e.g. JAX)

### Push and register

- [ ] Tag never reused; Dockerfile and launcher saved before push
- [ ] Push confirmed by the user
- [ ] Registry package visible so the platform can pull the image
- [ ] Registration confirmed by the user; correct
      `--minimum-gpu-memory-gb` (never 0 for GPU models)
- [ ] User was offered a scored test run and given the choice

### Scored test run, if the user asked for one

- [ ] Benchmark chosen by the user; submit confirmed before running
- [ ] `manifold benchmark list` checked for a debug variant of the
      named benchmark's family; when one exists, the first run used it
- [ ] Debug run judged on episode completion, first frames, and
      action ranges, with no score comparison
- [ ] Test run completed; score compared to reference
- [ ] First frames and action ranges inspected
- [ ] Container OOM checked (from `manifold run get` and the container's
      stderr)
- [ ] If OOM: user asked before bumping VRAM; image re-tagged and pushed
      (no rebuild); new tag re-registered with a higher
      `--minimum-gpu-memory-gb`; `CONTEXT.md`'s peak VRAM updated to
      what worked

---

# Reference: import paths

- `manifold.recipes`. `read_pairing`, `launch_server`, `serve`, `evaluate`,
  `run_benchmark`, `run_sharded_benchmark`, `run_episodes`, `write_rollup`,
  `OpenLoopChunkQueue`, `ChunkEndpoint`, `PolicyProfile`, `resolve`,
  `from_lerobot_checkpoint` / `SignatureSuggestion`, `describe`, `Recorder`,
  `dump`, `load`, `NO_RECORDER`
- `manifold.recipes.serving`. `PolicyEndpoint` and `Session` protocols (NOT
  re-exported by `manifold.recipes`)
- `manifold.core.check`. `check_compatibility` returns `Report`
- `manifold.core.verify`. `verify` returns `VerifyReport`
- `manifold.core.pipeline`. `Pipeline`
- `manifold.core.policy`. `PolicySignature`
- `manifold.core.embodiment`. `EEActionSpace`, `JointActionSpace`,
  `UnifiedActionSpace`, `Proprioception`, `EEObservationSpec`,
  `GripperObservationSpec`
- `manifold.core.conventions`. `RotationFormat`, `GripperFormat`, `Frame`
  (pass enum members, never string values)
- `manifold.core.native_layout`. `NativeLayout`, `LayoutEntry`
  (`.from_camera` / `.from_state` / `.from_instruction`), `SourceKind`,
  `Slice`, `Split`, `BatchAxis`, `DtypeCast`, `Component`, `Assemble`
- `manifold.benchmarks`. `ALL`, `LIBERO`, `SIMPLER`, `ROBOCASA`
- `manifold.adapters`. `PackToNativeLayout`, `UnpackFromNativeLayout`,
  `ObservationTap`, `ActionTap` (convention adapters are one level down)
- `manifold.adapters.observation`. `ProprioRotationAdapter`,
  `FrameRebaseAdapter`, `DynamicFrameRebaseAdapter`, `ObservedGripperAdapter`,
  `ResizeCameras`, `Rotate180Cameras`, `FlipVerticalCameras`,
  `SwapChannelOrder`, `StackFrameHistory`
- `manifold.adapters.action`. `RotationFormatAdapter`,
  `GripperPolarityAdapter`, `GripperThresholdAdapter`,
  `UnifiedGripperThresholdAdapter`, `UnifiedSliceAdapter`, `BasePinWiden`,
  `DiscreteBinarize`
- `manifold.lib.rotation.convert`. Driver-side rotation re-encode
- `manifold.lib.gripper`. Action-side gripper remaps
