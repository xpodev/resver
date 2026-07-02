# RESVER — Resource Version

A CLI tool for managing ML model version groups in a monorepo. Acts as a **lock-file registry**: maps named group versions to specific model versions, and lets apps declare which group they depend on.

Storage and retrieval of actual model files is fully delegated to user-defined shell commands — RESVER is agnostic to your storage backend (DVC, S3, Git LFS, custom scripts, etc.).

---

## Installation

Requires Python 3.11+ and [uv](https://docs.astral.sh/uv/).

```bash
uv tool install git+https://github.com/xpodev/resver
```

Or add it as a dev dependency in your monorepo:

```bash
uv add --dev resver
```

---

## Core Concepts

| Term | Description |
|---|---|
| **Model** | A named ML artifact (e.g. `fraud-detector`) |
| **Model Version** | A specific version of a model (e.g. `2.1.0`), with a path and optional storage commands |
| **Model Group** | A named, versioned snapshot pinning multiple model versions (e.g. `production@1.4.0`) |
| **App** | Any package in the monorepo that declares a dependency on a model group via `.resver/app.yml` |

---

## File Structure

### `.resver/registry.yml` — monorepo root, source of truth

```yaml
models:
  fraud-detector:
    description: "Detects fraudulent transactions"
    versions:
      2.1.0:
        path: "models/fraud-detector/v2.1.0"
        created_at: "2024-01-15T10:30:00Z"
        created_by: "jane@company.com"
      2.0.0:
        path: "models/fraud-detector/v2.0.0"
        created_at: "2023-12-01T09:00:00Z"
        created_by: "john@company.com"
        pull_command: "aws s3 sync s3://legacy-bucket/{path} ./{path}"
        push_command: "aws s3 sync ./{path} s3://legacy-bucket/{path}"
  embedder:
    description: "Text embedding model"
    versions:
      0.9.4:
        path: "models/embedder/v0.9.4"
        created_at: "2024-01-10T11:00:00Z"
        created_by: "jane@company.com"
        pull_command: "./scripts/pull_from_hf.sh {model} {version}"

groups:
  production:
    versions:
      1.4.0:
        created_at: "2024-01-15T10:30:00Z"
        created_by: "jane@company.com"
        description: "Q1 release"
        models:
          fraud-detector: "2.1.0"
          embedder: "0.9.4"
```

### `.resver/config.yml` — monorepo root, global fallback commands

```yaml
pull_command: "dvc pull {path}"
push_command: "dvc push {path}"
```

### `.resver/app.yml` — per-app, at the app root

```yaml
group: production
version: "1.4.0"
```

---

## Command Reference

### Registry Discovery

```bash
resver where
```
Prints the path to the `.resver/registry.yml` being used. Resolved by walking up from the current directory, same convention as git.

---

### Model Management

```bash
# Register a new model
resver model add fraud-detector --description "Detects fraudulent transactions"

# List all models
resver model list

# Remove a model (fails if referenced by any group)
resver model remove fraud-detector
```

---

### Model Version Management

```bash
# Register a new version
resver model version add fraud-detector 2.1.0 \
  --path models/fraud-detector/v2.1.0 \
  --created-by jane@company.com

# With storage command overrides (otherwise global config is used)
resver model version add embedder 1.0.0 \
  --path models/embedder/v1.0.0 \
  --pull-command "./scripts/pull_from_hf.sh {model} {version}" \
  --push-command "./scripts/push_to_hf.sh {model} {version}"

# List all versions of a model
resver model version list fraud-detector

# Remove a version (fails if referenced by any group)
resver model version remove fraud-detector 2.0.0
```

---

### Group Management

```bash
# Create a new group
resver group create production

# Release a new group version (prompts for each model if --model not supplied)
resver group release production 1.4.0 \
  --description "Q1 release" \
  --model fraud-detector=2.1.0 \
  --model embedder=0.9.4

# List all groups and their latest version
resver group list

# Show all versions of a group
resver group show production

# Show a specific group version
resver group show production@1.4.0
```

---

### App Management

Run these commands from inside an app's directory.

```bash
# Declare which group version this app uses
resver app use production@1.4.0

# Show resolved model versions and paths
resver app show

# Validate that the declared group version still exists (use in CI)
resver app check
```

---

### Pull / Push

```bash
# Pull all models for the current app (reads .resver/app.yml)
resver pull

# Push all models for a specific group version
resver push production@1.4.0
```

Pull and push validate all command configurations before executing any, then run sequentially from the monorepo root. Halts on first failure.

---

### Config Management

```bash
# Set the global pull command
resver config set pull-command "dvc pull {path}"

# Set the global push command
resver config set push-command "dvc push {path}"

# Show current global config
resver config show
```

---

### Utility

```bash
# Validate the full registry for structural correctness and consistency
resver validate

# Show model version changes between two group versions
resver diff production@1.3.0 production@1.4.0
```

---

## Command Token Reference

These tokens are substituted at runtime in pull/push commands:

| Token | Resolves to |
|---|---|
| `{path}` | The `path` field from the model version entry |
| `{model}` | The model's name |
| `{version}` | The pinned model version |
| `{group}` | The group name |

---

## Command Resolution Order

When executing pull or push for a model version, RESVER resolves the command using:

1. `pull_command` / `push_command` on the specific model version entry
2. `pull_command` / `push_command` in `.resver/config.yml`
3. If neither exists — fail with a clear message before executing anything

---

## Workflow Example

```bash
# Initial setup (monorepo root)
resver config set pull-command "dvc pull {path}"
resver config set push-command "dvc push {path}"

# Register models
resver model add fraud-detector --description "Fraud detection model"
resver model version add fraud-detector 2.1.0 --path models/fraud-detector/v2.1.0

resver model add embedder --description "Text embedding model"
resver model version add embedder 0.9.4 --path models/embedder/v0.9.4

# Create a group and release a version
resver group create production
resver group release production 1.0.0 \
  --description "Initial production group" \
  --model fraud-detector=2.1.0 \
  --model embedder=0.9.4

# In each app that needs models
cd apps/my-service
resver app use production@1.0.0
git add .resver/app.yml

# Pull models locally
resver pull
```

---

## CI Integration

Add `resver app check` to your CI pipeline to fail fast if an app's declared group version has been removed from the registry:

```yaml
- name: Validate model group
  run: resver app check
  working-directory: apps/my-service
```

---

## Development

```bash
# Install dependencies
uv sync --all-groups

# Run tests
uv run pytest tests/ -v

# Smoke test
uv run resver --help
```

---

## Behavioral Notes

- The registry file is discovered by walking up from cwd — run `resver` commands from anywhere inside the monorepo.
- All writes preserve existing YAML formatting and comments (`ruamel.yaml`).
- All mutating commands print a reminder to commit changed files to git.
- No command automatically runs git operations.
- Pull/push commands are resolved at execution time — changing `.resver/config.yml` retroactively applies to all versions without their own override.
- Commands run as shell subprocesses from the **monorepo root**, not the app directory.
