---
name: osv
description: Operating system vault CLI for Mac Keychain, Linux keyring, and Windows Credential Manager. Use when storing or retrieving secrets in OS-native credential stores, or when needing secure cross-platform secret storage.
license: MIT
metadata:
  author: frostyeti
  version: "1.0"
---

# osv - Operating System Vaults

osv is a CLI for interacting with OS-native credential stores: Mac Keychain, Linux keyring (libsecret), and Windows Credential Manager. Provides secure secret storage with cross-platform support.

## Installation

Build from source:

```bash
git clone https://github.com/frostyeti/osv
cd osv
mise run build
./bin/osv --help
```

Or install with Go:

```bash
go install github.com/frostyeti/osv@latest
```

## Quick Start

```bash
osv config set service my-app                # Set default service name
osv set --key api-token --value "secret"     # Store a secret
osv get --key api-token                      # Retrieve a secret
osv ls                                       # List all secrets
osv rm --key api-token -y                    # Remove a secret
```

## Service Name

The service name identifies your application in the OS credential store. It groups all your secrets together.

```bash
osv config set service my-app-name           # Set default
export OSV_SERVICE=my-app-name               # Or use env var
osv --service my-app get --key token         # Or pass per-command
```

## Core Commands

### Set Secrets

```bash
osv set --key name --value "secret"
osv set name "secret"                        # Positional args
osv set --key name --file ./secret.txt       # From file
osv set --key name --var MY_ENV_VAR          # From env var
echo "secret" | osv set --key name --stdin   # From stdin
osv set --key name --generate --size 32      # Generate random
```

### Get Secrets

```bash
osv get --key name
osv get name                                 # Positional arg
osv get --key a --key b                      # Multiple secrets
osv get --key name --format json             # Output format
```

### List Secrets

```bash
osv ls                                       # All secrets
osv ls "app-*"                               # Glob filter
osv list "*-password"                        # Alias: list
```

### Remove Secrets

```bash
osv rm --key name                            # With confirmation
osv rm --key name --yes                      # Skip confirmation
osv rm name1 name2 -y                        # Multiple with short flags
```

## Output Formats

| Format | Description |
|--------|-------------|
| `text` | Plain text (default) |
| `dotenv`, `env`, `.env` | `KEY=value` format |
| `sh`, `bash`, `zsh` | `export KEY='value'` |
| `pwsh`, `powershell` | `$Env:KEY='value'` |
| `json` | JSON object |
| `null`, `null-terminated` | Null-delimited values |
| `github` | GitHub Actions masked output |
| `azure-devops`, `ado` | Azure DevOps secret variable |

### Format Examples

```bash
osv get --key API_KEY --format dotenv
# API_KEY='secret-value'

osv get --key API_KEY --key DB_PASS --format sh
# export API_KEY='secret1'
# export DB_PASS='secret2'

osv get --key token --format json
# {
#   "token": "secret-value"
# }
```

## Secret Generation

```bash
osv set --key name --generate --size 32      # 32 char secret
osv set --key name --generate -U -S          # No upper, no special
osv set --key name --generate --chars "abc123"  # Custom charset only
```

Generation flags:
- `--size N` - Length (default 16)
- `-U, --no-upper` - Exclude uppercase
- `-L, --no-lower` - Exclude lowercase
- `-D, --no-digits` - Exclude digits
- `-S, --no-special` - Exclude special chars
- `--special "<chars>"` - Custom special chars (default: `@_-{}|#!~:^`)
- `--chars "<chars>"` - Use only these characters

## Configuration

```bash
osv config set service my-app                # Default service name
osv config set libsecret.collection login    # Linux: collection name
osv config set keychain.name login           # macOS: keychain name
osv config get service                       # Get config value
osv config rm service                        # Remove config value
```

Config file location: `~/.config/osv/config`

## Platform-Specific Notes

### macOS (Keychain)

- Uses Keychain.app for storage
- Secrets are accessible via Keychain Access
- Default keychain: `login`
- Configure: `osv config set keychain.name login`

### Linux (libsecret/Keyring)

- Requires `libsecret` to be installed
- Uses GNOME Keyring or KDE Wallet
- Default collection: `login`
- Install: `sudo apt install libsecret-1-0 libsecret-1-dev`

### Windows (Credential Manager)

- Uses Windows Credential Manager
- Accessible via Control Panel > Credential Manager
- No additional configuration needed

## Pairing with Cast

osv integrates with cast for injecting secrets into tasks:

### Option 1: Direct Environment Injection

```yaml
tasks:
  deploy:
    uses: bash
    run: |
      eval $(osv get --key DB_PASSWORD --format sh)
      ./deploy.sh
```

### Option 2: Generate .env File

```yaml
tasks:
  setup-env:
    uses: bash
    run: |
      osv get --key DB_PASSWORD --key API_KEY --format dotenv > .env

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
      DB_PASS=$(osv get --key db-password)
      mysql -u user -p"$DB_PASS" database
```

### Option 4: Multiple Secrets

```yaml
tasks:
  run-with-secrets:
    uses: bash
    run: |
      eval $(osv get --key DB_URL --key API_KEY --key SECRET_TOKEN --format sh)
      ./application
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `OSV_SERVICE` | Default service name |

## Best Practices

1. Set a unique service name per project
2. Use `--format sh` or `--format dotenv` for shell integration
3. Prefer stdin or file input for sensitive values (avoids shell history)
4. Use glob patterns with `ls` to find related secrets
5. Pair with cast for CI/CD secret injection

## Troubleshooting

- **Linux keyring not working**: Install libsecret (`sudo apt install libsecret-1-0 libsecret-1-dev`)
- **Service name required**: Set via `osv config set service <name>` or `OSV_SERVICE` env var
- **Secret not found**: Use `osv ls` to verify entry exists
- **Permission denied**: Check OS credential store permissions
