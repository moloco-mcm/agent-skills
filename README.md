# moloco-mcm/agent-skills

Claude Code skills for MCM partner integrations.

## What's inside

A marketplace and plugin host. Each skill is packaged as its own plugin under `plugins/<skill-name>/` so partners install only what they need.

## Available skills

| Skill | Description |
| --- | --- |
| [`ad-preview-template`](plugins/ad-preview-template) | Generate valid HTML ad preview templates for MCM Sponsored Display and Sponsored Brands. |

## Install

### Claude Code

```bash
/plugin marketplace add moloco-mcm/agent-skills
/plugin install ad-preview-template@agent-skills
```

The skill is auto-detected on the next session.

### Other agents (Cursor, Copilot, etc.)

Reference the relevant `SKILL.md` directly:

- [`plugins/ad-preview-template/skills/ad-preview-template/SKILL.md`](plugins/ad-preview-template/skills/ad-preview-template/SKILL.md)

## Repository layout

```
agent-skills/
├── .claude-plugin/
│   └── marketplace.json         # marketplace catalog
├── plugins/
│   └── ad-preview-template/     # one plugin per skill
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
│           └── ad-preview-template/
│               └── SKILL.md
├── README.md
└── LICENSE
```

## Feedback

Open an issue or contact your Moloco account representative.

## License

Apache-2.0
