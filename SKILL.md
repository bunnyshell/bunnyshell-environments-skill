---
name: bunnyshell-environments
description: Manage Bunnyshell environments platform (environments.bunnyshell.com). Use this skill when the user asks to create, configure, deploy, stop, start, or delete environments in Bunnyshell. Also use for component management, port forwarding, remote development, SSH access, pipeline monitoring, and authoring bunnyshell.yaml configuration files. Triggers include mentions of Bunnyshell, bns CLI, environment deployment, ephemeral environments, or bunnyshell.yaml.
---

# Bunnyshell Environments Skill

Manage cloud environments using the Bunnyshell platform via the `bns` CLI.

## Prerequisites

- **CLI installed**: `bns` command available (install via Homebrew: `brew install bunnyshell/tap/bunnyshell-cli`)
- **CLI up to date**: Some flags/behaviors differ between versions. Update before debugging unexpected CLI errors:
  - macOS: `brew upgrade bunnyshell/tap/bunnyshell-cli`
  - Linux/Windows: Download latest from https://github.com/bunnyshell/cli/releases
- **Authentication**: API token from https://environments.bunnyshell.com/access-token

**Authenticate via environment variable (recommended):**
```bash
export BUNNYSHELL_TOKEN=YOUR_TOKEN
bns environments list
```

**Alternative — persistent profile (interactive terminal only):**
```bash
bns configure profiles add --name default --token YOUR_TOKEN --default
```

**Warning:** `bns configure profiles add` prompts interactively and will hang in non-interactive/scripting contexts. Prefer `export BUNNYSHELL_TOKEN` when automating.

## Core Workflows

### 1. Environment Operations

```bash
# List environments
bns environments list --project <PROJECT_ID>

# Show environment details
bns environments show --id <ENV_ID>

# Create from template
bns environments create \
  --from-template <TEMPLATE_ID> \
  --name "My Env" \
  --project <PROJECT_ID> \
  --k8s <CLUSTER_ID>

# Create from local bunnyshell.yaml
bns environments create \
  --from-path bunnyshell.yaml \
  --name "My Env" \
  --project <PROJECT_ID> \
  --k8s <CLUSTER_ID>

# Deploy environment (wait for completion)
bns environments deploy --id <ENV_ID> --wait

# Deploy environment (non-blocking)
# Note: --output json is NOT supported with deploy/stop/start commands
# These commands only support stylish (default) output format
bns environments deploy --id <ENV_ID> --no-wait

# Stop environment (scale down, preserve data)
bns environments stop --id <ENV_ID> --wait

# Start environment (resume from stopped)
bns environments start --id <ENV_ID> --wait

# Abort a running action (deploy, start, stop, etc.) (v0.26+)
bns environment abort --id <ENV_ID>

# Delete environment
bns environments delete --id <ENV_ID>

# Clone environment
bns environments clone --id <ENV_ID> --name "Cloned Env"

# Export configuration (get current YAML definition)
bns environments definition --id <ENV_ID> > bunnyshell.yaml

# Update configuration from local file
bns environments update-configuration --id <ENV_ID> --from-path bunnyshell.yaml

# Update configuration and deploy in one step
bns environments update-configuration --id <ENV_ID> --from-path bunnyshell.yaml --deploy
```

### 2. Component Operations

```bash
# List components
bns components list --environment <ENV_ID>

# Show component details
bns components show --id <COMPONENT_ID>
```

### 3. Executing Commands in Containers

#### Option A: Using `bns exec` (Recommended)

The `bns exec` command runs commands in a component's container. It handles kubeconfig automatically.

**IMPORTANT:** For pods with sidecars/multiple containers, ALWAYS specify `-c <CONTAINER_NAME>`. Omitting `-c` triggers an interactive container picker that hangs in non-interactive contexts.

```bash
# Run a command (non-interactive)
bns exec <COMPONENT_ID> -- ls -la /app
bns exec <COMPONENT_ID> -- cat /etc/nginx/sites-enabled/default
bns exec <COMPONENT_ID> -- python3 -c "print('hello')"

# Interactive shell
bns exec <COMPONENT_ID> --tty --stdin

# Specific container (REQUIRED for multi-container pods — omitting hangs)
bns exec <COMPONENT_ID> -c <CONTAINER_NAME> -- whoami

# Pipe a local script into the container
bns exec <COMPONENT_ID> --stdin -- python3 < local-script.py
```

#### Option B: Using `bns ssh` (Interactive Shell)

The `bns ssh` command opens an interactive `/bin/sh` session. Prefer `bns exec` for non-interactive use.

```bash
# Interactive shell session
bns ssh --component <COMPONENT_ID>

# With specific container
bns ssh --component <COMPONENT_ID> --container <CONTAINER_NAME>
```

#### Option C: Using `kubectl exec` (Direct Cluster Access)

Alternative when you have direct cluster access and kubeconfig configured. **Requires cluster credentials.**

```bash
# 1. Get the environment's namespace
bns environments show --id <ENV_ID> --output json | jq -r '.namespace'
# Output: "7uyzlw" → namespace is "env-7uyzlw"

# 2. List pods in the namespace
kubectl get pods -n env-<NAMESPACE>

# 3. Execute command
kubectl exec -n env-<NAMESPACE> <POD_NAME> -c <CONTAINER_NAME> -- <COMMAND>
```

### 4. Viewing Runtime / Container Logs

Use the top-level `bns logs` command (not `bns components logs`, which does not exist):

```bash
# Logs from a single component
bns logs --component <COMPONENT_ID>

# Stream all components in an environment
bns logs --environment <ENV_ID>

# Filter by component name (repeatable, requires --environment)
bns logs --environment <ENV_ID> --name api --name worker

# Follow / stream in real-time
bns logs --component <COMPONENT_ID> -f

# Last N lines
bns logs --component <COMPONENT_ID> --tail 200

# Time-based filtering
bns logs --component <COMPONENT_ID> --since 5m       # last 5 minutes
bns logs --component <COMPONENT_ID> --since-time 2026-02-18T10:00:00Z  # RFC3339

# Specific container (multi-container pods)
bns logs --component <COMPONENT_ID> -c <CONTAINER_NAME>

# All containers in the pod
bns logs --component <COMPONENT_ID> --all-containers

# Include timestamps in output
bns logs --component <COMPONENT_ID> --timestamps

# Logs from previous terminated container
bns logs --component <COMPONENT_ID> --previous

# Disable color coding / source prefix
bns logs --component <COMPONENT_ID> --no-color --prefix=false
```

**Alternative: `kubectl logs`** (requires direct cluster access and kubeconfig):

```bash
# Get the namespace
bns environments show --id <ENV_ID> --output json | jq -r '.namespace'
# Namespace format: "env-<UNIQUE>"

kubectl logs -n env-<NAMESPACE> <POD_NAME> --tail=200
kubectl logs -n env-<NAMESPACE> <POD_NAME> -c <CONTAINER_NAME> -f
```

### 5. Pipeline Monitoring & Logs

```bash
# List pipelines
bns pipeline list --environment <ENV_ID>

# List pipelines with sorting (v0.26+)
bns pipeline list --environment <ENV_ID> --sort createdAt:desc

# Filter pipelines by status
bns pipeline list --environment <ENV_ID> --status failed

# Show pipeline details
bns pipeline show --id <PIPELINE_ID>

# List jobs in a pipeline (use --id flag, NOT positional argument)
# WARNING: `bns pipeline jobs <PIPELINE_ID>` (positional) may trigger interactive mode
bns pipeline jobs --id <PIPELINE_ID>

# Filter jobs by status (v0.26+)
bns pipeline jobs --id <PIPELINE_ID> --jobStatus failed
# Possible values: pending, queued, in_progress, failed, abort_failed, success, skipped, aborting, aborted

# Monitor pipeline progress (waits until completion)
bns pipeline monitor --id <PIPELINE_ID>

# Abort a running environment action (deploy, start, stop, etc.) (v0.26+)
bns environment abort --id <ENV_ID>
```

#### Pipeline Logs

```bash
# All logs for a pipeline
bns pipeline logs --id <PIPELINE_ID>

# Only logs from failed jobs (v0.26+)
bns pipeline logs --id <PIPELINE_ID> --jobStatus failed

# Only failed steps within all jobs (v0.26+)
bns pipeline logs --id <PIPELINE_ID> --stepStatus failed

# Both: failed jobs AND failed steps only
bns pipeline logs --id <PIPELINE_ID> --jobStatus failed --stepStatus failed

# View logs for a specific job directly
bns pipeline logs --id <PIPELINE_ID> --job <JOB_ID>

# Logs for the latest pipeline in an environment
bns pipeline logs --id "$(bns pipeline list --environment <ENV_ID> --sort=createdAt:desc -o json | jq -r '._embedded.item[0].id')"

# Output formats: stylish (default), json, yaml, raw
bns pipeline logs --job <JOB_ID> -o json
```

### 6. Remote Development

```bash
# Shell into component
bns exec <COMPONENT_ID> --tty --stdin

# Port forwarding
bns port-forward 5432 --component <COMPONENT_ID>          # Same local/remote
bns port-forward 15432:5432 --component <COMPONENT_ID>    # Different local port
bns port-forward :5432 --component <COMPONENT_ID>         # Random local port

# Debug session (modifies pod for debugging, then reverts)
bns debug start --component <COMPONENT_ID>
bns debug stop

# Remote development (sync local code to container)
bns remote-development up --component <COMPONENT_ID>
bns remote-development down
```

**Note:** `bns debug up`/`down` do not exist. Use `bns debug start`/`stop` instead.

### 7. Variables & Secrets

```bash
# Import variables
bns variables import --var-file=/path/to/vars.env
bns variables import --secret-file=/path/to/secrets.env

# Encrypt a secret (positional argument — preferred)
bns secrets encrypt "my-secret" --organization <ORG_ID>

# Encrypt via stdin (use echo -n to avoid trailing newline!)
echo -n "my-secret-value" | bns secrets encrypt --organization <ORG_ID>
# WARNING: `echo` (without -n) adds a newline to the value,
# causing subtle auth failures (e.g., password becomes "secret\n" not "secret")

# Decrypt a secret
bns secrets decrypt "ENCRYPTED[...]" --organization <ORG_ID>

# Encrypt entire definition file
bns secrets encrypt-definition --file bunnyshell.yaml --organization <ORG_ID>
```

**CRITICAL: SECRET[] and ENCRYPTED[] are mutually exclusive — use ONE, never both:**
```yaml
# Option A: SECRET["plaintext"] → Bunnyshell encrypts on save
DB_PASSWORD: SECRET["my-password"]

# Option B: ENCRYPTED[...] → Already encrypted via `bns secrets encrypt`
DB_PASSWORD: ENCRYPTED[bXktcGFzc3dvcmQ...]

# WRONG: SECRET["ENCRYPTED[...]"] → Double-encrypts, value is corrupted!
DB_PASSWORD: SECRET["ENCRYPTED[bXktcGFzc3dvcmQ...]"]  # DO NOT DO THIS
```

## Configuration Authoring

When asked to create or modify `bunnyshell.yaml`:
- See [references/yaml-schema.md](references/yaml-schema.md) for complete schema
- See [references/components.md](references/components.md) for component types
- See [references/variables.md](references/variables.md) for interpolation syntax

### Minimal bunnyshell.yaml Structure

```yaml
kind: Environment
name: my-environment
type: primary  # or ephemeral

environmentVariables:
  BASE_DOMAIN: my-app.example.com   # Custom domain (or use {{ env.base_domain }} for auto)

environmentVariablesGroups:
  app:
    APP_KEY: SECRET["my-secret"]
    APP_URL: 'https://{{ env.vars.BASE_DOMAIN }}'

components:
  - kind: Application
    name: api
    gitRepo: 'https://github.com/org/repo.git'
    gitBranch: main
    gitApplicationPath: /
    dockerCompose:
      build:
        context: .
        dockerfile: Dockerfile
      environment:
        APP_KEY: '{{ env.varGroups.app.APP_KEY }}'
        APP_URL: '{{ env.varGroups.app.APP_URL }}'
    hosts:
      - hostname: 'api-{{ env.vars.BASE_DOMAIN }}'
        path: /
        servicePort: 8080
        selfManagedDns: true
        k8s:
          ingress:
            tlsSecretName: tls-secret
            annotations:
              cert-manager.io/cluster-issuer: letsencrypt-prod
```

For a complete PHP-FPM + Nginx sidecar example, see [references/yaml-schema.md](references/yaml-schema.md#php-fpm--nginx-sidecar-complete-example).

### Key Interpolation Patterns

```yaml
{{ env.unique }}              # Environment unique ID
{{ env.base_domain }}         # Auto-generated domain
{{ env.k8s.namespace }}       # Kubernetes namespace
{{ env.vars.MY_VAR }}         # Environment variable
{{ components.NAME.image }}   # Component's Docker image
{{ components.NAME.exported.VAR }}  # Exported variable from Helm/Terraform
```

### Custom Domains with SSL

For custom domains, set `selfManagedDns: true` on the host. Use ExternalDNS or manage DNS manually.

**With cert-manager (recommended for auto SSL):**

```yaml
hosts:
  - hostname: 'api.example.com'
    path: /
    servicePort: 8080
    selfManagedDns: true
    k8s:
      ingress:
        tlsSecretName: ''
        annotations:
          cert-manager.io/cluster-issuer: letsencrypt-prod
```

**With existing TLS certificate (K8s secret):**

```yaml
hosts:
  - hostname: 'api.example.com'
    path: /
    servicePort: 8080
    selfManagedDns: true
    k8s:
      ingress:
        tlsSecretName: my-tls-secret
```

See [references/yaml-schema.md](references/yaml-schema.md) for full custom domain documentation.

## Output Format

Use `--output json` or `--output yaml` for scripting.

**Important:** JSON responses use `_embedded.item[]` structure, not a direct array:

```bash
# Correct way to parse environment list
bns environments list --output json | jq '._embedded.item[] | {id, name}'

# Correct way to parse component list
bns components list --environment <ENV_ID> --output json | jq '._embedded.item[] | {id, name}'

# Get just the IDs
bns environments list --output json | jq -r '._embedded.item[].id'
```

### JSON Field Names by Resource Type

Field names differ across resource types. Key fields to know:

| Resource | Key Fields |
|----------|-----------|
| **Environments** | `id`, `name`, `operationStatus` (not `status`), `namespace`, `kubernetesIntegration` |
| **Components** | `id`, `name`, `operationStatus`, `publicURLs` (not `endpoints`) |
| **K8s Integrations** | `id`, `clusterName` (not `name`), `cloudProvider`, `status` |
| **Projects** | `id`, `name`, `organization` |
| **Pipelines** | `id`, `status`, `description` |

**Getting component URLs:**
```bash
bns components show --id <COMPONENT_ID> --output json | jq -r '.publicURLs[]'
```

**Getting a project's organization (needed for secret encryption):**
```bash
bns projects show --id <PROJECT_ID> --output json | jq -r '.organization'
```

## Clone and Customize Workflow

Complete workflow for cloning an environment and customizing it:

```bash
# 1. Clone the source environment
bns environments clone --id <SOURCE_ENV_ID> --name "new-environment"
# Note the new environment ID from the output

# 2. Export the new environment's configuration
bns environments definition --id <NEW_ENV_ID> > new-env.yaml

# 3. Edit new-env.yaml to customize:
#    - Update environment variables
#    - Change git branches
#    - Modify resource settings
#    - etc.

# 4. Apply the updated configuration
bns environments update-configuration --id <NEW_ENV_ID> --from-path new-env.yaml

# 5. Deploy the environment
bns environments deploy --id <NEW_ENV_ID> --no-wait

# 6. Monitor the deployment
bns pipeline list --environment <NEW_ENV_ID>
bns environments show --id <NEW_ENV_ID>
```

## Deploy from Scratch Workflow

Complete workflow for deploying a new application to Bunnyshell:

```bash
# 1. Find the project
bns projects list --search "my-project" --output json | jq '._embedded.item[] | {id, name}'

# 2. Find the cluster
bns k8s list --output json | jq '._embedded.item[] | {id, clusterName, status}'

# 3. Get the organization ID (needed for encrypting secrets)
bns projects show --id <PROJECT_ID> --output json | jq -r '.organization'

# 4. Encrypt any secrets for bunnyshell.yaml
bns secrets encrypt "my-secret-value" --organization <ORG_ID>
# Returns: ENCRYPTED[...] — use this DIRECTLY in bunnyshell.yaml (do NOT wrap in SECRET[])
# For stdin piping: echo -n "value" | bns secrets encrypt --organization <ORG_ID>

# 5. Create the environment from a local bunnyshell.yaml
bns environments create \
  --from-path bunnyshell.yaml \
  --name "my-env" \
  --project <PROJECT_ID> \
  --k8s <CLUSTER_ID>
# Note the environment ID from the output

# 6. Deploy (non-blocking)
bns environments deploy --id <ENV_ID> --no-wait

# 7. Monitor the deployment pipeline
bns pipeline list --environment <ENV_ID>
bns pipeline monitor --id <PIPELINE_ID>

# 8. Get the public URL
bns components list --environment <ENV_ID> --output json | \
  jq -r '._embedded.item[] | "\(.name): \(.publicURLs[])"'
```

**Shortcut — update config and deploy in one step:**
```bash
bns environments update-configuration --id <ENV_ID> --from-path bunnyshell.yaml --deploy
```

## REST API

For CI/CD integrations or direct API access, see [references/api.md](references/api.md).

Quick example — deploy an environment:
```bash
curl -X POST "https://api.environments.bunnyshell.com/v1/environments/ENV_ID/deploy" \
  -H "Authorization: Bearer $TOKEN"
```

## Common Issues

| Issue | Solution |
|-------|----------|
| Authentication failed | `export BUNNYSHELL_TOKEN=<token>` or run `bns configure profiles add` (interactive only) |
| 522 Connection timed out | Cluster may be behind a firewall. Verify Cloudflare IPs are whitelisted on the cluster's ingress controller. Check cluster network connectivity before debugging app config. |
| Environment stuck deploying | Check `bns pipeline monitor --id <PIPELINE_ID>` or view in web UI |
| Component not found in UI | Ensure Helm uses `--post-renderer /bns/helpers/helm/bns_post_renderer` |
| Variables not injecting | Check variable scope (project > environment > component) |
| Blank page / mixed content errors | TLS terminates at ingress; app sees HTTP. Configure the framework to trust proxies (see below) |
| Sidecar port not reachable | Parent component must declare the sidecar's port in its own `ports` list |
| File uploads rejected (413) | Add `nginx.ingress.kubernetes.io/proxy-body-size: 50m` annotation to hosts |

### Framework Proxy/TLS Configuration

Kubernetes ingress terminates TLS. The app receives plain HTTP and generates `http://` URLs, causing mixed-content blocks. Fix per framework:

| Framework | Fix |
|-----------|-----|
| **Laravel** | `$middleware->trustProxies(at: '*')` in `bootstrap/app.php` |
| **Symfony** | `framework.trusted_proxies: 'REMOTE_ADDR'` and `trusted_headers: ['x-forwarded-for', 'x-forwarded-proto']` |
| **Django** | `SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')` |
| **Express/Node** | `app.set('trust proxy', true)` |
| **Rails** | Ensure `config.force_ssl = true` or use `ActionDispatch::RemoteIp` middleware |

## Template Management

When asked to update or create Bunnyshell templates:
- See [references/template-update-playbook.md](references/template-update-playbook.md) for standard update process and best practices checklist
- See [references/image-versions.md](references/image-versions.md) for recommended Docker image versions
- See [references/troubleshooting.md](references/troubleshooting.md) for common issues and solutions

## Framework Deployment Checklists

See [references/framework-checklists.md](references/framework-checklists.md) for framework-specific deployment checklists (Laravel, Symfony, Node.js, Django, Rails).

## Reference Files

- [Framework Checklists](references/framework-checklists.md) - Framework-specific deployment checklists
- [CLI Reference](references/cli.md) - Complete CLI command documentation
- [API Reference](references/api.md) - REST API for CI/CD integrations
- [YAML Schema](references/yaml-schema.md) - bunnyshell.yaml configuration reference
- [Component Types](references/components.md) - Helm, Kubernetes, Terraform, Docker Compose, etc.
- [Variables & Interpolation](references/variables.md) - Secrets, scopes, Twig filters
- [Template Update Playbook](references/template-update-playbook.md) - Process for updating templates
- [Image Versions](references/image-versions.md) - Recommended Docker image versions
- [Troubleshooting](references/troubleshooting.md) - Common issues and solutions
