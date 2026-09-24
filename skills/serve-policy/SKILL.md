---
name: serve-policy
description: >
  Start here to serve a researcher's policy on Manifold. Read the policy
  codebase, ask the user which benchmark to prepare for, write a serving
  script, serve it from this machine with `manifold policy serve`, and run
  it against the debug variant of that benchmark. Use when the user asks to
  "setup", "connect to manifold", "run evals", "serve my policy", or
  equivalent.
compatibility: >
  Run from the user's policy project directory. Needs the manifold CLI,
  logged in, with `manifold policy serve NAME:VERSION -- COMMAND`.
  Needs a manifold-sdk revision that exports `manifold.serve`. This machine
  must be able to run the model, so it needs a GPU.
---

## Summary

The policy runs on the user's machine. Manifold runs the benchmark on its
own machines, and sends each observation to the policy.

The objective is to write one Python file with three critical components:

- A `PolicySignature` which faithfully describes the **native** inputs and
  outputs of the policy (observations, proprioception, actions,
  instructions, etc.).
- A `predict(obs: Observation) -> Action` function that passes data to the
  policy or model.
- A call to `manifold.serve(predict, SIGNATURE)`, inside
  `if __name__ == "__main__":`.

The policy signature **MUST** represent the raw inputs and outputs of the
policy. This is because `manifold.serve` searches for adapters when a run
connects. The adapters convert the benchmark's observations into the form
that the signature declares. They flip or rotate images, resize them, and
convert rotation formats. The signature must therefore describe what the
raw model expects, instead of what the benchmark sends.

`manifold policy serve` starts the script once, in the same directory and
with the same environment. When the script accepts connections, serve
registers the policy and this machine, and prints a line that starts with
`Ready`. Every run that the user submits for that version goes to this
script, so the model loads only once.

When complete, the user should have `manifold-sdk` installed in the
project, alongside a new serving script for their policy-benchmark pair.

## Rules before starting

1. **Speak to the user in their language, not the SDK's.**

The user has not read the SDK docs. They will not recognize class names,
method names, config fields, or enum values. The skill below names those
identifiers freely because you need them to write correct code. When
narrating progress to the user, translate.

Say things like:
- "I'll write a script that lets the benchmark run your policy."
- "The script loaded your model and is waiting for the test run."
- "The test run finished, and your policy scored 40%."
- "Your policy scored zero. I'll check the image orientation first."

2. **Ask with the structured question tool.**

In Claude Code that is `AskUserQuestion`. Ask all questions of Phase 2 in one round.

3. **All decisions on data shape and semantics must be backed.**

Each value in the signature must come from the project's code, with a file
and line, or from the user. A run with a wrong value can complete and
still score near zero.

4. **Plan the entire task in a to-do list before you start, and update it as
you go.**

Use whichever planning tool your harness provides:

- **Claude Code:** `TaskCreate` to seed the plan, `TaskUpdate` to move items
  between `pending` / `in_progress` / `completed`, `TaskList` / `TaskGet` to
  read state.
- **Codex:** use `update_plan` to create and maintain an ordered plan, with
  exactly one item `in_progress` at a time. Keep validation as an explicit item
  until it passes.
- **Other harnesses:** check the harness for a to-do list or planning tool
  before using the fallback below.
- **No planning tool available:** keep the plan as a plain-text checklist in
  your responses and re-post it (with statuses updated) each time you advance.

---

## Phase 1: Understand

**Check whether this machine can run the model, now.**

Run `uname -s && uname -m && nvidia-smi`.

Do this before asking the user anything. The last step loads the user's
model weights on this machine, and runs a debug benchmark that must score
above zero.

An error from `nvidia-smi` does not prove that the machine has no GPU.
Agent harnesses often run commands in a sandbox, and the sandbox can
block the GPU device files. When `nvidia-smi` fails, run it again outside the sandbox, and if that fails, request escalated permissions for the command, and the user approves it. If the harness has no such option, ask the user to run `nvidia-smi` in their own terminal and paste the output.

If the sandbox blocks the GPU, run each later command that loads the
model with escalated permissions too. This applies to
`manifold policy serve`.

The user or an escalated command may confirm that this machine has no
GPU. Then offer the user two options. The user can move this session to
a machine with a GPU. The user can also serve the policy from a GPU
container on a cloud platform, as the "Serve from a cloud container"
section describes. If the project deploys to a cloud platform, offer
that platform first.

**Tools.** Run `manifold policy serve --help`. The help must show
`<identifier>:<version>`. If it does not, stop and tell the user to upgrade the
CLI with `uv tool upgrade manifold-cli`. Run `manifold auth status`. If
the user is not logged in, ask them to run `manifold auth login`.

**The codebase.** Find the package manager from the lockfile or the
dependency file. Find the code that loads the model and runs inference,
usually in files named like `eval*`, `infer*`, `serve*` or `rollout*`.
Find the checkpoint that it loads. If there are multiple checkpoints,
ask the user which one they want to start with.

**The inputs and outputs.** Record each fact below with its file and
line:

- **Images.** The name of each camera, its shape, and its orientation.
  Images can come in any modality, such as EO/RGB, or depth maps.
  Declare orientation relative to the true scene: `UPRIGHT` means that
  the image matches the 3D scene exactly. Code comments about
  orientation can be wrong. Trust the training data or the user.
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
have a debug variant, named `debug-` plus the family name. These debug
variants output data in the same format as their full counterparts, but
with a low task and/or episode count for quick debugging.

---

## Phase 2: Clarify

Ask these questions in one round:

- **Which benchmark should the policy be prepared for?** Offer the
  benchmarks from `manifold benchmark list`, without the debug variants.
  The script targets one benchmark.
- **The inputs and outputs that Phase 1 could not settle.** Ask one
  question for each fact that has no clear source in the code. Image
  orientation, the gripper sign and the rotation format matter most.
  Explain in one sentence that a wrong answer can make the policy score
  zero.
- **What to call the policy on Manifold.** Offer the project folder name
  as the first option.
- **May the skill add `manifold-sdk` to the project?** Say that this
  installs one package.
- **Only when serving from a cloud container: which platform and which
  GPU?** Offer the platform and the GPU from the project's own
  deployment first, if the project has one. The GPU needs enough memory
  for the model.

---

## Phase 3: Execute

**Install the SDK.** Pin the newest commit on the default branch of the
`manifold-sdk` GitHub repository to its full SHA. Add it with the
project's package manager. With uv:

```
uv add "manifold-sdk @ git+https://github.com/bifrostai/manifold-sdk.git@<commit>"
```

**Read the benchmark.** Find the chosen benchmark in `manifold.benchmarks`.
Read its `sensors`, `embodiment.proprioception`, `embodiment.action` and
`instruction`. Compare them with the Phase 1 facts. If the model needs a
camera or a robot state that the benchmark does not publish, stop and
tell the user. The adapters convert differences in format when a run
connects.

**Write the script.**

First, create a `.manifold/` directory in the project root if it doesn't
exist.

Write one file in `.manifold/`, named `serve_<policy>_<benchmark>.py`.
Follow these rules:

- Build the `PolicySignature` from the Phase 1 and Phase 2 facts. Spell
  out every convention field (`rotation=`, `gripper=`, `delta=`). Do not
  copy the benchmark's values. Check that
  `SIGNATURE.action_space.expected_length()` equals the action width.
  Below is an example:
  ```
  POLICY_SIGNATURE = PolicySignature(
      cameras=(
          agentview(
              shape=(224, 224, 3),
              orientation=CameraOrientation.FLIPPED_HORIZONTAL,
          ),
          wrist(
              shape=(224, 224, 3),
              orientation=CameraOrientation.FLIPPED_HORIZONTAL,
          ),
      ),
      proprioception=Proprioception(
          ee_pose=EEObservationSpec(
              rotation=RotationFormat.AXIS_ANGLE,
              gripper=GripperObservationSpec(dim=2),
          ),
      ),
      action_space=EEActionSpace(
          rotation=RotationFormat.AXIS_ANGLE,
          gripper=GripperFormat.SIGNED_OPEN_LOW,
          delta=True,
      ),
      instruction=True,
  )
  ```
- Load the model at the top of the module, with the project's own
  loading code. The runner waits up to 15 minutes for the script to open
  its port.
- In `predict`, read images from `obs.sensors` by the camera names in
  the signature, the robot state from `obs.state`, and the instruction
  from `obs.instruction`. The adapters have converted them to the
  signature's form. Build the model's input as the project's inference
  code builds it. Example:
  ```
  def predict(obs: Observation) -> Action:
      out = POLICY.infer({
          "observation/image":       obs.sensors["agentview"],
          "observation/wrist_image": obs.sensors["wrist"],
          "observation/state":       obs.state["ee_pose"].astype(np.float32),
          "prompt":                  obs.instruction or "",
      })
      chunk = np.asarray(out["actions"])[:EXECUTION_STEPS]
      return Action.from_array(chunk)
  ```
- Return the executed steps of the chunk as one `Action`, in physical
  units. Pass `Action.from_array` an array with one row for each
  executed step.
- Inside `if __name__ == "__main__":`, call
  `manifold.serve(predict, SIGNATURE)` with no other arguments. The runner sets `MANIFOLD_SERVER_URL`, and
  `manifold.serve` reads it.
  ```
  if __name__ == "__main__":
      manifold.serve(predict, POLICY_SIGNATURE)
  ```
- Import the model from the project. Do not copy model code, and do not
  change other project files.

If the user serves from a cloud container, follow the "Serve from a
cloud container" section from here.

**Serve.** Start the serve command so that it keeps running after your
shell exits. Run it from the project root, with the project's run
command in place of `uv run`. Run it outside the sandbox, with escalated
permissions, because the script needs the GPU and the network. If the
harness cannot start a process outside the sandbox, ask the user to run
the commands below in their own terminal.

Do not start the serve command with a plain `&`. A process that the
agent's shell starts with `&` ends when that shell exits, or when the
user's SSH session closes.

Use tmux if `which tmux` finds it. Otherwise use `setsid nohup`.

1. With tmux. If `tmux has-session -t manifold-<policy>` succeeds, a
   serve process for this policy is running. Stop it as the "Stop
   serving" step describes before you start it again.

   ```
   tmux new -d -s manifold-<policy> 'manifold policy serve <policy>:0.0.1 -- uv run .manifold/serve_<policy>_<benchmark>.py > .manifold/serve.log 2>&1'
   ```

2. Without tmux:

   ```
   setsid nohup manifold policy serve <policy>:0.0.1 -- uv run .manifold/serve_<policy>_<benchmark>.py > .manifold/serve.log 2>&1 &
   echo $! > .manifold/serve.pid
   ```

   `setsid` moves the command out of the shell's session. `nohup` makes
   the command ignore SIGHUP. With both, the command keeps running when
   the agent's shell or the user's SSH session closes.

Use `0.0.1` unless the user named a version. The script loads the model
first, which can take minutes. Read the output with
`tail -n 50 .manifold/serve.log`, and wait for the line that starts with
`Ready`.

**Run the debug benchmark.** Submit a run against the debug variant of
the chosen benchmark, then follow it:

```
manifold run submit <policy>:0.0.1 debug-<family> --name <policy>-debug
manifold run watch <run-id>
```

Read `.manifold/serve.log` while the run is in progress. The log shows
the output of the script and any traceback. The run's log in the app
shows the same lines.

The run must get a non-zero score. If the run fails, or if the debug run
completes with all tasks and episodes scoring zero, go to
Troubleshooting.

**Stop serving.** Once the debug run scores above zero, send SIGINT to
the serve process. SIGINT lets the command remove this machine's
registration before it exits.

- If you started it with tmux, run `tmux send-keys -t manifold-<policy> C-c`.
- Otherwise, run `kill -INT $(cat .manifold/serve.pid)`.

Do not send SIGKILL with `kill -9`. SIGKILL stops the command before it
removes the registration. Wait for the process to exit, then read the
last lines of `.manifold/serve.log`. If they say `This machine is still
registered with Manifold`, remove the registration by hand:

1. Run `manifold runner list`. The serve command names the runner after
   the machine's hostname. It adds a short suffix when that name is
   taken.
2. Run `manifold runner show <name>` to get the runner ID.
3. Run `manifold runner revoke <runner-id>`.

---

## Serve from a cloud container

Use this section in place of the "Serve" and "Stop serving" steps when
the user serves the policy from a GPU container on a cloud platform.
Write the serving script as Phase 3 describes. The serving script does
not change.

**Check the platform's CLI.** Check that the platform's CLI is installed
and logged in. If it is not, ask the user to set it up.

**Write the container definition.** Write it in `.manifold/`, in the
format that the platform uses. Read the platform's docs for the format
of the installed version. If the project deploys to this platform, copy
its image definition. The container needs these parts:

- An image that holds the project, its dependencies and `manifold-sdk`.
  The image also needs the manifold CLI. Install the CLI with
  `uv tool install manifold-cli --index https://bifrost-manifold-releases.s3.us-west-2.amazonaws.com/simple/`.
  The CLI needs Python 3.13, and `uv` downloads it when the image lacks
  it.
- The GPU from Phase 2.
- A time limit long enough for a full benchmark run. Many platforms stop
  a container after a short default time limit.
- Storage that keeps the model weights between starts. Point the
  project's weight cache at it. Without this storage, the container
  downloads the weights at each start. The runner waits 15 minutes for
  the script to open its port, and a download can take longer.
- Storage that keeps the manifold login between starts. Set the
  environment variable `MANIFOLD_CONFIG_DIR` to its path. The CLI stores
  its login in that directory. The user then logs in at the first start
  only. Run a single container at a time with this storage, because the
  CLI rewrites the login when it refreshes.
- A start command that prints the container's hostname with `hostname`,
  then runs `manifold auth status`. That command exits with code 0 even
  when the user is logged out. So the start command reads its output. If
  the output does not start with `Logged in as`, the start command runs
  `manifold auth login`. Then it runs the serve command from the "Serve"
  step in the foreground, from the project root in the image.

Pass `MANIFOLD_API_URL` and `MANIFOLD_WEBAPP_URL` into the container if
they are set on this machine. The CLI in the container then uses the
same Manifold deployment.

**Start the container.** Start the platform's command that runs the
container and streams its output. Start it in the same way as the
"Serve" step starts the serve command. Use the tmux session name
`cloud-<policy>`. Write the output to `.manifold/cloud.log`, and write
the PID to `.manifold/cloud.pid`.

At the first start, `manifold auth login` prints a link after `Visit`,
and a code after `confirm code`. Show both to the user, and ask the user
to open the link. Then wait for a line that starts with `Ready` in
`.manifold/cloud.log`.

The platform bills the GPU while the container waits for work. Start
the container right before you submit a run.

**Run the debug benchmark.** Submit the debug run as Phase 3 describes.
Read `.manifold/cloud.log` for the script's output and any traceback.

**Stop the container.** Stop it with the platform's own command, or
send SIGINT to the local command, as the "Stop serving" step describes.
The platform may stop the serve command before the command removes its
registration. So run `manifold runner list` after the container stops.
The container printed its hostname at the start of
`.manifold/cloud.log`. If a runner with that name is listed, remove it
as the "Stop serving" step describes.

Tell the user that the two storage locations keep the weights and the
login for the next start. Deleting the login storage makes the next
container log in again.

---

## Troubleshooting

After each fix, stop serving as the "Stop serving" step describes, and
serve again under a new version, such as `0.0.2`. Manifold lists runs
under their version string. With a new string, the runs of the fixed
script appear apart from the failed runs. Then submit the debug run
again.

### The run does not start or finish

- **`run submit` says that the version is not ready yet.** The serve
  process has stopped, it has not printed `Ready` yet, or it serves a
  different version. Check that it is still running, and that the
  version in `run submit` matches the version in `policy serve`.
- **The run stays queued.** The serve process takes one run at a time.
  The run starts when the previous run ends.
- **The script exits before `Ready`.** The serve output shows the
  traceback. An import error usually means that the serve command used a
  different environment. Start the serve command from the project root,
  with the project's own run command.
- **Serve stops while the model loads.** Serve waits 15 minutes for the
  script to accept connections. If the weights take longer to download,
  download them once before you serve.
- **The run fails as it connects, with a `ValueError`.** The adapters
  cannot convert the benchmark's data into the signature. Compare the
  camera names, the robot state and the action type in the signature
  against the benchmark. If the signature is wrong, fix it to match the
  model. Do not change it to match the benchmark.
- **The script fails inside `predict`.** The traceback usually names a
  shape or a key. Print the shape of each value in `obs.sensors` and
  `obs.state` once, and compare it with the model's input.

### The run completes, but every episode scores zero

A zero score means that the script runs, but one of its values is
wrong. Check the causes below in order. The list starts with the most
common cause.

1. **Image orientation.** Temporarily save an image for each camera that
   `predict` receives, then compare it with a frame from the training
   data or show it to the user. The two must match. A mirrored or
   upside-down image means that the signature declares the wrong
   orientation. Remove the saving code after the check.
2. **Gripper sign.** Watch an episode from the run link. If the gripper
   opens when it should close, the signature declares the wrong gripper
   convention.
3. **Normalization.** Print the actions that `predict` returns for a few
   steps. Values that sit near -1 and 1 for every dimension suggest that
   the model returns normalized actions. Convert them to physical units
   with the project's own statistics.
4. **Rotation format and deltas.** If the arm spins or drifts away from
   the objects, compare the rotation format and the `delta` setting in
   the signature with the project's evaluation code.
5. **Robot state.** Compare the order and the width of `obs.state` with
   the state that the project's evaluation code (if any) passes to the model.
6. **Chunk length.** Return the number of steps that the policy
   executes, not the full chunk that the model predicts.
7. **Instruction.** Print `obs.instruction` once. A model that needs a
   text instruction fails when it receives an empty string.
8. **Checkpoint.** Confirm with the user that the checkpoint was trained
   for this benchmark.

If the score is still zero after all eight checks, stop. Show the user
what you checked and what you found.

---

## Hand back to the user

Once the debug run scores above zero and the serve process has exited,
tell the user four things:

- The path of the serving script, `.manifold/serve_<policy>_<benchmark>.py`.
- The debug score, and the link that `manifold run submit` printed, so
  that they can inspect the run.
- The serve command, so that they can serve the policy again. A run
  starts only while the serve command is running.
- How to stop serving. Give them the tmux command or the `kill -INT`
  command from the "Stop serving" step.

Then ask whether to submit a run against the full benchmark. If the user
says yes, start the serve command again with the same version, submit
the run, and stop serving after the run completes.
