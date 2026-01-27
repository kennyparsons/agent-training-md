# Agent Command Logging Standard

## Purpose

When running commands, the coding agent should log all output to a central log file with timestamped, tagged prefixes. This allows the user to monitor all agent activity in real-time via `tail -f` in a separate terminal.

## Log File

- **Location**: `logs/terminal.log` (relative to repo root)
- **Mode**: Always **append** (`>>` or `tee -a`)
- **Never truncate** the log file - preserve history across commands

## Command Format

All commands should be piped through a logging wrapper that:
1. Prefixes each line with a tag identifying the operation
2. Adds a timestamp (HH:MM:SS)
3. Appends to the log file
4. Still shows output in the terminal

### Pattern

```bash
<command> 2>&1 | while IFS= read -r line; do echo "[<TAG>-$(date +%H:%M:%S)] $line"; done | tee -a logs/terminal.log
```

### Components

| Part | Purpose |
|------|---------|
| `2>&1` | Capture both stdout and stderr |
| `while IFS= read -r line` | Process line-by-line, preserving whitespace |
| `[<TAG>-$(date +%H:%M:%S)]` | Prefix with tag and timestamp |
| `tee -a` | Append to log AND show in terminal |

## Tag Naming Convention

Tags should be:
- **UPPERCASE**
- **Short** (max ~15 chars)
- **Descriptive** of the operation

### Standard Tags

| Tag | Use Case |
|-----|----------|
| `TF-APPLY` | Terraform apply |
| `TF-PLAN` | Terraform plan |
| `TF-DESTROY` | Terraform destroy |
| `TF-IMPORT` | Terraform import |
| `DOCKER-BUILD` | Docker build operations |
| `DOCKER-PUSH` | Docker push operations |
| `TEST-HTTP` | HTTP endpoint testing (curl) |
| `TEST-UNIT` | Unit test runs |
| `TEST-E2E` | End-to-end tests |
| `BUILD` | General build commands |
| `DEPLOY` | Deployment operations |
| `GIT` | Git operations |
| `NPM` | npm/node commands |
| `UV` | Python uv commands |
| `GCLOUD` | gcloud CLI commands |
| `BQ` | BigQuery commands |
| `CURL` | API/HTTP requests |
| `LINT` | Linting operations |
| `MIGRATE` | Database migrations |

## Examples

### Terraform Apply
```bash
cd infrastructure && source ./load-secrets.sh ../.env && \
terraform apply -auto-approve 2>&1 | \
while IFS= read -r line; do echo "[TF-APPLY-$(date +%H:%M:%S)] $line"; done | \
tee -a ../logs/terminal.log
```

### HTTP Testing
```bash
curl -s -w "\nHTTP_CODE:%{http_code}\n" https://example.com/health 2>&1 | \
while IFS= read -r line; do echo "[TEST-HTTP-$(date +%H:%M:%S)] $line"; done | \
tee -a logs/terminal.log
```

### Running Tests
```bash
uv run pytest tests/ -v 2>&1 | \
while IFS= read -r line; do echo "[TEST-UNIT-$(date +%H:%M:%S)] $line"; done | \
tee -a logs/terminal.log
```

### Docker Build
```bash
docker build -t myapp:latest . 2>&1 | \
while IFS= read -r line; do echo "[DOCKER-BUILD-$(date +%H:%M:%S)] $line"; done | \
tee -a logs/terminal.log
```

### Multiple Sequential Commands
For related commands, use the same tag:
```bash
(
  echo "=== Starting deployment ===" && \
  terraform apply -auto-approve && \
  echo "=== Deployment complete ==="
) 2>&1 | while IFS= read -r line; do echo "[DEPLOY-$(date +%H:%M:%S)] $line"; done | tee -a logs/terminal.log
```

## User Monitoring

The user can monitor all agent activity with:
```bash
tail -f logs/terminal.log
```

Example output:
```
[TF-APPLY-14:55:41] google_cloud_run_v2_service.hubquiz: Modifying...
[TF-APPLY-14:55:43] module.quiz_sync[0].null_resource.sync_quizzes (local-exec):   ✅ Deleted 6 non-Terraform question(s)
[TF-APPLY-14:55:44] Apply complete! Resources: 1 added, 2 changed, 0 destroyed.
[TEST-HTTP-14:56:01] HTTP_CODE:200
[TEST-HTTP-14:56:01] {"status":"healthy"}
```

## Important Notes

1. **Always use `tee -a`** (append mode) - never overwrite the log
2. **Create logs/ directory** if it doesn't exist before first command
3. **Use consistent tags** for related operations
4. **Include separators** for long operations (e.g., `echo "=== Starting... ==="`)
5. **Ensure logs/ is in .gitignore** to avoid committing logs

