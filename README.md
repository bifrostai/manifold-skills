# manifold-skills

AI agent skills for the Manifold platform.

## Installation

Teach Codex, Claude Code, Cursor, and 70+ other AI agents to work with Manifold.

```
npx skills add bifrostai/manifold-skills --all
```

We recommend serving your policy directly on a GPU machine you own. In Claude Code:

```
/serve-policy
```

In Codex:

```
$serve-policy
```

## Workflow

Pick the path that matches where your policy runs. We recommend
self-hosting.

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 40}}}%%
flowchart TD
    start{"Where does<br/>your policy run?"}
    self["<b>Self‑hosted&nbsp;policy&nbsp;(recommended)</b><br/>Serve your policy locally,<br/>on a cloud server, or<br/>serverless deployment"]
    cloud["<b>Manifold cloud</b><br/>Containerize your<br/>policy to run on<br/>Manifold servers"]
    serve["/serve-policy<br/>Writes a single script<br/>that serves your policy<br/>where it is"]
    cont["/containerize-policy<br/>Records the project's context, wraps the policy and tests it, then builds the image, pushes it, and registers it"]
    run["Run evaluations"]

    start --> self --> serve --> run
    start --> cloud --> cont --> run
```

### Self-hosted policy

Run `/serve-policy` inside your policy project. Your agent reads your model
code and asks which benchmark to evaluate on. It writes a short Python
script that serves your policy from your own machine. Then it starts the
script with `manifold policy serve`.

To check the setup, your agent runs the debug version of your benchmark. A
debug benchmark sends the same data as the full benchmark, with fewer tasks
or episodes, so you get a first score quickly.

### Manifold cloud

Choose this path to run your policy on Manifold's GPUs. Run
`/containerize-policy` inside your policy project. Your agent asks about your
project and hardware. It writes a Python wrapper around your model and tests
it. Then it builds a container image with your model and registers it with
Manifold. Your agent asks before each step that pushes, registers, or spends
cloud time.

## Skills

| Skill | When to use it |
|---|---|
| [`serve-policy`](skills/serve-policy/SKILL.md) | Your policy runs locally, on a cloud server, or on a serverless deployment. Your agent writes a script that serves it from there. |
| [`containerize-policy`](skills/containerize-policy/SKILL.md) | Your policy should run on Manifold's GPUs. Your agent wraps your model, builds a container image, and registers it with Manifold. You need a container registry. |

## Structure

```
skills/
  <skill-name>/
    SKILL.md        # the skill definition (frontmatter + instructions)
```

Each `SKILL.md` has YAML frontmatter with `name`, `description`, and `compatibility` (prerequisites), followed by the procedural guide.

Skills produce structured **checkpoints** at each phase exit.
