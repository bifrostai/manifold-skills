---
name: wrap-policy
description: >
  Wrap a researcher's policy for the Manifold beta platform (PROFILE /
  BENCHMARK / PIPELINE), then prove it with check_compatibility, verify, and a
  live run of the driver. Use when asked to "wrap my policy for Manifold",
  "get this checkpoint running on beta", "pair <model> with LIBERO / SIMPLER /
  RoboCasa", "write a wrap", "why is check_compatibility INCOMPATIBLE",
  "why did the server reject my pairing", or when deciding whether a checkpoint
  can pair with a shipped benchmark.
compatibility: >
  Install the model stack FIRST, the SDK second: `uv pip install -e <sdk-repo>`.
  Re-pin what the model stack needs — the SDK's numpy/scipy floors conflict with
  older JAX/torch (typically `numpy<2`, `scipy<1.13`). Put the SDK repo root on
  PYTHONPATH: `export PYTHONPATH="$SDK_REPO:.:<project>"`. Declaring and gating
  need no GPU; loading and running need the model environment.
---

## Summary

Wrap one policy so a Manifold benchmark can drive it. The output is a Python
module that exports `PROFILE`, `BENCHMARK`, `PIPELINE`. You load it with
`manifold.recipes.read_pairing`. Done when both `check_compatibility` and
`verify` pass AND the driver has run live.

Three rules:

1. **The policy side converts; the benchmark never adapts.** Every mismatch is
   bridged on the policy side.
2. **The gates never execute your driver.** `check_compatibility` and `verify`
   fold specs and synthetic data through the PIPELINE only. The driver first
   runs when something connects (Phase 6).
3. **A green gate is not a correct wrap.** Both gates compare declarations. A
   wrong gripper polarity, action width, or proprioception slice gates green
   and scores zero.

## Pairing module structure

Export three names: `PROFILE`, `BENCHMARK`, `PIPELINE`.

| File | Holds | Imports model stack? |
|---|---|---|
| `driver.py` | endpoint + session | yes |
| `profile.py` | frozen dataclass: signature, weights, chunk, exec_steps, device, layouts, `load()` | no |
| `<pairing>.py` | `PROFILE` / `BENCHMARK` / `PIPELINE` | no |

Keep the model stack out of profile/pairing so Phases 2–5 run with no GPU.

The SDK repo root must be on PYTHONPATH for any imports beyond the installed
`manifold` package.

---

## Phase 1 — understand the policy

Do this before writing any Manifold code.

**Find the authoritative eval.** Look in `eval/`, `scripts/`, `run_*`,
`*server*`, `rollout*`, `eval*`. Check both the policy repo AND the benchmark
repo (`<bench-repo>/policies/<model>/`) — they can disagree on resize, image
size, gripper threshold, and normalization stats.

**Pick the real policy class** — the one the project runs to get actions.

**Read the eval's env adapter** for the conversions you'll need in Phase 4, but
don't reproduce it — the benchmark delivers typed form.

**Identify the chunk numbers:**
- *chunk* = rows predicted per forward
- *exec_steps* = rows executed before re-forwarding

Call the chunk-returning method (`predict_action_chunk`), never the
queue-owning one (`select_action`) — the queue would double-dispense.

If the eval ensembles instead of executing open-loop, take the open-loop mode
and record the deviation.

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

**Note observation preprocessing** — resizes, flips, channel swaps, frame
stacking, proprio re-encoding, normalization.

> **Phase 1 checkpoint** — record before writing code:
> ```
> action_width      = ?  (source: file:line)
> action_space      = EE|Joint|Unified
> rotation          = ?  (source: file:line)
> gripper           = ?  (source: file:line)
> delta             = ?  (source: file:line)
> frame             = ?  (source: file:line)
> chunk             = ?  (source: file:line)
> exec_steps        = ?  (source: file:line)
> normalization     = yes|no, location: ?
> checkpoint        = ?
> cameras           = [{name, shape}]
> instruction       = true|false
> proprioception    = ee_pose|joint_pos|none
> ```
> A claim without a source is a guess. It will gate green and score zero.

---

## Phase 2 — read the target benchmark spec

**Print the catalog:**

```python
from manifold.benchmarks import ALL, LIBERO, SIMPLER, ROBOCASA
```

Three suites, three embodiments, all end-effector delta with instruction.
`FRANKA_JOINT` exists as an embodiment no benchmark uses — a joint-space policy
cannot pair.

Read the `Benchmark`: `embodiment.action`, `embodiment.proprioception`,
`sensors` (name, resolution, mount), `instruction`.

**Use the catalog `Benchmark`, not the project's vendored copy.** The catalog
is what the runner advertises at HELLO.

**Cross-read the suite's runner.** If the runner passes actions through
unconverted while the upstream eval converts them, report a possible
benchmark-side bug — do not absorb it into your pipeline.

If the target is not in the catalog, stop — the benchmark must be authored
first.

> **Phase 2 checkpoint:**
> ```
> target_benchmark    = LIBERO|SIMPLER|ROBOCASA
> embodiment_action   = {type, rotation, gripper, delta, frame}
> cameras_published   = [{name, shape}]
> instruction         = true|false
> feasible            = true|false (if false: why, and stop)
> ```

---

## Phase 3 — declare the signature and profile

**`PolicySignature`** declares the action space the model emits and the
observation it consumes.

**Stop if the action is not `EEActionSpace`, `JointActionSpace`, or
`UnifiedActionSpace(payload=...)`**, or if its state is not `ee_pose` or
`joint_pos`. Anything else is benchmark work, not a wrap.

### Signature rules

- **Declare what the Session RETURNS and CONSUMES, in physical units.** Do
  `[-1,1]→physical` scaling in the session. Normalization is unmodeled.
- **A width or convention lie gates green.** A 2-wide model declared 7-wide
  passes both gates. It fails live as `PairingRejected("expected an action
  frame")`.
- **If a pipeline adapter reshapes a camera**, declare the reshaped form. A
  frame-history stack is rank-4: `(n_frames, H, W, C)`.
- **No proprioception needed?** Declare `Proprioception()`. Do not invent a
  channel — that feeds the model a tensor it never saw in training.
- **An unneeded instruction** is never an incompatibility. Declare
  `instruction=False`.
- **`EEActionSpace.chunk_size` is NOT the profile's `chunk`.** `chunk_size`
  means one emitted action carries N stacked steps; benchmarks declare 1. A
  chunked VLA stays `chunk_size=1`.
- **Chunking requires both `NativeLayout`s.** Without both `pack` and `unpack`,
  the serve loop calls `session.infer`, and `OpenLoopChunkQueue.infer` raises
  `NotImplementedError`.

Spell out every convention field (`rotation=`, `gripper=`, `delta=`, `frame=`)
from the project's eval. Do not copy the benchmark's action object.

### NativeLayout

`input_layout` maps `Observation` channels to the model's keys. `output_layout`
maps raw output back to an action. Key renames are entries with no ops.

- **Plain `(chunk, dim)` output**: use
  `LayoutEntry(key=..., source=SourceKind.STATE, source_name=None,
  ops=(Slice(start=0, stop=dim),))`. `Slice` takes keyword args only —
  `Slice(0, dim)` raises `TypeError`.
- **`state["ee_pose"]` layout**: `[pos3, rotation, gripper_qpos]`. Widths from
  `ee_step_layout` / `expected_length()`. No accessor exists — driver code
  slices by hand and wrong slices pass every gate.
- **Instruction entry**: use `SourceKind.INSTRUCTION`. Packing runs every step,
  so the model sees the current task by construction. Never stash the
  instruction on the endpoint.

### Profile

Frozen dataclass satisfying `recipes.PolicyProfile`: `signature`,
`default_weights`, `input_layout`, `output_layout`, `chunk`, `exec_steps`,
`device`, `load(weights, device) -> PolicyEndpoint`.

Defer the driver import into `load()`. Carry `device` explicitly — checkpoints
bake in the training device.

- **`chunk`/`exec_steps` have no gate.** Missing `exec_steps` by exact name
  fails at the first step in `advance`. Missing `.profile` on the endpoint
  kills the connection pre-READY: `PairingRejected("policy rejected the
  pairing (no READY)")`.

> **Phase 3 checkpoint:**
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

## Phase 4 — write the driver and assemble the pipeline

### The split rule

| Where | What belongs | Visible to gates? |
|---|---|---|
| **Pipeline** (typed adapters) | rotation format, gripper polarity, camera resize/flip/channel-order, frame rebase, frame history | yes |
| **NativeLayout** (packing) | key renames, dtype casts, batch/time axes, slices, splits | verify only |
| **Driver session** | normalization, axis order transpose, derived tensors, gripper qpos→openness | no |

Name each session-side operation in the pairing docstring.

### Driver

- Expose **the profile's `SIGNATURE` object** as `endpoint.signature`, not a
  copy. The live gate checks `endpoint.signature`.
- Take a `threading.Lock` around the model in `forward`.
- **Keep nothing mutable on the endpoint.** Per-episode state goes in the
  session or a stateful adapter.
- **Chunked policy**: subclass `OpenLoopChunkQueue`, implement only `_forward`,
  return the full chunk. Do not build your own chunk buffer.
- **Instruction**: `native[key]` arrives as `("text",)`. Unwrap with
  `str(x[0])`.
- **Axis transpose**: no permute op exists; `SwapChannelOrder` is RGB↔BGR only.
  Torch-vision models need `(C, H, W)` — transpose in the session.
- Override `reset()` (calling `super().reset()`) only if the model holds
  per-episode state.

### Pipeline

```python
from manifold.core.pipeline import Pipeline
from manifold.adapters import PackToNativeLayout, UnpackFromNativeLayout

PIPELINE = Pipeline(
    observation=[...],   # benchmark form -> signature's consumed form
    action=[...],        # signature's emitted form -> benchmark's
    pack=PackToNativeLayout(PROFILE.input_layout),
    unpack=UnpackFromNativeLayout(PROFILE.output_layout, SIGNATURE.action_space),
)
```

`recipes.resolve(policy, benchmark, adapters)` finds a chain that bridges the
pairing — use as a head start. It cannot discover stateful adapters (filters
them out).

### Pipeline rules

- **Convert encodings, never spaces.** Rotation re-encode, gripper polarity,
  resize: encodings. Joint↔EE, position↔velocity: different controllers — wrong
  checkpoint, stop.
- **No loose convention math in the driver.** Use catalog adapters or write a
  custom adapter (4 class attributes + `applies`/`produce`/`adapt`).
- **Replicate the project's action postprocessing end to end.** Read every line
  — helpers routinely scale, then negate, then clip.
- **Prove each conversion is needed.** Three-extremes test (low, middle, high).
  If values already match, add no adapter.
- **Wire every input the signature consumes.** Zeros for a trained state input
  scores zero silently.

### Trap: `ProprioRotationAdapter(wrap=True)` default is wrong for robosuite

`wrap=True` (the default) routes quaternion→axis-angle through scipy's `[0, π]`
canonicalization. Robosuite/LIBERO/robomimic evals need `wrap=False`. Difference:
5.03 rad per component whenever `w < 0`, which happens routinely. Both settings
gate green.

### Trap: `StackFrameHistory` stacks cameras only

If the checkpoint also windows proprioception, the packed state arrives
unstacked — a live `IndexError` after a green gate. Buffer state history in the
session or write a custom adapter.

### Trap: cross-side stateful adapters fail `verify`

A correct absolute→delta adapter pair sharing a `state_key` will be rejected.
`verify` creates a fresh empty `PipelineState()` for the action probe and never
folds the observation chain first, so the shared slot is always empty. The
server refuses the pairing because `_gate` requires `verified.ok`.

The only implementation that passes treats a missing reference as the identity
pose — the gate selects for the silently-degrading variant. If you take that
path, make it an explicit opt-in flag and name it in the docstring.

Alternative: do the conversion in the session (invisible to the gate). Either
way, raise as an SDK issue.

### Trap: no absolute↔delta adapter ships

The action catalog has seven adapters; none converts absolute↔delta. Write a
custom stateful adapter pair: observation caches `ee_pose` under a `state_key`,
action subtracts it. For chunked policies (`exec_steps > 1`), rows after the
first have no observed pose to subtract — flag it, do not attempt it.

> **Phase 4 checkpoint:**
> ```
> Pipeline observation adapters: [list, in order]
> Pipeline action adapters:      [list, in order]
> Session-side operations:       [list]
>
> Trap decisions:
>   ProprioRotationAdapter wrap  = true|false|n/a  (reason)
>   StackFrameHistory proprio    = handled|n/a
>   Cross-side stateful adapter  = identity-fallback|session-side|n/a
>   Axis transpose               = present|n/a
>
> Conversions NOT added: [list, with three-extremes justification]
> ```

---

## Phase 5 — gate

```python
from manifold.core.check import check_compatibility
from manifold.core.verify import verify

report = check_compatibility(SIGNATURE, BENCHMARK, PIPELINE)
verified = verify(SIGNATURE, BENCHMARK, PIPELINE)
```

Fix the pipeline or declarations until both pass. **Do not redeclare the
signature to make the check pass** — a redeclared signature that does not match
the model ships and cannot run.

### Reading an INCOMPATIBLE reason

Dump both specs with `model_dump(mode="json")` and diff key by key.

| Mismatch | Fix |
|---|---|
| action-space class (Joint vs EE vs Unified) | STOP — different controller |
| `rotation` (action) | `RotationFormatAdapter` |
| `rotation` (proprio) | `ProprioRotationAdapter` — choose `wrap=` deliberately |
| `gripper` polarity (action) | `GripperPolarityAdapter`; `GripperThresholdAdapter` for continuous |
| gripper on Unified action | `UnifiedGripperThresholdAdapter` or `DiscreteBinarize` |
| gripper encoding (observed) | `ObservedGripperAdapter` |
| `frame` (proprio) | `FrameRebaseAdapter` / `DynamicFrameRebaseAdapter` |
| camera shape | `ResizeCameras` |
| camera orientation/channel | `Rotate180Cameras` / `FlipVerticalCameras` / `SwapChannelOrder` |
| camera name mismatch | no rename adapter — custom adapter needed |
| camera rank (clip vs frame) | `StackFrameHistory` + clip shape in signature |
| action width (EE → padded) | `BasePinWiden` (EE→Unified); `UnifiedSliceAdapter` (Unified→EE) |
| `chunk_size` | set to 1 — the chunk lives in raw output |
| `delta` (single-step) | custom stateful adapter pair (see Phase 4 trap) |
| `delta` (chunked) | open problem — flag it |
| `frame` (action) | no adapter — needs kinematics, STOP |
| joint ↔ EE | STOP — different controller |

### Verify

`verify` pushes asymmetric probe data through the pipeline. Read `VerifyReport`
as a coverage manifest: `not_checked` entries are things only Phase 6 can
touch. Quote `not_checked` in the handoff — do not claim "verified."

### Confirm module is well-formed

```sh
python -c "
import importlib
from manifold.recipes import read_pairing
print(read_pairing(importlib.import_module('<your wrap module>')))
"
```

> **Phase 5 checkpoint** — record verbatim:
> ```
> check_compatibility  = COMPATIBLE|COMPATIBLE_VIA_PIPELINE|INCOMPATIBLE
>   lossless           = true|false
>   reasons            = [if any]
>
> verify               = N checked, M failed, K not checked
>   failed             = [list]
>   not_checked        = [list — quote in handoff]
>
> read_pairing         = success|failure
> ```

---

## Phase 6 — run the driver (MANDATORY)

The gates exercise the Pipeline only. The driver first runs when something
connects.

### Stand-in runner

Publish the real catalog `Benchmark`. Copy the env-side `NativeLayout` from
the suite's own runner source. Stub the physics — integrate deltas into
a pose, render a frame that changes. This runner **scores nothing**: it proves
the driver, not the policy.

### Rung 1 — `evaluate`, in process (mandatory first)

```python
from manifold.recipes import evaluate
result = evaluate(
    endpoint, BENCHMARK, reset, step,
    pipeline=PIPELINE, episodes=2, max_steps=12,
)
```

Same gate, same per-step fold, no server. Tracebacks come straight back.

### Rung 2 — served + sharded (mandatory)

Launcher (the SDK does not ship a CLI that launches out-of-tree pairings):

```python
"""Serve an out-of-tree pairing via recipes.launch_server."""
import argparse, importlib
from manifold.recipes import launch_server, read_pairing

def main():
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--port", type=int, required=True)
    parser.add_argument("--host", default="127.0.0.1")
    parser.add_argument("--pairing", required=True, help="dotted module path")
    args = parser.parse_args()
    pairing = read_pairing(importlib.import_module(args.pairing))
    launch_server([pairing], port=args.port, host=args.host)

if __name__ == "__main__":
    main()
```

```sh
export PYTHONPATH="$SDK_REPO:.:<project>"
.venv/bin/python -m wrap.serve --port 8901 --pairing wrap.<your_pairing> &
.venv/bin/python -m <your_runner> --server 127.0.0.1:8901 --episodes 3 \
    --max-steps 12 --output-dir ./smoke-out
```

### Rung 3 — SMOKE runner (only when target is SMOKE)

SMOKE publishes one 64×64 `agentview` on Franka EE. Against LIBERO (256×256,
two cameras), SIMPLER (480×640, WidowX), or RoboCasa (three cameras, unified),
it is refused before READY.

Multi-pairing on one server: `assert_shared_profile` requires every pairing to
share the **same profile object by identity** (`is`, not `==`).

### Rung 4 — real target suite (when available)

Non-zero success on a suite the checkpoint handles is the strongest evidence.
`manifold policy bridge` and `manifold benchmark bridge` are byte-transparent
TCP relays for cross-machine runs.

### Instrumentation

- **Forward cadence**: print per `_forward`. N steps = exactly
  `ceil(N / exec_steps)` forwards.
- **`ObservationTap` / `ActionTap`** (from `manifold.adapters`): identity
  adapters that log a seam.
- **Sanity-read actions**: does the gripper close near the object? Missed
  denormalization shows as ±1.0 magnitudes.
- **Prove the reset**: episode 1's first steps should reproduce episode 0's.

### Diagnostics

The runner reports `PairingRejected("expected an action frame")` or `"policy
rejected the pairing (no READY)"` regardless of cause. The real exception is
one line in the **server log**. Reproduce in process (Rung 1) for a traceback.

**Sharding**: give each shard its own `--output-dir` — `write_rollup` writes
`<output-dir>/results/<benchmark>.json` with no shard component; shards sharing
a directory overwrite each other.

> **Phase 6 checkpoint:**
> ```
> Rung 1 (evaluate):
>   episodes      = ?
>   forward_count = ? (expected: ceil(steps / exec_steps) * episodes)
>   ep 2 reproduced ep 1 = yes|no
>
> Rung 2 (served):
>   episodes      = ?
>   shards        = ?
>   forward_count matches = yes|no
>   PairingRejected       = no|yes (message)
>
> Rung 3/4 = ran|skipped (reason)
> ```

---

## Final checklist

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
- [ ] Gates green; `verify` zero failed (or documented exception);
      `not_checked` quoted in handoff
- [ ] **Rung 1 and Rung 2 both ran**; forward cadence counted; second episode
      clean
- [ ] Handoff states what was and was not proven

---

## Reference: import paths

- `manifold.recipes` — `read_pairing`, `launch_server`, `serve`, `evaluate`,
  `run_benchmark`, `run_sharded_benchmark`, `run_episodes`, `write_rollup`,
  `OpenLoopChunkQueue`, `ChunkEndpoint`, `PolicyProfile`, `resolve`,
  `from_lerobot_checkpoint` / `SignatureSuggestion`, `describe`, `Recorder`,
  `dump`, `load`, `NO_RECORDER`
- `manifold.recipes.serving` — `PolicyEndpoint` and `Session` protocols (NOT
  re-exported by `manifold.recipes`)
- `manifold.core.check` — `check_compatibility` → `Report`
- `manifold.core.verify` — `verify` → `VerifyReport`
- `manifold.core.pipeline` — `Pipeline`
- `manifold.core.policy` — `PolicySignature`
- `manifold.core.embodiment` — `EEActionSpace`, `JointActionSpace`,
  `UnifiedActionSpace`, `Proprioception`, `EEObservationSpec`,
  `GripperObservationSpec`
- `manifold.core.conventions` — `RotationFormat`, `GripperFormat`, `Frame`
  (pass enum members, never string values)
- `manifold.core.native_layout` — `NativeLayout`, `LayoutEntry`
  (`.from_camera` / `.from_state` / `.from_instruction`), `SourceKind`,
  `Slice`, `Split`, `BatchAxis`, `DtypeCast`, `Component`, `Assemble`
- `manifold.benchmarks` — `ALL`, `LIBERO`, `SIMPLER`, `ROBOCASA`
- `manifold.adapters` — `PackToNativeLayout`, `UnpackFromNativeLayout`,
  `ObservationTap`, `ActionTap` (convention adapters are one level down)
- `manifold.adapters.observation` — `ProprioRotationAdapter`,
  `FrameRebaseAdapter`, `DynamicFrameRebaseAdapter`, `ObservedGripperAdapter`,
  `ResizeCameras`, `Rotate180Cameras`, `FlipVerticalCameras`,
  `SwapChannelOrder`, `StackFrameHistory`
- `manifold.adapters.action` — `RotationFormatAdapter`,
  `GripperPolarityAdapter`, `GripperThresholdAdapter`,
  `UnifiedGripperThresholdAdapter`, `UnifiedSliceAdapter`, `BasePinWiden`,
  `DiscreteBinarize`
- `manifold.lib.rotation.convert` — driver-side rotation re-encode
- `manifold.lib.gripper` — action-side gripper remaps
