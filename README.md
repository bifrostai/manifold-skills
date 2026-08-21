# manifold-skills

AI agent skills for the Manifold platform. Each skill guides an agent through a specific workflow against the `manifold-sdk`.

## Skills

| Skill | What it does |
|---|---|
| [`wrap-policy`](skills/wrap-policy/SKILL.md) | Wrap a researcher's policy checkpoint as a Manifold pairing module (`PROFILE` / `BENCHMARK` / `PIPELINE`), gate it with `check_compatibility` and `verify`, and prove it with a live driver run. |

## Usage

Install a skill into your agent's skill directory so it loads when the agent is asked to perform that workflow. Each skill is a single `SKILL.md` file — no dependencies beyond the SDK.

## Structure

```
skills/
  <skill-name>/
    SKILL.md        # the skill definition (frontmatter + instructions)
```

Each `SKILL.md` has YAML frontmatter with `name`, `description`, and `compatibility` (prerequisites), followed by the procedural guide.

Skills produce structured **checkpoints** at each phase exit. When a wrap fails in production, pull the checkpoints from the conversation to see where the agent's understanding diverged from reality.
