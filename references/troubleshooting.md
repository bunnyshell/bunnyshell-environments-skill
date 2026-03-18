# Troubleshooting Guide

Common issues encountered during template development and deployment.

_(This guide is actively updated as issues are encountered)_

## Deployment Failures

### General Debugging Steps

1. Check pipeline status: `bns pipeline list --environment <ENV_ID>`
2. Monitor pipeline: `bns pipeline monitor --id <PIPELINE_ID>`
3. Check component status: `bns components list --environment <ENV_ID>`
4. Check pod logs via kubectl (see SKILL.md for kubectl setup)

## Image Pull Errors

### Symptoms
- Pipeline stuck at "Pulling image" stage
- Error: `ImagePullBackOff` or `ErrImagePull`

### Common Causes
- Image tag doesn't exist on Docker Hub
- Private registry without credentials configured
- Rate limiting on Docker Hub (anonymous pulls)

### Fix
- Verify image tag exists: check Docker Hub directly
- For rate limits: configure `imagePullSecrets` in the cluster

## Database Connection Issues

### PostgreSQL password change not taking effect

**Symptom:** After changing `POSTGRES_PASSWORD`, connections fail with "password authentication failed"

**Cause:** PostgreSQL only reads `POSTGRES_PASSWORD` on first initialization (when the data directory is empty). The password is persisted in the volume. Subsequent env var changes are ignored.

**Fix:** Delete the environment and recreate it (to destroy the PG volume), or exec into the PG container and change the password manually:
```bash
bns exec <PG_COMPONENT_ID> -c <CONTAINER_NAME> -- psql -U postgres -c "ALTER USER myuser PASSWORD 'newpassword'"
```

**Note:** This applies to all PostgreSQL Docker images (official, Bitnami, etc.) that use a persistent volume for `/var/lib/postgresql/data`.

## Bitnami Docker Hub Changes (Aug 2025)

Bitnami restructured their public Docker Hub catalog:
- **Community tier (free)**: Only `latest` tag available at `docker.io/bitnamisecure`
- **Legacy**: Old versioned tags moved to `docker.io/bitnamilegacy` (unsupported)
- **Enterprise**: Full version support requires paid subscription

**Impact on templates**: Any template using `bitnami/*` images with specific version tags needs migration to official images or `bitnamilegacy`.

**MinIO migration**: Switch from `bitnami/minio:VERSION` to `minio/minio:RELEASE.YYYY-MM-DDThh-mm-ssZ`
- Bitnami env vars `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD` work with official image too
- Official image uses `/data` as default data dir (bitnami used `/bitnami/minio/data`)
- Console runs on port 9001 by default with `--console-address :9001`

## bns CLI Gotchas

### `bns components show` requires `--id` flag
```bash
# WRONG - goes interactive
bns components show pYmOB67AGr

# CORRECT
bns components show --id pYmOB67AGr
```

### `bns components list` needs `--organization` for some accounts
If `bns components list --environment <ENV_ID>` returns empty, try adding `--organization <ORG_ID>`.

### Non-interactive mode
Always use `--no-wait` for deploy/delete when scripting, and avoid commands that may prompt for input.

### `bns exec` hangs on multi-container pods
Always pass `-c <CONTAINER_NAME>` when the component has sidecars or init containers. Without it, the CLI prompts for interactive container selection, which hangs in non-interactive contexts.

### `bns pipeline jobs` goes interactive with positional argument
Use `bns pipeline jobs --id <PIPELINE_ID>`, not `bns pipeline jobs <PIPELINE_ID>`. The positional form may trigger interactive mode and produce massive output.

### `--output json` not supported with deploy/stop/start
`bns environments deploy --output json` returns "Error: only stylish format is supported when following pipelines". Deploy, stop, and start commands only support the default stylish output.

### `bns pipeline logs --failed` does not exist
The `--failed` flag is not a valid option. To find failed jobs, use `bns pipeline jobs --id <PIPELINE_ID>` and check individual job status.

## Build Failures

_(Updated as issues are encountered)_
