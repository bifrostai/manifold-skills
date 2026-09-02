---
name: containerize-remote-wrap
description: >
  Package a remote-endpoint policy wrap as a container image for the
  Manifold platform, push it to a container registry, register it with
  the endpoint URL in its config env, and offer a scored test run. Use
  after `/wrap-remote-policy`, when the model runs on the user's own
  inference server and the container only dials it.
compatibility: >
  Run from the user's policy project directory, after
  `/wrap-remote-policy` has written the wrap files under
  `<project>/.manifold/<slug>/`. Requires docker, a push credential
  for the target container registry (ghcr by default), and the
  Manifold MCP server available so the skill can call
  `register_policy_version` with `config.env`. Input is a wrap that
  passes check_compatibility and verify under wrap-remote-policy.
---

## Summary

Take a wrap from `/wrap-remote-policy` (a Python module that exports
`PROFILE`, `BENCHMARK`, and `PIPELINE` and dials the user's inference
server) and get it running on the Manifold platform.

The output is four things:

1. **A `Dockerfile`** written to `<project>/.manifold/<slug>/Dockerfile`.
   It builds a slim image with the driver and its HTTP client, using
   the project root as the build context.
2. **A launcher script** written to `<project>/.manifold/<slug>/serve.py`.
   It imports the pairing module and calls `launch_server`. Phase 3
   shows the exact code to paste in.
3. **A container image** pushed to the registry at the agreed name
   and tag (for example
   `ghcr.io/<org>/policy-<slug>-<benchmark>:0.1.0`).
4. **A registered policy version** on the Manifold platform, with the
   endpoint URL in its `config.env` and `minimum_gpu_memory_gb: 0`.
   Registered via the Manifold MCP server's `register_policy_version`
   tool, because that path carries `config.env`; the `manifold policy
   init` CLI does not today.

Files 1 and 2 are new files on disk. The other two live on the
registry and on the platform.

The image is different from the in-container-model case in three
ways:

- **No model, no CUDA.** The image is a slim Python base plus
  manifold-sdk, `httpx`, and any decoders the benchmark needs (for
  example `pillow` if the benchmark ships colored frames as JPEG).
- **No project source.** The wrap does not import from the user's
  project; only the pairing folder ships.
- **No GPU claim.** Registers with `minimum_gpu_memory_gb: 0`, so
  placement can put it on a GPU-less runner rather than occupying a
  card.

Once all four exist, the skill's job is done. After that, ask the
user whether to submit a scored test run against a benchmark of
their choice (Phase 4 walks through it). Do not submit a run on your
own.

This skill does not fix wrap bugs. If the wrap does not pass both
`check_compatibility` and `verify`, go back to
`/wrap-remote-policy` first.

## Rules

**Confirm before doing anything else.** First thing after this skill
loads, tell the user in your own words what it will do and why:
building a small Docker image (a few hundred MB, most of it scipy and
numpy), pushing it to their registry, and registering it on Manifold
with the endpoint URL in `config.env`, so the platform can pull and
run their driver against benchmarks. Wait for a yes before reading
`CONTEXT.md` or touching anything. Push, register, and submit still
have their own per-step confirmations later.

**Speak to the user in their language, not the SDK's.** The user has
not read the SDK docs. They will not recognize Docker fields,
registry commands, or the names of Manifold's tools and config
fields. The skill below names those identifiers freely because you
need them to write correct code. When narrating progress to the
user, translate.

Say things like:
- "I'll build the image that dials your server."
- "The image built and pushed to your registry."
- "Registered on Manifold with your endpoint URL attached to the
  version's config."
- "The scored test run finished. The score is X."

Not the identifiers from the Dockerfile snippets or tool calls
below. If the user uses one of those terms themselves, follow
their lead. Otherwise, describe what happened and why it matters.

**Run in the user's policy directory.** Once the user has said yes,
confirm the current working directory is their policy project. The
same one `/setup-manifold` and `/wrap-remote-policy` ran in. Look for
`<project>/.manifold/CONTEXT.md` and `.manifold/<slug>/` (with the
wrap files inside). If either is missing, stop and ask the user to
run the earlier skills first.

**Read `.manifold/CONTEXT.md` first.** setup-manifold already
recorded the package manager, registry, endpoint URL, auth
situation, wire contract, and benchmarks of interest; wrap-remote-policy
added anything else it learned. Read `CONTEXT.md` before asking the
user anything; ask only about details it does not cover. If
`CONTEXT.md` has `model_runtime = in_container`, stop and tell the
user this skill is for hosted-endpoint policies; `/containerize-wrap`
is the one for in-container models.

**Write into `.manifold/<slug>/`.** The `Dockerfile` and `serve.py`
this skill produces both live under
`<project>/.manifold/<slug>/`, next to the wrap files. `docker
build` runs with the project root as the build context so the whole
project is available to `COPY`.

**Do not run this skill without an explicit user invocation.** If
another skill (for example `/wrap-remote-policy`) has just finished,
stop and wait for the user to ask for containerization by name. Do
**not** chain into this skill on your own after "the wrap is proven"
or any similar success line. Each step in the Manifold flow is a
separate skill so the user can review the previous step's output
before spending registry storage and cloud time; auto-chaining
skips that review.

**Plan the entire task in a to-do list before you start, and update
it as you go.** Use whichever planning tool your harness provides:

- **Claude Code:** `TaskCreate` to seed the plan, `TaskUpdate` to
  move items between `pending` / `in_progress` / `completed`,
  `TaskList` / `TaskGet` to read state.
- **Codex:** use `update_plan` to create and maintain an ordered
  plan, with exactly one item `in_progress` at a time. Keep the
  scored test run as an explicit item until it passes.
- **Other harnesses:** check the harness for a to-do list or
  planning tool before using the fallback below.
- **No planning tool available:** keep the plan as a plain-text
  checklist in your responses and re-post it (with statuses
  updated) each time you advance.

The intent is (1) to **structure the work** so nothing gets skipped,
and (2) to **stay accountable and informative** by updating the list
as steps start and finish, so the user can follow along without
asking.

**Ask the user before every step that costs money or writes to a
shared system.** Each of the choices below costs the user time,
cloud credits, or registry storage. Do not pick any of them
yourself:

- The target benchmark for the scored test run. Never pick "a
  reasonable benchmark" for the user.
- Pushing the image to the registry.
- Registering the policy on the platform.
- Submitting a run.

At each of those points: state what you are about to do, wait for a
confirmation, and only then proceed. Do not chain "wrap built
successfully, now submitting a run against <a benchmark>" into one
action. **If the user declines any of these, stop the skill there;
do not skip to the next step.**

The **policy name** (the thing that becomes `<slug>` in the registered
version) is **not** on that list. setup-manifold picks it and hands
it here. Do not rename the policy on the user's behalf. If the slug
is missing, ask the user once for it.

The container image name (registry, namespace, tag) is a
container-registry concept, not a user-facing choice. This skill
constructs it from the policy slug plus the org's registry and
namespace.

## How the platform runs a policy

The platform runs policies as registered container images. When a
run is dispatched, the runner pulls the image, starts it, waits for
the server inside to accept connections on port 8000, and then drives
it through the benchmark.

Consequences for the remote case:

1. **One image per wrap.** The `CMD` in the Dockerfile picks which
   wrap the container serves. Registration cannot override it.
2. **Tags become versions.** Registration reads the image tag and
   uses it as the version string. Never reuse a tag; old tags are
   immutable.
3. **The runner injects `config.env` at container start.** The env
   map from the registered version becomes environment variables in
   the container. The driver reads the endpoint URL from one of
   those vars (for example `MY_SERVER_URL`), and any tunable
   overrides the profile exposes.
4. **Nothing local proves the image works in production conditions.**
   A clean local `docker run` shows that the driver reaches the
   endpoint and the server accepts connections on 8000. Whether the
   platform can pull, schedule, drive, and score the image is only
   settled by an actual scored run, which is up to the user to
   submit.

---

## Phase 1: Understand the inputs

Do this before writing a Dockerfile. You need the wrap module path,
the policy name, the manifold-sdk version, and the env vars the
driver reads pinned down first.

**The wrap.** Confirm the wrap passes both `check_compatibility` and
`verify` under `/wrap-remote-policy`. Record the module path that
exports `PROFILE`, `BENCHMARK`, and `PIPELINE`. The Dockerfile's
`CMD` will name it.

**The policy name.** This is the slug used in the registered policy
version. setup-manifold picks it (or the user does directly). If it
is missing, ask the user once for it. Do not invent one.

**The image name.** Built from the policy slug plus the org's
registry and namespace, in the shape:
`<registry>/<namespace>/policy-<slug>:<tag>`. For example,
`ghcr.io/<your-org>/policy-mypolicy-mybench:0.1.0`. The tag is a
version string; bump it for every change to the wrap or the
Dockerfile. This skill constructs the image name from those parts;
do not ask the user to type it out.

Examples below use `ghcr.io/<your-org>`. Substitute the actual
registry and namespace throughout.

**The manifold-sdk version.** Install the same manifold-sdk version
the wrap was written against. Read it from the wrap's environment
(`uv.lock`, `pyproject.toml`, or `uv pip freeze`). The policy and
the benchmark exchange messages defined by the SDK, and mismatched
versions can silently disagree at run time.

**The endpoint URL env var.** Read `profile.py` and find the env
var the driver reads to get the server URL (for example
`MY_SERVER_URL`). This will go into the registered version's
`config.env`. If you can't find it in one grep, stop and ask the
user.

**Any tunable env vars.** Some fields on the profile are exposed as
env-var overrides (timeout, action steps, gripper convention). Read
`_ENV_OVERRIDES` in `profile.py` (or equivalent) and list them.
Only the endpoint URL is required at registration; overrides are
optional and can be added later.

**The auth model** from `CONTEXT.md`. If the endpoint needs a header
token, flag it. `config.env` is not a secret store today, so the
practical answer is a network restriction on the endpoint (IP
allowlist to the runner's egress, or VPC-internal). If the user has
not arranged that, stop and ask them to before registration.

**The benchmark decoders the image needs.** Look at the benchmark's
sensors. If the benchmark ships colored frames as JPEG, the image
needs `pillow`. If it ships depth as TIFF, same. If unsure, add
`pillow` (small, common); a missing decoder surfaces as an import
error the moment the pack tries to decode.

> **Phase 1 checkpoint.** Record before designing:
> ```
> wrap_module           = ?  (for example mywrap.mypolicy_mybench)
> checks_pass           = yes (link to wrap-remote-policy handoff)
> policy_slug           = ?  (source: setup-manifold | user)
> image_name            = <registry>/<namespace>/policy-<slug>:<tag>
>                         (constructed by this skill from the slug)
> sdk_version           = ?  (from wrap-remote-policy environment)
> endpoint_url_env      = ?  (for example MY_SERVER_URL)
> tunable_env_vars      = [list, or empty]
> auth_model            = open | network_restricted | header_token
> network_restriction_arranged = yes | no | n/a
> benchmark_decoders    = [list, for example httpx, pillow]
> ```

---

## Phase 2: Design the Dockerfile

Decide what the image will contain before writing any of it.
Everything in this phase is a decision, not implementation.

### What goes in the image

Assume the user has a project folder with the wrap files under
`.manifold/<slug>/`. The image does **not** contain the model, the
model's dependencies, or the project's source code.

The image contains, in this order:

1. **A slim Python base image**, pinned by digest, not by a floating
   tag like `latest`, so rebuilds are reproducible. `python:3.12-slim-bookworm`
   is a safe default. Debian, not Alpine: the runner probes the
   container's readiness with a bash `/dev/tcp` trick that busybox
   ash does not have.
2. **manifold-sdk**, installed at the version the wrap was written
   against. The recommended install path is the project's `uv.lock`
   (exported to `requirements.txt` in a discarded stage), so the
   image's dependency set matches the wrap's environment exactly.
3. **The driver's HTTP client and any benchmark decoders.** `httpx`
   for the transport. `pillow` if the target benchmark ships
   colored frames as JPEG (LIBERO does). Pin exact versions; do not
   let a fresh resolution at build time drift.
4. **The wrap files.** Only `.manifold/<slug>/`, not the project's
   `src/`. The wrap does not import from the project.

### The wrap files

The whole `.manifold/<slug>/` folder is what ships:
`driver.py`, `profile.py`, the pairing file, and `serve.py`. Copy
them into the `WORKDIR`, then set `PYTHONPATH` so the launcher's
`import` finds them.

### Weights

There are none. The checkpoint lives on the user's inference server.
Skip every "fetch or bake" decision from the in-container-model case.

### Auth to the endpoint

Not baked into the image, not in `config.env`. The Manifold platform
does not currently give the container a secret store, so anything
requiring a secret at run time must be handled by a network
restriction on the endpoint (IP allowlist, VPC-internal). If the
endpoint needs a header token today, the answer is to drop the token
and add the network restriction instead, until a runner-side secret
channel exists.

> **Phase 2 checkpoint:**
> ```
> base_image          = python:3.12-slim-bookworm@sha256:...
> deps_in_image       = manifold-sdk (pinned), httpx (pinned), pillow (pinned, if needed)
> project_source      = none (wrap does not import from it)
> weights             = none (server holds the checkpoint)
> auth_strategy       = network_restriction (endpoint accepts unauthenticated from runner IPs)
> ```

---

## Phase 3: Build and push the image

Write the Dockerfile and build it. A local `docker run` is
meaningful here even on a CPU-only box, because the container has
no GPU work to do. Then push to the registry.

### Dockerfile rules

These apply to every image:

**Port 8000.** The container must listen on port 8000.
`DEFAULT_CONTAINER_PORT` is set to 8000 in the runner code and
registration cannot change it. Bind `0.0.0.0:8000` in the `CMD`.

**Bash is required.** The readiness probe runs
`bash -c 'exec 3<>/dev/tcp/127.0.0.1/8000'` inside the container.
`python:3.12-slim-bookworm` has bash; do not switch to Alpine or a
distroless base.

**`PYTHONUNBUFFERED=1`.** Set this in the Dockerfile. Without it, a
crashing container's last output lines stay in a write buffer and
are lost. That output is how you diagnose failures (readiness poll
timing out, endpoint URL wrong, wire codec broken).

**Layer order.** Put manifold-sdk and its transitive deps
(scipy, numpy) in early layers. Put the wrap files in late layers.
That way, editing the wrap rebuilds only the cheap layers at the end.

**Do NOT apply** the following rules from the in-container-model
case: CUDA base image, layer ordering for torch/jax/CUDA wheels,
pinning model deps (numpy, scipy) for the model's tolerance (still
pin them, but for wire-format stability rather than model
tolerance), GPU memory pre-allocation flags. No model runs here.

### Launcher and CMD

Same launcher shape as the in-container case:

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

`COPY` the wrap module and the launcher into the `WORKDIR`, then:

```dockerfile
CMD ["python", "serve.py", "--pairing", "my_wrap.mypolicy_mybench", \
     "--host", "0.0.0.0", "--port", "8000"]
```

### Build

```sh
docker build -f <path-to-Dockerfile> \
    -t <registry>/<namespace>/policy-<slug>:<tag> .
```

A clean `docker build` means the image assembled without errors,
nothing more. The driver has not run yet.

### Run it locally

CPU-only local runs are meaningful for a remote wrap. The container
has no GPU work to do; its main job at startup is dialing the
endpoint and passing the readiness probe.

**If the endpoint is reachable from your local box**, run it:

```sh
docker run -p 8000:8000 -e MY_SERVER_URL=<endpoint-url> \
    <image>:<tag>
```

Look for:

- The driver's startup log line printing the endpoint URL, the
  camera list, and the state window read from the description route.
- The listening line: `listening on 0.0.0.0:8000 (TCP), up to 8
  concurrent shard(s)`.

Both must appear. If the container hangs before either, the
readiness poll is blocked. Check that the URL is right, the
endpoint is up, and (if the endpoint is network-restricted) your
local box's IP is allowlisted. If the container starts printing
`RuntimeError` from the driver's readiness poll and exits, the URL
or the description route is wrong.

**If the endpoint is not reachable from your local box** (private
VPC, internal-only), skip this step. The first real proof will be a
scored run in Phase 4.

### Push

**Save the Dockerfile and launcher before you push.** If they get
lost or overwritten later, no one can rebuild the image or see what
code went into it.

**Ask the user before pushing.** Show them the exact image name and
tag, and wait for confirmation.

```sh
docker login ghcr.io   # or the registry the user chose
docker push <registry>/<namespace>/policy-<slug>:<tag>
```

**Make sure the platform can pull the image.** Two paths:

- **Public registry** (a public ghcr package, a public Docker Hub
  repo). Pull just works, no credentials involved. A new ghcr
  package starts private; flip it to public at
  `https://github.com/orgs/<org>/packages/container/<package>/settings`
  before any run.
- **Private registry** (private ghcr package, private Docker Hub
  repo, ECR, Artifact Registry). The platform needs a pull
  credential configured on its side. Confirm with the Bifrost team
  before submitting a run.

Either way, a pull failure at run time surfaces as a scheduling or
pull error, not a permission error, so if the platform can't reach
the image, the run just looks broken.

> **Phase 3 checkpoint:**
> ```
> build_exit_code        = 0
> startup_log_printed    = yes | no | skipped (endpoint unreachable from local box)
> listening_line         = yes | no | skipped
> source_saved           = yes (where: git commit hash | folder | ...)
> push_confirmed_by_user = yes | no
> push_exit_code         = 0
> visibility             = public | private (action needed)
> ```

---

## Phase 4: Register with env, then offer a test run

Register the image via the Manifold MCP server's
`register_policy_version` tool, because that path carries
`config.env`. The `manifold policy init` CLI does not today, so the
CLI is not the right entry point for a remote wrap.

Then ask the user whether to submit a scored test run. Do not
submit one on your own.

### Register

**Ask the user to confirm the slug, version, image name, and
endpoint URL before calling the tool.** Registration creates a
catalog entry for their organization, and the tag becomes an
immutable version string.

Call `register_policy_version` with a payload like:

```json
{
  "organization": "<org>",
  "slug": "<slug>",
  "version": "<tag>",
  "adapter": "container",
  "config": {
    "image": "<registry>/<namespace>/policy-<slug>:<tag>",
    "env": {
      "MY_SERVER_URL": "<endpoint-url>"
    }
  },
  "minimum_gpu_memory_gb": 0
}
```

The image tag becomes the version. So a payload with
`"image": ".../policy-mypolicy-mybench:0.1.0"` registers version
`0.1.0`.

**`minimum_gpu_memory_gb: 0`.** The container has no GPU work to
do, and 0 lets placement take a GPU-less runner instead of
occupying a card. This is the opposite of the in-container-model
case, where 0 would silently deny GPU access to a model that needs
it.

**Add tunable overrides only if the user asked for them.** If the
user wants to change a timeout or a chunk stride at registration
time, add the matching env var to `config.env` (for example
`"MY_TIMEOUT_S": "240"`). Otherwise leave `config.env` with just
the endpoint URL.

**Auth caveat.** `config.env` is not a secret store. If the endpoint
needs a token, do not put it here. Restrict the endpoint by
network (IP allowlist, VPC-internal) instead.

**Visibility.** By default a new policy is visible only to its own
organization. Only make it public if the user asks for it.

### Offer a scored test run

Registration is done. The image is live on the platform, but you
have not yet seen the platform pull it, schedule it, drive it, and
score it. Only a scored run does that.

**Ask the user whether to submit one.** Something like: "The image
is registered as version `X` and points at your endpoint at `URL`.
Do you want to submit a scored test run? If so, which registered
benchmark should I pair it against? A run costs cloud time."

Do not pick a benchmark. Do not submit on your own. If the user
says no, stop here; the skill is done.

If the user says yes and names a benchmark, run:

```sh
manifold run submit <policy-slug> <benchmark-slug>
manifold run watch <run-id>
```

Before submitting:

- The named benchmark must be registered and must have a working
  image. If the run fails immediately with an error naming the
  benchmark image (missing image, benchmark container crash on
  start), that is a benchmark-side problem, not a wrap bug; flag
  it to the Bifrost team.
- Runs are visible to everyone in the organization; use a clear
  `--name` if the user wants the run labeled.
- The first request per task carries a fresh task sentence and the
  endpoint's language encoder may take 30 to 60 seconds on that
  request. The runner's per-request timeout must cover it; the
  wrap's `timeout_s` covers the driver's side.

### Validate the results

A completed run does not mean the wrap is correct. Check the
actual output.

**Episode completion.** All episodes should reach `completed`:

```sh
manifold run get <run-id> --episodes
```

**First frames.** Dump the first observation frame from each
episode. A mostly black first frame means the scene did not load,
regardless of what the wrap did with it.

**Action magnitudes.** Actions should be in the physical range for
the embodiment (millimeters, radians). Actions stuck at the edges
(all 1.0 or all -1.0) suggest either the driver's action scaling
is off or the server is returning normalized outputs the driver
never denormalized.

**Score vs reference.** Compare `mean_success` against the
checkpoint's published or previously measured score on this suite.
A checkpoint trained on a real rig will not score well in a
simulator's scenes, so the first run tells you the connection
works, not whether the model is any good.

**This is the only point in the whole flow where convention bugs
become visible.** A wrong gripper sign, a wrong component order, a
wrong rotation encoding, a wrong depth scale, all pass
`check_compatibility` and `verify`. They only show up here as a
score near zero. For a remote wrap, add: a wrong wire codec, a
wrong description-route contract, or a stale action scaling
constant.

A near-zero score on a suite the checkpoint is known to handle
means a wrap bug until proven otherwise. Go back to
`/wrap-remote-policy` Phase 3, fix the driver, bump the tag, and
re-enter this skill at Phase 3.

### If the run fails on connection to the endpoint

The container printed a driver error and exited, or all requests
returned 5xx. Check:

- The endpoint URL in `config.env` is right (typo, missing
  scheme, trailing slash).
- The endpoint is up and its description route responds. Test it
  from a machine that can reach it: `curl <url>/config`.
- The runner's egress addresses are allowlisted on the endpoint.
  If the endpoint is network-restricted and the platform's IPs are
  not allowed, every request fails.
- The wire codec matches. If `/config` answers but `/infer`
  returns a decode error, the codec assumption is wrong.

Fix, bump the tag, push, re-register the new tag with the same
`config.env`, and offer another scored test run.

### If the run is slow but works

The container has no GPU claim. That is expected; the model runs on
the user's server, and network round-trips dominate. If throughput
is a concern, the answer is either raising the sharded
concurrency (talk to the Bifrost team) or lowering the endpoint's
own response time; the wrap does not gate throughput here.

> **Phase 4 checkpoint:**
> ```
> slug                          = ?
> version                       = ? (from image tag)
> config_env_endpoint_url       = ?
> minimum_gpu_memory_gb         = 0 (always for a remote wrap)
> register_confirmed_by_user    = yes | no
> registration_result           = success | failure (message)
> test_run_offered_to_user      = yes  (must be yes; offering is required)
> user_asked_for_test_run       = yes | no  (if no: stop, skill is done)
>
> (fill in below only if the user asked for a test run)
> benchmark_slug                = ? (chosen by user)
> submit_confirmed_by_user      = yes | no
> run_id                        = ?
> episodes_completed            = ? / ?
> mean_success                  = ?
> reference_score               = ? (source)
> score_within_noise            = yes | no (if no: investigate)
> first_frames_checked          = yes | no
> action_range_checked          = yes | no
> connection_failure_seen       = yes | no
> (if yes:)
> fix_applied                   = ? (what changed)
> new_tag                       = ?
> ```

---

## Final checklist

Inputs

- [ ] Wrap passes `check_compatibility` and `verify` under
      `/wrap-remote-policy`
- [ ] Policy slug received from setup-manifold (not invented); image
      name constructed from it
- [ ] manifold-sdk version in the image matches the version the
      wrap was written against
- [ ] Endpoint URL env var identified from `profile.py`
- [ ] Tunable env vars listed (may be empty)
- [ ] Auth model recorded; if `header_token`, network restriction
      arranged on the endpoint before registration

Image contents

- [ ] Slim Python base image, pinned by digest, not by a floating
      tag like `latest`
- [ ] No CUDA base, no torch/jax, no model stack in the image
- [ ] manifold-sdk installed at the wrap's pinned version
- [ ] `httpx` installed at a pinned version
- [ ] Benchmark decoders installed if the benchmark ships encoded
      frames (for example `pillow` for JPEG)
- [ ] Only `.manifold/<slug>/` copied in; no project `src/`
- [ ] No credentials baked into the image (no `ENV MY_TOKEN=...`,
      no copied credential files)

Dockerfile rules

- [ ] Dockerfile builds without errors
- [ ] Container starts locally, prints the driver's startup log
      line (endpoint URL, cameras, state window) and the listening
      line, if the endpoint is reachable from the local box;
      skipped otherwise
- [ ] Port 8000, bound on `0.0.0.0`
- [ ] `PYTHONUNBUFFERED=1` set
- [ ] Bash present in the image

Push and register

- [ ] Tag never reused; Dockerfile and launcher saved before push
- [ ] Push confirmed by the user
- [ ] Registry package visible so the platform can pull the image
- [ ] Registration went through `register_policy_version` (MCP
      tool), not the `manifold policy init` CLI, because the CLI
      does not carry `config.env`
- [ ] `config.env` has the endpoint URL, plus any tunable
      overrides the user asked for
- [ ] `minimum_gpu_memory_gb` is `0`
- [ ] User was offered a scored test run and given the choice

If the user asked for a test run:

- [ ] Benchmark chosen by the user; submit confirmed before running
- [ ] Test run completed; score compared to reference
- [ ] First frames and action ranges inspected
- [ ] Connection failures diagnosed against the endpoint (URL, up,
      allowlisted, codec) rather than assumed to be wrap bugs
- [ ] If a fix was applied: image re-tagged and pushed, new tag
      re-registered with the same `config.env`
