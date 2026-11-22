# Forward Proxy for Restricted Environments

One Linux VM that can reach the internet. Everything else on the restricted
network cannot. This VM runs a **forward proxy** with a strict allowlist:
machines inside point their update tools at it, approved URLs pass,
everything else gets `403`.

```
[restricted machines] ──proxy:3128──> [proxy VM] ──> internet
                                        only approved URLs pass
```

Two ways to build it — pick **one**:

- **Option A — nginx** (Docker container, you control the exact version)
- **Option B — Squid** (installed from the Rocky repo, least maintenance)

---

## Tested and verified

Every config in this guide was built and functionally tested before writing:

| Test | nginx | Squid |
|---|---|---|
| HTTP through proxy → `200` | ✔ | ✔ |
| HTTPS `CONNECT` tunnel → `200` | ✔ | ✔ |
| Non-allowlisted site → `403` from the proxy | ✔ | ✔ |
| Add a URL + reload without downtime | ✔ | ✔ |
| Config syntax check command | ✔ | ✔ |

Software tested: nginx 1.26.3 + `ngx_http_proxy_connect_module`
(patch `proxy_connect_rewrite_102101.patch`), Squid 5.7.

---

## Part 0 — Prepare the VM

- Rocky Linux 9 VM, updated (`sudo dnf update -y`), with internet access.
- Static IP on the interface facing the restricted network.
- Working DNS.
- Know these two values (used below as examples — replace with yours):
  - VM internal IP: `192.168.10.5`
  - Restricted network range: `10.10.0.0/16`

---
---

# OPTION A — nginx (Docker)

## A.1 Install Docker

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER        # log out/in once after this
```

## A.2 Create the files

```bash
sudo mkdir -p /opt/forward-proxy/proxy
cd /opt/forward-proxy
```

Three files (create with `sudo vi <file>`).

### `/opt/forward-proxy/docker-compose.yml`

```yaml
services:
  proxy:
    build:
      context: ./proxy
      args:
        NGINX_VERSION: "1.26.3"      # <-- bump this to update nginx
    image: forward-proxy:1.26.3      # <-- keep tag = version (easy rollback)
    container_name: forward-proxy
    network_mode: host               # nginx listens on the VM's own IP
    user: "33:33"                    # non-root (www-data)
    read_only: true                  # container filesystem cannot be written
    tmpfs:
      - /tmp                         # the only writable space nginx gets
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    volumes:
      - ./proxy/nginx.conf:/usr/local/nginx/conf/nginx.conf:ro,Z
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### `/opt/forward-proxy/proxy/Dockerfile`

```dockerfile
# nginx alone cannot proxy HTTPS (no CONNECT method) - we compile in
# the standard community module: github.com/chobits/ngx_http_proxy_connect_module
ARG NGINX_VERSION=1.26.3
FROM debian:bookworm-slim
ARG NGINX_VERSION

RUN apt-get update && apt-get install -y --no-install-recommends \
        gcc make patch libc6-dev libpcre2-dev zlib1g-dev libssl-dev \
        curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build
RUN curl -fsSL https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz -o nginx.tgz \
    && tar xzf nginx.tgz \
    && curl -fsSL https://github.com/chobits/ngx_http_proxy_connect_module/archive/refs/heads/master.tar.gz -o connect.tgz \
    && tar xzf connect.tgz \
    && cd nginx-${NGINX_VERSION} \
    && patch -p1 < ../ngx_http_proxy_connect_module-master/patch/proxy_connect_rewrite_102101.patch \
    && ./configure --add-module=../ngx_http_proxy_connect_module-master --with-http_ssl_module \
    && make -j"$(nproc)" && make install

COPY nginx.conf /usr/local/nginx/conf/nginx.conf
CMD ["/usr/local/nginx/sbin/nginx", "-g", "daemon off;"]
```

> When you raise `NGINX_VERSION` beyond 1.26.x, check the module repo's
> patch table and change the patch filename if the table says so.
> `--with-http_ssl_module` is required by the CONNECT module — do not remove it.

### `/opt/forward-proxy/proxy/nginx.conf`

```nginx
worker_processes  auto;
error_log  /dev/stderr warn;
pid        /tmp/nginx.pid;

events { worker_connections  1024; }

http {
    access_log  /dev/stdout;
    server_tokens off;

    # temp dirs inside the writable /tmp (container is read-only)
    client_body_temp_path /tmp/client_body;
    proxy_temp_path       /tmp/proxy;
    fastcgi_temp_path     /tmp/fastcgi;
    uwsgi_temp_path       /tmp/uwsgi;
    scgi_temp_path        /tmp/scgi;

    server {
        # CHANGE: IP facing the restricted network
        listen 192.168.10.5:3128;

        # CHANGE: your DNS server (the proxy resolves names for the clients)
        resolver 192.168.10.53 valid=300s;
        resolver_timeout 5s;

        # HTTPS tunneling (the compiled-in module)
        proxy_connect;
        proxy_connect_allow 443 80;          # tunnels to these ports only
        proxy_connect_connect_timeout 10s;
        proxy_connect_read_timeout 300s;

        # ---- ALLOWED DESTINATIONS: edit THIS line to add URLs ----
        # Examples below - replace with YOUR approved update endpoints.
        # 2026-11-22 initial set (change log: date, what, who approved)
        set $allowed 0;
        if ($host ~* ^(cdn\.redhat\.com|subscription\.rhsm\.redhat\.com|nozomi-contents\.s3\.amazonaws\.com|neverssl\.com)$) { set $allowed 1; }
        if ($allowed = 0) { return 403; }

        # plain-HTTP forwarding
        location / {
            proxy_pass http://$http_host$request_uri;
            proxy_set_header Host $http_host;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }
}
```

Rules that will save you pain:

- Keep the allowlist regex on **one single line** — nginx rejects multi-line
  conditions with `invalid condition "$host"` and the container exits.
- Escape dots as `\.` and separate entries with `|`.
- `proxy_http_version 1.1` + empty `Connection` header prevent random
  `502`s from upstreams that dislike HTTP/1.0 — both lines are required.

## A.3 Build and start

```bash
cd /opt/forward-proxy
docker compose up -d --build        # first build ~5 minutes (compiling)
docker ps                           # forward-proxy should stay "Up"
docker logs forward-proxy           # must be free of errors
```

## A.4 Firewall (allow only the restricted network)

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=10.10.0.0/16 port port=3128 protocol=tcp accept'
sudo firewall-cmd --reload
```

## A.5 Add a new URL (daily operation, ~2 minutes)

```bash
# 1. Add |new\.domain\.com to the allowlist line (before )$)
sudo vi /opt/forward-proxy/proxy/nginx.conf

# 2. Check syntax, then reload - no downtime
docker exec forward-proxy /usr/local/nginx/sbin/nginx -t
docker exec forward-proxy /usr/local/nginx/sbin/nginx -s reload

# 3. Verify
curl -x http://192.168.10.5:3128 -I https://new.domain.com
docker logs --tail 20 forward-proxy
```

## A.6 Update nginx (security patches)

- Watch <https://nginx.org/en/security_advisories.html> monthly.
- Monthly rebuild even without a version bump (picks up base-image fixes):

```bash
cd /opt/forward-proxy
docker compose build --pull
docker compose up -d
```

- Version upgrade: change `NGINX_VERSION` **and** the `image:` tag in
  `docker-compose.yml`, then the same two commands. Re-run the tests in Part 1.
- Rollback: set `image:` back to the previous tag, `docker compose up -d`.

---
---

# OPTION B — Squid (package install)

Simpler to run: HTTPS tunneling (`CONNECT`) is built in, and Rocky ships
security fixes through normal `dnf update`. No compiling, no containers.

## B.1 Install

```bash
sudo dnf install -y squid
sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.orig   # keep a backup
```

## B.2 Two files

### `/etc/squid/squid.conf` (replace the whole file)

```
visible_hostname forward-proxy

# CHANGE: IP facing the restricted network
http_port 192.168.10.5:3128

# who may use this proxy (CHANGE: your restricted network range)
acl clients src 10.10.0.0/16

# approved destinations live in this file (one per line)
acl allowed_sites dstdomain "/etc/squid/allowed_sites.txt"

# ports and methods
acl allowed_ports port 80 443
acl SSL_ports port 443
acl CONNECT method CONNECT
acl Safe_methods method GET POST OPTIONS CONNECT

# rules (order matters - first match wins)
http_access deny CONNECT !SSL_ports
http_access allow clients allowed_sites allowed_ports Safe_methods
http_access deny all

# update proxy, not a cache
cache deny all

# privacy: don't leak client IPs, the proxy itself, or its version
forwarded_for delete
via off
httpd_suppress_version_string on
request_header_access X-Forwarded-For deny all
```

### `/etc/squid/allowed_sites.txt`

```
# Approved destinations - one per line.
# Exact host:      cdn.redhat.com
# Host + subdomains: .redhat.com
# Examples below - replace with YOUR approved update endpoints.
cdn.redhat.com
subscription.rhsm.redhat.com
nozomi-contents.s3.amazonaws.com
neverssl.com
```

## B.3 Start

```bash
sudo squid -k parse                    # must print no FATAL/ERROR
sudo systemctl enable --now squid
sudo systemctl status squid --no-pager
```

## B.4 Firewall (same as Option A)

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=10.10.0.0/16 port port=3128 protocol=tcp accept'
sudo firewall-cmd --reload
```

## B.5 Add a new URL (daily operation, ~1 minute)

```bash
# 1. Append one line
echo "new.domain.com" | sudo tee -a /etc/squid/allowed_sites.txt

# 2. Check syntax, then reload - no downtime
sudo squid -k parse
sudo squid -k reconfigure

# 3. Verify
curl -x http://192.168.10.5:3128 -I https://new.domain.com
sudo tail -5 /var/log/squid/access.log
```

## B.6 Update Squid

```bash
sudo dnf update squid          # Rocky backports security fixes into its version
```

Do this on your normal patch schedule. Watch `dnf updateinfo list security`.

---
---

# Part 1 — Test it (same for both options)

Run from a machine inside the restricted network:

```bash
# 1. Direct access must FAIL (proves the restriction)
curl -m 8 -I https://cdn.redhat.com

# 2. HTTP via proxy -> expect 200
curl -x http://192.168.10.5:3128 -I http://neverssl.com

# 3. HTTPS via proxy -> expect "HTTP/1.1 200 Connection Established"
curl -vx http://192.168.10.5:3128 https://cdn.redhat.com -o /dev/null

# 4. Non-approved site -> expect 403 FROM THE PROXY
curl -vx http://192.168.10.5:3128 https://example.com -o /dev/null
```

What good looks like in the proxy logs:

```
# nginx:  docker logs forward-proxy
"CONNECT cdn.redhat.com:443 HTTP/1.1" 200
"CONNECT example.com:443 HTTP/1.1" 403

# Squid:  sudo tail /var/log/squid/access.log
TCP_TUNNEL/200 ... CONNECT cdn.redhat.com:443
TCP_DENIED/403 ... CONNECT example.com:443
```

> Note: some sites answer a bare request with their own 403 (Red Hat's CDN,
> Amazon S3). A `CONNECT ... 200` / `TCP_TUNNEL/200` in the log is the proof
> of success — the tunnel was built and the real server answered.

# Part 2 — Client quick reference

The proxy does nothing until each client is told to use it. One line each:

| Client | Setting |
|---|---|
| curl (one command) | `curl -x http://192.168.10.5:3128 ...` |
| Most Linux tools (session) | `export https_proxy=http://192.168.10.5:3128` |
| dnf / yum (Rocky, RHEL, Amazon Linux) | `proxy=http://192.168.10.5:3128` in `/etc/dnf/dnf.conf` |
| RHEL subscription-manager | `proxy_hostname` + `proxy_port` in `/etc/rhsm/rhsm.conf` |
| Windows | Settings → Network → Proxy, or `netsh winhttp set proxy 192.168.10.5:3128` |
| Appliances with a proxy option | enter `192.168.10.5` port `3128` in their update/proxy settings |

# Part 3 — Which one should you pick?

| | nginx (Option A) | Squid (Option B) |
|---|---|---|
| HTTPS tunneling | needs compile (automated in the Dockerfile) | built in |
| Updates | you rebuild the image (full control) | `dnf update` (zero effort) |
| Config style | one file, one regex line | one config + one plain list file |
| Best if | you want pinned versions & container isolation | you want the least maintenance |

Both enforce the exact same policy model: known clients → approved URLs →
ports 80/443 only → everything else denied and logged.

# Part 4 — Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Container/service dies at start | config error | nginx: `docker logs forward-proxy` · Squid: `sudo squid -k parse` |
| nginx: `invalid condition "$host"` | allowlist regex spans multiple lines | put it on **one line** |
| Client: "403 from proxy after CONNECT" | site not in allowlist | expected — go through the add-URL steps |
| Random `502` on HTTP sites | flaky upstream or HTTP/1.0 issue | nginx.conf already includes the HTTP/1.1 fix; retry |
| Squid reconfigure seems ignored | config test failed silently | always run `squid -k parse` first |
| Nothing in logs | wrong log location | nginx: `docker logs` · Squid: `/var/log/squid/access.log` |

# Part 5 — Security checklist

- Allowlist contains **only** approved update endpoints — every entry verified against vendor documentation, every change dated in a comment.
- Proxy listens **only** on the internal-facing IP; firewall accepts 3128 **only** from the restricted network range.
- Tunnels restricted to ports **443/80**; methods restricted to GET/POST/OPTIONS/CONNECT (Squid).
- nginx container: non-root, read-only filesystem, all capabilities dropped, no-new-privileges, pinned version.
- Squid: `forwarded_for delete` + `via off` + version hidden — clients' IPs and the proxy's identity are not leaked upstream.
- Outbound internet from this VM limited to ports 80/443 on the network firewall.
- Logs reviewed regularly — every allowed and denied attempt is recorded.
- Updates: monthly rebuild (nginx) / `dnf update` (Squid) + advisories.
