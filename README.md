# Pangolin + Proxmox + Guacamole Runbook

Professional runbook for deploying Apache Guacamole on a hardened Proxmox host and exposing services via Pangolin with subdomain-based routing. This avoids path-based rewrites and keeps each service on its own hostname.

## Scope
Why: establishes what is covered and what is not.

- Guacamole deployed via Docker Compose with hardened runtime flags
- TLS termination for Guacamole using an nginx sidecar (self-signed certificate)
- Pangolin/Newt exposure using subdomains (no path-based routing)
- Proxmox exposed via its native HTTPS service

## Architecture summary
Why: quick mental model for data flow and responsibility boundaries.

- Pangolin Cloud handles public DNS/edge and forwards traffic to Newt.
- Newt runs on the host and maintains the secure tunnel plus health checks.
- Proxmox serves its UI/API directly over HTTPS on its native port.
- Guacamole runs in Docker; guacd handles protocol traffic, Postgres stores config, nginx terminates TLS.

## Guacamole deployment (hardened)
Why: ensures a stable Guacamole stack on hardened kernels and exposes it safely.

Location: /opt/guacamole

### Docker Compose (example)
Why: defines the full Guacamole stack and the TLS front-end in one place.

```
services:
  guacd:
    image: guacamole/guacd:1.5.5
    restart: unless-stopped

  db:
    image: postgres:15
    restart: unless-stopped
    command: ["postgres", "-c", "unix_socket_directories="]
    environment:
      POSTGRES_DB: guacamole
      POSTGRES_USER: guacamole
      POSTGRES_PASSWORD: <CHANGE_ME>
    volumes:
      - db-data:/var/lib/postgresql/data

  guacamole:
    image: guacamole/guacamole:1.5.5
    restart: unless-stopped
    depends_on:
      - guacd
      - db
    environment:
      GUACD_HOSTNAME: guacd
      POSTGRES_HOSTNAME: db
      POSTGRES_DATABASE: guacamole
      POSTGRES_USER: guacamole
      POSTGRES_PASSWORD: <CHANGE_ME>
      JAVA_TOOL_OPTIONS: "-XX:-UseContainerSupport -Dorg.apache.catalina.core.AsyncContextImpl.USE_NIO=false"
    security_opt:
      - seccomp:unconfined
    privileged: true

  web:
    image: nginx:1.27-alpine
    restart: unless-stopped
    depends_on:
      - guacamole
    ports:
      - "8081:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
    security_opt:
      - seccomp:unconfined
    privileged: true

volumes:
  db-data:
```

### nginx.conf (TLS termination and proxy)
Why: serves HTTPS to Pangolin/Newt and maps / to /guacamole/.

```
server {
  listen 80;
  server_name _;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  server_name _;

  ssl_certificate     /etc/nginx/certs/guac.crt;
  ssl_certificate_key /etc/nginx/certs/guac.key;

  location / {
    proxy_pass http://guacamole:8080/guacamole/;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_redirect off;
  }
}
```

### Self-signed certs
Why: provides local TLS for health checks; public TLS is handled by Pangolin.

```
mkdir -p /opt/guacamole/certs
openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -keyout /opt/guacamole/certs/guac.key \
  -out /opt/guacamole/certs/guac.crt \
  -subj "/CN=guacamole.local"
```

### Database initialization
Why: Guacamole will not authenticate without its schema and admin user.

Guacamole needs its schema loaded into Postgres:
```
# Create database
PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -U guacamole -d postgres -c "CREATE DATABASE guacamole;"

# Load schema
cat /opt/guacamole/postgresql/schema/001-create-schema.sql \
  /opt/guacamole/postgresql/schema/002-create-admin-user.sql | \
  PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -U guacamole -d guacamole
```

Password rotation note:
- If you change POSTGRES_PASSWORD, run `docker compose up -d` (recreate) so guacamole.properties regenerates.
- A simple `restart` can leave stale credentials inside the guacamole container.

### Default login
- Username: guacadmin
- Password: guacadmin
- Change immediately

## Pangolin routing (subdomains)
Why: subdomains avoid path rewrite edge cases and keep services isolated.

- guacamole.contoso.com -> internal host 192.168.1.x:8081 (HTTPS)
- proxmox.contoso.com -> internal host 192.168.1.x:8006 (HTTPS)

No path-based routing or regex rewrites are required.

## Verification
Why: confirms stack health before exposing to users.

- Guacamole (local): curl -k https://127.0.0.1:8081/ returns 200
- Pangolin target: healthy
- guacamole.contoso.com loads Guacamole login
- proxmox.contoso.com loads Proxmox login and realm dropdown

## Troubleshooting
Why: common failure modes and their fixes.

- Pangolin shows unhealthy or 503: backend must speak HTTPS (Newt health checks use HTTPS).
- Guacamole shows raw translation keys: assets are not served; avoid path routing or preserve subpaths.
- Postgres errors: database does not exist; create the DB and load schema as shown above.
