1|# Pangolin + Proxmox + Guacamole Runbook
2|
3|Professional runbook for deploying Apache Guacamole on a hardened Proxmox host and exposing services via Pangolin with subdomain-based routing. This avoids path-based rewrites and keeps each service on its own hostname.
4|
5|## Scope
Why: establishes what is covered and what is not.

6|- Guacamole deployed via Docker Compose with hardened runtime flags
7|- TLS termination for Guacamole using an nginx sidecar (self-signed certificate)
8|- Pangolin/Newt exposure using subdomains (no path-based routing)
9|- Proxmox exposed via its native HTTPS service
10|
11|## Architecture summary
Why: quick mental model for data flow and responsibility boundaries.

12|- Pangolin Cloud handles public DNS/edge and forwards traffic to Newt.
13|- Newt runs on the host and maintains the secure tunnel plus health checks.
14|- Proxmox serves its UI/API directly over HTTPS on its native port.
15|- Guacamole runs in Docker; guacd handles protocol traffic, Postgres stores config, nginx terminates TLS.
16|
17|## Guacamole deployment (hardened)
Why: ensures a stable Guacamole stack on hardened kernels and exposes it safely.

18|Location: /opt/guacamole
19|
20|### Docker Compose (example)
Why: defines the full Guacamole stack and the TLS front-end in one place.

21|```
22|services:
23|  guacd:
24|    image: guacamole/guacd:1.5.5
25|    restart: unless-stopped
26|
27|  db:
28|    image: postgres:15
29|    restart: unless-stopped
30|    command: ["postgres", "-c", "unix_socket_directories="]
31|    environment:
32|      POSTGRES_DB: guacamole
33|      POSTGRES_USER: guacamole
34|      POSTGRES_PASSWORD: <CHANGE_ME>
35|    volumes:
36|      - db-data:/var/lib/postgresql/data
37|
38|  guacamole:
39|    image: guacamole/guacamole:1.5.5
40|    restart: unless-stopped
41|    depends_on:
42|      - guacd
43|      - db
44|    environment:
45|      GUACD_HOSTNAME: guacd
46|      POSTGRES_HOSTNAME: db
47|      POSTGRES_DATABASE: guacamole
48|      POSTGRES_USER: guacamole
49|      POSTGRES_PASSWORD: <CHANGE_ME>
50|      JAVA_TOOL_OPTIONS: "-XX:-UseContainerSupport -Dorg.apache.catalina.core.AsyncContextImpl.USE_NIO=false"
51|    security_opt:
52|      - seccomp:unconfined
53|    privileged: true
54|
55|  web:
56|    image: nginx:1.27-alpine
57|    restart: unless-stopped
58|    depends_on:
59|      - guacamole
60|    ports:
61|      - "8081:443"
62|    volumes:
63|      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
64|      - ./certs:/etc/nginx/certs:ro
65|    security_opt:
66|      - seccomp:unconfined
67|    privileged: true
68|
69|volumes:
70|  db-data:
71|```
72|
73|### nginx.conf (TLS termination and proxy)
Why: serves HTTPS to Pangolin/Newt and maps / to /guacamole/.

74|```
75|server {
76|  listen 80;
77|  server_name _;
78|  return 301 https://$host$request_uri;
79|}
80|
81|server {
82|  listen 443 ssl;
83|  server_name _;
84|
85|  ssl_certificate     /etc/nginx/certs/guac.crt;
86|  ssl_certificate_key /etc/nginx/certs/guac.key;
87|
88|  location / {
89|    proxy_pass http://guacamole:8080/guacamole/;
90|    proxy_set_header Host $host;
91|    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
92|    proxy_set_header X-Forwarded-Proto https;
93|    proxy_set_header X-Forwarded-Host $host;
94|    proxy_set_header X-Forwarded-Port $server_port;
95|    proxy_redirect off;
96|  }
97|}
98|```
99|
100|### Self-signed certs
Why: provides local TLS for health checks; public TLS is handled by Pangolin.

101|```
102|mkdir -p /opt/guacamole/certs
103|openssl req -x509 -nodes -newkey rsa:2048 -days 3650   -keyout /opt/guacamole/certs/guac.key   -out /opt/guacamole/certs/guac.crt   -subj "/CN=guacamole.local"
104|```
105|
106|### Database initialization
Why: Guacamole will not authenticate without its schema and admin user.

107|Guacamole needs its schema loaded into Postgres:
108|```
109|# Create database
110|PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -U guacamole -d postgres -c "CREATE DATABASE guacamole;"
111|
112|# Load schema
113|cat /opt/guacamole/postgresql/schema/001-create-schema.sql   /opt/guacamole/postgresql/schema/002-create-admin-user.sql |   PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -U guacamole -d guacamole
114|```
115|
116|
Password rotation note:
- If you change POSTGRES_PASSWORD, run `docker compose up -d` (recreate) so guacamole.properties regenerates.
- A simple `restart` can leave stale credentials inside the guacamole container.

### Default login
117|- Username: guacadmin
118|- Password: guacadmin
119|- Change immediately
120|
121|## Pangolin routing (subdomains)
Why: subdomains avoid path rewrite edge cases and keep services isolated.

122|- guacamole.contoso.com -> internal host 192.168.1.x:8081 (HTTPS)
123|- proxmox.contoso.com -> internal host 192.168.1.x:8006 (HTTPS)
124|
125|No path-based routing or regex rewrites are required.
126|
127|## Verification
Why: confirms stack health before exposing to users.

128|- Guacamole (local): curl -k https://127.0.0.1:8081/ returns 200
129|- Pangolin target: healthy
130|- guacamole.contoso.com loads Guacamole login
131|- proxmox.contoso.com loads Proxmox login and realm dropdown
132|
133|## Troubleshooting
Why: common failure modes and their fixes.

134|- Pangolin shows unhealthy or 503: backend must speak HTTPS (Newt health checks use HTTPS).
135|- Guacamole shows raw translation keys: assets are not served; avoid path routing or preserve subpaths.
136|- Postgres errors: database does not exist; create the DB and load schema as shown above.
137|