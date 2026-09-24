# Serve from Modal

Read this file together with the "Serve from a cloud container" section
of the `serve-policy` skill, in
[`skills/serve-policy/SKILL.md`](../SKILL.md). That section lists the
parts that the container needs. This
file maps those parts to Modal. The skeleton below follows the Modal
docs as of 2026-09-24. If the installed Modal version rejects a call,
read the Modal docs for that call.

## Check the CLI

Run `modal --version` and `modal profile current`. If `modal --version`
fails, ask the user to install the Modal CLI with `uv tool install modal`.
uv installs it in its own environment, so the project's dependencies stay
the same. The app file imports only `modal` and the Python standard
library, so the CLI can run it from that environment. If
`modal profile current` fails, ask the user to log in with `modal setup`.

## Write the app

Write `.manifold/modal_<policy>.py` from this skeleton. Fill in each
`<...>`, and change nothing else unless the project needs it. If the
project has a Modal app, copy its system packages and build steps into
the image.

```python
import os
import subprocess
from pathlib import Path

import modal

PROJECT = Path(__file__).resolve().parent.parent
REMOTE_PROJECT = "/root/project"
SERVE_COMMAND = [
    "manifold", "policy", "serve", "<policy>:<version>",
    "--", "uv", "run", ".manifold/serve_<policy>_<benchmark>.py",
]

app = modal.App("manifold-<policy>")

image = (
    modal.Image.debian_slim(python_version="<project python version>")
    .apt_install("git")
    .pip_install("uv")
    .run_commands(
        "uv tool install manifold-cli --index "
        "https://bifrost-manifold-releases.s3.us-west-2.amazonaws.com/simple/"
    )
    .env({"PATH": "/root/.local/bin:/usr/local/bin:/usr/bin:/bin"})
    .add_local_dir(
        PROJECT, REMOTE_PROJECT, copy=True, ignore=[".venv", ".git"]
    )
    .workdir(REMOTE_PROJECT)
    .run_commands("uv sync --frozen")
)

weights = modal.Volume.from_name("manifold-<policy>-weights", create_if_missing=True)
login = modal.Volume.from_name("manifold-<policy>-login", create_if_missing=True)

deployment = {
    name: os.environ.get(name)
    for name in ("MANIFOLD_API_URL", "MANIFOLD_WEBAPP_URL")
}


@app.function(
    image=image,
    gpu="<gpu>",
    timeout=24 * 60 * 60,
    volumes={"/cache": weights, "/manifold": login},
    secrets=[
        modal.Secret.from_dict(
            {"MANIFOLD_CONFIG_DIR": "/manifold", "HF_HOME": "/cache/huggingface", **deployment}
        )
    ],
)
def serve() -> None:
    subprocess.run(["hostname"], check=True)
    status = subprocess.run(
        ["manifold", "auth", "status"], capture_output=True, text=True, check=True
    )
    if not status.stdout.startswith("Logged in as"):
        subprocess.run(["manifold", "auth", "login"], check=True)
        login.commit()
    subprocess.run(SERVE_COMMAND, check=True)
```

Notes on the skeleton:

- `modal run` runs `serve` because the app has a single function.
- `copy=True` bakes the project into the image. Modal allows build
  steps after `add_local_dir` only with `copy=True`, and `uv sync` is a
  build step. `ignore` keeps the local `.venv` and `.git` out of the
  image.
- `uv sync --frozen` installs the dependencies from `uv.lock`. If the
  project uses another package manager, replace this step and the
  `uv run` in `SERVE_COMMAND` with that manager's commands.
- `uv tool install` puts the `manifold` command in `/root/.local/bin`.
  The `PATH` setting makes it available to the function.
- `HF_HOME` points Hugging Face downloads at the weights volume. If the
  project caches weights somewhere else, set that project's cache
  variable instead.
- `Secret.from_dict` drops a key with the value `None`. So the two
  deployment URLs reach the container only if they are set on this
  machine.
- `timeout` takes seconds. Modal stops a function after 300 seconds by
  default, and allows at most 24 hours.
- Modal commits volume changes every few seconds, and again when the
  container shuts down. `login.commit()` saves the login at once. If two
  containers write the same file, Modal keeps the last write, so run a
  single container with the login volume.
- `gpu=` takes a Modal GPU name, such as `"A100"` or `"H100"`.

## Start and stop

Start the app with `modal run .manifold/modal_<policy>.py`. `modal run`
streams the container's output to the local terminal, so
`.manifold/cloud.log` receives the login link and the serve output. Do
not pass `--detach`. With `--detach`, the app keeps running after
`modal run` exits, and Modal keeps billing the GPU.

Stop the app with SIGINT to `modal run`, which Ctrl-C also sends.
Modal stops an app that `modal run` started when `modal run` exits. Then check `manifold runner list`
as the "Stop serving" step in `skills/serve-policy/SKILL.md` describes.

## If the app dies during a run

Tell the three cases apart with `.manifold/cloud.log` and
`manifold run get <run-id>`.

1. **The script fails, and the container stays up.** The log shows a
   traceback from the script, such as an error in `predict` or CUDA out
   of memory. The runner reports the failure, and the run fails. Fix the
   script, then serve again under a new version, as the Troubleshooting
   section in `skills/serve-policy/SKILL.md` describes.
2. **The container or the app dies.** `modal run` exits although you did
   not stop it, or the log stops. The log shows no traceback from the
   script. The usual causes are the function's `timeout`, a crash of the
   container, and a closed `modal run` process. Manifold stops receiving
   heartbeats from the policy side. After about 90 seconds, Manifold
   puts the run back in the queue and deletes the episodes that the run
   recorded. `manifold run get` then shows the run as `queued`. Do these
   steps in order:
   1. Revoke the runner `modal`, as the "Stop serving" step in
      `skills/serve-policy/SKILL.md` describes. For about 5
      minutes after the container dies, Manifold can still place the run
      on the dead runner. The run then waits until that placement lapses.
   2. Start the app again with the same version. The queued run waits
      for that version. With a new version, the run stays queued.
   3. Tell the user that the run starts again from its first episode.
   4. If the `timeout` ended the container, raise it, or ask the user to
      run a smaller benchmark. Otherwise the container dies again at the
      same point.
3. **You stopped the app with Ctrl-C.** This is the normal stop, which
   "Start and stop" describes.
