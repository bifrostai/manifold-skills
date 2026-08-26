---
name: containerize-wrap
description: >
  Package a verified policy wrap as a container image for the Manifold platform,
  then build it, push it to a container registry, register it, and run it. Use
  when asked to "containerize my policy", "build the policy image", "push my
  wrap to ghcr", "register my policy on the platform", or "get my wrap running
  on the platform".
compatibility: >
  Requires docker and a push credential for the target container registry (ghcr
  by default). Requires the manifold CLI (manifold auth login). Input is a
  wrap that passes check_compatibility and verify under the wrap-policy skill.
---

## Summary

Take the output of `wrap-policy` and make it runnable on the Manifold platform.

That output is a Python module that exports `PROFILE`, `BENCHMARK`, and
`PIPELINE`. You load it with `manifold.recipes.read_pairing`.

The result is a container image on a registry, a catalog entry pointing at that
image, and a scored test run against a registered benchmark.

This skill does not do wrap authoring. If the wrap does not pass both
`check_compatibility` and `verify`, go back to the `wrap-policy` skill first.

**Why a container?** The platform runs policies as registered container images.
When a run is dispatched, the image is pulled, started, and the server inside
must accept connections before the run proceeds.

Three rules:

1. **One image per wrap.** The `CMD` in the Dockerfile specifies which wrap
   the container runs. Registration cannot override the command.
2. **Tags become versions.** When you run `manifold policy init`, the image
   tag becomes the version string. Never reuse a tag.
3. **Test before merge.** The full cycle (build, push, register, run, score)
   finishes before any code is merged. Merging is the release, not the test.

---

## Phase 0: prerequisites and naming

**Input:** one wrap (profile, driver, pipeline) that passes both
`check_compatibility` and `verify` under the `wrap-policy` skill.

**Ask the user which container registry to use.** Default to `ghcr.io`, but the
user may prefer another registry (Docker Hub, AWS ECR, GCP Artifact Registry,
etc.). Confirm the registry and the repository namespace before building. The
rest of this skill uses `ghcr.io/<your-org>` in examples. Substitute the
user's actual registry and namespace.

**Image name:** `<registry>/<namespace>/policy-<family>-<pairing>:<tag>`

For example: `ghcr.io/<your-org>/policy-pi05-libero:0.1.0`

The `<tag>` is the version. Bump it for every change to the wrap or the
Dockerfile. Old tags stay immutable.

**SDK version:** if a benchmark image exists for the target suite, the policy
image and the benchmark image must install the same SDK version. The action
format and the message format between the two sides come from the SDK. A
version mismatch can break them silently.

> **Phase 0 checkpoint:**
> ```
> wrap_module        = ?
> checks_pass           = yes (link to wrap-policy handoff)
> registry              = ghcr.io (or user's choice)
> image_name            = <registry>/<namespace>/policy-<family>-<pairing>:<tag>
> sdk_version           = ?
> ```

---

## Phase 1: choose the Dockerfile shape

There are two shapes. The choice depends on where the model stack's Python
environment comes from.

### Shape A: SDK first

Use this when the model stack installs from a package index (PyPI, a private
index). Build the SDK's environment first, then add the model dependencies on
top.

This is simpler. The SDK's `uv.lock` pins the whole environment. Model
dependencies add to it.

The SDK repo's `examples/policies/smoke/Dockerfile` follows this shape.

### Shape B: upstream first

Use this when the model stack has its own lock file or conda environment and is
not on PyPI (a research repo with its own `uv` workspace, for example).

Build the model stack's environment first, pinned at a specific commit with a
build `ARG`. Then install the SDK into that environment. Bumping the upstream
commit is a separate decision from bumping the SDK version.

The SDK repo's `examples/policies/pi05/Dockerfile` follows this shape.

### Writing the Dockerfile

Regardless of shape, the following apply:

**Base image.** If a benchmark image exists for the target suite, use the same
base image. Pin it by digest when possible. When a runner box runs both sides of
a pair, shared base layers save disk and pull time.

**Port 8000.** The container must listen on port 8000. The constant
`DEFAULT_CONTAINER_PORT` is set to 8000 in the runner code. Registration cannot
change it. Bind `0.0.0.0:8000` in the `CMD`.

**Bash is required.** The readiness probe runs
`bash -c 'exec 3<>/dev/tcp/127.0.0.1/8000'` inside the container. If the base
image does not have bash, install it.

**`PYTHONUNBUFFERED=1`.** Set this in the Dockerfile. Without it, a crashing
container's last output lines stay in a write buffer and are lost. Container
output is used to diagnose failures.

**Layer order.** Put the model stack's large dependencies (torch, jax, CUDA
wheels) in early layers. Put the SDK and your wrap code in later layers. That
way, editing the wrap rebuilds only the cheap layers at the end.

**Pin the model stack's dependencies.** The SDK declares dependency floors with
no ceilings. A fresh dependency resolution inside a rebuild can upgrade a
package the model stack cannot use. Pin the model stack's `numpy`, `scipy`, and
similar packages in the image to prevent this.

**No network fetches at model construction.** If the model's loader downloads
anything at construction time (pretrained weights, tokenizer files), disable
that or copy the files into the image at build time. A container in an offline
environment will fail at load otherwise.

**GPU memory pre-allocation.** Some frameworks claim all GPU memory at startup.
JAX does this by default, for example. Disable that behavior in the Dockerfile
with the framework's environment variable. Otherwise, on a shared box, all GPU
memory is consumed before the benchmark renderer starts.

### How to get your wrap into the image

The SDK's example server (`examples/policies/serve.py`) uses
`importlib.import_module(f".{name}", __package__)` to load wraps. This
import is relative to the `examples.policies` package. If your code is somewhere
else, it will not be found.

Most wraps are not inside the SDK's `examples/policies/` directory. For those,
do not use the example server. Instead:

1. `COPY` your wrap (the module with `PROFILE`, `BENCHMARK`, `PIPELINE`, plus
   `profile.py` and `driver.py`) into the `WORKDIR`.
2. Write a small launcher script (the same one from `wrap-policy` Phase 6):

```python
"""Serve a wrap that is not inside the SDK examples."""
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

3. `COPY` the launcher into the image and make it the `CMD`:

```dockerfile
CMD ["python", "serve.py", "--pairing", "my_wrap.franka_libero", \
     "--host", "0.0.0.0", "--port", "8000"]
```

If your wrap has been added to the SDK's `examples/policies/` directory, you can
use the SDK's own `serve.py` as shown in the smoke and pi05 Dockerfiles.

### Weights: baked or fetched?

**Fetched at first load** (what the SDK examples do): the profile's
`default_weights` contains a hub id or path. Set the model stack's cache
environment variable to point at `/opt/manifold/cache`. If the runner was
registered with `--cache-dir`, a host directory is mounted at that path inside
the container. The download happens once per box and is reused across runs.
Without that mount, every run re-downloads.

A large checkpoint may not download fast enough before the 10-minute readiness
timeout. Warm the cache on each box by hand before the first run.

**Baked into the image**: copy the checkpoint files into the image at build time.
Only copy what inference needs. Hub snapshots often include optimizer states,
training configs, and alternate checkpoints that add size but are not used.
Baking trades larger image size and slower pushes for no cache warming step.
Prefer this only for small checkpoints.

Record which approach you chose and why.

> **Phase 1 checkpoint:**
> ```
> dockerfile_shape      = SDK first | upstream first
> base_image            = ?
> wrap_location          = inside SDK examples | outside (custom launcher)
> weights_approach      = fetched (cache) | baked
> env_vars_set          = [PYTHONUNBUFFERED=1, ...]
> port                  = 8000
> bash_in_image         = yes
> ```

---

## Phase 2: build and test locally

### Build

```sh
docker build -f <path-to-Dockerfile> \
    -t <registry>/<namespace>/policy-<family>-<pairing>:<tag> .
```

`docker build` exiting 0 does not mean the server works. It only means the
image assembled without errors. The server has not run yet.

### Test the container

Run the image and confirm the server starts and prints its listening line:

```sh
docker run --gpus all <image>:<tag>
```

Look for the line: `listening on 0.0.0.0:8000 (TCP), up to 8 concurrent shard(s)`

Dependency conflicts, import errors, and model loading failures will show up
here. None of them appear during the build step.

For a stronger check, point the stand-in benchmark runner from `wrap-policy`
Phase 6 at the containerized server and run one episode:

```sh
# In one terminal:
docker run --gpus all -p 8000:8000 <image>:<tag>

# In another terminal:
python -m <your_runner> --server 127.0.0.1:8000 --episodes 1 --max-steps 12
```

If this works, the containerized server is handling the same protocol exchange
that happens during a platform run.

**Commit the Dockerfile and wrap source before pushing the image.** A
published tag whose source cannot be found in git cannot be audited or rebuilt.

> **Phase 2 checkpoint:**
> ```
> build_exit_code       = 0
> listening_line        = yes | no
> stand_in_episode      = ran | skipped (reason)
> source_committed      = yes (commit hash)
> ```

---

## Phase 3: push to the container registry

### Authenticate

For ghcr:
```sh
docker login ghcr.io
```

For other registries, use the appropriate login command.

### Push

```sh
docker push <registry>/<namespace>/policy-<family>-<pairing>:<tag>
```

### ghcr visibility (ghcr only, first push of a new package name)

A new ghcr package starts private. While private, the image cannot be pulled,
and runs will fail. The failure message does not say "permission denied". It
looks like a scheduling failure or a pull failure.

To make the package visible, go to the GitHub package settings page:
`https://github.com/orgs/<org>/packages/container/<package>/settings`

Change the visibility before testing any runs against this image.

For other registries, make sure the runner boxes can pull the image. The
registry must be reachable and the credentials must be configured.

> **Phase 3 checkpoint:**
> ```
> push_exit_code        = 0
> registry              = ?
> visibility            = public | private (action needed)
> ```

---

## Phase 4: register on the platform

Registration creates a catalog entry that points at the image. The CLI creates
the policy entity (if it does not exist) and adds a version in one step.

```sh
manifold policy init <slug> \
  --image <registry>/<namespace>/policy-<family>-<pairing>:<tag> \
  --minimum-gpu-memory-gb <N>
```

The image tag becomes the version string. So
`--image ghcr.io/<your-org>/policy-pi05-libero:0.1.0` registers version `0.1.0`.

### GPU memory

`--minimum-gpu-memory-gb` controls two things: which runners can accept this
image, and whether the container gets GPU access.

The scheduler filters runners by their reported GPU memory.

`--gpus all` is only added to the `docker run` command when the version's config
has `"gpu": true`. That flag is set only when `minimum_gpu_memory_gb` is greater
than 0.

**If this value is 0, the container starts without GPU access.** A GPU model
registered with 0 will run very slowly on CPU or crash at model load.

Set `<N>` to the model's actual GPU memory footprint, rounded up.

### Visibility

By default, a new policy is visible to its organization. Add `--visibility
public` if other organizations need to run against it.

> **Phase 4 checkpoint:**
> ```
> slug                  = ?
> version               = ? (from image tag)
> minimum_gpu_memory_gb = ?
> gpu_config            = true (confirm > 0 for GPU models)
> ```

---

## Phase 5: submit a test run

Pair the registered policy against a registered benchmark that you know works:

```sh
manifold run submit <policy-slug> <benchmark-slug>
```

Then watch it:

```sh
manifold run watch <run-id>
```

### Before submitting

- The benchmark you name must be registered and must have a working image. If
  you do not have a working benchmark to test against, you cannot validate the
  image on the platform yet.
- Runs are visible to everyone in the organization. Use a clear `--name` if you
  want it labeled.
- Start with one shard (no `--shards` flag, or `--shards 1`). Add shards only
  after a single shard run has completed.
- There is no hardware check before scheduling. A benchmark that needs a
  specific GPU will be scheduled onto a box without one, and fail at runtime.

> **Phase 5 checkpoint:**
> ```
> run_id                = ?
> policy_slug           = ?
> benchmark_slug        = ?
> shards                = 1
> status                = queued | dispatched | running | completed | failed
> ```

---

## Phase 6: validate the results

A completed run does not mean the wrap is correct. Check the actual output.

### Check episode completion

All episodes should reach `completed` status:

```sh
manifold run get <run-id> --episodes
```

### Inspect the results

If you have access to the results files
(`/tmp/results/results/<benchmark>.json` inside the benchmark container, or the
output directory on the runner box), examine them:

- **First frames.** Dump the first observation frame from each episode. A run
  once scored 0.4 in a scene with no background. A mostly black first frame
  means the scene did not load.
- **Action magnitudes.** Actions should be in the physical range for the
  embodiment (millimeters, radians). Actions stuck at the edges (all 1.0 or all
  -1.0) suggest missing denormalization.

### Compare against a reference score

Compare the run's `mean_success` against the checkpoint's published or
previously measured score on this suite.

The score should be within noise of the reference. **This is the only point in
the entire flow where convention bugs become visible.** A wrong gripper sign, a
wrong component order, a wrong rotation encoding, or a wrong proprioception
mapping all pass `check_compatibility` and `verify`. They only show up here, as
a score near zero.

A near zero score on a suite the checkpoint is known to handle means a wrap bug
until you prove otherwise. Go back to `wrap-policy` Phase 4, fix the pipeline or
session, rebuild the image with a bumped tag, and re-enter this skill at
Phase 1.

> **Phase 6 checkpoint:**
> ```
> episodes_completed    = ? / ?
> mean_success          = ?
> reference_score       = ? (source)
> score_within_noise    = yes | no (if no: investigate)
> first_frames_checked  = yes | no
> action_range_checked  = yes | no
> ```

---

## Phase 7: merge

When the test run validates:

- The image is live and registered. Nothing more to deploy.
- Merge the Dockerfile and wrap source into the main branch.

The merge makes the source for the tested tag findable in git. The image was the
test artifact. Merging is the release.

---

## Runner box prep (once per box)

When a new wrap runs on a box for the first time, someone must prepare the
box by hand.

1. **`docker login`** on the box, so the runner daemon can pull from the
   registry.

2. **Register the runner with `--cache-dir`**, so downloaded weights persist
   across runs:
   ```sh
   manifold runner register <name> --cache-dir /path/to/cache --gpu-device 0
   ```

3. **Warm the cache by hand** for large checkpoints. Run the image once:
   ```sh
   docker run --gpus all -v /path/to/cache:/opt/manifold/cache <image>:<tag>
   ```
   Wait for the model to download and load. Then stop the container. The
   checkpoint is now in the host cache directory and will be mounted into future
   runs.

---

## Final checklist

- [ ] Wrap passes `check_compatibility` and `verify` under `wrap-policy`
- [ ] Dockerfile builds without errors
- [ ] Container starts and prints its listening line
- [ ] Port 8000, bound on `0.0.0.0`
- [ ] `PYTHONUNBUFFERED=1` set
- [ ] Bash present in the image (readiness probe needs it)
- [ ] Model stack dependency versions pinned in the image
- [ ] No network fetch at model construction time (or files copied into image)
- [ ] GPU pre-allocation disabled for shared boxes
- [ ] Weights approach decided and documented (baked or cache)
- [ ] Tag never reused; Dockerfile and source committed before push
- [ ] Registry package visible to runner boxes
- [ ] Registered with correct `--minimum-gpu-memory-gb` (never 0 for GPU models)
- [ ] Single shard test run completed; score compared to reference
- [ ] First frames and action ranges inspected
- [ ] Runner boxes prepped (docker login, cache dir, cache warmed)
