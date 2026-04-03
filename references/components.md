# Bunnyshell Component Types Reference

## Component Type Selection Guide

| Type | Use Case |
|------|----------|
| `Application` | Git-stored code requiring builds |
| `Service` | Third-party software (caches, queues, proxies) |
| `Database` | Persistent data stores |
| `DockerImage` | Build images for other components |
| `StaticApplication` | Nginx-served HTML/CSS/JS |
| `Helm` | Deploy Helm charts |
| `KubernetesManifest` | Raw Kubernetes manifests |
| `Terraform` | Infrastructure-as-code |
| `GenericComponent` | Fully custom bash workflows |
| `CustomDockerImage` | Full control over image builds |
| `InitContainer` | Initialization before main container |
| `SidecarContainer` | Co-located helper container |

## Lifecycle Scripts

Components with `runnerImage` support lifecycle scripts:

| Script | When Run | Purpose |
|--------|----------|---------|
| `deploy` | On deploy | Create/update resources |
| `destroy` | On delete | Clean up resources |
| `start` | On start | Resume stopped environment |
| `stop` | On stop | Pause environment, preserve data |

## Component Dependencies

Control execution order with `dependsOn`:

```yaml
components:
  - kind: Database
    name: postgres

  - kind: GenericComponent
    name: migrations
    dependsOn: [postgres]

  - kind: Application
    name: api
    dependsOn: [migrations]

  - kind: Application
    name: frontend
    dependsOn: [api]
```

**Execution order:**
- Deploy: postgres → migrations → api → frontend
- Start: postgres → migrations → api → frontend  
- Stop: All in parallel
- Destroy: All in parallel

## Exported Variables

Components can export variables for other components:

```yaml
# Terraform component
- kind: Terraform
  name: database
  exportVariables:
    - DB_HOST
    - DB_PORT
  deploy:
    - 'DB_HOST=terraform output --raw host'
    - 'DB_PORT=terraform output --raw port'

# Application using exported variables
- kind: Application
  name: api
  dependsOn: [database]
  dockerCompose:
    environment:
      DATABASE_HOST: '{{ components.database.exported.DB_HOST }}'
      DATABASE_PORT: '{{ components.database.exported.DB_PORT }}'
```

## Bunnyshell Helpers

### Helm Post-Renderer

Required for Bunnyshell UI to discover Helm resources:

```yaml
deploy:
  - 'helm upgrade --install ... --post-renderer /bns/helpers/helm/bns_post_renderer ...'
```

### Kubernetes Kustomize Init

Adds required labels for resource discovery:

```yaml
deploy:
  - '/bns/helpers/kubernetes/bns_kustomize_init --namespace {{ env.k8s.namespace }}'
  - 'kubectl apply -k .'
```

### Terraform Backend

Bunnyshell-managed state backend:

```yaml
deploy:
  - '/bns/helpers/terraform/get_managed_backend > zz_backend_override.tf'
  - 'terraform init -input=false'
```

Capture state for UI:

```yaml
deploy:
  - 'BNS_TF_STATE_LIST=terraform show -json'
```

## Docker Compose Key Support

Bunnyshell uses **Kompose** under the hood to translate Docker Compose definitions into Kubernetes resources. This means any Docker Compose key that Kompose supports will work, even if not explicitly listed below.

### Explicitly Documented Keys

| Key | Kubernetes Mapping |
|-----|-------------------|
| `build`, `build:context`, `build:dockerfile`, `build:args` | Builds/pushes to Docker repository |
| `command` | Pod.Spec.Container.Args |
| `container_name` | Metadata.Name |
| `deploy:replicas` | Deployment.Spec.Replicas |
| `deploy:resources` | Containers.Resources.Limits |
| `environment` | Environment variables |
| `expose`, `ports` | Service.Spec.Ports |
| `image` | Deployment.Spec.Containers.Image |
| `volumes` | PersistentVolumeClaim |
| `configs` | ConfigMap |
| `secrets` | Secret |
| `tmpfs` | emptyDir with Memory medium |

### Additional Kompose-Supported Keys

These are not documented by Bunnyshell but work because Kompose translates them:

| Key | Kubernetes Mapping |
|-----|-------------------|
| `privileged: true` | `securityContext.privileged: true` |
| `cap_add` | `securityContext.capabilities.add` |
| `cap_drop` | `securityContext.capabilities.drop` |
| `devices` | Volume mounts for devices |
| `user` | `securityContext.runAsUser` |
| `working_dir` | Container `workingDir` |
| `entrypoint` | Container `command` |
| `stdin_open` | Container `stdin` |
| `tty` | Container `tty` |

### Not Supported

- `depends_on` (use `dependsOn` at component level)
- `healthcheck`
- `networks`, `network_mode`
- `links`, `external_links`
- `restart` (Kubernetes handles this)
- `logging` (use Kubernetes built-in)

## Runner Images

Recommended images for script-based components:

| Use Case | Image |
|----------|-------|
| Helm | `dtzar/helm-kubectl:3.8.2` |
| Kubernetes | `alpine/k8s:1.22.15` |
| Terraform | `hashicorp/terraform:1.5.1` |
| AWS CLI | `amazon/aws-cli:latest` |
| GCloud | `google/cloud-sdk:alpine` |
| Azure | `mcr.microsoft.com/azure-cli:latest` |
| Generic | `ubuntu:22.04` |

### Using Component's Built Image

Use `@self` to run scripts in the component's own built image:

```yaml
- kind: GenericComponent
  name: my-component
  runnerImage: '@self'
  dockerCompose:
    build:
      context: .
  deploy:
    - './my-deploy-script.sh'
```

## HPA (Horizontal Pod Autoscaler)

```yaml
hpa:
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - cpu: 70%                    # CPU percentage
    - memory: 80%                 # Memory percentage
    - type: ContainerResource
      cpu: 75m
      container: nginx
    - type: Pods
      metricName: requests_per_second
      averageValue: '100'
  scaleUp:
    policies:
      - value: 100%
        periodSeconds: 30
  scaleDown:
    stabilizationWindowSeconds: 300
    selectPolicy: Min
    policies:
      - value: 25%
        periodSeconds: 30
```

## CronJobs

```yaml
cronJobs:
  - name: cleanup
    schedule: '0 * * * *'         # Cron expression
    command: ['/bin/sh', '-c', 'cleanup.sh']
    containers: [api]             # Which container to run in
    volumes: true                 # Mount volumes
    executionTimeout: 120         # Seconds
    resources:
      limits:
        cpus: 1
        memory: 512M
```

## Runner Image Authentication

For private registries:

```yaml
- kind: CustomDockerImage
  name: my-image
  runnerImage: 'private.registry.com/builder:latest'
  runnerImageAuth:
    provider: docker_hub | aws | gcp | azure | github | gitlab | harbor | jfrog
    username: myuser
    password: SECRET[mypassword]
```
