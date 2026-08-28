# manifold-skills

AI agent skills for the Manifold platform.

## Skills

| Skill | What it does |
|---|---|
| [`wrap-policy`](skills/wrap-policy/SKILL.md) | Wrap a researcher's policy for Manifold: write the driver, profile, and pairing files; pass `check_compatibility` and `verify`; prove it with a live `evaluate` run. Three phases: understand, design, implement. |
| [`containerize-wrap`](skills/containerize-wrap/SKILL.md) | Package a verified policy wrap as a container image, push it to a registry, register it on the platform, and validate it with a scored test run. |

## Installation

Teach Codex, Claude Code, Cursor, and 70+ other AI agents to work with Manifold.

```
npx skills add bifrostai/manifold-skills --all
```

## Structure

```
skills/
  <skill-name>/
    SKILL.md        # the skill definition (frontmatter + instructions)
```

Each `SKILL.md` has YAML frontmatter with `name`, `description`, and `compatibility` (prerequisites), followed by the procedural guide.

Skills produce structured **checkpoints** at each phase exit.
