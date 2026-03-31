# Ansible Role: Nginx

[![CI](https://github.com/DenZen1988/ansible-nginx/actions/workflows/ci.yml/badge.svg)](https://github.com/DenZen1988/ansible-nginx/actions/workflows/ci.yml)

A production-ready, fully variable-driven Ansible role for Nginx on Debian/Ubuntu.

## Features

- **Flexible installation** — apt packages (distro or official nginx.org repo) or compile from source
- **Full `nginx.conf` control** — every directive configurable via variables
- **Virtual hosts** — unlimited server blocks via `nginx_vhosts` list
- **SSL/TLS** — Mozilla Intermediate defaults, DH params, HSTS, OCSP stapling
- **Let's Encrypt** — automated Certbot certificate issuance and renewal
- **Custom certificates** — deploy your own certs from local files
- **Reverse proxy** — proxy headers snippet, timeouts, buffering, WebSocket support
- **Load balancing** — upstream groups with round-robin, least_conn, ip_hash, random
- **TCP/UDP stream proxy** — raw stream blocks for database proxying, etc.
- **Monitoring** — optional stub_status endpoint
- **Fully composable** — override anything at group or host level

## Supported Platforms

| OS | Versions |
| --- | --- |
| Ubuntu | 20.04 (Focal), 22.04 (Jammy), 24.04 (Noble) |
| Debian | 11 (Bullseye), 12 (Bookworm) |

## Requirements

- Ansible ≥ 2.14
- `community.crypto` collection (for DH param generation): `ansible-galaxy collection install community.crypto`
- `community.general` collection (for source compilation): `ansible-galaxy collection install community.general`

## Installation

### Ansible Galaxy

```bash
ansible-galaxy install DenZen1988.nginx
```

### Git

```bash
git clone https://github.com/DenZen1988/ansible-nginx.git roles/ansible-nginx
```

## Quick Start

### Minimal — install Nginx with defaults

```yaml
- hosts: webservers
  become: true
  roles:
    - role: ansible-nginx
```

### Static site with Let's Encrypt

```yaml
# group_vars/webservers.yml
nginx_ssl_enabled: true
nginx_letsencrypt_enabled: true
nginx_certbot_email: "admin@example.com"
nginx_letsencrypt_domains:
  - domain: "example.com"
    extra_domains: ["www.example.com"]
    webroot: "/var/www/example.com/html"
 
nginx_vhosts:
  - name: "example.com-redirect"
    listen: "80"
    server_name: "example.com www.example.com"
    enabled: true
    extra_directives:
      - "return 301 https://$host$request_uri"
    locations: []
 
  - name: "example.com"
    listen: "443 ssl http2"
    server_name: "example.com www.example.com"
    root: "/var/www/example.com/html"
    index: "index.html"
    enabled: true
    ssl: true
    ssl_certificate: "/etc/letsencrypt/live/example.com/fullchain.pem"
    ssl_certificate_key: "/etc/letsencrypt/live/example.com/privkey.pem"
    locations:
      - match: "/"
        directives:
          - "try_files $uri $uri/ =404"
```

### Reverse proxy with load balancing

```yaml
# group_vars/reverse_proxies.yml
nginx_ssl_enabled: true
 
nginx_upstreams:
  - name: "app_backend"
    strategy: "least_conn"
    keepalive: 32
    servers:
      - "10.0.1.10:8080 weight=3"
      - "10.0.1.11:8080"
      - "10.0.1.12:8080 backup"
 
nginx_vhosts:
  - name: "app.example.com"
    listen: "443 ssl http2"
    server_name: "app.example.com"
    enabled: true
    ssl: true
    ssl_certificate: "/etc/nginx/ssl/app.crt"
    ssl_certificate_key: "/etc/nginx/ssl/app.key"
    locations:
      - match: "/"
        directives:
          - "proxy_pass http://app_backend"
          - "proxy_set_header Host $host"
          - "proxy_set_header X-Real-IP $remote_addr"
          - "proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for"
          - "proxy_set_header X-Forwarded-Proto $scheme"
```

## Role Variables

All variables are defined in `defaults/main.yml` with sensible defaults. Here is a summary of the major sections:

### Installation

| Variable | Default | Description |
| --- | --- | --- |
| `nginx_install_method` | `"package"` | `"package"` or `"source"` |
| `nginx_official_repo_enabled` | `true` | Use nginx.org repo instead of distro packages |
| `nginx_source_version` | `"1.26.2"` | Version to compile (source only) |
| `nginx_extra_packages` | `[]` | Additional apt packages to install |

### Core Configuration

| Variable | Default | Description |
| --- | --- | --- |
| `nginx_worker_processes` | `"auto"` | Number of worker processes |
| `nginx_worker_connections` | `1024` | Connections per worker |
| `nginx_client_max_body_size` | `"64m"` | Max request body size |
| `nginx_server_tokens` | `"off"` | Hide version in headers |
| `nginx_keepalive_timeout` | `65` | Keepalive timeout in seconds |
| `nginx_extra_http_directives` | `[]` | Additional directives in http context |

### Virtual Hosts

| Variable | Default | Description |
| --- | --- | --- |
| `nginx_vhosts` | `[]` | List of server block definitions |
| `nginx_remove_default_vhost` | `true` | Remove the default site |
| `nginx_create_docroots` | `true` | Auto-create document root directories |

### SSL/TLS

| Variable | Default | Description |
| --- | --- | --- |
| `nginx_ssl_enabled` | `false` | Enable SSL snippet and settings |
| `nginx_ssl_protocols` | `"TLSv1.2 TLSv1.3"` | Allowed protocols |
| `nginx_ssl_hsts` | `true` | Enable HSTS header |
| `nginx_ssl_dhparam` | `""` | Path to DH params (generated if set) |

### Let's Encrypt

| Variable | Default | Description |
| --- | --- | --- |
| `nginx_letsencrypt_enabled` | `false` | Install Certbot and issue certs |
| `nginx_certbot_email` | `""` | Email for cert notifications |
| `nginx_letsencrypt_domains` | `[]` | Domains to certify |
| `nginx_certbot_auto_renew` | `true` | Enable renewal cron job |

### Reverse Proxy & Load Balancing

| Variable | Default | Description |
| --- | --- | --- |
| `nginx_upstreams` | `[]` | Upstream group definitions |
| `nginx_proxy_connect_timeout` | `60` | Proxy connect timeout |
| `nginx_proxy_read_timeout` | `60` | Proxy read timeout |

See `defaults/main.yml` for the complete list with documentation.

## Variable Precedence: Host vs Group

Ansible's variable precedence means host_vars override group_vars. This works naturally for scalar values like `nginx_worker_connections`.
However, **list variables** (like `nginx_vhosts`) are **replaced entirely**, not merged.

If a host needs to add a vhost to the group's list, you must redefine the full list in `host_vars/`. See `examples/inventory/host_vars/web01.example.com.yml`
for a pattern.

To work around this, consider:

- Using `hash_behaviour: merge` (not recommended globally)
- Using a custom variable like `nginx_vhosts_extra` and combining in the playbook
- Keeping host-specific sites in separate variables and combining with `+`

## Directory Structure

```text
ansible-nginx/
├── defaults/main.yml        # All configurable variables
├── vars/Debian.yml           # OS-specific internals
├── tasks/
│   ├── main.yml              # Task orchestrator
│   ├── install-package.yml   # apt-based install
│   ├── install-source.yml    # compile from source
│   ├── configure.yml         # nginx.conf + snippets
│   ├── ssl.yml               # SSL/TLS setup
│   ├── letsencrypt.yml       # Certbot integration
│   ├── custom-certs.yml      # Deploy custom certs
│   ├── upstreams.yml         # Load balancer config
│   ├── vhosts.yml            # Server blocks
│   ├── stream.yml            # TCP/UDP proxy
│   └── service.yml           # Service management
├── templates/
│   ├── nginx.conf.j2         # Main config
│   ├── vhost.conf.j2         # Server block
│   ├── upstream.conf.j2      # Upstream group
│   ├── ssl_params.conf.j2    # SSL snippet
│   ├── proxy_params.conf.j2  # Proxy snippet
│   ├── stub_status.conf.j2   # Monitoring
│   ├── stream.conf.j2        # Stream config
│   └── nginx.service.j2      # Systemd unit (source)
├── handlers/main.yml
├── meta/main.yml
├── examples/
│   ├── playbook.yml
│   └── inventory/
└── README.md
```

## License

MIT

## Author

Denis Walther — [GitHub](https://github.com/DenZen1988)
