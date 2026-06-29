---
name: raspberry-deploy
description: |
  **Raspberry Pi Deployment Rules**: Defines rules, templates, and Traefik configurations for deploying applications to the self-hosted Raspberry Pi runner.
  Ensures that:
  - Secrets are managed via GitHub Secrets and written to .env files at deploy time.
  - Deployments run on the self-hosted runner, with branch-based isolation.
  - Subdomains use hyphenated single-level format (e.g., dev-app.20112013.xyz) to match wildcard certificates.
  - docker-compose configurations use optional env_file to prevent missing-file crashes.
  - Persistent volumes map to SSD storage isolated by project and branch.
license: Apache-2.0
metadata:
  version: v2
  publisher: jmateosj
---

# Raspberry Pi Deployment Rules (GitOps, Traefik & Docker)

This skill defines the mandatory rules, patterns, and configurations for all applications deployed on the self-hosted Raspberry Pi runner. Any agent attempting to create, update, or troubleshoot deployments must follow these rules strictly.

---

## 🔑 1. GitHub Secrets & Dynamic `.env` Generation

- **Zero committed secrets**: Credentials, database passwords, and API keys must **never** be committed to git.
- **Dynamic file construction**: Workflows running on the self-hosted runner must dynamically write secrets to a `.env` file in the workspace directory.
- **URL Encoding for Database Passwords**: If a database password contains special characters (like `@`, `:`, `/`, etc.), it must be URL-encoded before constructing connection strings. Python is pre-installed on the Raspberry Pi and must be used to perform this encoding:

### Recommended Workflow Step for `.env` Generation:
```yaml
      - name: Create Environment Variables
        run: |
          # Encode database password for safe URI connection string
          DB_PASS_ENCODED=$(python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1], safe=''))" "${{ secrets.DB_PASS }}")
          
          echo "DB_USER=${{ secrets.DB_USER }}" > .env
          echo "DB_PASS=${{ secrets.DB_PASS }}" >> .env
          echo "DATABASE_URL=postgresql://${{ secrets.DB_USER }}:${DB_PASS_ENCODED}@postgres:5432/${{ secrets.DB_NAME }}" >> .env
          echo "AUTH_SECRET=${{ secrets.AUTH_SECRET }}" >> .env
          # (Add other application-specific variables)
```

---

## 🐙 2. Branch-Based Isolation

To prevent conflicts between the development (`dev`) and production (`main`) versions of the same application on the single Raspberry Pi:

1. **Docker Compose Project Name (`-p`)**:
   Always pass a branch-specific project name to `docker compose`. This avoids container name collisions and groups containers logically.
   - Production: `docker compose -p [appname]-main ...`
   - Development: `docker compose -p [appname]-dev ...`
   
2. **Persistent Storage in External SSD**:
   All persistent data (PostgreSQL, Redis, uploads, certs) must be stored in the external SSD under `/mnt/ssd/` and isolated by project and branch:
   - Volume path format: `/mnt/ssd/[appname]/${BRANCH}/[service_name]`
   - Example: `/mnt/ssd/sat-manager/${BRANCH:-dev}/postgres:/var/lib/postgresql/data`

---

## 🌐 3. Traefik Routing & Wildcard SSL

The Raspberry Pi has a single Traefik reverse proxy that manages certificates and routes incoming requests.

1. **Host Header Formatting**:
   - Cloudflare and Let's Encrypt certificates are generated for the wildcard domain `*.20112013.xyz`.
   - Wildcards cover only **one level** of subdomains.
   - **Main/Prod Branch**: `[appname].20112013.xyz`
   - **Dev Branch**: `dev-[appname].20112013.xyz` (or `[branch]-[appname].20112013.xyz`).
   - ⚠️ **DO NOT** use `dev.[appname].20112013.xyz` as it is a two-level subdomain and will fail SSL handshake.

2. **Network Mapping**:
   - The application container must connect to the external network `proxy` in addition to any internal application networks.
   - Define `proxy` as external at the bottom of `docker-compose.prod.yml`:
     ```yaml
     networks:
       proxy:
         external: true
     ```

3. **Required Traefik Labels (App Container)**:
   ```yaml
   labels:
     - "traefik.enable=true"
     - "traefik.docker.network=proxy"
     - "traefik.http.routers.${APP_NAME}-${BRANCH}.rule=Host(`${DOMAIN}`)"
     - "traefik.http.routers.${APP_NAME}-${BRANCH}.entrypoints=web,websecure"
     - "traefik.http.routers.${APP_NAME}-${BRANCH}.tls=true"
     - "traefik.http.routers.${APP_NAME}-${BRANCH}.tls.certresolver=myresolver"
     # Specify the internal port the container listens on (e.g. NextJS default 3000)
     - "traefik.http.services.${APP_NAME}-${BRANCH}.loadbalancer.server.port=3000"
   ```

---

## 🐳 4. Resilient Docker Compose Patterns

1. **Optional Environment Files**:
   To avoid container crashes if `.env` or `.env.local` files do not exist at startup time, mark them as optional using the `required: false` directive:
   ```yaml
   services:
     app:
       env_file:
         - path: .env
           required: false
         - path: .env.local
           required: false
   ```

2. **Resource Limits**:
   Since the Raspberry Pi has shared memory/CPU resources, always specify reasonable memory limits on heavy services:
   ```yaml
   deploy:
     resources:
       limits:
         memory: 512M
   ```

3. **Database Migration and Seeding**:
   Use Docker compose profiles or isolated one-off containers to run migrations and seeds after container startup.
   - Example:
     ```yaml
     seeder:
       image: node:20-alpine
       profiles: ["tools"]
       working_dir: /app
       volumes:
         - .:/app
       environment:
         DATABASE_URL: ${DATABASE_URL}
       command: sh -c "npm install && npx prisma db push --accept-data-loss && npx tsx prisma/seed.ts"
       networks:
         - app_internal
     ```
     Execute in the workflow via: `docker compose -p app-${branch} -f docker-compose.prod.yml run --rm seeder`
