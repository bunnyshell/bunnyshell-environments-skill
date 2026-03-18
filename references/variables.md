# Bunnyshell Variables & Interpolation Reference

## Variable Scopes (Priority: Low → High)

```
Project Variables         ← Lowest priority, applies to all environments
    ↓
Environment Variables     ← Overrides project vars
    ↓
Environment Variable Groups  ← Must be explicitly included
    ↓
Component Variables       ← Highest priority, final value
```

## Secrets

### SECRET[] Syntax

```yaml
environmentVariables:
  PLAIN_VALUE: my_value
  SECRET_VALUE: SECRET["my-password"]
```

After saving, displays as encrypted:
```yaml
SECRET_VALUE: ENCRYPTED[bXktcGFzc3dvcmQ...]
```

### Quoting Rules

| Value | Syntax |
|-------|--------|
| `abcd` | `SECRET[abcd]` |
| `ab,cd` | `SECRET["ab,cd"]` (comma requires quotes) |
| `ab cd` | `SECRET["ab cd"]` (space requires quotes) |
| `ab"cd` | `SECRET['ab"cd']` or `SECRET["ab\"cd"]` |
| `ab'cd` | `SECRET["ab'cd"]` or `SECRET['ab\'cd']` |

### Common Mistakes

**CRITICAL: SECRET[] and ENCRYPTED[] are mutually exclusive — never combine them:**

```yaml
# CORRECT: Use SECRET[] with plaintext — Bunnyshell encrypts on save
DB_PASSWORD: SECRET["my-password"]

# CORRECT: Use ENCRYPTED[] with pre-encrypted value from `bns secrets encrypt`
DB_PASSWORD: ENCRYPTED[bXktcGFzc3dvcmQ...]

# WRONG: Wrapping ENCRYPTED[] in SECRET[] double-encrypts — the container
# receives the literal "ENCRYPTED[...]" string instead of the actual value
DB_PASSWORD: SECRET["ENCRYPTED[bXktcGFzc3dvcmQ...]"]  # CORRUPTED!
```

**When encrypting via stdin, use `echo -n` to avoid trailing newlines:**

```bash
# CORRECT
echo -n "my-password" | bns secrets encrypt --organization <ORG_ID>

# WRONG — encrypts "my-password\n" (with newline), causing auth failures
echo "my-password" | bns secrets encrypt --organization <ORG_ID>
```

### Where SECRET[] Works

- Project variables
- `templateVariables`
- `environmentVariablesGroups`
- `environmentVariables`
- Docker-compose build args and secrets
- Init/sidecar container environment
- Component environment

## Variable Naming Rules

- Alphanumeric, underscore, dash, dot
- Cannot start with digit
- Cannot start with `BNS_`
- Length: 3-255 characters

## Interpolation Syntax

```yaml
{{ expression }}                    # Simple interpolation
{{ expression | filter }}           # With filter
{% if condition %}...{% endif %}    # Conditional
```

## Available Values

### Organization

```yaml
{{ org.name }}
{{ org.unique }}
```

### Project

```yaml
{{ project.name }}
```

### Template Variables

```yaml
{{ template.vars.VAR_NAME }}
```

### Environment

```yaml
{{ env.unique }}              # Unique environment ID (e.g., "qmrxve")
{{ env.name }}                # Environment name
{{ env.type }}                # "primary" or "ephemeral"
{{ env.base_domain }}         # Auto-generated domain
{{ env.urlHandle }}           # URL handle
{{ env.k8s.namespace }}       # Kubernetes namespace
{{ env.k8s.version }}         # Kubernetes version (e.g., "1.33")
{{ env.vars.VAR_NAME }}       # Environment variable value
```

### Environment Variable Groups

```yaml
{{ env.varGroups.GROUP_NAME }}
{{ env.varGroups.GROUP_NAME.VAR_NAME }}
```

### Cluster Info

```yaml
{{ env.cluster.displayName }}
{{ env.cluster.name }}
{{ env.cluster.cloud }}       # aws, azure, gcp, etc.
```

### Components

```yaml
{{ components.NAME.name }}
{{ components.NAME.gitBranch }}
{{ components.NAME.gitBranchSha }}
{{ components.NAME.vars.VAR_NAME }}

# Hosts/URLs
{{ components.NAME.ingress.hosts[0].url }}
{{ components.NAME.ingress.hosts[0].hostname }}
{{ components.NAME.ingress.hosts[0].path }}

# Docker images (DockerImage, Application, Service, Database)
{{ components.NAME.image }}       # Full: nginx:abc123
{{ components.NAME.imageName }}   # Name: nginx
{{ components.NAME.imageTag }}    # Tag: abc123

# Exported variables (Helm, Kubernetes, Terraform, Generic)
{{ components.NAME.exported.VAR_NAME }}

# Container variables
{{ components.NAME.containers.CONTAINER.vars.VAR_NAME }}
```

### Component Self-Reference (Inside Component)

```yaml
{{ component.name }}
{{ component.gitBranch }}
{{ component.gitApplicationPath }}
{{ component.image }}
```

## Interpolation Filters

### String Filters

| Filter | Example | Result |
|--------|---------|--------|
| `lower` | `{{ "Hello" \| lower }}` | `hello` |
| `upper` | `{{ "Hello" \| upper }}` | `HELLO` |
| `capitalize` | `{{ "hello" \| capitalize }}` | `Hello` |
| `title` | `{{ "hello world" \| title }}` | `Hello World` |
| `trim` | `{{ "  hi  " \| trim }}` | `hi` |
| `slug` | `{{ "Hello World" \| slug }}` | `hello-world` |
| `replace` | `{{ "brown" \| replace({'brown': 'green'}) }}` | `green` |
| `indent` | `{{ "text" \| indent(4) }}` | `    text` |
| `slice` | `{{ "hello" \| slice(0,2) }}` | `he` |
| `default` | `{{ undefined \| default('N/A') }}` | `N/A` |
| `url_encode` | `{{ "a b" \| url_encode }}` | `a%20b` |
| `json_encode` | `{{ value \| json_encode }}` | JSON string |
| `yaml_encode` | `{{ value \| yaml_encode }}` | YAML string |

### Collection Filters

```yaml
# Merge arrays
{{ [1, 2] | merge([3, 4]) }}  # [1, 2, 3, 4]

# Merge objects
{{ {"a": 1} | merge({"b": 2}) }}  # {"a": 1, "b": 2}

# Slice
{{ [1, 2, 3, 4] | slice(1, 2) }}  # [2, 3]

# URL encode object
{{ {"page": 1, "q": "test"} | url_encode }}  # page=1&q=test

# Key-value format
{{ {"a": 1, "b": 2} | kv_encode({glue: " = "}) }}
# a = 1
# b = 2
```

### Date Filters

```yaml
{{ "now" | date('Y-m-d') }}                        # 2024-07-04
{{ "now" | date_modify('+1 day') | date('Y-m-d') }} # 2024-07-05
```

## Conditional Logic

```yaml
{% if env.type == "ephemeral" %}
  replicas: 1
{% else %}
  replicas: 3
{% endif %}
```

## Where to Use Interpolation

### Environment Variables

```yaml
environmentVariables:
  API_URL: 'https://api-{{ env.base_domain }}'
  ENV_TYPE: '{{ env.type }}'
```

### Docker Compose

```yaml
dockerCompose:
  build:
    context: '{{ component.gitApplicationPath }}'
    args:
      VERSION: '{{ env.vars.APP_VERSION }}'
    secrets:
      NPM_TOKEN: '{{ env.vars.NPM_TOKEN }}'
  environment:
    API_HOST: '{{ components.api.ingress.hosts[0].url }}'
```

### Helm/Kubernetes/Terraform Scripts

```yaml
deploy:
  - 'helm upgrade --namespace {{ env.k8s.namespace }} ...'
  - 'kubectl apply -n {{ env.k8s.namespace }} ...'
  - 'terraform apply -var "name={{ env.unique }}" ...'
```

### Hosts Configuration

```yaml
hosts:
  - hostname: 'api-{{ env.base_domain }}'
    path: '{{ env.vars.API_PATH }}'
    k8s:
      ingress:
        className: '{{ env.vars.INGRESS_CLASS }}'
```

### Remote Dev Configuration

```yaml
dev:
  api:
    - kind: Deployment
      name: 'api-{{ env.unique }}'
      namespace: '{{ env.k8s.namespace }}'
```

## Environment Variable Groups

### Define Groups

```yaml
environmentVariablesGroups:
  db-creds:
    MYSQL_USER: admin
    MYSQL_PASS: SECRET[password]
  cache-config:
    REDIS_HOST: redis
    REDIS_PORT: 6379
```

### Include in Components

```yaml
components:
  - name: api
    kind: Application
    dockerCompose:
      environmentVariablesGroups:
        - db-creds
        - cache-config
```

### Access via Interpolation

```yaml
{{ env.varGroups.db-creds.MYSQL_USER }}
```

## Common Patterns

### Organize Variables by Service (Recommended)

Use `environmentVariablesGroups` to group related variables by service. This keeps secrets and config co-located and avoids duplication:

```yaml
environmentVariablesGroups:
  mysql:
    DB_PASSWORD: SECRET['secret']
    MYSQL_DATABASE: my_app_db
    MYSQL_ROOT_PASSWORD: SECRET['rootpass']
  app:
    APP_KEY: SECRET['base64:...']
    APP_URL: 'https://{{ env.vars.BASE_DOMAIN }}'
    MONOLITH_BASE_URL: 'https://monolith.example.com'

environmentVariables:
  BASE_DOMAIN: my-app.example.com

components:
  - kind: Database
    name: mysql
    dockerCompose:
      environment:
        MYSQL_ROOT_PASSWORD: '{{ env.varGroups.mysql.DB_PASSWORD }}'
        MYSQL_DATABASE: '{{ env.varGroups.mysql.MYSQL_DATABASE }}'

  - kind: Application
    name: app
    dockerCompose:
      environment:
        APP_KEY: '{{ env.varGroups.app.APP_KEY }}'
        APP_URL: '{{ env.varGroups.app.APP_URL }}'
        DB_DATABASE: '{{ env.varGroups.mysql.MYSQL_DATABASE }}'
        DB_PASSWORD: '{{ env.varGroups.mysql.DB_PASSWORD }}'
    hosts:
      - hostname: 'app-{{ env.vars.BASE_DOMAIN }}'
```

**Why this pattern works:**
- Database credentials defined once in `mysql` group, referenced by both the database and app components
- `BASE_DOMAIN` as a top-level env var makes it easy to change the domain in one place
- `APP_URL` in the `app` group references `BASE_DOMAIN`, keeping URLs consistent
- After saving, `SECRET[]` values become `ENCRYPTED[...]` in the stored definition

### Database Connection String

```yaml
environmentVariables:
  DATABASE_URL: 'postgres://{{ env.vars.DB_USER }}:{{ env.vars.DB_PASS }}@{{ components.postgres.exported.HOST }}:5432/{{ env.vars.DB_NAME }}'
```

### Conditional Replicas

```yaml
deploy:
  - |
    cat << EOF > values.yaml
    replicas: {% if env.type == "ephemeral" %}1{% else %}3{% endif %}
    EOF
```

### Dynamic Hostnames

```yaml
hosts:
  - hostname: '{{ env.vars.SERVICE_PREFIX }}-{{ env.base_domain }}'
```

### Using Exported Variables

```yaml
components:
  - kind: Terraform
    name: rds
    exportVariables: [DB_HOST, DB_PORT]
    deploy:
      - 'DB_HOST=terraform output --raw endpoint'
      - 'DB_PORT=terraform output --raw port'

  - kind: Application
    name: api
    dependsOn: [rds]
    dockerCompose:
      environment:
        DATABASE_HOST: '{{ components.rds.exported.DB_HOST }}'
        DATABASE_PORT: '{{ components.rds.exported.DB_PORT }}'
```
