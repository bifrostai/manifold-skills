# manifold-skills

AI agent skills for the Manifold platform.

## Installation

Teach Codex, Claude Code, Cursor, and 70+ other AI agents to work with Manifold.

```
npx skills add bifrostai/manifold-skills --all
```

Then start one of two skills. Pick the one that matches where your policy
will run.

To serve the policy from your own machine, in Claude Code:

```
/serve-policy
```

To package the policy as a container image that runs on Manifold's
machines:

```
/setup-manifold
```

In Codex, start each skill with `$` instead of `/`, for example
`$serve-policy`.

## Workflow

### Serve from your own machine

Run `/serve-policy` from your policy project. Your agent reads the code
that runs your model, asks which benchmark to prepare for, and writes one
serving script. It serves the policy from your machine with
`manifold policy serve`, then runs it against the debug variant of that
benchmark.

### Package as a container image

Run `/setup-manifold` to begin. Your agent may ask you questions about your project.

```mermaid
flowchart TD
    setup["/setup-manifold<br/>Understand the user's project, workflow, and hardware"]
    branch{"Where will<br/>the model run?"}
    rwrap["/wrap-remote-policy<br/>Prepare a container image that forwards to the user's endpoint. Ensures compatibility, then sends off a live test run."]
    wrap["/wrap-policy<br/>Prepare the user's policy as a standalone container image. Ensures compatibility, then sends off a live test run (skipped when the current machine cannot run the model)."]
    rcont["/containerize-remote-wrap<br/>Build a small image that forwards to the user's endpoint."]
    cont["/containerize-wrap<br/>Build the image containing the model. Push it and register it."]
    run["Run evaluations"]

    setup --> branch
    branch -->|"Your own cloud"| rwrap
    branch -->|"Manifold cloud"| wrap
    rwrap --> rcont
    wrap --> cont
    rcont --> run
    cont --> run
```

## Skills

| Skill | What it does |
|---|---|
| [`serve-policy`](skills/serve-policy/SKILL.md) | Serve a policy from the user's own machine. Reads the policy codebase, asks which benchmark to prepare for, writes one script with the signature and a `predict` function, starts `manifold policy serve`, and runs the debug variant of the benchmark. Three phases: understand, clarify, execute. |
| [`setup-manifold`](skills/setup-manifold/SKILL.md) | Set up a project for Manifold: gather what the wrap and container steps will need, and write it to `.manifold/CONTEXT.md`. Asks first where the model runs (in the built container, or on the user's own inference server), then branches the follow-up questions and picks the next skill accordingly. Three phases: look at the project, interview, write CONTEXT.md. |
| [`wrap-policy`](skills/wrap-policy/SKILL.md) | Wrap a researcher's policy for Manifold when the model loads into the container: write the driver, profile, and pairing files; pass `check_compatibility` and `verify`; prove it with a live `evaluate` run (skipped, with the user's consent, on a machine that cannot run the model). Three phases: understand, design, implement. |
| [`containerize-wrap`](skills/containerize-wrap/SKILL.md) | Package a verified policy wrap as a container image, push it to a registry, and register it on the platform. Then ask the user whether to submit a scored test run. Four phases: understand, design, build & push, register & offer a test run. |
| [`wrap-remote-policy`](skills/wrap-remote-policy/SKILL.md) | Wrap a researcher's policy for Manifold when the model runs on the user's own inference server (Modal endpoint, private HTTPS box): write a driver to dial the server, plus the profile and pairing files; pass `check_compatibility` and `verify`; prove it with a live run against the endpoint. Three phases: understand, design, implement. Run `setup-manifold` first to write `CONTEXT.md`; this skill reads it. |
| [`containerize-remote-wrap`](skills/containerize-remote-wrap/SKILL.md) | Package a remote-endpoint wrap as a container image, push it to a registry, and register it with the endpoint URL in `config.env` and `minimum_gpu_memory_gb: 0`. Then ask the user whether to submit a scored test run. Four phases: understand, design, build & push, register & offer a test run. |

## Structure

```
skills/
  <skill-name>/
    SKILL.md        # the skill definition (frontmatter + instructions)
```

Each `SKILL.md` has YAML frontmatter with `name`, `description`, and `compatibility` (prerequisites), followed by the procedural guide.

Skills produce structured **checkpoints** at each phase exit.
