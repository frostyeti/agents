---
name: kpv
description: KeePass vault CLI for secrets management with automation. Use when working with KeePass databases, managing secrets in .kdbx files, or injecting secrets into environment variables and dotenv files. Pairs with cast task runner.
license: MIT
metadata:
  author: Grimfrost Mage
  version: "0.0.0"
---

# kpv - KeePass Vault CLI

kpv is a CLI for automating secrets management with KeePass (`.kdbx`) vaults. It supports initializing vaults, getting/setting secrets, listing, importing, exporting, removing, ensuring, and syncing entries with OS keyring integration.

## Installation

Build from source:

```bash
git clone https://github.com/frostyeti/kpv
cd kpv
mise run build
./bin/kpv --help
```

Or install with Go:

```bash
go install github.com/frostyeti/kpv@latest
```

Or install with mise:

```bash
mise install go:github.com/frostyeti/kpv@latest
```



## Quick Start

```bash
kpv init                                    # Initialize default vault
kpv secrets set --key api-token --value "secret"
kpv secrets get --key api-token
kpv secrets ls
```

## Global Flags

| Flag | Env Var | Description |
|------|---------|-------------|
| `-V, --vault` | `KPV_VAULT` | Vault name or path |
| `-P, --vault-password` | `KPV_PASSWORD` | Vault password |
| `-F, --vault-password-file` | `KPV_PASSWORD_FILE` | Password file path |
| `--vault-os-secret` | `KPV_VAULT_OS_SECRET` | OS keyring key for password |

Password resolution order: `--password` > `--password-file` > env vars > OS keyring > `.key` file

## Core Commands

### Vault Management

```bash
kpv init                                    # Default vault at ~/.local/share/kpv/default.kdbx
kpv init --vault myvault --global           # Named vault in default directory
kpv init --vault /path/to/vault.kdbx        # Specific path
```

### Secrets Operations

```bash
kpv secrets set --key name --value "secret"
kpv secrets set --key name --file ./secret.txt
kpv secrets set --key name --stdin          # Pipe: echo "secret" | kpv secrets set --key name --stdin
kpv secrets set --key name --generate --size 32

kpv secrets get --key name
kpv secrets get --key name --format dotenv  # Output as dotenv format
kpv secrets get --key name --format sh      # Output as shell export
kpv secrets get --key a --key b             # Multiple secrets

kpv secrets ensure --key name               # Get or generate if missing
kpv secrets ls
kpv secrets ls "app-*"                      # Glob filter
kpv secrets rm --key name --yes

kpv secrets get-string --key entry --field custom-field
kpv secrets set-string --key entry --field name --value "val"
```

### Output Formats

| Format | Description |
|--------|-------------|
| `text` | Plain text (default) |
| `dotenv` | `KEY=value` format |
| `sh`, `bash`, `zsh` | `export KEY="value"` |
| `pwsh`, `powershell` | PowerShell format |
| `json` | JSON object |
| `github-actions` | GitHub Actions `::set-output` |
| `azure-devops` | Azure DevOps logging command |

### Import/Export

```bash
kpv secrets export --json --file secrets.json
kpv secrets export --json --pretty
kpv secrets import --json --file secrets.json
kpv secrets sync --json --file secrets.json --dry-run
```

## Pairing with Cast

kpv integrates with cast to inject secrets into tasks:

### Option 1: Direct Environment Injection

```yaml
tasks:
  deploy:
    uses: bash
    run: |
      export $(kpv secrets get --key DB_PASSWORD --format sh)
      ./deploy.sh
```

### Option 2: Generate .env File

```yaml
tasks:
  setup-env:
    uses: bash
    run: |
      kpv secrets get --key DB_PASSWORD --key API_KEY --format dotenv > .env

  run-app:
    needs: [setup-env]
    dotenv: [".env"]
    uses: bash
    run: npm start
```

### Option 3: Inline Secret Resolution

```yaml
tasks:
  database:
    uses: bash
    run: |
      DB_PASS=$(kpv secrets get --key db-password)
      mysql -u user -p"$DB_PASS" database
```

### Option 4: Multiple Secrets to Environment

```yaml
tasks:
  run-with-secrets:
    uses: bash
    run: |
      eval $(kpv secrets get --key DB_URL --key API_KEY --key SECRET_TOKEN --format sh)
      ./application
```

## Custom String Fields

KeePass entries can have custom fields beyond the standard Password/Username/URL:

```bash
kpv secrets set-string --key my-api --field client-id --value "abc123"
kpv secrets set-string --key my-api --field private-key --file key.pem
kpv secrets get-string --key my-api --field client-id
```

## Secret Generation

```bash
kpv secrets set --key name --generate --size 32
kpv secrets ensure --key name --size 24 -U -S   # Generate if missing, no upper, no special
```

Generation flags:
- `--size N` - Length (default 16)
- `-U, --no-upper` - No uppercase
- `-L, --no-lower` - No lowercase  
- `-D, --no-digits` - No digits
- `-S, --no-special` - No special chars
- `--special "<chars>"` - Custom special chars

## Configuration

```bash
kpv config set defaults.path /path/to/default.kdbx
kpv config get defaults.path
kpv config ls

kpv config aliases set work /path/to/work.kdbx
kpv config aliases get work
```

## OS Keyring Integration

```bash
kpv config secret set /path/to/vault.kdbx    # Store password in OS keyring
kpv config secret get /path/to/vault.kdbx    # Retrieve from OS keyring
```

## Default Paths

- Linux/macOS: `~/.local/share/kpv/default.kdbx`
- Windows: `%LOCALAPPDATA%/kpv/default.kdbx`
- Key backup: `~/.local/share/kpv/default.key`

## Best Practices

1. Use OS keyring for vault passwords - more secure than `.key` files
2. Use `--format sh` or `--format dotenv` for shell integration
3. Use `secrets ensure` for idempotent secret generation
4. Pair with cast for automated secret injection in CI/CD
5. Use custom fields for multi-value entries (API keys + tokens)

## Troubleshooting

- **Vault locked**: Ensure password is provided via flag, env, keyring, or `.key` file
- **Secret not found**: Use `kpv secrets ls` to verify entry exists
- **Format issues**: Try `--format text` for debugging, then switch to desired format
