# Bunnyshell CLI Reference

## Global Flags

```
--configFile string   Config file (default "$HOME/.bunnyshell/config.yaml")
-d, --debug           Debug network requests
-h, --help            Help
--no-progress         Disable progress spinners
--non-interactive     Disable interactive terminal
-o, --output string   Output format: stylish | json | yaml (default "stylish")
--profile string      Use profile from config file
-v, --verbose         Increase log verbosity
```

## Configuration & Profiles

```bash
# Add profile
bns configure profiles add \
  --name <NAME> \
  --token <TOKEN> \
  --default \
  --organization <ORG_ID> \
  --project <PROJECT_ID> \
  --environment <ENV_ID>

# List profiles
bns configure profiles list

# Set default profile
bns configure profiles default

# Remove profile
bns configure profiles remove

# Manage context
bns configure profiles context
```

## Organizations

```bash
bns organizations list
bns organizations show --id <ORG_ID>
```

## Projects

```bash
bns projects list --organization <ORG_ID>
bns projects show --id <PROJECT_ID>
bns projects create --name "My Project" --organization <ORG_ID>
```

## Environments

```bash
# List & show
bns environments list --project <PROJECT_ID>
bns environments show --id <ENV_ID>

# Create from template
bns environments create \
  --from-template <TEMPLATE_ID> \
  --name "My Env" \
  --project <PROJECT_ID> \
  --k8s <CLUSTER_ID>

# Create from local file
bns environments create \
  --from-path bunnyshell.yaml \
  --name "My Env" \
  --project <PROJECT_ID> \
  --k8s <CLUSTER_ID>

# Clone
bns environments clone --id <ENV_ID> --name "Cloned Env"

# Lifecycle
bns environments deploy --id <ENV_ID> --wait
bns environments stop --id <ENV_ID> --wait
bns environments start --id <ENV_ID> --wait
bns environments delete --id <ENV_ID> --force

# Configuration (export/import)
bns environments definition --id <ENV_ID> > bunnyshell.yaml
bns environments update-configuration --id <ENV_ID> --from-path bunnyshell.yaml

# Settings
bns environments update-settings --id <ENV_ID> --auto-deploy-ephemerals
bns environments update-settings --id <ENV_ID> --termination-protection
```

**Note:** Environment variables are managed via `bns variables` commands (see Variables section below).

## Components

```bash
# List & show
bns components list --environment <ENV_ID>
bns components show --id <COMPONENT_ID>

# Update component
bns components update --id <COMPONENT_ID>

# SSH into component
bns components ssh --id <COMPONENT_ID>
bns components ssh --id <COMPONENT_ID> --container <CONTAINER_NAME>

# Port forwarding via component
bns components port-forward --id <COMPONENT_ID> 8080:80
```

**Note:** `bns components redeploy`, `delete`, `logs`, and `exec` do not exist. Use:
- For redeployment: `bns environments deploy` or redeploy via web UI
- For logs: `bns logs --component <COMPONENT_ID>` (top-level command)
- For exec: `bns ssh --component <COMPONENT_ID> --shell "command"` or `kubectl exec`

## Pipelines

```bash
# List & show
bns pipeline list --environment <ENV_ID>
bns pipeline show --id <PIPELINE_ID>

# List jobs in a pipeline
bns pipeline jobs --id <PIPELINE_ID>

# Monitor progress (waits until completion)
bns pipeline monitor --id <PIPELINE_ID>
bns pipeline monitor --id <PIPELINE_ID> --interval 5s  # Custom check interval

# View logs (interactive job selection when --environment or --id used)
bns pipeline logs --environment <ENV_ID>               # Pick from latest pipeline
bns pipeline logs --id <PIPELINE_ID>                   # Pick from specific pipeline
bns pipeline logs --job <JOB_ID>                       # Specific job directly
bns pipeline logs --environment <ENV_ID> --failed      # Only failed jobs
bns pipeline logs --environment <ENV_ID> -f            # Follow in real-time
bns pipeline logs --job <JOB_ID> --step deploy         # Filter by step name
bns pipeline logs --job <JOB_ID> --tail 50             # Last N lines per step
bns pipeline logs --job <JOB_ID> -o json               # Output: stylish|json|yaml|raw
```

**Note:** `bns pipeline cancel` does not exist. Use the web UI to cancel pipelines.

## Kubernetes Clusters

```bash
bns k8s-clusters list --organization <ORG_ID>
bns k8s-clusters show --id <CLUSTER_ID>
```

## Variables

```bash
# Environment variables
bns variables list --environment <ENV_ID>
bns variables show --id <VAR_ID>
bns variables create --environment <ENV_ID> --name KEY --value VALUE
bns variables edit --id <VAR_ID> --value NEW_VALUE
bns variables edit --id <VAR_ID> --secret              # Mark as secret
bns variables delete --id <VAR_ID>

# Import from file
bns variables import --var-file=/path/to/vars.env
bns variables import --secret-file=/path/to/secrets.env
bns variables import --var-file=vars.env --secret-file=secrets.env --ignore-duplicates
```

**Note:** Use `bns variables edit` (not `update`) to modify existing variables.

## Project Variables

```bash
bns project-variables list --project <PROJECT_ID>
bns project-variables show --id <VAR_ID>
bns project-variables create --project <PROJECT_ID> --name KEY --value VALUE
bns project-variables edit --id <VAR_ID> --value NEW_VALUE
bns project-variables delete --id <VAR_ID>
```

**Note:** Use `bns project-variables edit` (not `update`) to modify existing variables.

## Secrets

```bash
# Encrypt/decrypt standalone values
bns secrets encrypt "my password" --organization <ORG_ID>
cat mysecret.txt | bns secrets encrypt --organization <ORG_ID>
bns secrets decrypt "ENCRYPTED[...]" --organization <ORG_ID>

# Encrypt/decrypt definition files
bns secrets encrypt-definition --file bunnyshell.yaml --organization <ORG_ID>
bns secrets decrypt-definition --file bunnyshell.yaml --organization <ORG_ID>
bns secrets decrypt-definition --file bunnyshell.yaml --resolved --organization <ORG_ID>
```

## Logs (Runtime / Container)

```bash
# Single component
bns logs --component <COMPONENT_ID>

# All components in an environment
bns logs --environment <ENV_ID>

# Filter by component name (repeatable, requires --environment)
bns logs --environment <ENV_ID> --name api --name worker

# Follow / stream
bns logs --component <COMPONENT_ID> -f

# Last N lines
bns logs --component <COMPONENT_ID> --tail 200

# Time-based filtering
bns logs --component <COMPONENT_ID> --since 5m
bns logs --component <COMPONENT_ID> --since-time 2026-02-18T10:00:00Z

# Specific container / all containers
bns logs --component <COMPONENT_ID> -c <CONTAINER_NAME>
bns logs --component <COMPONENT_ID> --all-containers

# Include timestamps / previous container
bns logs --component <COMPONENT_ID> --timestamps
bns logs --component <COMPONENT_ID> --previous

# Formatting
bns logs --component <COMPONENT_ID> --no-color
bns logs --component <COMPONENT_ID> --prefix=false
```

**Note:** `bns components logs` does not exist. Use the top-level `bns logs` command instead.

## SSH

```bash
bns ssh --component <COMPONENT_ID>
bns ssh --component <COMPONENT_ID> --shell /bin/bash
```

## Port Forwarding

```bash
# Basic forwarding
bns port-forward 5432                              # Same local/remote port
bns port-forward 15432:5432                        # Different local port
bns port-forward :5432                             # Random local port
bns port-forward 15432:5432 5432 :5432             # Multiple forwards

# With component (non-interactive)
bns port-forward :5432 --component <COMPONENT_ID>
```

## Debug (Remote Development)

```bash
bns debug start --component <COMPONENT_ID>
bns debug start --component <COMPONENT_ID> --shell /bin/bash
bns debug start --component <COMPONENT_ID> --force-recreate-resource
bns debug stop
```

**Note:** `bns debug up` and `bns debug down` do not exist. Use `start`/`stop` instead.

## Remote Development

```bash
bns remote-development up --component <COMPONENT_ID>
bns remote-development down
bns remote-development config   # Manage remote development config
```

**Note:** Commands are `up`/`down` (not `start`/`stop`). `status` does not exist.

## Git Operations

```bash
bns git info
bns git branch
```

## Shell Completion

```bash
# ZSH
echo 'source <(bns completion zsh)' >> ~/.zshrc
echo 'compdef _bns bunnyshell-cli' >> ~/.zshrc

# Bash
echo 'source <(bns completion bash)' >> ~/.bashrc
```
