# Framework Deployment Checklists

Pre-deployment checklists for common frameworks on Bunnyshell. Check these before your first deploy to avoid iterative debugging.

## Laravel

- [ ] Dockerfile includes all required PHP extensions (`pdo_pgsql`, `pdo_mysql`, `redis`, `gd`, etc.)
- [ ] pecl extensions (e.g., `redis`) need `$PHPIZE_DEPS`: `apk add --virtual .build-deps $PHPIZE_DEPS`
- [ ] `APP_KEY` is set (generate with `php artisan key:generate --show`)
- [ ] `SESSION_DRIVER` is set to `array`, `redis`, or `file` (default `database` requires sessions migration)
- [ ] `trustProxies` middleware configured for K8s ingress TLS termination: `$middleware->trustProxies(at: '*')` in `bootstrap/app.php`
- [ ] If using Passport: auto-generate keys in entrypoint with correct permissions (`660`, `www-data`)
- [ ] Run migrations post-deploy: `bns exec <ID> -c <CONTAINER> -- php artisan migrate --force`
- [ ] `APP_URL` matches the Bunnyshell host URL (use interpolation: `https://{{ env.vars.BASE_DOMAIN }}`)
- [ ] For queue workers: ensure `QUEUE_CONNECTION` is set and Redis/database is reachable

## Symfony

- [ ] `APP_SECRET` is set
- [ ] `trusted_proxies` configured: `framework.trusted_proxies: 'REMOTE_ADDR'` with `trusted_headers: ['x-forwarded-for', 'x-forwarded-proto']`
- [ ] `DATABASE_URL` uses correct driver and host (use Bunnyshell interpolation for DB component)
- [ ] Run migrations post-deploy: `bns exec <ID> -c <CONTAINER> -- php bin/console doctrine:migrations:migrate --no-interaction`
- [ ] Cache clear after deploy: `php bin/console cache:clear --env=prod`

## Node.js / Express

- [ ] `app.set('trust proxy', true)` for correct `req.protocol` behind K8s ingress
- [ ] `NODE_ENV` set to `production`
- [ ] Health check endpoint exists (for K8s liveness/readiness probes)
- [ ] If using Prisma: run `npx prisma migrate deploy` post-deploy
- [ ] Ensure the app listens on `0.0.0.0`, not `localhost` (required for container networking)

## Next.js

- [ ] `NEXTAUTH_URL` / `NEXT_PUBLIC_*` env vars set at build time (baked into the bundle)
- [ ] Standalone output mode in `next.config.js`: `output: 'standalone'`
- [ ] Server listens on `0.0.0.0` (Next.js default is fine, but verify custom servers)
- [ ] If using API routes with DB: ensure `DATABASE_URL` is available at runtime

## Django

- [ ] `SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')` for TLS behind ingress
- [ ] `ALLOWED_HOSTS` includes the Bunnyshell domain
- [ ] `SECRET_KEY` is set via environment variable
- [ ] Run migrations post-deploy: `bns exec <ID> -c <CONTAINER> -- python manage.py migrate --noinput`
- [ ] Static files collected: `python manage.py collectstatic --noinput` (in Dockerfile or entrypoint)
- [ ] `CSRF_TRUSTED_ORIGINS` includes `https://<your-bunnyshell-domain>`

## Rails

- [ ] `config.force_ssl = true` or `ActionDispatch::RemoteIp` middleware for TLS behind ingress
- [ ] `SECRET_KEY_BASE` is set
- [ ] `RAILS_ENV=production`
- [ ] Run migrations post-deploy: `bns exec <ID> -c <CONTAINER> -- rails db:migrate`
- [ ] Asset precompilation in Dockerfile: `rails assets:precompile`
- [ ] Database adapter gem included (`pg`, `mysql2`, etc.)

## General (All Frameworks)

- [ ] Database component uses a persistent volume — password changes require volume wipe (see troubleshooting.md)
- [ ] Secrets encrypted with `bns secrets encrypt` go directly in YAML as `ENCRYPTED[...]` — do NOT wrap in `SECRET[]`
- [ ] When piping secrets via stdin: use `echo -n` to avoid trailing newline
- [ ] For multi-container pods (e.g., PHP-FPM + Nginx): always use `bns exec <ID> -c <CONTAINER>` to avoid interactive hang
- [ ] Verify the app binds to `0.0.0.0` (not `127.0.0.1` or `localhost`)
- [ ] Check that exposed ports in `bunnyshell.yaml` match what the app actually listens on
