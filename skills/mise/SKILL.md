---
name: mise
description: A polyglot tool version manager that replaces asdf, nvm, pyenv, rbenv, and more.
license: MIT
metadata:
  author: Grimfrost Mage
  version: "0.0.0"
---

# Mise Skill

You are an expert in using mise (https://mise.jdx.dev/), a polyglot tool version manager that replaces asdf, nvm, pyenv, rbenv, and more.

## Core Concepts

- Mise manages versions of development tools and languages.
- Uses `mise.toml` (preferred) and `.tool-versions` files to specify versions per project and child directories.
- Supports global config at `~/.config/mise/config.toml`

## Common Commands

### Installing Tools

```bash
# Install a specific version
mise install nodejs@20.10.0

# Install the latest version
mise install nodejs@latest

# Install all tools from .tool-versions
mise install

# Install without confirmation
mise install -y
```

### Using Tools (Temporary)

```bash
# Use a specific version for this shell session
mise use nodejs@20.10.0

# Use latest version
mise use nodejs@latest

# Set as local default (writes to .tool-versions)
mise use --local nodejs@20.10.0

# Set as global default (writes to ~/.config/mise/config.toml)
mise use --global nodejs@20.10.0
```

### Executing Commands

```bash
# Run a command with a specific tool version
mise exec nodejs@18 -- node --version

# Run with multiple tools
mise exec nodejs@18 python@3.11 -- ./script.js

# Run without specifying version (uses active version)
mise exec -- node --version
```

### Locating Tools

```bash
# Show where a tool is installed
mise where nodejs
mise where nodejs@20.10.0

# Show all tool paths
mise where --all

# Show bin path
mise bin-paths
```

### Listing and Status

```bash
# List installed tools
mise list

# List available versions
mise list-all nodejs

# Show current active tools
mise current

# Show outdated tools
mise outdated
```

## Shell Activation (CRITICAL)

Mise must have shell activation enabled to automatically switch tool versions when entering directories using a shell.

If a shell is not used, then invoke the tool using `mise exec` to ensure the correct version is used or through a task runner that supports mise
or the built in task command, `mise run`.

### Check if activated:
```bash
# Should show mise version if active
mise --version

# Check if hook is in shell config
grep -i mise ~/.bashrc ~/.zshrc ~/.config/fish/config.fish 2>/dev/null
```

### Enable Activation

**Bash:**
```bash
echo 'eval "$(mise activate bash)"' >> ~/.bashrc
source ~/.bashrc
```

**Zsh:**
```bash
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc
source ~/.zshrc
```

**Fish:**
```bash
echo 'mise activate fish | source' >> ~/.config/fish/config.fish
source ~/.config/fish/config.fish
```

### Immediate Activation (Current Shell)
```bash
eval "$(mise activate bash)"  # or zsh
```

## Best Practices

1. **Always check activation first** - Most issues are due to missing shell activation
2. **Use mise.toml** - Pin project-specific versions in version control
3. **Prefer mise exec** - For CI/CD or one-off commands, use exec instead of modifying global state
4. **Regular updates** - Run `mise self-update` and `mise install` regularly
5. **Check where** - Use `mise where` to debug path issues

## Troubleshooting

- Tool not found: Check `mise list` to see if installed
- Wrong version: Verify shell activation with `mise current`
- PATH issues: Compare `which node` vs `mise where node`
- Version not switching: Ensure `mise.toml` exists and shell is activated

## References

- [Mise Documentation](https://mise.jdx.dev/)
- [GitHub Repository](
- [Toml Tasks](https://mise.jdx.dev/tasks/toml-tasks.html)
- [Running Tasks](https://mise.jdx.dev/tasks/running-tasks.html)
- [Tools Overview](https://mise.jdx.dev/dev-tools/)
- [Shims Overview](https://mise.jdx.dev/dev-tools/shims.html)

## Backends

Backends are the supported tools and languages that mise can manage. Each backend has its own implementation for installing, switching, and managing versions of the tool.

These are the primary backends that cover a wide range of development tools and languages:

- [Backends](https://mise.jdx.dev/dev-tools/backends/)
- [Github backend](https://mise.jdx.dev/dev-tools/backends/github.html)
- [Go backend](https://mise.jdx.dev/dev-tools/backends/go.html)
- [Dotnet backend](https://mise.jdx.dev/dev-tools/backends/dotnet.html)
- [Cargo backend](https://mise.jdx.dev/dev-tools/backends/cargo.html)
- [Npm backend](https://mise.jdx.dev/dev-tools/backends/npm.html)
- [Aqua backend](https://mise.jdx.dev/dev-tools/backends/aqua.html)
- [Vfox backend](https://mise.jdx.dev/dev-tools/backends/vfox.html)