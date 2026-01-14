---
name: bunnyshell environments
description: Manage Bunnyshell environments platform (environments.bunnyshell.com). Use this skill when the user asks to create, configure, deploy, stop, start, or delete environments in Bunnyshell. Also use for component management, port forwarding, remote development, SSH access, pipeline monitoring, and authoring bunnyshell.yaml configuration files. Triggers include mentions of Bunnyshell, bns CLI, environment deployment, ephemeral environments, or bunnyshell.yaml.
---

# Bunnyshell Environments Skill

Manage cloud environments using the Bunnyshell platform via the `bns` CLI.

## Prerequisites

- **CLI installed**: `bns` command available (install via Homebrew: `brew install bunnyshell/tap/bunnyshell-cli`)
- **Authentication**: Profile configured with API token from https://environments.bunnyshell.com/access-token

Check if CLI is configured:
```bash
bns environments list
```

If not configured, add a profile:
```bash
bns configure profiles add --name default --token YOUR_TOKEN --default
```

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

# Deploy environment (wait for completion)
bns environments deploy --id <ENV_ID> --wait

# Deploy environment (non-blocking)
bns environments deploy --id <ENV_ID> --no-wait

# Stop environment (scale down, preserve data)
bns environments stop --id <ENV_ID> --wait

# Start environment (resume from stopped)
bns environments start --id <ENV_ID> --wait

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
bns components show <COMPONENT_ID>

# Interactive shell session (execs /bin/sh in the container)
bns ssh --component <COMPONENT_ID>

# With specific container (for multi-container pods)
bns ssh --component <COMPONENT_ID> --container <CONTAINER_NAME>

# Execute commands non-interactively (preferred for scripting)
bns ssh --component <COMPONENT_ID> --no-banner --non-interactive --shell "whoami; pwd; ls -lah"
```

**Note:** `bns ssh` is not real SSH - it executes `/bin/sh` in the container. Use the `--shell` flag to pass commands for non-interactive execution. `bns components exec` does not exist.

### 3. Executing Commands in Containers

There are two main approaches to execute commands in component containers:

#### Option A: Using `bns ssh` (Recommended)

The `bns ssh` command executes `/bin/sh` in the container (not real SSH). It handles kubeconfig automatically.

```bash
# Interactive shell session
bns ssh --component <COMPONENT_ID>

# With specific container (if pod has multiple containers)
bns ssh --component <COMPONENT_ID> --container <CONTAINER_NAME>

# Non-interactive command execution (best for scripting)
bns ssh --component <COMPONENT_ID> --no-banner --non-interactive --shell "whoami"
bns ssh --component <COMPONENT_ID> --no-banner --non-interactive --shell "php bin/console --version"
bns ssh --component <COMPONENT_ID> --no-banner --non-interactive --shell "cat /etc/hosts; pwd; ls -la"
```

**Flags for non-interactive use:**

- `--no-banner` - Suppresses the welcome banner
- `--non-interactive` - Disables interactive prompts
- `--shell "<commands>"` - Commands to execute (semicolon-separated for multiple)

#### Option B: Using `kubectl exec` (Direct Cluster Access)

Alternative when you have direct cluster access and kubeconfig configured. **Requires cluster credentials.**

**Prerequisites for kubectl:**
1. **kubectl installed**: The Kubernetes CLI must be available
2. **Kubeconfig configured**: You need a valid kubeconfig with credentials for the cluster
   - The kubeconfig is typically at `~/.kube/config` or set via `KUBECONFIG` env var
   - Must have credentials for the Kubernetes cluster where the environment runs
   - Bunnyshell environments run on clusters configured via "Kubernetes Integrations"
3. **Network access**: Your machine must be able to reach the Kubernetes API server

**Step-by-step workflow:**

```bash
# 1. Get the environment's namespace (format: env-<UNIQUE>)
bns environments show --id <ENV_ID> --output json | jq -r '.namespace'
# Example output: "7uyzlw" → namespace is "env-7uyzlw"

# 2. List pods in the namespace
kubectl get pods -n env-<NAMESPACE>

# 3. Execute command in a specific pod/container
kubectl exec -n env-<NAMESPACE> <POD_NAME> -c <CONTAINER_NAME> -- <COMMAND>

# Example: Check Symfony version in dashboard-php
kubectl exec -n env-7uyzlw dashboard-php-7bb5d69574-2k6jc -c dashboard-php -- php bin/console --version
```

**Finding the right pod and container names:**
- Pod names typically follow the pattern: `<component-name>-<replicaset-hash>-<pod-hash>`
- Container names often match the component name, but multi-container pods may differ
- Use `kubectl describe pod <POD_NAME> -n env-<NAMESPACE>` to see all containers

### 4. Viewing Component Logs

**Note:** `bns components logs` does not exist. Use these alternatives:

```bash
# Option 1: Via kubectl (requires cluster access - see prerequisites above)
# First get the namespace from environment details
bns environments show --id <ENV_ID> --output json | jq -r '.namespace'
# Namespace format is "env-<UNIQUE>" (e.g., if output is "7uyzlw", namespace is "env-7uyzlw")

# List pods to find the one you need
kubectl get pods -n env-<NAMESPACE>

# Get logs from a specific pod/container
kubectl logs -n env-<NAMESPACE> <POD_NAME> --tail=200
kubectl logs -n env-<NAMESPACE> <POD_NAME> -c <CONTAINER_NAME> --tail=200  # specific container
kubectl logs -n env-<NAMESPACE> <POD_NAME> -f  # follow/stream logs

# Option 2: SSH and view logs interactively
bns ssh --component <COMPONENT_ID>
# Then run: tail -f /var/log/app.log (or wherever logs are stored)
```

### 5. Pipeline Monitoring

```bash
# List pipelines
bns pipeline list --environment <ENV_ID>

# Show pipeline details
bns pipeline show --id <PIPELINE_ID>

# Monitor pipeline progress (waits until completion)
bns pipeline monitor --id <PIPELINE_ID>
```

**Note:** `bns pipeline logs` and `bns pipeline cancel` do not exist. Use `monitor` to follow progress or the web UI to cancel pipelines.

### 6. Remote Development

```bash
# Shell into component (execs /bin/sh, not real SSH)
bns ssh --component <COMPONENT_ID>

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

# Encrypt/decrypt secrets
bns secrets encrypt "my-secret" --organization <ORG_ID>
bns secrets decrypt "ENCRYPTED[...]" --organization <ORG_ID>

# Encrypt entire definition file
bns secrets encrypt-definition --file bunnyshell.yaml --organization <ORG_ID>
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
  API_KEY: SECRET["my-secret"]
  BASE_URL: 'https://api-{{ env.base_domain }}'

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
    hosts:
      - hostname: 'api-{{ env.base_domain }}'
        path: /
        servicePort: 8080
```

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
| Authentication failed | Run `bns configure profiles add` with valid token |
| Environment stuck deploying | Check `bns pipeline monitor --id <PIPELINE_ID>` or view in web UI |
| Component not found in UI | Ensure Helm uses `--post-renderer /bns/helpers/helm/bns_post_renderer` |
| Variables not injecting | Check variable scope (project > environment > component) |

## Reference Files

- [CLI Reference](references/cli.md) - Complete CLI command documentation
- [API Reference](references/api.md) - REST API for CI/CD integrations
- [YAML Schema](references/yaml-schema.md) - bunnyshell.yaml configuration reference
- [Component Types](references/components.md) - Helm, Kubernetes, Terraform, Docker Compose, etc.
- [Variables & Interpolation](references/variables.md) - Secrets, scopes, Twig filters
