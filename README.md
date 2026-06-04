1|1|1|# Pangolin + Proxmox + Guacamole Runbook
2|2|2|
3|3|3|Professional runbook for deploying Apache Guacamole on a hardened Proxmox host and exposing services via Pangolin with subdomain-based routing. This avoids path-based rewrites and keeps each service on its own hostname.
4|4|4|
5|5|5|## Scope
6|6|6|- Guacamole deployed via Docker Compose with hardened runtime flags
7|7|7|- TLS termination for Guacamole using an nginx sidecar (self-signed certificate)
8|8|8|- Pangolin/Newt exposure using **subdomains** (no path-based routing)
9|9|9|- Proxmox exposed via its native HTTPS service
10|10|10|
11|11|11|## Redaction policy
12|12|12|This document redacts all internal IPs and ports. Examples use RFC 1918 placeholders and generic ports:
13|13|13|- Internal IPs: 192.168.1.x
14|14|14|- Internal ports: 8081
15|15|15|- Public domain: contoso.com
16|16|16|
17|17|17|## Architecture summary
18|18|18|- guacamole.contoso.com -> Pangolin -> Newt -> internal host (HTTPS, nginx sidecar)
19|19|19|- proxmox.contoso.com -> Pangolin -> Newt -> internal host (HTTPS, Proxmox UI)
20|20|20|
21|21|21|## Guacamole deployment (hardened)
22|22|22|Location: `/opt/guacamole`
23|23|23|
24|24|24|### Docker Compose (example)
25|25|25|```
26|26|26|services:
27|27|27|  guacd:
28|28|28|    image: guacamole/guacd:1.5.5
29|29|29|    restart: unless-stopped
30|30|30|
31|31|31|  db:
32|32|32|    image: postgres:15
33|33|33|    restart: unless-stopped
34|34|34|    command: ["postgres", "-c", "unix_socket_directories="]
35|35|35|    environment:
36|36|36|      POSTGRES_DB: guacamole
37|37|37|      POSTGRES_USER: guacamole
38|38|38|      POSTGRES_PASSWORD: K77evtnfmXU8eVgBmg5gEnqQj5d3UWOH
39|39|39|    volumes:
40|40|40|      - db-data:/var/lib/postgresql/data
41|41|41|
42|42|42|  guacamole:
43|43|43|    image: guacamole/guacamole:1.5.5
44|44|44|    restart: unless-stopped
45|45|45|    depends_on:
46|46|46|      - guacd
47|47|47|      - db
48|48|48|    environment:
49|49|49|      GUACD_HOSTNAME: guacd
50|50|50|      POSTGRES_HOSTNAME: db
51|51|51|      POSTGRES_DATABASE: guacamole
52|52|52|      POSTGRES_USER: guacamole
53|53|53|      POSTGRES_PASSWORD: K77evtnfmXU8eVgBmg5gEnqQj5d3UWOH
54|54|54|      JAVA_TOOL_OPTIONS: "-XX:-UseContainerSupport -Dorg.apache.catalina.core.AsyncContextImpl.USE_NIO=false"
55|55|55|    security_opt:
56|56|56|      - seccomp:unconfined
57|57|57|    privileged: true
58|58|58|
59|59|59|  web:
60|60|60|    image: nginx:1.27-alpine
61|61|61|    restart: unless-stopped
62|62|62|    depends_on:
63|63|63|      - guacamole
64|64|64|    ports:
65|65|65|      - "8081:443"
66|66|66|    volumes:
67|67|67|      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
68|68|68|      - ./certs:/etc/nginx/certs:ro
69|69|69|    security_opt:
70|70|70|      - seccomp:unconfined
71|71|71|    privileged: true
72|72|72|
73|73|73|volumes:
74|74|74|  db-data:
75|75|75|```
76|76|76|
77|77|77|### nginx.conf (TLS termination and proxy)
78|78|78|```
79|79|79|server {
80|80|80|  listen 80;
81|81|81|  server_name _;
82|82|82|  return 301 https://$host$request_uri;
83|83|83|}
84|84|84|
85|85|85|server {
86|86|86|  listen 443 ssl;
87|87|87|  server_name _;
88|88|88|
89|89|89|  ssl_certificate     /etc/nginx/certs/guac.crt;
90|90|90|  ssl_certificate_key /etc/nginx/certs/guac.key;
91|91|91|
92|92|92|  location / {
93|93|93|    proxy_pass http://guacamole:8080/guacamole/;
94|94|94|    proxy_set_header Host $host;
95|95|95|    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
96|96|96|    proxy_set_header X-Forwarded-Proto https;
97|97|97|    proxy_set_header X-Forwarded-Host $host;
98|98|98|    proxy_set_header X-Forwarded-Port $server_port;
99|99|99|    proxy_redirect off;
100|100|100|  }
101|101|101|}
102|102|102|```
103|103|103|
104|104|104|### Self-signed certs
105|105|105|```
106|106|106|mkdir -p /opt/guacamole/certs
107|107|107|openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
108|108|108|  -keyout /opt/guacamole/certs/guac.key \
109|109|109|  -out /opt/guacamole/certs/guac.crt \
110|110|110|  -subj "/CN=guacamole.local"
111|111|111|```
112|112|112|
113|113|113|### Database initialization
114|114|114|Guacamole needs its schema loaded into Postgres:
115|115|115|```
116|116|116|# Create database
117|117|117|PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -U guacamole -d postgres -c "CREATE DATABASE guacamole;"
118|118|118|
119|119|119|# Load schema
120|120|120|cat /opt/guacamole/postgresql/schema/001-create-schema.sql \
121|121|121|  /opt/guacamole/postgresql/schema/002-create-admin-user.sql | \
122|122|122|  PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -U guacamole -d guacamole
123|123|123|```
124|124|124|
125|125|125|### Default login
126|126|126|- Username: guacadmin
127|127|127|- Password: guacadmin
128|128|128|- Change immediately
129|129|129|
130|130|130|## Pangolin routing (subdomains)
131|131|131|Set up two hostnames in Pangolin Cloud, each mapped to a different backend port on the same internal host:
132|132|132|
133|133|133|- guacamole.contoso.com -> internal host 192.168.1.x:8081 (HTTPS)
134|134|134|- proxmox.contoso.com -> internal host 192.168.1.x:8006 (HTTPS)
135|135|135|
136|136|136|No path-based routing or regex rewrites are required.
137|137|137|
138|138|138|## Verification
139|139|139|- Guacamole (local): `curl -k https://127.0.0.1:8081/` returns 200
140|140|140|- Pangolin target: healthy
141|141|141|- guacamole.contoso.com loads Guacamole login
142|142|142|- proxmox.contoso.com loads Proxmox login and realm dropdown
143|143|143|
144|144|144|## Troubleshooting
145|145|145|### Pangolin shows unhealthy / 503
146|146|146|- Newt health checks default to HTTPS. Ensure backend speaks HTTPS.
147|147|147|
148|148|148|### Guacamole UI shows raw translation keys
149|149|149|- This occurs when static assets are not served. If you use path routing, ensure subpaths are preserved. Prefer subdomains.
150|150|150|
151|151|151|### Postgres errors: database does not exist
152|152|152|- Create the database and load schema as shown above.
153|153|153|
154|154|154|