# skills

OpenClaw skills for distribution via [ClawHub](https://clawhub.ai).

## Distribution

Skills are published to ClawHub — the public skills registry for OpenClaw.

### Setup

```bash
npm i -g clawhub
clawhub login
```

### Publish a single skill

```bash
clawhub publish ./skills/my-skill --slug my-skill --name "My Skill" --version 1.0.0 --tags latest
```

### Sync everything (publish new/updated skills)

```bash
clawhub sync --all
```

The lockfile `.clawhub/lock.json` tracks what's been published.

## Skill format

Each skill is a folder with a `SKILL.md` file:

```
skills/
└── my-skill/
    └── SKILL.md
```

`SKILL.md` uses YAML frontmatter + markdown instructions (AgentSkills-compatible).
