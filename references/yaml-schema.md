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

security:                   # Optional: IP restrictions
  access:
    allowedIps:
      - '192.168.0.0/24'

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
        selfManagedDns: false      # Use external DNS
        displayPaths: []           # Override displayed URLs
        k8s:
          ingress:
            className: bns-nginx
            tlsSecretName: string
            annotations: {}
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
