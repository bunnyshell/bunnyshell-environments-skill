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

# Create
bns environments create-from-template \
  --template <TEMPLATE_ID> \
  --name "My Env" \
  --project <PROJECT_ID> \
  --k8s-integration <CLUSTER_ID>

# Clone
bns environments clone --id <ENV_ID> --name "Cloned Env"

# Lifecycle
bns environments deploy --id <ENV_ID> --wait
bns environments stop --id <ENV_ID> --wait
bns environments start --id <ENV_ID> --wait
bns environments delete --id <ENV_ID> --force

# Configuration
bns environments export-configuration --id <ENV_ID> > bunnyshell.yaml
bns environments update-configuration --id <ENV_ID> < bunnyshell.yaml

# Variables
bns environments show-variables --id <ENV_ID>
bns environments set-variable --id <ENV_ID> --name KEY --value VALUE

# Settings
bns environments update --id <ENV_ID> --auto-deploy true
bns environments update --id <ENV_ID> --auto-stop true --auto-stop-hours 8
```

## Components

```bash
# List & show
bns components list --environment <ENV_ID>
bns components show --id <COMPONENT_ID>

# Operations
bns components redeploy --id <COMPONENT_ID>
bns components delete --id <COMPONENT_ID>

# Logs & exec
bns components logs --id <COMPONENT_ID>
bns components exec --id <COMPONENT_ID> -- <COMMAND>
```

## Pipelines

```bash
bns pipeline list --environment <ENV_ID>
bns pipeline show --id <PIPELINE_ID>
bns pipeline logs --id <PIPELINE_ID> --follow
bns pipeline cancel --id <PIPELINE_ID>
```

## Kubernetes Clusters

```bash
bns k8s-clusters list --organization <ORG_ID>
bns k8s-clusters show --id <CLUSTER_ID>
```

## Variables

```bash
# Environment variables
bns variables list --environment <ENV_ID>
bns variables create --environment <ENV_ID> --name KEY --value VALUE
bns variables update --id <VAR_ID> --value NEW_VALUE
bns variables delete --id <VAR_ID>

# Import from file
bns variables import --var-file=/path/to/vars.env
bns variables import --secret-file=/path/to/secrets.env
bns variables import --var-file=vars.env --secret-file=secrets.env --ignore-duplicates
```

## Project Variables

```bash
bns project-variables list --project <PROJECT_ID>
bns project-variables create --project <PROJECT_ID> --name KEY --value VALUE
bns project-variables update --id <VAR_ID> --value NEW_VALUE
bns project-variables delete --id <VAR_ID>
```

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
bns debug up    # Start SSH connection in debug
bns debug down  # Revert pod to original state
```

## Remote Development

```bash
bns remote-development start --component <COMPONENT_ID>
bns remote-development stop
bns remote-development status
```

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
