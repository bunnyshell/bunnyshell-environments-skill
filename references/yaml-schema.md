# bunnyshell.yaml Schema Reference

## Top-Level Structure

```yaml
kind: Environment           # Required: always "Environment"
name: string                # Required: environment name
type: primary | ephemeral   # Required: environment type

deployment:                 # Optional: Kubernetes deployment strategy
  strategy:
    type: RollingUpdate | Recreate
    maxUnavailable: int | percent    # Only for RollingUpdate
    maxSurge: int | percent          # Only for RollingUpdate

environmentVariables:       # Optional: environment-level variables
  KEY: value
  SECRET_KEY: SECRET["value"]

environmentVariablesGroups: # Optional: reusable variable groups
  group-name:
    KEY: value

components: []              # Required: list of components

volumes: []                 # Optional: persistent volumes

security:                   # Optional: environment-wide IP restrictions
  access:
    allowedIps:
      - '192.168.0.0/24'    # All hosts restricted unless public: true
    realIpTrustedProxies: null  # Optional: trusted proxy CIDRs for real IP detection

dev: {}                     # Optional: remote development config
```

## Components Common Attributes

All component types share:

```yaml
components:
  - kind: string          # Required: component type
    name: string          # Required: unique component name
    version: v1           # Optional: parser version

    gitRepo: string       # Git repository URL (https://)
    gitBranch: string     # Branch, tag, or commit SHA
    gitApplicationPath: / # Path within repository

    dependsOn: []         # Components that must deploy first

    hosts:                # Public URL configuration
      - hostname: string  # Must include {{ env.base_domain }} or custom domain
        path: /
        servicePort: 8080
        public: false     # Bypass allowedIps restriction
        externalAddress: string    # CNAME target
        selfManagedDns: false      # Required: true when using custom domains
        displayPaths: []           # Override displayed URLs
        k8s:
          ingress:
            className: bns-nginx
            tlsSecretName: string  # K8s secret with TLS cert, or '' for cert-manager auto
            annotations: {}        # e.g. cert-manager.io/cluster-issuer: letsencrypt-prod
```

## Docker Compose Components

### Application / Service / Database

```yaml
- kind: Application | Service | Database
  name: my-app
  gitRepo: 'https://github.com/org/repo.git'
  gitBranch: main
  gitApplicationPath: /

  dockerCompose:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        BUILD_ENV: production
      secrets:
        NPM_TOKEN: SECRET[token]
    image: nginx:latest          # Or use pre-built image
    environment:
      KEY: value
    environmentVariablesGroups:
      - my-group
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: 2
          memory: 2000M
        reservations:
          cpus: '0.5'
          memory: 500M

  volumes:
    - name: data-volume
      mount: /var/lib/data
      subPath: ''

  cronJobs:
    - name: cleanup
      schedule: '0 * * * *'
      command: ['/bin/sh', '-c', 'cleanup.sh']
      containers: [my-app]
      volumes: true
      executionTimeout: 120

  hpa:                           # Horizontal Pod Autoscaler
    minReplicas: 1
    maxReplicas: 5
    metrics:
      - cpu: 70%
      - memory: 80%

  hosts:
    - hostname: 'app-{{ env.base_domain }}'
      path: /
      servicePort: 8080
```

### InitContainer / SidecarContainer

```yaml
- kind: InitContainer
  name: init-db
  gitRepo: 'https://github.com/org/repo.git'
  gitBranch: main
  dockerCompose:
    build:
      context: ./init

- kind: Application
  name: api
  pod:
    init_containers:
      - from: init-db
        name: custom-name
        environment:
          INIT_MODE: full
        shared_paths:
          - path: /init/data
            target:
              path: /app/data
              container: '@parent'
            initial_contents: '@self'
    sidecar_containers:
      - from: sidecar-name
        name: custom-sidecar
```

#### shared_paths.initial_contents

Controls which container's files populate the shared path on startup:

| Value | Meaning | Use When |
|-------|---------|----------|
| `@self` | Copy from the sidecar/init container | Sidecar owns the files (e.g., config injector) |
| `@target` | Copy from the parent container | Parent owns the files (e.g., app builds assets, sidecar serves them) |

#### Sidecar Port Exposure

**Important:** When a sidecar exposes a port (e.g., nginx on 8080), the parent Application component must also declare that port in its `ports` list, since they share the same pod network:

```yaml
- kind: Application
  name: app
  dockerCompose:
    ports:
      - '9000:9000'    # app's own port
      - '8080:8080'    # sidecar's port — must be declared here too
  pod:
    sidecar_containers:
      - from: nginx
        name: nginx
  hosts:
    - hostname: 'app-{{ env.base_domain }}'
      path: /
      servicePort: 8080  # routes to the sidecar's port
```

### PHP-FPM + Nginx Sidecar (Complete Example)

The most common pattern for Laravel/Symfony/PHP applications. Nginx and PHP-FPM run in the same pod, sharing the application directory.

```yaml
components:
  - kind: Database
    name: mysql
    dockerCompose:
      image: 'mysql:8.0'
      environment:
        MYSQL_ROOT_PASSWORD: '{{ env.varGroups.mysql.DB_PASSWORD }}'
        MYSQL_DATABASE: '{{ env.varGroups.mysql.MYSQL_DATABASE }}'
      ports:
        - '3306:3306'
    volumes:
      - name: mysql-data
        mount: /var/lib/mysql
        subPath: ''

  - kind: SidecarContainer
    name: nginx
    gitRepo: 'https://github.com/org/repo.git'
    gitBranch: main
    gitApplicationPath: docker/nginx
    dockerCompose:
      build:
        context: docker/nginx
      ports:
        - '8080:8080'

  - kind: Application
    name: app
    gitRepo: 'https://github.com/org/repo.git'
    gitBranch: main
    gitApplicationPath: /
    dockerCompose:
      build:
        context: .
        dockerfile: Dockerfile
      environment:
        APP_URL: '{{ env.varGroups.app.APP_URL }}'
        DB_HOST: mysql
        DB_DATABASE: '{{ env.varGroups.mysql.MYSQL_DATABASE }}'
        DB_PASSWORD: '{{ env.varGroups.mysql.DB_PASSWORD }}'
      ports:
        - '9000:9000'   # PHP-FPM
        - '8080:8080'   # Nginx sidecar port
    pod:
      sidecar_containers:
        - from: nginx
          name: nginx
          shared_paths:
            - path: /app
              target:
                path: /app
                container: '@parent'
              initial_contents: '@target'  # Copy built assets FROM app TO nginx
    dependsOn:
      - mysql
    hosts:
      - hostname: 'app-{{ env.base_domain }}'
        path: /
        servicePort: 8080
```

**Nginx sidecar config** must point to `localhost:9000` (same pod), not the service name:

```nginx
upstream php {
    server localhost:9000;
}
```

**Framework gotcha:** PHP frameworks behind a Kubernetes ingress (which terminates TLS) will see requests as HTTP. This causes asset URLs to use `http://` instead of `https://`, resulting in mixed-content errors and blank pages. Fix per framework:

| Framework | Fix |
|-----------|-----|
| Laravel | `$middleware->trustProxies(at: '*')` in `bootstrap/app.php` |
| Symfony | `framework.trusted_proxies: 'REMOTE_ADDR'` in config |
| Generic PHP | Check `X-Forwarded-Proto` header |

## Helm Component

```yaml
- kind: Helm
  name: my-release
  runnerImage: 'dtzar/helm-kubectl:3.8.2'
  gitRepo: 'https://github.com/org/charts.git'  # Or omit for registry
  gitBranch: main
  gitApplicationPath: /charts/myapp

  deploy:
    - |
      cat << EOF > values.yaml
        image: {{ components.my-image.image }}
        replicas: 2
        ingress:
          host: api-{{ env.base_domain }}
      EOF
    - 'helm upgrade --install --namespace {{ env.k8s.namespace }} --dependency-update --post-renderer /bns/helpers/helm/bns_post_renderer -f values.yaml my-release-{{ env.unique }} .'
    - 'SERVICE_URL=https://api-{{ env.base_domain }}'

  destroy:
    - 'helm uninstall my-release-{{ env.unique }} --namespace {{ env.k8s.namespace }}'

  start:
    - 'helm upgrade --namespace {{ env.k8s.namespace }} --post-renderer /bns/helpers/helm/bns_post_renderer --reuse-values --set replicas=2 my-release-{{ env.unique }} .'

  stop:
    - 'helm upgrade --namespace {{ env.k8s.namespace }} --post-renderer /bns/helpers/helm/bns_post_renderer --reuse-values --set replicas=0 my-release-{{ env.unique }} .'

  exportVariables:
    - SERVICE_URL

  environment:
    KUBECONFIG: /root/.kube/config
```

## KubernetesManifest Component

```yaml
- kind: KubernetesManifest
  name: myapp
  runnerImage: 'alpine/k8s:1.22.15'
  gitRepo: 'https://github.com/org/manifests.git'
  gitBranch: main
  gitApplicationPath: /k8s

  deploy:
    - '/bns/helpers/kubernetes/bns_kustomize_init --namespace {{ env.k8s.namespace }}'
    - 'kustomize edit set image app={{ components.app-image.image }}'
    - 'kubectl apply -k .'
    - 'ENDPOINT=https://myapp-{{ env.base_domain }}'

  destroy:
    - 'kustomize create --autodetect --recursive --namespace {{ env.k8s.namespace }}'
    - 'kubectl delete -k .'

  start:
    - 'kubectl scale --replicas=2 --namespace {{ env.k8s.namespace }} deployment/myapp'

  stop:
    - 'kubectl scale --replicas=0 --namespace {{ env.k8s.namespace }} deployment/myapp'

  exportVariables:
    - ENDPOINT
```

## Terraform Component

```yaml
- kind: Terraform
  name: s3-bucket
  runnerImage: 'hashicorp/terraform:1.5.1'
  gitRepo: 'https://github.com/org/infra.git'
  gitBranch: main
  gitApplicationPath: /terraform/s3

  deploy:
    - 'cd terraform/s3'
    - '/bns/helpers/terraform/get_managed_backend > zz_backend_override.tf'
    - 'terraform init -input=false -no-color'
    - 'terraform apply -var "name=bucket-{{ env.unique }}" -input=false -auto-approve -no-color'
    - 'BNS_TF_STATE_LIST=terraform show -json'
    - 'BUCKET_NAME=terraform output --raw bucket_name'

  destroy:
    - 'cd terraform/s3'
    - '/bns/helpers/terraform/get_managed_backend > zz_backend_override.tf'
    - 'terraform init -input=false -no-color'
    - 'terraform destroy -var "name=bucket-{{ env.unique }}" -input=false -auto-approve -no-color'

  exportVariables:
    - BUCKET_NAME

  environment:
    AWS_ACCESS_KEY_ID: '{{ env.vars.AWS_KEY }}'
    AWS_SECRET_ACCESS_KEY: '{{ env.vars.AWS_SECRET }}'
    AWS_REGION: us-east-1
```

## GenericComponent

```yaml
- kind: GenericComponent
  name: cloud-function
  runnerImage: 'google/cloud-sdk:alpine'
  gitRepo: 'https://github.com/org/functions.git'
  gitBranch: main

  deploy:
    - 'gcloud auth activate-service-account --key-file /tmp/key.json'
    - 'gcloud run deploy fn-{{ env.unique }} --image={{ components.fn-image.image }} --region us-central1'
    - 'SERVICE_URL=$(gcloud run services describe fn-{{ env.unique }} --format "value(status.url)")'

  destroy:
    - 'gcloud run services delete fn-{{ env.unique }} --quiet --region us-central1'

  exportVariables:
    - SERVICE_URL

  environment:
    GCLOUD_KEY: '{{ env.vars.GCLOUD_SA_KEY }}'
```

## DockerImage (Build-Only)

```yaml
- kind: DockerImage
  name: api-image
  gitRepo: 'https://github.com/org/api.git'
  gitBranch: main
  gitApplicationPath: /
  context: /
  dockerfile: Dockerfile
  target: production
  args:
    BUILD_ENV: production
  secrets:
    NPM_TOKEN: SECRET[token]
```

## StaticApplication

```yaml
- kind: StaticApplication
  name: frontend
  gitRepo: 'https://github.com/org/frontend.git'
  gitBranch: main
  gitApplicationPath: /

  buildNodeVersion: lts
  buildCommand: 'npm run build'
  buildOutputDir: dist
  buildArguments:
    VITE_API_URL: '{{ components.api.ingress.hosts[0].url }}'
  buildSettings:
    NPM_TOKEN: SECRET[token]

  hosts:
    - hostname: 'app-{{ env.base_domain }}'
      path: /
      servicePort: 8080
```

## Volumes

```yaml
volumes:
  - name: db-data
    size: 10Gi
    type: disk
```

| Type | Backing | Access Mode | Use When |
|------|---------|-------------|----------|
| `disk` | Block storage (EBS, PD) | ReadWriteOnce | Default; single-pod workloads (databases) |
| `network` | Network filesystem (EFS, Filestore) | ReadWriteMany | Multiple pods need the same volume, or pod rescheduling across nodes |

## Custom Domains and SSL

### Custom Domain with ExternalDNS

Use `selfManagedDns: true` when using your own domain. Bunnyshell will not create DNS records - use ExternalDNS or manage DNS manually.

```yaml
hosts:
  - hostname: 'api.example.com'
    path: /
    servicePort: 8080
    selfManagedDns: true
```

### SSL with Existing Certificate

Reference a Kubernetes secret containing your TLS certificate:

```yaml
hosts:
  - hostname: 'api.example.com'
    path: /
    servicePort: 8080
    selfManagedDns: true
    k8s:
      ingress:
        tlsSecretName: my-custom-domain-cert
```

### SSL with cert-manager (Auto-Generated Certificates)

Use cert-manager to automatically provision and renew certificates. Set `tlsSecretName` to empty string and add the cluster-issuer annotation:

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

**Prerequisites:** Cluster must have cert-manager installed with a ClusterIssuer named `letsencrypt-prod` (or your issuer name).

**Note:** Let's Encrypt has rate limits. For ephemeral environments, use a wildcard certificate (`*.example.com`) with DNS challenge instead of per-environment certificates.

### SSL with cert-manager + Named TLS Secret

When you want cert-manager to store the certificate in a specific named secret:

```yaml
hosts:
  - hostname: 'app-{{ env.vars.BASE_DOMAIN }}'
    path: /
    servicePort: 8080
    selfManagedDns: true
    k8s:
      ingress:
        tlsSecretName: tls-secret
        annotations:
          cert-manager.io/cluster-issuer: letsencrypt-prod
          nginx.ingress.kubernetes.io/proxy-body-size: 50m  # increase for file uploads
```

### Common Ingress Annotations

| Annotation | Purpose | Default |
|-----------|---------|---------|
| `cert-manager.io/cluster-issuer` | Auto-provision TLS certificate | — |
| `nginx.ingress.kubernetes.io/proxy-body-size` | Max upload size | `1m` |
| `nginx.ingress.kubernetes.io/proxy-read-timeout` | Backend read timeout | `60` |
| `nginx.ingress.kubernetes.io/whitelist-source-range` | IP allowlist (CIDR) | — |

### IP Restrictions

**Option 1: Environment-wide restriction**

Restricts all hosts unless `public: true` is set on the host:

```yaml
security:
  access:
    allowedIps:
      - '192.168.0.0/24'
      - '10.0.0.1/32'
```

**Option 2: Per-host restriction (nginx annotation)**

Restricts access to a specific host only:

```yaml
hosts:
  - hostname: 'admin.example.com'
    path: /
    servicePort: 8080
    selfManagedDns: true
    k8s:
      ingress:
        tlsSecretName: ''
        annotations:
          cert-manager.io/cluster-issuer: letsencrypt-prod
          nginx.ingress.kubernetes.io/whitelist-source-range: '192.168.1.0/24,10.0.0.1/32'
```

Use comma-separated CIDR ranges for multiple IPs.

## Remote Development Config

```yaml
dev:
  api:                           # Component name
    - containers:
        api:                     # Container name
          remoteDevProfile: api_rdev
          command: ['php-fpm']
          syncPaths:
            - remotePath: /var/www/
              localPath: ~/project
          portForwards:
            - "9003<9003"        # Reverse (remote to local)
            - "80>8080"          # Forward (local to remote)
          environment:
            DEBUG: 'true'
          resources:
            limits:
              memory: 750M
            requests:
              cpu: '0.15'
              memory: 500M

  helm-component:                # For Helm/Kubernetes components
    - kind: Deployment
      name: 'api-{{ env.unique }}'
      namespace: '{{ env.k8s.namespace }}'
      containers:
        api:
          command: ['npm', 'run', 'dev']
          syncPaths:
            - remotePath: /app
              localPath: ~/project
```
