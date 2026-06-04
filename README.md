# Pangolin + Proxmox + Guacamole Runbook

Professional runbook for deploying Apache Guacamole on a hardened Proxmox host and exposing services via Pangolin with subdomain-based routing. This avoids path-based rewrites and keeps each service on its own hostname.

## Scope
Defines what is covered and what is not.

- Guacamole deployed via Docker Compose with hardened runtime flags
- TLS termination for Guacamole using an nginx sidecar (self-signed certificate)
- Pangolin/Newt exposure using subdomains (no path-based routing)
- Proxmox exposed via its native HTTPS service

## Architecture summary
Quick mental model for data flow and responsibility boundaries.

- Pangolin Cloud handles public DNS/edge and forwards traffic to Newt.
- Newt runs on the host and maintains the secure tunnel plus health checks.
- Proxmox serves its UI/API directly over HTTPS on its native port.
- Guacamole runs in Docker; guacd handles protocol traffic, Postgres stores config, nginx terminates TLS.

## Guacamole deployment (hardened)
Ensures a stable Guacamole stack on hardened kernels and exposes it safely.

Location: /opt/guacamole

### Docker Compose (example)
Defines the full Guacamole stack and the TLS front-end in one place.

```
services:
  guacd:
    image: guacamole/guacd:1.5.5
    restart: unless-stopped
    security_opt:
      - seccomp:unconfined
    privileged: true

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
Serves HTTPS to Pangolin/Newt and maps / to /guacamole/.

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

  # WebSocket tunnel endpoint for Guacamole
  location /websocket-tunnel {
    proxy_pass http://guacamole:8080/guacamole/websocket-tunnel;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Sec-WebSocket-Protocol $http_sec_websocket_protocol;
    proxy_set_header Sec-WebSocket-Extensions $http_sec_websocket_extensions;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_read_timeout 3600;
    proxy_send_timeout 3600;
    proxy_buffering off;
  }

  # Support explicit /guacamole/websocket-tunnel paths
  location /guacamole/websocket-tunnel {
    proxy_pass http://guacamole:8080/guacamole/websocket-tunnel;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Sec-WebSocket-Protocol $http_sec_websocket_protocol;
    proxy_set_header Sec-WebSocket-Extensions $http_sec_websocket_extensions;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_read_timeout 3600;
    proxy_send_timeout 3600;
    proxy_buffering off;
  }

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
Provides local TLS for health checks; public TLS is handled by Pangolin.

```
mkdir -p /opt/guacamole/certs
openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -keyout /opt/guacamole/certs/guac.key \
  -out /opt/guacamole/certs/guac.crt \
  -subj "/CN=guacamole.local"
```

### Database initialization
Guacamole will not authenticate without its schema and admin user.

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
Subdomains avoid path rewrite edge cases and keep services isolated.

- guacamole.contoso.com -> internal host 192.168.1.x:8081 (HTTPS)
- proxmox.contoso.com -> internal host 192.168.1.x:8006 (HTTPS)

No path-based routing or regex rewrites are required.

## Verification
Confirms stack health before exposing to users.

- Guacamole (local): curl -k https://127.0.0.1:8081/ returns 200
- WebSocket tunnel: curl -i -k -H 'Connection: Upgrade' -H 'Upgrade: websocket' -H 'Sec-WebSocket-Protocol: guacamole' https://127.0.0.1:8081/websocket-tunnel returns 101
- Pangolin target: healthy
- guacamole.contoso.com loads Guacamole login
- proxmox.contoso.com loads Proxmox login

## Troubleshooting
Common failure modes and their fixes.

- Laggy Guacamole sessions: WebSocket tunnel must return 101. If it falls back to HTTP tunnel, keyboard/mouse lag is severe.
- Pangolin shows unhealthy or 503: backend must speak HTTPS (Newt health checks use HTTPS).
- Guacamole connection fails with internal error: guacd needs seccomp:unconfined + privileged to open socket pairs for RDP/VNC.
- Guacamole shows raw translation keys: assets are not served; avoid path routing or preserve subpaths.
- Postgres errors: database does not exist; create the DB and load schema as shown above.
- Pangolin TLS shows Traefik default cert: remove and re-add the domain in Pangolin Cloud to re-issue the certificate.

## Pangolin API notes
Use the Integration API when automating hostnames/resources.

- Base URL: https://api.pangolin.net/v1
- Auth: Authorization: Bearer <API_KEY>
- Requires orgId (example: timsonner)
- Sites: GET /org/{orgId}/sites
- Domains: GET /org/{orgId}/domains
- Domains can only update certResolver on wildcard domains (non-wildcard updates return 400).
- If API returns 403 on site-resources, the key lacks Site Resource permissions.

Information needed from the user to automate:
- orgId
- siteId or site niceId
- API key with Site Resource permissions (if editing site-resources)
- Domains/hostnames to attach
