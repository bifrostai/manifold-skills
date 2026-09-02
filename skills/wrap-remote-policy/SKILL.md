---
name: wrap-remote-policy
description: >
  Wrap a policy that runs on the user's own inference server (Modal
  endpoint, private HTTPS box) for the Manifold
  platform. Write a driver that dials that server, plus the profile and
  pairing that pass check_compatibility and verify. Use when the model
  does not load in the built container.
compatibility: >
  Run from the user's policy project directory, after `/setup-manifold`
  has written `<project>/.manifold/CONTEXT.md` with `model_runtime =
  hosted_endpoint`. Everything this skill writes goes inside
  `<project>/.manifold/<slug>/`, where `<slug>` is the policy name
  recorded in `CONTEXT.md`.
---

## Summary

Use the manifold-sdk to write the files that let a Manifold benchmark
call the user's own inference server. The output is a folder of
Python files. Three kinds of file, each with its own role. The skill
uses these three words as labels for the files throughout:

- The **driver** makes the HTTP calls to the user's server.
- The **profile** describes what the server expects as input and
  returns as output.
- The **pairing** ties one driver-plus-profile to one benchmark. There
  is one pairing file per benchmark the user wants to run against.

Together these files are called the **wrap**.

All conversions between what the benchmark publishes and what the
server accepts happen in the wrap. The benchmark side is fixed. The
server side is fixed too. This skill does not change what routes the
server exposes or the shape of its request and response bodies. It
only changes how the driver talks to them.

Your task is complete when `check_compatibility` and `verify` both
pass AND the driver reaches the endpoint and answers one full episode
under `evaluate`. The two checks pass specs and dummy data through
the PIPELINE only; they do not touch the driver. Only a real request
against the real endpoint proves the wire format, the readiness poll,
and the action conversion.

If the endpoint is not reachable from the wrap author's machine, the
final live proof shifts to a scored run under
`/containerize-remote-wrap` instead. Call that out in the handoff.

## Rules

**Confirm before doing anything else.** First thing after this skill
loads, tell the user in your own words what it will do and why:
adding `manifold-sdk` and an HTTP client (`httpx`) to the project's
deps, creating files under `<project>/.manifold/`, and writing code
that will dial their inference server at run time. Wait for a yes
before reading `CONTEXT.md` or touching anything.

**Speak to the user in their language, not the SDK's.** The user has
not read the SDK docs. They will not recognize class names, method
names, config fields, enum values, or HTTP route paths. The skill
below names those identifiers freely because you need them to write
correct code. When narrating progress to the user, translate.

Say things like:
- "I'll write the files that let the benchmark call your server."
- "The wrap passes the SDK's compatibility check."
- "The driver reached your server and finished the test run."
- "Some parts of the wrap were not tested by the check."

Not the identifiers from the code blocks below. If the user uses
one of those terms themselves, follow their lead. Otherwise,
describe what happened and why it matters.

**Run in the user's policy directory.** Once the user has said yes,
confirm the current working directory is their policy project: the
same one `/setup-manifold` ran in. Look for
`<project>/.manifold/CONTEXT.md` at the root; if it does not exist,
stop and ask the user to run `/setup-manifold` first. If
`CONTEXT.md` has `model_runtime = in_container`, stop and tell the
user this skill is for hosted-endpoint policies only; `/wrap-policy`
is the one for in-container models.

**Read `.manifold/CONTEXT.md` next.** setup-manifold already
interviewed the user and recorded the policy slug, the registry, the
benchmarks of interest, the endpoint URL, the auth situation, and
the wire contract summary. Read that file before asking the user
anything else; ask only about details `CONTEXT.md` does not already
cover.

**Jobs setup-manifold delegated to this skill.** setup-manifold wrote
`CONTEXT.md` and nothing else. Once you've read it, do these before
writing any wrap code:

- Add `manifold-sdk` and `httpx` to the project's dependency file
  **and** install them into the project's environment. With uv, one
  command per package:

  ```
  uv add "manifold-sdk @ git+https://github.com/bifrostai/manifold-sdk.git"
  uv add httpx
  ```

  With plain `requirements.txt`, append both lines to the file, then
  run `pip install -r requirements.txt`. Recording without installing
  (or the reverse) leaves the project half-set-up. If the install
  fails on a dependency conflict, stop and hand the error to the
  user.
- Create the folder `<project>/.manifold/<slug>/`. The slug is in
  `CONTEXT.md`. All wrap files below live inside it.

**Plan the entire task in a to-do list before you start, and update
it as you go.** Use whichever planning tool your harness provides:

- **Claude Code:** `TaskCreate` to seed the plan, `TaskUpdate` to
  move items between `pending` / `in_progress` / `completed`,
  `TaskList` / `TaskGet` to read state.
- **Codex:** use `update_plan` to create and maintain an ordered
  plan, with exactly one item `in_progress` at a time. Keep the live
  run as an explicit item until it passes.
- **Other harnesses:** check the harness for a to-do list or planning
  tool before using the fallback below.
- **No planning tool available:** keep the plan as a plain-text
  checklist in your responses and re-post it (with statuses updated)
  each time you advance.

The intent is (1) to **structure the work** so nothing gets skipped,
and (2) to **stay accountable and informative** by updating the list
as steps start and finish, so the user can follow along without
asking.

## Folder structure

All wrap files live under `<project>/.manifold/<slug>/`, where
`<slug>` is the policy name recorded in `CONTEXT.md`:

| File | Description | Dials the endpoint? |
|---|---|---|
| `.manifold/<slug>/driver.py` | HTTP client + endpoint + session | yes |
| `.manifold/<slug>/profile.py` | frozen dataclass: signature, layouts, wire conventions, env overrides, `load()` | no |
| `.manifold/<slug>/<benchmark>.py` | pairing file, exports `PROFILE` / `BENCHMARK` / `PIPELINE` (one file per benchmark) | no |

- `driver.py` holds the HTTP client for the server, and the endpoint
  and session classes that use it.
- `profile.py` is a lightweight spec describing what the driver sends
  to the server and what shape it expects back (wire encoding,
  timeouts, action scaling, gripper convention).
- `<benchmark>.py` is the pairing file. It builds the signature and
  layouts, instantiates the profile, and declares `PROFILE`,
  `BENCHMARK`, and `PIPELINE`.

`PROFILE` points at the wrap's spec (from profile.py).
`BENCHMARK` points at the test to run.
`PIPELINE` is the list of adapters that translate between the
benchmark's data and what the driver forwards to the server.

---

## Phase 1: Understand the endpoint and target benchmark

Do this before writing any Manifold code. You need a full picture of
both sides to design anything.

### Endpoint side

**Read the server's route contract.** Two routes are common:

- A **description route** (for example `GET /config`) that returns
  what the loaded checkpoint expects. Camera names, state-window
  depth, chunk length, the metadata key the server reads the task
  sentence from. A cold container may answer with empty fields
  while its checkpoint is still loading, so treat empty fields as
  "not ready yet."
- An **inference route** (for example `POST /infer`) that takes one
  observation and returns one chunk of actions.

Get the exact URL path, HTTP method, request body shape, and response
body shape for each route. The user or their handoff document has
this. If it does not, stop and ask the user for it; do not guess.

**Note the wire encoding.** JSON, msgpack, msgpack-numpy, protobuf,
custom. If msgpack-numpy: check whether it is the PyPI package's
encoding or a project-specific variant. Two variants that look
similar often use different map keys and dtype tokens and do not
interoperate silently.

**Note the cold-start behavior.** How long from container start until
the server describes a loaded checkpoint. Whether the first request
carrying a new task sentence takes longer than later ones (a language
encoder often takes 30 to 60 seconds on the first task).

**Note whether requests are self-contained.** If the inference route
answers each request from its own payload alone, without server-side
session state between requests, any replica can answer any request
and the container needs no instance pinning. If the server carries
per-connection state (a warm KV cache tied to a session id, per-episode
memory), the container must pin one session to one server instance.

**Note the auth situation** from `CONTEXT.md` and the server itself.
Header token, mutual TLS, or none. If a header token is required, the
Manifold platform does not currently give the container a secret
store, so the workaround is to restrict the endpoint by network and
drop the token for now.

**Note the action convention the server emits.** Absolute pose or
delta pose. Rotation format. Gripper polarity and range. Chunk size
and the frame rate the chunk was trained at. Where you send state
and how much history the server consumes.

### Benchmark side

**Find the target Manifold benchmark** in `manifold.benchmarks` (for
example LIBERO, SIMPLER, ROBOCASA). Use the manifold-sdk's
`Benchmark`, not the user's project's vendored copy.

**Read the `Benchmark`:** `embodiment.action`,
`embodiment.proprioception`, `sensors` (name, resolution, mount),
`instruction`.

**If the benchmark is not available**, stop and notify the user. The
benchmark must be authored first.

> **Phase 1 checkpoint.** Record before designing anything:
> ```
> Endpoint side:
>   description_route  = ?  (method, path, body)
>   inference_route    = ?  (method, path, body)
>   wire_encoding      = ?
>   cold_start_budget  = ?  (seconds until /config describes a loaded checkpoint)
>   first_task_latency = ?  (seconds on the first request per task)
>   requests_self_contained = yes | no  (if no: session pinning is required)
>   auth               = open | network_restricted | header_token
>   action_convention  = {absolute|delta}, rotation=?, gripper=?, frame=?
>   chunk_size         = ?  (poses returned per request)
>   chunk_hz           = ?  (frame rate the chunk was trained at)
>   state_window       = ?  (rows of history the server reads)
>
> Benchmark side:
>   target_benchmark   = LIBERO | SIMPLER | ROBOCASA | ...
>   embodiment_action  = {type, rotation, gripper, delta, frame}
>   cameras_published  = [{name, shape}]
>   instruction        = true | false
>
> Feasible: true | false  (if false: why, and stop)
> ```

---

## Phase 2: Design the signature, layouts, and profile

Decide what your wrap will look like before writing any code.
Everything here is a decision, not implementation.

**Which file holds which thing:**

| Thing | File |
|---|---|
| `Profile` class (the dataclass template) | `profile.py` |
| `PolicySignature(...)` instance | pairing file |
| Two `NativeLayout(...)` instances (input + output) | pairing file |
| Profile instance (`POLICYNAME_BENCHMARKNAME = MyProfile(...)`) | pairing file |
| `PROFILE`, `BENCHMARK`, `PIPELINE` module-level names | pairing file |

`profile.py` defines the Profile class and its fields (layouts, wire
conventions, env overrides). The pairing file creates one and fills
those fields with real values.

### The signature: what the driver emits and consumes

The pairing file declares a signature. This is a description of the
action space the driver returns and the observation it consumes.
Same rules as any wrap. In code it is a `PolicySignature`.

**Stop if the action is not end-effector, joint, or a unified
variant** (in code: `EEActionSpace`, `JointActionSpace`, or
`UnifiedActionSpace(payload=...)`). Same if the state is not
end-effector pose or joint position (`ee_pose` or `joint_pos`).
Anything else is benchmark work, not a wrap.

- Report real-world units. Meters, radians, gripper state. If the
  server emits normalized values, convert them in the driver before
  returning.
- The checks compare your signature to the benchmark, not to what
  the server actually does. Wrong shapes here pass the checks and
  crash at run time.
- Describe cameras as what the driver actually forwards. Include
  extra dimensions from frame stacking if the server needs them.
- No proprioception? Say so with an empty `Proprioception()`. Don't
  invent fake data to fill the gap.
- No instruction? Set `instruction=False`.
- Spell out every convention field (`rotation=`, `gripper=`,
  `delta=`, `frame=`) from the server's contract. Do not copy the
  benchmark's action object.

### The layout: benchmark observation to driver input, driver output to action

The benchmark hands the driver an observation object. The driver's
session code reads a dictionary. A layout describes how each
dictionary key gets filled from the observation, and how the
driver's returned dictionary is sliced back into an action. In code
these are two `NativeLayout` instances.

- The input layout maps observation channels to dictionary keys.
- The output layout maps the driver's returned dictionary back to an
  action.

Same layout entry rules as any wrap. `Slice` takes keyword args only
(`Slice(start=0, stop=dim)`).

**Camera pose entries.** If the server needs per-step camera poses,
use `ExposeCameraPoses` (a pipeline observation adapter) to publish
each camera's flattened 4x4 world pose as a state channel, then
declare `LayoutEntry(source=SourceKind.STATE, source_name="<name>_pose",
ops=(Slice(start=0, stop=16), ...))`. `verify` probes state keys by
their declared length, so pin the width with `Slice` even when it is
an identity window.

**Instruction entry.** Use `SourceKind.INSTRUCTION`. Every step, the
current task's instruction is passed to the driver. Do not cache it
on the endpoint.

### The profile: the wrap's spec

The profile carries the signature, the input and output layouts,
the request and response constants the driver needs at run time,
and a `load()` method that opens a connection to the server. In
code it is a frozen dataclass with these fields:

- `signature: PolicySignature`
- `default_weights: str` (set to `""`. The server holds the
  checkpoint.)
- `input_layout: NativeLayout`
- `output_layout: NativeLayout`
- Request and response constants. Any values the driver needs when
  building requests or interpreting responses. For example:
  `scene_intrinsics`, `wrist_intrinsics`, `depth_scale_*`,
  `action_steps` (poses used per reply), `chunk_stride`,
  `max_pos_delta_m`, `max_rot_delta_rad`, `gripper_open_span`,
  `gripper_closed`, `timeout_s`.
- A `load(weights, device) -> Endpoint` method.

`load()` ignores both `weights` and `device`. It reads the endpoint
URL from an environment variable (for example `NT_SERVER_URL`) and
opens a connection to the server. Both `load()` arguments exist to
match the `PolicyProfile` protocol. They select nothing for a
remote wrap.

**Env-tunable knobs.** Some profile fields should be adjustable
without an image rebuild. Timeout, action steps per reply, chunk
stride, gripper convention. Provide an env-var override for each,
applied by `load()` via `dataclasses.replace`, so a registered
version's `config.env` can tweak them. Bake the map into the module:

```python
_ENV_OVERRIDES: dict[str, tuple[str, type]] = {
    "MY_ACTION_STEPS":  ("action_steps",     int),
    "MY_CHUNK_STRIDE":  ("chunk_stride",     int),
    "MY_TIMEOUT_S":     ("timeout_s",        float),
    "MY_GRIPPER_CLOSED":("gripper_closed",   float),
}
```

Defer the driver import into `load()` so `profile.py` imports without
`httpx` on the sys.path. This keeps the pairing gates
(`check_compatibility` / `verify`) runnable without the transport.

> **Phase 2 checkpoint:**
> ```
> PolicySignature:
>   action_space        = {type}(rotation=?, gripper=?, delta=?, frame=?, chunk_size=1)
>   proprioception      = {ee_pose: ..., joint_pos: ...}
>   cameras             = [{name, shape, dtype}]
>   instruction         = true | false
>
> NativeLayout input keys:  [list with source_kind and source_name]
> NativeLayout output keys: [list with ops]
>
> Profile fields:
>   default_weights     = "" (empty; server holds the checkpoint)
>   wire constants      = [list]
>   env overrides       = [list of (env var, field, type)]
>   timeout_s           = ?
> ```

---

## Phase 3: Implement the wrap

Write the driver, assemble the pipeline, pass both checks, then prove
it live.

### File skeletons

Write these three files. Each snippet below is the minimum shape;
fill in the wire-specific logic, then flesh out with the rules that
follow.

**`profile.py`.** Frozen dataclass, no `httpx` import at top level:

```python
from __future__ import annotations
import os
from dataclasses import dataclass, replace
from typing import TYPE_CHECKING
from manifold.core.native_layout import NativeLayout
from manifold.core.policy import PolicySignature

if TYPE_CHECKING:
    from mywrap.driver import MyEndpoint

SERVER_URL_ENV = "MY_SERVER_URL"

_ENV_OVERRIDES: dict[str, tuple[str, type]] = {
    "MY_TIMEOUT_S":     ("timeout_s",     float),
    "MY_ACTION_STEPS":  ("action_steps",  int),
}

@dataclass(frozen=True)
class MyProfile:
    signature: PolicySignature
    default_weights: str
    input_layout: NativeLayout
    output_layout: NativeLayout
    # wire convention fields
    action_steps: int
    timeout_s: float
    # ...more constants the driver needs

    def load(self, _weights: str, _device: str | None):
        # deferred so profile.py imports without httpx
        from mywrap.driver import MyEndpoint
        return MyEndpoint(self._with_env_overrides(), os.environ[SERVER_URL_ENV])

    def _with_env_overrides(self) -> "MyProfile":
        overrides = {
            field: parse(os.environ[name])
            for name, (field, parse) in _ENV_OVERRIDES.items()
            if name in os.environ
        }
        return replace(self, **overrides) if overrides else self
```

**`driver.py`.** HTTP client + endpoint (dials the server once) +
session (one per runner):

```python
from __future__ import annotations
import threading, time
from typing import Any, Protocol
import httpx

_READY_TIMEOUT_S = 540.0
_READY_POLL_DELAY_S = 5.0

class _HttpClient:
    def __init__(self, base_url: str, timeout_s: float):
        self._http = httpx.Client(base_url=base_url.rstrip("/"), timeout=timeout_s)

    def config(self) -> dict[str, Any]:
        deadline = time.monotonic() + _READY_TIMEOUT_S
        last = "no attempt completed"
        while True:
            try:
                r = self._http.get("/config", timeout=15.0)
                r.raise_for_status()
                body = r.json()
                if body.get("camera_keys"):
                    return body
                last = "server is up but checkpoint is not loaded yet"
            except (httpx.HTTPError, ValueError) as exc:
                last = str(exc)
            if time.monotonic() >= deadline:
                raise RuntimeError(f"/config did not describe a loaded checkpoint: {last}")
            time.sleep(_READY_POLL_DELAY_S)

    def infer(self, sample: dict[str, Any]) -> dict[str, Any]:
        r = self._http.post("/infer", json=sample)  # or msgpack, per contract
        r.raise_for_status()
        return r.json()

class MyEndpoint:
    def __init__(self, profile, server_url: str):
        self._profile = profile
        self.signature = profile.signature  # expose by identity, not copy
        self._client = _HttpClient(server_url, profile.timeout_s)
        self._lock = threading.Lock()
        config = self._client.config()
        self.camera_keys = tuple(config.get("camera_keys") or ())
        # ...store anything the session needs from /config

    @property
    def profile(self):
        return self._profile

    def session(self):
        return MySession(self)

    def forward(self, sample: dict[str, Any]) -> dict[str, Any]:
        with self._lock:
            return self._client.infer(sample)

class SessionEndpoint(Protocol):
    camera_keys: tuple[str, ...]
    @property
    def profile(self): ...
    def forward(self, sample: dict[str, Any]) -> dict[str, Any]: ...

class MySession:
    def __init__(self, endpoint: SessionEndpoint):
        self._endpoint = endpoint
        # ...per-connection state (history, pose queue)

    def advance(self, native: dict[str, Any]) -> tuple[Any, int]:
        # build the sample, forward it, convert the reply to an action
        ...

    def reset(self) -> None: ...
    def close(self) -> None: ...
```

**`<policyname>_<benchmarkname>.py`.** The pairing file. Same shape
as any wrap:

```python
from manifold.core.pipeline import Pipeline
from manifold.adapters import PackToNativeLayout, UnpackFromNativeLayout
from mywrap.profile import MyProfile

SIGNATURE = PolicySignature(...)
INPUT_LAYOUT = NativeLayout(entries=(...))
OUTPUT_LAYOUT = NativeLayout(entries=(...))

MYPOLICY_MYBENCH = MyProfile(
    signature=SIGNATURE,
    default_weights="",
    input_layout=INPUT_LAYOUT,
    output_layout=OUTPUT_LAYOUT,
    action_steps=20,
    timeout_s=180.0,
    # ...more wire constants
)

PROFILE = MYPOLICY_MYBENCH
BENCHMARK = ...    # the manifold.benchmarks entry
PIPELINE = Pipeline(
    observation=[...],   # benchmark form to signature's consumed form
    action=[],           # often empty; driver emits the final action
    pack=PackToNativeLayout(PROFILE.input_layout),
    unpack=UnpackFromNativeLayout(PROFILE.output_layout, SIGNATURE.action_space),
)
```

### Where to put each translation

The benchmark's data won't match what the server expects. Something
has to convert between them. Three places to put those conversions:

- **Pipeline adapters** for reusable typed conversions. Built-in
  adapters cover rotation format, gripper polarity, camera
  resize/flip/channel-order, frame rebase, frame history, camera
  pose exposure. `check_compatibility` inspects the chain and
  validates it.
- **NativeLayout entries** for building the input dictionary the
  driver reads: renaming keys,
  casting dtypes, slicing arrays. Not typed, so `check_compatibility`
  can't reason about them, but `verify` pushes data through and
  confirms the shapes come out right.
- **Driver session code** for per-endpoint idiosyncrasies: the wire
  codec, the readiness poll, absolute-to-delta conversion measured
  from the current pose, depth-format changes (metric floats to
  uint16 counts), gripper openness computed from finger positions,
  building the wire sample dict.

Prefer the highest place that fits. Every session-side operation is
invisible to the checks, so name each one in the pairing docstring.

### Wire codec

Verify byte-for-byte against the upstream. Do **not** assume a common
name like "msgpack-numpy" means the same thing as the PyPI package;
implementations differ in map keys and dtype tokens. If the server's
codec is project-specific, inline it in `driver.py` rather than
depending on a package that might drift.

### Readiness poll

At load, poll the description route until the server describes a
loaded checkpoint. Budget the poll under the runner's readiness
timeout (600 seconds today; leave headroom, for example 540 seconds).
The container's readiness probe hits port 8000 within that window,
so a dead upstream must fail with a driver-side message rather than
letting the runner time out first.

Treat these as retries, not failures:

- Empty description fields. The server is up but its checkpoint has
  not loaded yet.
- Undecodable response body. A proxy in front of a cold container
  can answer its own interstitial page under HTTP 200.

Treat these as failures:

- Connection refused past the deadline.
- Real HTTP errors (401, 5xx) with a decodable error message.

### Session shape: per-step or chunked?

If the server consumes a **state window** of consecutive rows, or
returns **absolute poses**, serve one step per `advance` and refill
the pose queue only when it drains. Two reasons:

- A state window sampled every N steps (instead of every step) skips
  observations the server needs, so a buffered open-loop chunk
  corrupts the input.
- An absolute-pose reply becomes stale between when it was predicted
  and when the arm reaches the target; measure the delta from where
  the arm is NOW, not from the previous target, so tracking error is
  corrected on the following step instead of building up.

If the server returns per-step deltas measured against a state window
you can supply consecutively, the standard `OpenLoopChunkQueue` still
applies. Otherwise, hand-roll the pose queue in the session (see the
newtheory example in the SDK).

### Absolute-to-delta conversion

Do it in the session, after each request. Read the current EE pose
from `native`, subtract it from the next queued absolute pose, scale
by the benchmark controller's per-step limits
(`max_pos_delta_m` for position, `max_rot_delta_rad` for rotation),
and clip to `[-1, 1]`.

The SDK ships no adapter for this, and a pipeline action adapter
cannot see per-step observations. See wrap-policy Phase 3 Traps for
why the pipeline-side workaround is fragile; the driver-side solve
is the answer for the remote case.

### Testable session

Declare a `SessionEndpoint` `Protocol` that names only the surface
the session reads. `camera_keys`, `state_window`, wire constants,
`forward()`. Type the session against the Protocol, not against the
real endpoint class. Then session tests can run against a stub that
returns canned poses instead of POSTing to the real server:

```python
class _StubEndpoint:
    camera_keys = ("agentview",)
    state_window = 1
    # ...whatever else the Protocol names
    def forward(self, sample):
        return {"actions": np.zeros((10, 7), dtype=np.float32)}
```

Session tests over the stub prove the delta conversion, gripper
mapping, and history bookkeeping without a live server.

### Endpoint hygiene

- **Expose the profile's `SIGNATURE` object as `endpoint.signature`**,
  not a copy. The live check reads it by identity.
- **Take a `threading.Lock` around `forward()`.** The server runs
  one checkpoint on one GPU; concurrent shards queue here rather
  than time each other out on the server.
- **Keep nothing mutable on the endpoint.** Per-episode state goes
  in the session.
- **`endpoint.profile` must return the profile the endpoint was
  built from.** Missing this kills the connection pre-READY with
  `PairingRejected("policy rejected the pairing (no READY)")`.

### Check the wrap

Two checks confirm the wrap is well-formed before you try to run it.

- **`check_compatibility`** compares your signature to the benchmark,
  walking the pipeline's adapter chain. Verdicts: `COMPATIBLE`,
  `COMPATIBLE_VIA_PIPELINE`, `INCOMPATIBLE`.
- **`verify`** pushes fake data through the pipeline and reports
  which pieces it managed to exercise. Anything under `not_checked`
  is a piece only a live run can prove.

```python
from manifold.core.check import check_compatibility
from manifold.core.verify import verify

report = check_compatibility(SIGNATURE, BENCHMARK, PIPELINE)
verified = verify(SIGNATURE, BENCHMARK, PIPELINE)
```

**These checks do not touch the driver.** They pass or fail based on
the pairing alone. A wrap can pass both and still crash on the first
POST because the wire codec is wrong or the readiness poll gets a
different route. Only a live request proves the driver.

Fix your code until both pass. Never change the signature to shut a
check up. A signature that doesn't actually match what the driver
sends will pass the checks but crash the moment a real observation
arrives.

Once both pass, confirm the module itself is well-formed:

```sh
python -c "
import importlib
from manifold.recipes import read_pairing
print(read_pairing(importlib.import_module('<your wrap module>')))
"
```

If `verify` reports `not_checked` entries, list them in the handoff.
Do not claim the wrap is "verified" if pieces went untested.

### Run the wrap live

The checks do not touch the driver. The driver is only proven by a
real request against the real endpoint. Two ways to do it:

- **Endpoint reachable from your machine.** Set the endpoint URL
  env var (for example `MY_SERVER_URL=https://...`) and run
  `evaluate` with a minimal `reset`/`step` pair:

  ```python
  from manifold.recipes import evaluate
  result = evaluate(
      endpoint, BENCHMARK, reset, step,
      pipeline=PIPELINE, episodes=2, max_steps=12,
  )
  ```

  `evaluate` runs the driver, dials the endpoint, and reports any
  exception as a traceback. Look for the driver's startup log line
  (upstream URL, camera list, state window read from `/config`) and
  a clean run of two episodes.

- **Endpoint not reachable** (private VPC, internal-only, or the
  user is on a build box without egress). Build a stub
  `SessionEndpoint` (see "Testable session" above), plug it into
  `MyEndpoint` for the duration of the test, and run `evaluate`
  against it. This exercises the wrap's Python code. It does not
  test the wire format against the real server. Note that in the
  handoff. Make the first real proof a scored run under
  `/containerize-remote-wrap`.

**How to describe this run to the user.** Call it "a test run to
check that the driver reaches your server and finishes without
errors." Do NOT say you are "faking the physics" or "using fake
data." Those phrases are correct SDK jargon but read as
untrustworthy to a non-engineer.

**Do not treat episode-to-episode differences as a wrap bug.** With
hand-written `reset` and `step` (or a stub `SessionEndpoint`), the
two episodes will not exactly reproduce each other unless the
stand-in is deterministic. That is expected. The live run only
tests one thing. Did the driver reach the server and answer
without raising an exception? If both episodes finish
without an exception, proceed to the handoff. Do not go back and
edit the wrap to make the episodes match.

> **Phase 3 checkpoint:**
> ```
> Pipeline observation adapters: [list, in order]
> Pipeline action adapters:      [list, in order]
> Session-side operations:       [list]
>
> Wire codec verified byte-for-byte against upstream: yes | no (justify)
> Readiness poll budget:                              540s (below runner's 600s)
> Session shape:                                      per_step | chunked (justify)
> Absolute-to-delta:                                  n/a | driver-side (against current pose)
>
> Checks:
>   check_compatibility  = COMPATIBLE | COMPATIBLE_VIA_PIPELINE | INCOMPATIBLE
>     lossless           = true | false
>     reasons            = [if any]
>   verify               = N checked, M failed, K not checked
>     failed             = [list]
>     not_checked        = [list, quote in handoff]
>   read_pairing         = success | failure
>
> Live run:
>   endpoint reachable   = yes | no
>   evaluate mode        = real endpoint | stub SessionEndpoint
>   episodes             = ?
>   forward_count        = ?
>   startup log printed  = yes | no
> ```

---

## Final checklist

- [ ] `read_pairing` accepts the module without `httpx` on the path
- [ ] `default_weights` is `""`; `load()` ignores both arguments and
      reads the endpoint URL from an env var
- [ ] Signature declares what the driver actually sends and receives,
      in physical units
- [ ] Wire codec verified byte-for-byte against upstream (inlined if
      project-specific)
- [ ] Readiness poll under the runner's 600s timeout; empty-fields
      and undecodable-body treated as retries
- [ ] Session shape matches the server's contract: per-step for
      state-window or absolute-pose servers, chunked otherwise
- [ ] Absolute-to-delta done driver-side, measured from the current
      pose (if applicable)
- [ ] `SessionEndpoint` `Protocol` declared; session typed against it
- [ ] `endpoint.signature` is the profile's object; `endpoint.profile`
      returns the profile
- [ ] `threading.Lock` around `forward()`; nothing mutable on the
      endpoint
- [ ] Env overrides for deployment-tunable knobs (timeout, action
      steps, gripper convention)
- [ ] Checks pass; `verify` zero failed (or documented exception);
      `not_checked` quoted in handoff
- [ ] **Live run** ran at least 2 episodes without error, either
      against the real endpoint or against a stub `SessionEndpoint`
- [ ] Handoff states what was and was not proven, including whether
      the first real wire-format proof is deferred to a scored run

---

## Stop here, hand back to the user

The skill ends at the checklist above. Do **not** invoke another
skill automatically. In particular, do **not** call
`/containerize-remote-wrap` on your own after saying "the wrap is
proven"; that skill costs registry storage and often cloud time, and
the user has to opt in first. Each step in the Manifold flow is a
separate skill by design; auto-chaining skips the user's chance to
review.

**Keep the handoff SHORT.** A long summary reads as "we're done."
The user needs to see, at a glance, that there is a next step. Aim
for under 8 lines total, and put the next step in a visible box so
it does not get buried in prose.

Do NOT restate the wrap's design. Do NOT list what was done in Phase
1 or Phase 2. Do NOT review the checks. The user watched those
happen; the handoff is about what is next.

Use this exact shape:

1. **One line: status.** Example: "Wrap is written and passes the
   checks."
2. **One line: what got made.** The module path and the pairing name.
3. **Caveats, only if any.** One line each. `verify` entries under
   `not_checked`, whether the wire format was tested against a stub
   rather than the real server, or an auth situation that still
   needs the user to arrange a network restriction. Skip this
   entirely if there are none.
4. **Next step, boxed.** Present the next skill as a bordered
   call-out so it stands apart from the prose. For example:

   ```
   ┌────────────────────────────────────────────────────────────┐
   │  NEXT: run  /containerize-remote-wrap                      │
   │  Packages the driver and registers it on Manifold with     │
   │  the endpoint URL in the version's config.                 │
   └────────────────────────────────────────────────────────────┘
   ```

   A one-row markdown table works too, if box-drawing characters
   render badly in the harness:

   | Next step | What it does |
   |---|---|
   | Run `/containerize-remote-wrap` | Package the driver and register it on Manifold with the endpoint URL in the version's config. |

Then stop. Wait for the user to invoke the next skill.

---

## Reference: import paths

- `manifold.recipes`. `read_pairing`, `launch_server`, `serve`,
  `evaluate`, `run_benchmark`, `run_sharded_benchmark`,
  `run_episodes`, `write_rollup`, `OpenLoopChunkQueue`,
  `PolicyProfile`, `resolve`, `describe`, `Recorder`, `dump`,
  `load`, `NO_RECORDER`
- `manifold.recipes.serving`. `PolicyEndpoint` and `Session`
  protocols (NOT re-exported by `manifold.recipes`)
- `manifold.core.check`. `check_compatibility` returns `Report`
- `manifold.core.verify`. `verify` returns `VerifyReport`
- `manifold.core.pipeline`. `Pipeline`
- `manifold.core.policy`. `PolicySignature`
- `manifold.core.embodiment`. `EEActionSpace`, `JointActionSpace`,
  `UnifiedActionSpace`, `Proprioception`, `EEObservationSpec`,
  `GripperObservationSpec`
- `manifold.core.conventions`. `RotationFormat`, `GripperFormat`,
  `Frame` (pass enum members, never string values)
- `manifold.core.native_layout`. `NativeLayout`, `LayoutEntry`
  (`.from_camera` / `.from_state` / `.from_instruction`),
  `SourceKind`, `Slice`, `Split`, `BatchAxis`, `DtypeCast`,
  `Component`, `Assemble`
- `manifold.core.sensor`. `CameraIntrinsics`
- `manifold.benchmarks`. `ALL`, `LIBERO`, `SIMPLER`, `ROBOCASA`
- `manifold.adapters`. `PackToNativeLayout`,
  `UnpackFromNativeLayout`, `ObservationTap`, `ActionTap`
- `manifold.adapters.observation`. `ProprioRotationAdapter`,
  `FrameRebaseAdapter`, `DynamicFrameRebaseAdapter`,
  `ObservedGripperAdapter`, `ResizeCameras`, `Rotate180Cameras`,
  `FlipVerticalCameras`, `SwapChannelOrder`, `StackFrameHistory`,
  `ExposeCameraPoses`
- `manifold.adapters.action`. `RotationFormatAdapter`,
  `GripperPolarityAdapter`, `GripperThresholdAdapter`,
  `UnifiedGripperThresholdAdapter`, `UnifiedSliceAdapter`,
  `BasePinWiden`, `DiscreteBinarize`
- `httpx`. HTTP client (add to the project's deps; not an SDK
  dependency)
