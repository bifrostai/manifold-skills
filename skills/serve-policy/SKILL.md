---
name: serve-policy
description: >
  Start here to put a researcher's policy on Manifold. Read the policy
  codebase, ask the user which benchmark to prepare for, write one serving
  script, serve it from this machine with `manifold policy serve`, and run
  it against the debug variant of that benchmark. Use when the user asks to
  "get my policy on Manifold", "evaluate my policy on Manifold", or "serve
  my policy".
compatibility: >
  Run from the user's policy project directory. Needs the manifold CLI,
  logged in, with `manifold policy serve NAME --version VERSION -- COMMAND`.
  Needs a manifold-sdk revision that exports `manifold.serve`. This machine
  must be able to run the model, which usually means Linux and an NVIDIA GPU.
---

## Summary

The policy runs on the user's machine. Manifold runs the benchmark on its
own machines, and sends each observation to the policy.

The skill writes one Python file. That file holds three things:

- A `PolicySignature`. It describes the images, the robot state and the
  instruction that the model takes, and the action that it returns.
- A `predict(obs: Observation) -> Action` function. It calls the user's
  model.
- A call to `manifold.serve(predict, SIGNATURE)`, inside
  `if __name__ == "__main__":`.

`manifold.serve` searches for adapters when a run connects. The adapters
convert the benchmark's observations into the form that the signature
declares. They flip or rotate images, resize them, and convert rotation
formats. The signature must therefore describe what the model expects,
not what the benchmark sends.

`manifold policy serve` registers the policy and this machine, then waits.
When a run arrives, it starts the script as a child process in the same
directory and with the same environment.

The skill writes nothing else. It writes no context file, no check
script, and no copy of the model code. It changes one project file, the
dependency file, to add `manifold-sdk`.

## Rules

**Speak to the user in their language.** The user has not read the SDK
docs. Say "which cameras your model uses", and do not say "the
`PolicySignature` cameras". Follow the user's lead if they use a term.

**Ask with the structured question tool.** In Claude Code that is
`AskUserQuestion`. Ask all questions of Phase 2 in one round.

**Ask, do not invent.** Each value in the signature comes from the
project's code, with a file and line, or from the user. A wrong value
passes every check and then scores near zero.

**Keep a to-do list.** Use the planning tool of the harness, and update
it as each step starts and ends.

---

## Phase 1: Understand

Do this before asking the user anything.

**Tools.** Run `manifold policy serve --help`. The help must show a
`--version` option. If it does not, stop and tell the user to upgrade the
CLI with `uv tool upgrade manifold-cli`. Run `manifold auth status`. If
the user is not logged in, ask them to run `manifold auth login`. Run
`nvidia-smi`. If this machine has no NVIDIA GPU, tell the user that it
probably cannot run the model, and ask whether to continue.

**The codebase.** Find the package manager from the lockfile or the
dependency file. Find the code that loads the model and runs inference,
usually in files named like `eval*`, `infer*`, `serve*` or `rollout*`.
Find the checkpoint that it loads.

**The inputs and outputs.** Record each fact below with its file and
line:

- **Images.** The name of each camera, its shape, and its orientation.
  Declare orientation relative to the real scene: `UPRIGHT` means that
  the image looks correct to a person standing in the scene. Code
  comments about orientation are often wrong. Trust the training data or
  the user.
- **Robot state.** End effector pose or joint positions, the rotation
  format, and the gripper width.
- **Instruction.** Whether the model takes a text instruction.
- **Action.** End effector or joint control, the rotation format, the
  gripper sign convention, whether actions are deltas, and the action
  width.
- **Chunks.** How many actions one forward pass predicts, and how many
  of them the project executes before it predicts again.
- **Normalization.** Whether the model returns normalized values. The
  script must return actions in physical units.

**The benchmarks.** Run `manifold benchmark list`. Note which benchmarks
have a debug variant, named `debug-` plus the family name.

---

## Phase 2: Clarify

Ask these questions in one round:

- **Which benchmark should the policy be prepared for?** Offer the
  benchmarks from `manifold benchmark list`, without the debug variants.
  The script targets one benchmark.
- **The inputs and outputs that Phase 1 could not settle.** Ask one
  question for each fact that has no clear source in the code. Image
  orientation, the gripper sign and the rotation format matter most.
  Explain in one sentence that a wrong answer makes the policy score near
  zero.
- **What to call the policy on Manifold.** Offer the project folder name
  as the first option.
- **May the skill add `manifold-sdk` to the project?** Say that this
  changes the dependency file and installs one package.

If the user says no to the install, stop.

---

## Phase 3: Execute

**Install the SDK.** Pin the newest commit on the default branch of the
`manifold-sdk` GitHub repository to its full SHA. Add it with the
project's package manager. With uv:

```
uv add "manifold-sdk @ git+https://github.com/bifrostai/manifold-sdk.git@<commit>"
```

Then run `python -c "import manifold; manifold.serve"` in the project's
environment. An `AttributeError` means that this SDK revision is too old.
Stop and tell the user.

**Read the benchmark.** Find the chosen benchmark in `manifold.benchmarks`.
Read its `sensors`, `embodiment.proprioception`, `embodiment.action` and
`instruction`. Compare them with the Phase 1 facts. If the model needs a
camera or a robot state that the benchmark does not publish, stop and
tell the user.

**Write the script.** Write one file at the project root, named
`serve_<benchmark>.py`. Follow these rules:

- Build the `PolicySignature` from the Phase 1 and Phase 2 facts. Spell
  out every convention field (`rotation=`, `gripper=`, `delta=`). Do not
  copy the benchmark's values. Check that
  `SIGNATURE.action_space.expected_length()` equals the action width.
- Load the model at the top of the module, with the project's own
  loading code. The runner waits up to 15 minutes for the script to open
  its port.
- In `predict`, read images from `obs.sensors` by the camera names in
  the signature, the robot state from `obs.state`, and the instruction
  from `obs.instruction`. The adapters have converted them to the
  signature's form. Build the model's input as the project's inference
  code builds it.
- Return the executed steps of the chunk as one `Action`, in physical
  units. Pass `Action.from_array` an array with one row for each
  executed step.
- Call `manifold.serve(predict, SIGNATURE)` with no other arguments. The
  runner sets `MANIFOLD_SERVER_URL`, and `manifold.serve` reads it.
- Import the model from the project. Do not copy model code, and do not
  change other project files.

**Serve.** Run this from the project root, in the background, with the
project's run command in place of `uv run`:

```
manifold policy serve <policy> --version v1 -- uv run serve_<benchmark>.py
```

Use `v1` unless the user named a version. Wait for the line
`Waiting for work...`.

**Run the debug benchmark.** Submit a run against the debug variant of
the chosen benchmark, then follow it:

```
manifold run submit <policy> debug-<family> --name <policy>-debug
manifold run watch <run-id>
```

Read the output of `manifold policy serve` while the run is in progress.
It shows the script's start and any traceback.

- **The run stays queued.** Check that the serve process is still
  running, with the same policy name and version.
- **The run fails as it connects, with a `ValueError`.** The adapters
  cannot convert the benchmark into the signature. Check the camera
  names, the gripper width and the action type against the benchmark.
- **The script fails.** Fix the script, stop the serve process with
  Ctrl-C, and serve again under a new version, such as `v2`.

---

## Hand back to the user

Tell the user these facts:

- The path of the script, and the serve command.
- The result of the debug run, with the link that `manifold run submit`
  printed, and `manifold run get <run-id> --episodes` for the episodes.
- A debug run proves that the policy runs. It does not prove a good
  score. A score near zero on the full benchmark usually means a wrong
  image orientation, gripper sign or rotation format.
- The serve process must keep running for any run to use the policy.
  Ctrl-C stops serving, and Manifold then stops sending runs to this
  machine.
- Manifold groups runs by the version string, and nothing checks the
  code or weights behind it. After any change to the script or the
  weights, serve under a new version.

Then ask whether to submit a run against the full benchmark.
