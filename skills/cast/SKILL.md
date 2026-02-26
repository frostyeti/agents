# Cast Task Runner

Cast is a task runner and automation tool written in Go. It provides a declarative way to define and run tasks, workflows, and remote execution using a `castfile` configuration file.

## Installation

Cast can be installed using official scripts:

### Linux / macOS
```bash
curl -sL https://raw.githubusercontent.com/frostyeti/cast/master/eng/scripts/install.sh | bash
```

### Windows (PowerShell)
```powershell
irm https://raw.githubusercontent.com/frostyeti/cast/master/eng/scripts/install.ps1 | iex
```

*By default, Cast installs to `~/.local/bin` (Linux/Mac) or `~/AppData/Local/Programs/bin` (Windows). Override this by setting the `CAST_INSTALL_DIR` environment variable before running.*

## Quick Reference

| Command | Description |
|---------|-------------|
| `cast` | Run the default task |
| `cast <task>` | Run a specific task |
| `cast run <task>` | Explicitly run a task |
| `cast list` or `cast ls` | List all available tasks |
| `cast -p <path>` | Specify project file or directory |
| `cast -c <context>` | Use a specific context |
| `cast @workspace <task>` | Run a task from a specific workspace child project |
| `cast run --job <job>` | Run a job and all its downstream jobs |
| `cast exec -- <cmd>` | Ad-hoc execution wrapped in the Castfile's environment |

## Autocompletion
Cast supports shell autocompletion for tasks, jobs, and workspace projects!

**Bash:** `echo 'source <(cast completion bash)' >> ~/.bashrc`
**Zsh:** `echo 'source <(cast completion zsh)' >> ~/.zshrc`
**PowerShell:** `Invoke-Expression (&cast completion powershell)`

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
  # Support for command substitution
  DB_PASS: $(aws secretsmanager get-secret-value --secret-id pass)

dotenv:
  - path: ".env"
  - path: ".env.production"
    contexts: [prod]

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
| `uses` | string | Runtime: `bash`, `python`, `node`, `deno`, `go`, `ssh`, `docker://image:tag`, or relative path `./script.sh` |
| `run` | string | The script/command to execute |
| `desc` | string | Brief description of the task |
| `env` | object/array | Environment variables for the task |
| `dotenv` | array | .env files to load specific to the task |
| `cwd` | string | Working directory |
| `needs` | array | Tasks that must run first (dependencies) |
| `hooks` | object | `{ before: [task], after: [task] }` execution hooks |
| `timeout` | string | Timeout (e.g., `30s`, `2m`, `1h`) |
| `if` | string | Conditional execution (e.g., `env.BRANCH == 'main'`) evaluated via the 'expr' library |
| `force` | string | Run even if dependencies fail |
| `hosts` | array | Remote hosts to run task on |
| `template` | string | Render text dynamically |
| `with` | object | Additional parameters |

### Example Tasks

```yaml
tasks:
  build:
    desc: "Build the project"
    uses: bash
    run: npm run build

  test-in-docker:
    uses: docker://golang:1.21
    run: go test ./...

  remote-deploy:
    hosts: [web-server]
    uses: ssh
    # Go text templates evaluate locally before remote execution!
    run: |
      echo "Deploying to {{ .Host.Host }} as {{ .Host.User }}"
      cd /var/www/app
      docker-compose up -d

  conditional:
    cwd: ./build
    uses: bash
    run: npm publish
    if: env.BRANCH == 'main'
```

## Jobs

Jobs group tasks into complex CI/CD-style pipelines.

```yaml
jobs:
  build-all:
    steps:
      - run: build
  
  deploy-all:
    needs: [build-all]
    steps:
      - run: deploy
```

Run a job (and downstream): `cast run --job build-all`

## Contexts

Contexts dynamically route tasks and configuration. 

```yaml
tasks:
  "deploy:prod":
    run: echo "Production Deploy"
  
  "deploy:dev":
    run: echo "Dev Deploy"
```
Command `cast run deploy -c prod` will automatically route to `deploy:prod`.

## Remote Workloads & Modularity

### Standalone Inventories
Load remote hosts from separate yaml files:
```yaml
inventories:
  - ./production.yaml
  - ./staging.yaml
```

### Remote Git Imports
Import tasks from Git repositories directly:
```yaml
imports:
  - from: github.com/frostyeti/shared-tasks
    ns: core

tasks:
  deploy:
    needs: [core:build]
    run: echo "Deploying..."
```

### Workspaces
Manage sub-projects efficiently without needing to `cd` into their directories.
```yaml
workspace:
  include:
    - "services/**"
```
Execute child task: `cast @frontend deploy -c prod`

## Environment Variables

Environment variables can be defined globally or per task, and support both string interpolation and command substitution (`$()`).

Dotenv files support multiple files, optional files (`?` prefix), and context targeting.

```yaml
dotenv:
  - path: .env
  - path: ?.env.local # Optional, won't error if missing

tasks:
  task:
    uses: bash
    run: echo $GLOBAL_VAR
```

## CAST_ Environment Variables

### CLI Configuration

| Variable | Description |
|----------|-------------|
| `CAST_PROJECT` | Default project file path (used by `-p` flag) |
| `CAST_CONTEXT` | Default context name (used by `-c` flag) |

### Task Runtime (available within tasks)

| Variable | Description |
|----------|-------------|
| `CAST_ENV` | Path to temp file. Write `KEY=value` here to share environment to downstream tasks |
| `CAST_PATH` | Path to temp file. Write directories here to prepend them to the `$PATH` |
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
