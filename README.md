# Pangolin + Proxmox + Guacamole Runbook

This repository is a runbook for deploying Apache Guacamole on a Proxmox host and publishing Guacamole and the Proxmox web UI through Pangolin. Each service uses its own hostname (`guacamole.example.com`, `proxmox.example.com`) so traffic stays on subdomain routing.

Pangolin Cloud handles public DNS and the edge, then forwards traffic to Newt. Newt runs on the host, keeps the tunnel, and health-checks the backends. Proxmox serves its UI and API over its native HTTPS port. Guacamole runs in Docker: `guacd` carries the remote-desktop protocols, Postgres stores Guacamole data, and an nginx sidecar terminates TLS for the tunnel.

The repository contains this runbook and `.gitignore`. It does not contain application source, a Compose file, an nginx config file, scripts, or tests. Copy the snippets below onto the Proxmox host.

## Prerequisites

On the Proxmox host:

- Docker with the Compose plugin
- `openssl`
- `psql` (PostgreSQL client), able to reach Postgres at `127.0.0.1` as the `guacamole` user
- A Pangolin site with Newt connected to that host

Images named in this runbook:

- `guacamole/guacd:1.5.5`
- `postgres:15`
- `guacamole/guacamole:1.5.5`
- `nginx:1.27-alpine`

Published hostnames:

- `guacamole.example.com` to `https://<proxmox-host>:8081`
- `proxmox.example.com` to `https://<proxmox-host>:8006`

## Install, build, run, and test

There is no package manifest, Dockerfile, Makefile, or test suite in this repository. Nothing here is compiled or packaged. The commands below are the host procedure.

### Certificate

The self-signed certificate is for local TLS and Newt health checks. Public TLS stays with Pangolin.

```bash
mkdir -p /opt/guacamole/certs
openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -keyout /opt/guacamole/certs/guac.key \
  -out /opt/guacamole/certs/guac.crt \
  -subj "/CN=guacamole.example.com"
```

### Compose file

Save this as `/opt/guacamole/docker-compose.yml`. Replace `<CHANGE_ME>` with the Postgres password before starting the stack.

`guacd` and the Guacamole container use `seccomp:unconfined` and `privileged: true` so `guacd` can open the socket pairs used for RDP and VNC. Postgres is started with an empty `unix_socket_directories` value. `JAVA_TOOL_OPTIONS` turns off container memory detection and Catalina NIO async.

```yaml
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

The Compose file does not publish Postgres to the host. The database commands in the next section assume a `psql` client can already reach Postgres on `127.0.0.1`.

### nginx config

Save this as `/opt/guacamole/nginx.conf`. It redirects port 80 to HTTPS, proxies `/` to `/guacamole/`, and proxies both WebSocket tunnel paths to `guacamole:8080`.

```nginx
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

### Start the stack

```bash
cd /opt/guacamole
docker compose up -d
```

After a Postgres password change, run `docker compose up -d` again so the Guacamole container recreates `guacamole.properties`. A container restart leaves the previous credentials in place.

### Database initialization

Guacamole authentication needs the schema and the admin user loaded into Postgres.

```bash
PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -U guacamole -d postgres -c "CREATE DATABASE guacamole;"

cat /opt/guacamole/postgresql/schema/001-create-schema.sql \
  /opt/guacamole/postgresql/schema/002-create-admin-user.sql | \
  PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -U guacamole -d guacamole
```

`$POSTGRES_PASSWORD` must match the `<CHANGE_ME>` value in the Compose file. The schema files are the Guacamole Postgres schema files on the host at those paths.

### Checks

From the Proxmox host, after the stack is up:

```bash
curl -k https://127.0.0.1:8081/
curl -i -k \
  -H 'Connection: Upgrade' \
  -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Protocol: guacamole' \
  https://127.0.0.1:8081/websocket-tunnel
```

The first command should return HTTP 200. The second should return HTTP 101. In Pangolin, the Guacamole target should be healthy. `https://guacamole.example.com` should show the Guacamole login. `https://proxmox.example.com` should show the Proxmox login.

## Usage

Sign in to Guacamole with the default account, then change it:

- Username: `guacadmin`
- Password: `guacadmin`

Routing:

- `guacamole.example.com` goes to `https://<proxmox-host>:8081` (nginx in front of Guacamole).
- `proxmox.example.com` goes to `https://<proxmox-host>:8006` (Proxmox native HTTPS).

Hostname routing is sufficient. Path rewrites are not part of this setup.

### Pangolin Integration API

Use the Integration API when automating hostnames and resources.

- Base URL: `https://<pangolin-api-host>/v1`
- Auth header: `Authorization: Bearer <API_KEY>`
- Org id: `<org-id>`
- Sites: `GET /org/<org-id>/sites`
- Domains: `GET /org/<org-id>/domains`

`certResolver` updates apply to wildcard domains. A non-wildcard domain update returns HTTP 400. An HTTP 403 on site resources means the API key is missing Site Resource permissions.

To automate a change, you need:

- `<org-id>`
- A site id or site nice id
- An API key with Site Resource permissions when editing site resources
- The hostnames to attach

### Troubleshooting

- Laggy Guacamole sessions: the WebSocket check above must return HTTP 101. An HTTP tunnel fallback makes keyboard and mouse input lag.
- Pangolin shows the target unhealthy or returns HTTP 503: the backend must speak HTTPS. Newt health checks use HTTPS.
- Guacamole reports an internal error on connect: `guacd` needs `seccomp:unconfined` and `privileged: true`.
- Guacamole shows raw translation keys: static assets are not being served. Keep Guacamole on its own hostname and preserve subpaths.
- Postgres says the database does not exist: create `guacamole` and load the schema with the commands above.
- Pangolin presents the Traefik default certificate: remove the domain in Pangolin Cloud and add it again so the certificate is reissued.

## Repository layout

```text
.
├── .gitignore
└── README.md
```

`.gitignore` ignores `.DS_Store`, `*.log`, `*.env`, `*.key`, `*.pem`, `*.pfx`, `*.tmp`, and `*.swp`.

This repository does not include a license file.
