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
bns environments create-from-template \
  --template <TEMPLATE_ID> \
  --name "My Env" \
  --project <PROJECT_ID> \
  --k8s-integration <CLUSTER_ID>

# Deploy environment
bns environments deploy --id <ENV_ID> --wait

# Stop environment (scale down, preserve data)
bns environments stop --id <ENV_ID> --wait

# Start environment (resume from stopped)
bns environments start --id <ENV_ID> --wait

# Delete environment
bns environments delete --id <ENV_ID>

# Clone environment
bns environments clone --id <ENV_ID> --name "Cloned Env"

# Export/update configuration
bns environments export-configuration --id <ENV_ID> > bunnyshell.yaml
bns environments update-configuration --id <ENV_ID> < bunnyshell.yaml
```

### 2. Component Operations

```bash
# List components
bns components list --environment <ENV_ID>

# Show component details
bns components show --id <COMPONENT_ID>

# View logs
bns components logs --id <COMPONENT_ID>

# Execute command in container
bns components exec --id <COMPONENT_ID> -- ls -la

# Redeploy single component
bns components redeploy --id <COMPONENT_ID>
```

### 3. Pipeline Monitoring

```bash
# List pipelines
bns pipeline list --environment <ENV_ID>

# View logs (follow mode)
bns pipeline logs --id <PIPELINE_ID> --follow

# Cancel running pipeline
bns pipeline cancel --id <PIPELINE_ID>
```

### 4. Remote Development

```bash
# SSH into component
bns ssh --component <COMPONENT_ID>

# Port forwarding
bns port-forward 5432 --component <COMPONENT_ID>          # Same local/remote
bns port-forward 15432:5432 --component <COMPONENT_ID>    # Different local port
bns port-forward :5432 --component <COMPONENT_ID>         # Random local port

# Debug session
bns debug start --component <COMPONENT_ID>
bns debug stop
bns debug up    # Start SSH in debug mode
bns debug down  # Revert pod to original state
```

### 5. Variables & Secrets

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

## Output Format

Use `--output json` or `--output yaml` for scripting:
```bash
bns environments list --output json | jq '.[] | .id'
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
| Environment stuck deploying | Check `bns pipeline logs --id <PIPELINE_ID>` |
| Component not found in UI | Ensure Helm uses `--post-renderer /bns/helpers/helm/bns_post_renderer` |
| Variables not injecting | Check variable scope (project > environment > component) |

## Reference Files

- [CLI Reference](references/cli.md) - Complete CLI command documentation
- [API Reference](references/api.md) - REST API for CI/CD integrations
- [YAML Schema](references/yaml-schema.md) - bunnyshell.yaml configuration reference
- [Component Types](references/components.md) - Helm, Kubernetes, Terraform, Docker Compose, etc.
- [Variables & Interpolation](references/variables.md) - Secrets, scopes, Twig filters
