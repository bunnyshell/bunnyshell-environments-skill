# Bunnyshell API Reference

## Authentication

All requests require a Bearer token in the `Authorization` header.

Get your token from: https://environments.bunnyshell.com/access-token

```bash
Authorization: Bearer YOUR_API_TOKEN
```

## Base URL

```
https://api.environments.bunnyshell.com/v1
```

## Common Operations

### List Environments

```bash
curl -X GET "https://api.environments.bunnyshell.com/v1/environments?project=PROJECT_ID" \
  -H "Authorization: Bearer $TOKEN"
```

Response:
```json
{
  "data": [
    {
      "id": "8AEJQ1oXW7",
      "name": "staging",
      "type": "primary",
      "operationStatus": "running",
      "project": "vK4JwdXoGE",
      "kubernetesIntegration": "abc123"
    }
  ]
}
```

### Create Environment from Template

```bash
curl -X POST "https://api.environments.bunnyshell.com/v1/environments/from-template" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-environment",
    "project": "PROJECT_ID",
    "template": "TEMPLATE_ID",
    "kubernetesIntegration": "CLUSTER_ID"
  }'
```

### Create Environment from Configuration

```bash
curl -X POST "https://api.environments.bunnyshell.com/v1/environments" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-environment",
    "project": "PROJECT_ID",
    "kubernetesIntegration": "CLUSTER_ID",
    "configuration": "kind: Environment\nname: my-environment\ntype: primary\ncomponents: []"
  }'
```

### Deploy Environment

```bash
curl -X POST "https://api.environments.bunnyshell.com/v1/environments/ENV_ID/deploy" \
  -H "Authorization: Bearer $TOKEN"
```

Response includes pipeline ID for tracking:
```json
{
  "id": "8AEJQ1oXW7",
  "operationStatus": "deploy_in_progress",
  "lastPipeline": "PIPELINE_ID"
}
```

### Stop Environment

```bash
curl -X POST "https://api.environments.bunnyshell.com/v1/environments/ENV_ID/stop" \
  -H "Authorization: Bearer $TOKEN"
```

### Start Environment

```bash
curl -X POST "https://api.environments.bunnyshell.com/v1/environments/ENV_ID/start" \
  -H "Authorization: Bearer $TOKEN"
```

### Delete Environment

```bash
curl -X DELETE "https://api.environments.bunnyshell.com/v1/environments/ENV_ID" \
  -H "Authorization: Bearer $TOKEN"
```

### Get Environment Details

```bash
curl -X GET "https://api.environments.bunnyshell.com/v1/environments/ENV_ID" \
  -H "Authorization: Bearer $TOKEN"
```

## Environment Variables

### List Environment Variables

```bash
curl -X GET "https://api.environments.bunnyshell.com/v1/variables?environment=ENV_ID" \
  -H "Authorization: Bearer $TOKEN"
```

### Create Environment Variable

```bash
curl -X POST "https://api.environments.bunnyshell.com/v1/variables" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "API_KEY",
    "value": "my-secret-value",
    "isSecret": true,
    "environment": "ENV_ID"
  }'
```

### Update Environment Variable

```bash
curl -X PATCH "https://api.environments.bunnyshell.com/v1/variables/VAR_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/merge-patch+json" \
  -d '{
    "value": "new-value"
  }'
```

### Delete Environment Variable

```bash
curl -X DELETE "https://api.environments.bunnyshell.com/v1/variables/VAR_ID" \
  -H "Authorization: Bearer $TOKEN"
```

## Pipeline Status

### Get Pipeline Details

```bash
curl -X GET "https://api.environments.bunnyshell.com/v1/pipelines/PIPELINE_ID" \
  -H "Authorization: Bearer $TOKEN"
```

Response:
```json
{
  "id": "PIPELINE_ID",
  "status": "running",
  "type": "deploy",
  "environment": "ENV_ID"
}
```

Pipeline statuses: `queued`, `running`, `succeeded`, `failed`, `cancelled`

### Get Pipeline Logs

```bash
curl -X GET "https://api.environments.bunnyshell.com/v1/pipelines/PIPELINE_ID/logs" \
  -H "Authorization: Bearer $TOKEN"
```

## Common Patterns

### Create and Deploy in One Flow

```bash
# 1. Create environment
ENV_RESPONSE=$(curl -s -X POST "https://api.environments.bunnyshell.com/v1/environments/from-template" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "pr-123",
    "project": "PROJECT_ID",
    "template": "TEMPLATE_ID",
    "kubernetesIntegration": "CLUSTER_ID"
  }')

ENV_ID=$(echo $ENV_RESPONSE | jq -r '.id')

# 2. Set variables
curl -X POST "https://api.environments.bunnyshell.com/v1/variables" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"BRANCH\",
    \"value\": \"feature/my-branch\",
    \"environment\": \"$ENV_ID\"
  }"

# 3. Deploy
curl -X POST "https://api.environments.bunnyshell.com/v1/environments/$ENV_ID/deploy" \
  -H "Authorization: Bearer $TOKEN"
```

### Poll for Deploy Completion

```bash
while true; do
  STATUS=$(curl -s -X GET "https://api.environments.bunnyshell.com/v1/environments/$ENV_ID" \
    -H "Authorization: Bearer $TOKEN" | jq -r '.operationStatus')
  
  if [[ "$STATUS" == "running" ]]; then
    echo "Deploy complete"
    break
  elif [[ "$STATUS" == *"failed"* ]]; then
    echo "Deploy failed"
    exit 1
  fi
  
  sleep 10
done
```

## Error Responses

```json
{
  "type": "https://tools.ietf.org/html/rfc2616#section-10",
  "title": "An error occurred",
  "status": 400,
  "detail": "Validation failed"
}
```

| Status | Meaning |
|--------|---------|
| 400 | Bad request / validation error |
| 401 | Invalid or missing token |
| 403 | Permission denied |
| 404 | Resource not found |
| 409 | Conflict (e.g., environment already deploying) |
