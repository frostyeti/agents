# Skills

This directory contains skills for agents. Skills are reusable capabilities that agents can use to perform tasks.

## Available Skills

| Skill | Description |
|-------|-------------|
| [cast](./cast/SKILL.md) | Task runner and automation tool with declarative configuration |
| [golang](./golang/SKILL.md) | Best practices for writing production Go code (Google, Uber style guides) |
| [kpv](./kpv/SKILL.md) | KeePass vault CLI for secrets management with automation |
| [mise](./mise/SKILL.md) | Polyglot tool version manager (replaces asdf, nvm, pyenv, rbenv) |
| [osv](./osv/SKILL.md) | OS vault CLI for Mac Keychain, Linux keyring, Windows Credential Manager |

## Structure

Each skill follows the [Agent Skills specification](https://agentskills.io/specification):

```
skill-name/
└── SKILL.md          # Required: YAML frontmatter + Markdown instructions
├── scripts/          # Optional: Executable scripts
├── references/       # Optional: Additional documentation
└── assets/           # Optional: Static resources (schemas, templates)
```

## Adding a New Skill

1. Create a new directory for your skill: `skills/<skill-name>/`
2. Add a `SKILL.md` file with YAML frontmatter and instructions
3. Include optional supporting files as needed

See the [Agent Skills specification](https://agentskills.io/specification) for details.
