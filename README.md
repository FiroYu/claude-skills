# claude-skills

Public collection of [Claude Code](https://claude.com/claude-code) skills in the standard `SKILL.md` format.

## Skills

| Skill | Description |
| --- | --- |
| [limit-break · 限界突破](limit-break/) | Pursue outcomes you would not seriously attempt: raise speed, altitude, depth, or range, and let external evidence judge the result. 追求原本不会认真尝试的结果，以真实反馈裁决突破成败。 |

Each skill folder is self-contained. See [limit-break/README.md](limit-break/README.md) for the full introduction, method source, and method notes.

## Layout

Each folder is one skill:

```
<skill-name>/
├── README.md      # introduction, method source, and method notes
├── SKILL.md       # the skill itself, standard SKILL.md format
├── assets/        # banner images
└── references/    # optional files the skill loads on demand
```

## Install

Copy a skill folder into your Claude Code skills directory:

```bash
git clone https://github.com/FiroYu/claude-skills.git
cp -r claude-skills/<skill-name> ~/.claude/skills/
```

The skill loads automatically on the next session.

## Scope

Only general-purpose, self-authored skills live here. Business-specific and personal skills stay in a private repo.
