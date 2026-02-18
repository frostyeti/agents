# Cast Task Runner

Cast is a task runner and automation tool written in Go. It provides a declarative way to define and run tasks using a `castfile` configuration file.

## Installation

Cast can be installed from the GitHub repository:

```bash
go install github.com/frostyeti/cast@latest
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `cast` | Run the default task |
| `cast <task>` | Run a specific task |
| `cast run <task>` | Explicitly run a task |
| `cast list` or `cast ls` | List all available tasks |
| `cast -p <path>` | Specify project file or directory |
| `cast -c <context>` | Use a specific context |

## Configuration File

Cast looks for a configuration file named `castfile`, `.castfile`, `castfile.yaml`, or `castfile.yml` in the current directory.

### Basic Structure

```yaml
id: "@org/project-id"
name: "project-name"
description: "Project description"
version: "0.1.0"

env:
  VAR_NAME: "value"

dotenv:
  - ".env"

tasks:
  task-name:
    uses: bash
    run: |
      echo "Hello, World!"
```

## Task Definition

### Task Properties

| Property | Type | Description |
|----------|------|-------------|
| `uses` | string | Runtime: `bash`, `sh`, `pwsh`, `powershell`, `python`, `node`, `bun`, `deno`, `ruby`, `go`, `dotnet`, `ssh`, `scp`, `tmpl`, `shell` |
| `run` | string | The script/command to execute |
| `desc` | string | Brief description of the task |
| `help` | string | Detailed help message |
| `env` | object/array | Environment variables for the task |
| `dotenv` | array | .env files to load |
| `cwd` | string | Working directory |
| `needs` | array | Tasks that must run first (dependencies) |
| `timeout` | string | Timeout (e.g., `30s`, `2m`, `1h`) |
| `if` | boolean/string | Conditional execution (e.g., `os == 'windows'`) |
| `force` | boolean/string | Run even if dependencies fail |
| `hosts` | array | Remote hosts to run task on |
| `template` | boolean/string | Evaluate `run` as a template |
| `with` | object | Additional parameters (e.g., `script` for external files) |

### Example Tasks

```yaml
tasks:
  build:
    desc: "Build the project"
    uses: bash
    run: |
      npm run build

  test:
    desc: "Run tests"
    uses: bash
    run: npm test
    needs:
      - build
    timeout: "5m"

  deno-script:
    uses: deno
    run: |
      console.log('Hello from Deno!');

  external-script:
    uses: bash
    with:
      script: "./scripts/deploy.sh"

  conditional:
    uses: bash
    run: echo "Windows only"
    if: "os == 'windows'"

  remote-task:
    uses: ssh
    run: |
      echo "Running on remote host"
    hosts: ["server1"]
```

## Listing Tasks

```bash
cast list
cast ls
```

Output shows task names with descriptions.

## Running Tasks

```bash
cast                 # Run default task
cast build           # Run 'build' task
cast run build       # Explicit run command
cast build test      # Run multiple tasks
cast @project build  # Run task from workspace project
```

### CLI Flags

```bash
cast build -p ./path/to/castfile    # Specify project file
cast build -p ./myproject           # Specify project directory
cast build -c production            # Use context
cast build -e KEY=value             # Set environment variable
cast build -E .env.local            # Load dotenv file
cast build -- --arg1 --arg2         # Pass arguments to task
```

## Dependencies

Tasks can depend on other tasks using the `needs` property:

```yaml
tasks:
  install:
    uses: bash
    run: npm install

  build:
    uses: bash
    run: npm run build
    needs:
      - install

  deploy:
    uses: bash
    run: npm run deploy
    needs:
      - build
```

## Remote Execution

Define hosts in the inventory:

```yaml
inventory:
  hosts:
    server1:
      host: 192.168.1.100
      user: deploy
      tags: ["production"]

tasks:
  deploy:
    uses: ssh
    run: |
      systemctl restart myservice
    hosts: ["server1"]
```

## Environment Variables

Environment variables can be defined globally or per task.

Environment variables can also be loaded from `.env` files
using the `dotenv` property, which supports multiple files and optional files (with `?` suffix).

```yaml
env:
  GLOBAL_VAR: "value"
  SECRET_VAR:
    name: "API_KEY"
    value: "secret"
    secret: true
  PG_PASS: $(kpv ensure --name "PG_PASS" --size 32)

dotenv:
  - ".env"
  - ".env.local?" # Optional .env file, won't error if missing

tasks:
  task:
    uses: bash
    run: echo $GLOBAL_VAR && echo $SECRET_VAR && echo $PG_PASS
    env:
      LOCAL_VAR: "local"
```

## When to Use Cast vs Alternatives

### Prefer Bash for Simple Tasks

For simple, single-command tasks, prefer using bash directly:

```bash
npm run build
npm test
```

Even on Windows, bash is available through Git for Windows installation.

### Prefer Cast When

- Multiple related tasks need coordination
- Tasks have dependencies
- Need declarative configuration
- Working with remote hosts
- Need cross-platform task definitions
- Tasks require specific runtimes (deno, bun, etc.)

### For Complex Scripting

**Prefer Deno** for complex tasks because:
- Single binary install (no package.json required)
- Can reference npm/jsr modules with versions in-script
- TypeScript support out of the box
- Secure by default with permission flags

```yaml
tasks:
  complex:
    uses: deno
    run: |
      import { parse } from "https://deno.land/std@0.208.0/flags/mod.ts";
      const args = parse(Deno.args);
      console.log(args);
```

**Bun** is also a good option:
- Single binary install
- Fast execution
- npm ecosystem compatibility

```yaml
tasks:
  complex:
    uses: bun
    run: |
      console.log("Hello from Bun");
```

## JSON Schemas

JSON schemas are available for IDE validation:
- `castfile.schema.json` - Main configuration schema
- `cast.module.schema.json` - Module schema for imports

## CAST_ Environment Variables

### CLI Configuration

| Variable | Description |
|----------|-------------|
| `CAST_PROJECT` | Default project file path (used by `-p` flag) |
| `CAST_CONTEXT` | Default context name (used by `-c` flag) |

### XDG Directories (set globally)

| Variable | Description |
|----------|-------------|
| `CAST_XDG_DATA_HOME` | XDG data directory (defaults to `~/.local/share`) |
| `CAST_XDG_CONFIG_HOME` | XDG config directory (defaults to `~/.config`) |
| `CAST_XDG_CACHE_HOME` | XDG cache directory (defaults to `~/.cache`) |
| `CAST_XDG_BIN_HOME` | XDG bin directory (defaults to `~/.local/bin`) |

### Task Runtime (available within tasks)

| Variable | Description |
|----------|-------------|
| `CAST_ENV` | Path to temp file for sharing environment between tasks |
| `CAST_PATH` | Path to temp file for sharing PATH entries between tasks |
| `CAST_OUTPUTS` | Path to temp file for sharing outputs between tasks |

### Module Tasks (set on tasks from imported modules)

| Variable | Description |
|----------|-------------|
| `CAST_FILE` | Path to the module's castfile |
| `CAST_DIR` | Directory containing the module's castfile |
| `CAST_PARENT_FILE` | Path to the parent project's castfile |
| `CAST_PARENT_DIR` | Directory containing the parent project's castfile |
| `CAST_MODULE_ID` | Module's unique identifier |
| `CAST_MODULE_NAME` | Module's name |
| `CAST_MODULE_VERSION` | Module's version |
| `CAST_MODULE_DESCRIPTION` | Module's description |

### Using CAST_ENV for Task Communication

Tasks can share environment variables by writing to `CAST_ENV`:

```yaml
tasks:
  load-env:
    uses: bash
    run: |
      API_KEY="$(kpv ensure --name "API_KEY" --size 32)"
      echo "API_KEY=$API_KEY" >> $CAST_ENV

  get-env:
    uses: bash
    run: |
      echo "API_KEY from env: $API_KEY"
    needs:
      - load-env
```

### Using CAST_PATH for PATH Updates

Tasks can add directories to PATH for subsequent tasks:

```yaml
tasks:
  setup-path:
    uses: bash
    run: |
      echo "/custom/bin" >> $CAST_PATH

  use-custom-tool:
    uses: bash
    run: custom-tool --version
    needs:
      - setup-path
```
