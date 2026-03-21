# Recommended Docker Image Versions

Last verified: 2026-02-14

## Databases

| Image | Recommended Version | Notes |
|---|---|---|
| PostgreSQL | `postgres:17-alpine` | LTS |
| MySQL | `mysql:8.4` | Use `--mysql-native-password=ON` for backward compat |
| MongoDB | `mongo:7.0` | LTS. Requires Mongoose 7+ driver |
| Redis | `redis:7.4-alpine` | |
| MinIO | `minio/minio:latest` or `minio/minio:RELEASE.2025-02-07T23-21-09Z` | **Switched from bitnami/minio** - Bitnami restructured Docker Hub (Aug 2025), versioned tags moved to unsupported `bitnamilegacy`. Use official MinIO image. Note: env var names differ from bitnami (see troubleshooting). |

## Runtimes

| Image | Recommended Version | Notes |
|---|---|---|
| Node.js | `node:22-alpine` | LTS (Oct 2024) |
| PHP | `php:8.3-fpm-alpine` | |
| Java | `eclipse-temurin:21-jdk-alpine` | LTS. Replaces deprecated `openjdk` images |
| Python/PyTorch | `bitnami/pytorch:2.5` or `pytorch/pytorch:2.5.1` | |

## Web Servers

| Image | Recommended Version | Notes |
|---|---|---|
| Nginx | `nginx:1.27-alpine` | Pin to specific minor for reproducibility |

## Runner Images

| Image | Recommended Version | Notes |
|---|---|---|
| Terraform | `hashicorp/terraform:1.9` | Latest 1.x line |
| Helm/kubectl | `dtzar/helm-kubectl:3.16` | Latest Helm 3 |
| Kubernetes tools | `alpine/k8s:1.31` | Match current K8s stable |

## Email Testing

| Old Image | Replacement | Notes |
|---|---|---|
| `mailhog/mailhog:latest` | `axllent/mailpit:latest` | Drop-in replacement. Same ports: 1025 (SMTP), 8025 (web UI) |

## Developer Tools

| Tool | Recommended Version | Install Method | Notes |
|---|---|---|---|
| ttyd (web terminal) | `1.7.7` | Binary from [GitHub releases](https://github.com/tsl0922/ttyd/releases) | NOT in Debian apt repos. Use: `wget -qO /usr/local/bin/ttyd https://github.com/tsl0922/ttyd/releases/download/1.7.7/ttyd.x86_64 && chmod +x /usr/local/bin/ttyd`. Supports basic auth via `-c user:pass` flag. |

## Magento-Specific

| Component | Recommended Image | Notes |
|---|---|---|
| PHP-FPM | `magento/magento-cloud-docker-php:8.3-fpm-*` | Magento 2.4.7+ supports PHP 8.3 |
| PHP CLI | `magento/magento-cloud-docker-php:8.3-cli-*` | For init containers |
| Search | `opensearchproject/opensearch:2.17` | Replaces Elasticsearch 7.x (EOL) |
| Nginx | Check `magento/magento-cloud-docker-nginx:1.27-*` | Or use stock nginx with custom config |
