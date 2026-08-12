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

## An operator's admission webhook doesn't see my configuration

**Symptom:** an operator that injects sidecars or init containers (APM agents,
OpenTelemetry, service meshes, secret injectors) never matches your pods, even though the
variable or label looks correct in `bunnyshell.yaml`.

**Cause:** the webhook only sees the **pod spec**. Of the things you can set per component:

| Set in `bunnyshell.yaml` | Lands on | Visible to a webhook? |
|---|---|---|
| `environment:` | `env-file-<component>` ConfigMap, via `envFrom` | ❌ |
| `labels:` (Kompose → annotations) | pod **annotations** | ✅ |
| `deploy: labels:` (Kompose → workload) | Deployment labels only | ❌ pod not labelled |
| `pod: labels/annotations:` | not a supported key | ❌ not applied |

**Diagnose:**

```bash
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].env}'   # inline env only
kubectl get pod <pod> -o jsonpath='{.metadata.labels}'
kubectl get pod <pod> -o jsonpath='{.metadata.annotations}'
kubectl logs -n <operator-ns> deploy/<operator> | grep -i <pod-name>
```

**Workarounds**, in order of preference:

1. **Select by container name** if the operator supports it — container names match the
   component name and are stable.
2. **Select by pod annotation** via `labels:`, if the operator reads annotations. Check the
   semantics for pods *without* the annotation: some operators treat "no annotation" as
   "match everything", which will inject into sidecars too.
3. **Namespace selector** — coarse, but every operator supports it, and each Bunnyshell
   environment gets its own namespace.

Verified against New Relic's `k8s-agents-operator` (2026-08).

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

## Definition validation errors

### `This port is not a public port declared in dockerCompose.ports`

**Cause:** a `hosts:` entry targets a `servicePort` that the **parent** component does not
declare — commonly when the port belongs to a sidecar.

**Fix:** declare the sidecar's port on the parent component too. The parent owns the
Service, so every port routed by `hosts:` must appear in its `dockerCompose.ports`, even
when another container is what listens on it.

```yaml
- kind: Service
  name: app
  dockerCompose:
    ports:
      - '8080:8080'   # this container
      - '80:80'       # the nginx sidecar's port, declared here so hosts: can route to it
  pod:
    sidecar_containers:
      - from: app-nginx
```

### `Value "env.vars.X" not found in the current context`

**Cause:** the definition interpolates `{{ env.vars.X }}`, but environment variables are
scoped to an environment that does not exist yet. On `environments create` there is nothing
to resolve against.

**Fix:** declare the values inline in the definition under `environmentVariables:` when
creating, then manage them normally afterwards.

```yaml
environmentVariables:
  MY_VAR: value
```

Note that values declared this way are stored in the definition in plaintext — see the
`SECRET[]` syntax in `variables.md` for anything sensitive.

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

### `bns pipeline logs --failed` does not exist (pre-v0.26)
The `--failed` flag was never valid. As of v0.26, use `--jobStatus` and `--stepStatus` instead:
```bash
bns pipeline logs --id <PIPELINE_ID> --jobStatus failed
bns pipeline logs --id <PIPELINE_ID> --stepStatus failed
```

### `bns environments list` paginates interactively

With more environments than fit one page, the command prints a "Navigate to a different
page?" prompt and exits with `Error: EOF` in a non-interactive shell. Piping it to `grep`
therefore searches **only the first page** and silently misses the rest — including
environments you have just created. Filter server-side instead:

```bash
bns environments list --project <PROJECT_ID>
```

### Fetching a definition is `definition`, not `definition-get`

```bash
bns environments definition --id <ENV_ID> -o raw > env.yaml
```

## Build Failures

### Package not available in Debian repos

**Symptom:** `E: Package 'ttyd' has no installation candidate` (or similar for other packages)

**Cause:** The package is not in the standard Debian/Ubuntu apt repositories for the target release (e.g., `bookworm`). This is common for newer or niche tools.

**Fix:** Install from GitHub releases or other sources instead of apt:
```dockerfile
# WRONG - ttyd is not in Debian bookworm repos
RUN apt-get install -y ttyd

# CORRECT - install binary from GitHub releases
RUN wget -qO /usr/local/bin/ttyd https://github.com/tsl0922/ttyd/releases/download/1.7.7/ttyd.x86_64 \
    && chmod +x /usr/local/bin/ttyd
```

**General rule:** Always verify package availability for your target distro before using `apt-get install`. If unavailable, check the project's GitHub releases for static binaries.

### Go version mismatch in multi-stage builds

**Symptom:** `go install` fails with `requires go >= 1.24.0 (running go 1.23.x; GOTOOLCHAIN=local)`

**Cause:** The Go module requires a newer Go version than the builder stage provides.

**Fix:** Check the project's `go.mod` for the required Go version and match your builder image:
```dockerfile
# Check required version first, then match:
FROM golang:1.24-bookworm AS builder
RUN go install github.com/example/tool@latest
```

### Always test Dockerfiles locally before deploying

**Best practice:** Build with `--platform linux/amd64` locally before pushing to Bunnyshell, since Bunnyshell builds on amd64:
```bash
docker build --platform linux/amd64 -f .docker/Dockerfile -t myimage:test .
```
This catches package availability issues, version mismatches, and build errors before wasting a deploy cycle.

## Web Terminal (ttyd) Setup

When creating workspace/CLI templates that need browser-based terminal access:

### Installation
ttyd is NOT in Debian apt repos. Install the binary directly:
```dockerfile
RUN wget -qO /usr/local/bin/ttyd https://github.com/tsl0922/ttyd/releases/download/1.7.7/ttyd.x86_64 \
    && chmod +x /usr/local/bin/ttyd
```

### Basic Authentication
Use the `-c` flag for HTTP basic auth:
```dockerfile
CMD ["sh", "-c", "ttyd -W -c ${TTYD_USER}:${TTYD_PASS} bash"]
```
Pass credentials via environment variables in `bunnyshell.yaml`:
```yaml
environmentVariables:
    TTYD_USER: 'admin'
    TTYD_PASS: 'changeme'
```

### Port
ttyd defaults to port 7681. Expose it in the Dockerfile and map it in `bunnyshell.yaml`:
```yaml
hosts:
    - hostname: 'terminal-{{ env.base_domain }}'
      path: /
      servicePort: 7681
```
