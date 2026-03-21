# Template Update Playbook

## Standard Update Process

1. Review current `bunnyshell.yaml` and `template.yaml`
2. Identify outdated Docker images (compare against [image-versions.md](image-versions.md))
3. Check component dependency map (which other templates share these components)
4. Apply best practices checklist (below)
5. Deploy to test environment
6. Verify all components healthy
7. Test application functionality (CRUD, auth, etc.)
8. Document any issues/workarounds in [troubleshooting.md](troubleshooting.md)
9. Tear down
10. Commit

## Best Practices Checklist

### bunnyshell.yaml

- [ ] All Docker images use specific version tags (never `:latest` for databases/runtimes)
- [ ] All images are latest stable LTS versions
- [ ] Runner images are up to date
- [ ] Database credentials use `environmentVariables` section (not hardcoded in component)
- [ ] Passwords use `SECRET[]` syntax where possible
- [ ] Shared values defined once at environment level, referenced via `{{ env.vars.X }}`
- [ ] Host references use `components.X.ingress.hosts[0].hostname` (not just `hosts[0]`)
- [ ] All hostname patterns follow: `prefix-{{ env.base_domain }}`
- [ ] Application components have resource limits (`deploy.resources.limits.memory`)
- [ ] Volume sizes are appropriate
- [ ] Volume types correct: `disk` for single-pod, `network` for multi-pod
- [ ] `security.access.allowedIps` present with guidance comment
- [ ] Consistent YAML formatting (2-space indent)
- [ ] Components ordered logically (databases first, then APIs, then frontends)
- [ ] `dependsOn` used where there are deployment order requirements
- [ ] `dev` section present for Application components

### template.yaml

- [ ] `name` matches the environment name
- [ ] `description` is accurate and up-to-date
- [ ] `tags` cover all technologies used
- [ ] `icons` list all stack technologies
- [ ] `stack.packages` versions match actual versions used
- [ ] `discoverable: true`

### Dockerfiles

- [ ] Base images use specific version tags
- [ ] Alpine-based images preferred
- [ ] Multi-stage builds (dev / build / prod)
- [ ] VS Code Remote Development tools in dev stage
- [ ] `.dockerignore` present and comprehensive
- [ ] EXPOSE declarations match actual ports

## MySQL 8.4 Migration Guide

MySQL 8.4 removes the `--default-authentication-plugin` flag.

**Before (MySQL 8.0):**
```yaml
command: ['--default-authentication-plugin=mysql_native_password']
```

**After (MySQL 8.4):**
```yaml
command: ['--mysql-native-password=ON']
```

## mailhog to Mailpit Migration

mailhog is abandoned. Mailpit is the active successor and is a drop-in replacement.

- Image: `mailhog/mailhog:latest` -> `axllent/mailpit:latest`
- SMTP port: 1025 (unchanged)
- Web UI port: 8025 (unchanged)
- MAILER_DSN: `smtp://mailhog:1025` -> `smtp://mailpit:1025`
- Update component name references from `mailhog` to `mailpit`

## Verified Deploy-Test Workflow

```bash
# Source environment variables
source .env

# Create environment from local config
bns environments create \
  --name "test-<template-name>" \
  --from-path .bunnyshell/templates/<template>/bunnyshell.yaml \
  --project <PROJECT_ID> \
  --k8s <CLUSTER_ID> \
  --output json | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'ENV ID: {d[\"id\"]}')"

# Deploy (non-blocking)
bns environments deploy --id <ENV_ID> --no-wait

# Monitor status
bns environments show --id <ENV_ID>  # check operationStatus

# List components (may need --organization flag)
bns components list --environment <ENV_ID> --organization <ORG_ID> --output json

# Show component URLs
bns components show --id <COMPONENT_ID>

# Verify endpoints
curl -sI https://<hostname>/

# Tear down
bns environments delete --id <ENV_ID> --no-wait
```

## Template vs Deployed Config Pattern

When templates need different security or config between the committed version and live deployments:

1. Commit templates with open defaults (e.g., `allowedIps: 0.0.0.0/0`)
2. Create a temp copy with overrides for deployment:
   ```bash
   cp bunnyshell.yaml /tmp/deploy.yaml
   sed -i 's|0.0.0.0/0|185.181.102.37/32|' /tmp/deploy.yaml
   bns environments update-configuration --id <ENV_ID> --from-path /tmp/deploy.yaml --deploy
   ```
3. This keeps templates reusable while allowing per-deployment restrictions

## Efficient Multi-Environment Deploy

When deploying multiple environments in parallel:

```bash
# Create all environments (parallel)
for yaml in template1/bunnyshell.yaml template2/bunnyshell.yaml; do
  bns environments create --from-path $yaml --name "..." --project <PID> --k8s <KID> &
done
wait

# Deploy all (non-blocking)
for env_id in ENV1 ENV2 ENV3; do
  bns environments deploy --id $env_id --no-wait
done

# Check statuses
for env_id in ENV1 ENV2 ENV3; do
  bns environments show --id $env_id --output json | \
    python3 -c "import sys,json; d=json.load(sys.stdin); print(f\"{d['name']}: {d['operationStatus']}\")"
done
```

## One-Step Update + Deploy

The most efficient way to update config and redeploy:
```bash
bns environments update-configuration --id <ENV_ID> --from-path bunnyshell.yaml --deploy
```
This waits for completion and prints endpoints when done.

## Common Pitfalls

- **Bitnami images**: As of Aug 2025, Bitnami restructured Docker Hub. Use official images instead of `bitnami/*` with version tags.
- **bns components list**: May return empty without `--organization` flag
- **bns components show**: Requires `--id` flag, positional arg triggers interactive mode
- **ttyd not in apt**: The `ttyd` web terminal package is not in Debian bookworm repos. Install binary from GitHub releases (see [troubleshooting.md](troubleshooting.md#web-terminal-ttyd-setup))
- **Go version mismatches**: Always check `go.mod` requirements before using `go install` in multi-stage Dockerfiles
- **Always local build first**: Run `docker build --platform linux/amd64` locally before deploying to catch build errors early
- **`gitRepo` must be remote**: `--from-path` reads YAML locally but the `gitRepo` field must point to an accessible remote Git repo (Bunnyshell pulls source remotely to build)
- **Component URLs**: Public URLs only appear in `bns components show --id <ID>`, not in list output
