# Pangolin + Proxmox + Guacamole Runbook

Professional runbook for deploying Apache Guacamole on a hardened Proxmox host and exposing services via Pangolin with subdomain-based routing. This avoids path-based rewrites and keeps each service on its own hostname.

## Scope
- Guacamole deployed via Docker Compose with hardened runtime flags
- TLS termination for Guacamole using an nginx sidecar (self-signed certificate)
- Pangolin/Newt exposure using **subdomains** (no path-based routing)
- Proxmox exposed via its native HTTPS service

## Redaction policy
This document redacts all internal IPs and ports. Examples use RFC 1918 placeholders and generic ports:
- Internal IPs: 192.168.1.x
- Internal ports: xxxxx
- Public domain: contoso.com

## Architecture summary
- guacamole.contoso.com -> Pangolin -> Newt -> internal host (HTTPS, nginx sidecar)
- proxmox.contoso.com -> Pangolin -> Newt -> internal host (HTTPS, Proxmox UI)

## Guacamole deployment (hardened)
Location: `/opt/guacamole`

### Docker Compose (example)
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
      POSTGRES_PASSWORD: <REDACTED>
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
      POSTGRES_PASSWORD: <REDACTED>
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
      - "xxxxx:443"
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
```
mkdir -p /opt/guacamole/certs
openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -keyout /opt/guacamole/certs/guac.key \
  -out /opt/guacamole/certs/guac.crt \
  -subj "/CN=guacamole.local"
```

### Database initialization
Guacamole needs its schema loaded into Postgres:
```
# Create database
PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -U guacamole -d postgres -c "CREATE DATABASE guacamole;"

# Load schema
cat /opt/guacamole/postgresql/schema/001-create-schema.sql \
  /opt/guacamole/postgresql/schema/002-create-admin-user.sql | \
  PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -U guacamole -d guacamole
```

### Default login
- Username: guacadmin
- Password: guacadmin
- Change immediately

## Pangolin routing (subdomains)
Set up two hostnames in Pangolin Cloud, each mapped to a different backend port on the same internal host:

- guacamole.contoso.com -> internal host 192.168.1.x:xxxxx (HTTPS)
- proxmox.contoso.com -> internal host 192.168.1.x:xxxxx (HTTPS)

No path-based routing or regex rewrites are required.

## Verification
- Guacamole (local): `curl -k https://127.0.0.1:xxxxx/` returns 200
- Pangolin target: healthy
- guacamole.contoso.com loads Guacamole login
- proxmox.contoso.com loads Proxmox login and realm dropdown

## Troubleshooting
### Pangolin shows unhealthy / 503
- Newt health checks default to HTTPS. Ensure backend speaks HTTPS.

### Guacamole UI shows raw translation keys
- This occurs when static assets are not served. If you use path routing, ensure subpaths are preserved. Prefer subdomains.

### Postgres errors: database does not exist
- Create the database and load schema as shown above.

## Notes
This runbook intentionally omits internal IPs, ports, and secrets.
