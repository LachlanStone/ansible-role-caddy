# ansible-role-caddy

An Ansible role to deploy and configure [Caddy](https://caddyserver.com/) as a reverse proxy with optional Cloudflare DNS plugin, modular Caddyfile templating, and systemd integration.

Supports RHEL-family systems (Fedora, Rocky Linux, RHEL) via DNF/COPR or a custom pre-built binary.

---

## Requirements

- Ansible ≥ 2.14
- Target OS: Fedora, Rocky Linux, or RHEL 9/10
- `community.general` collection (for `copr` module in `pkg` mode)

---

## Role Variables

### Install mode

| Variable | Default | Description |
|---|---|---|
| `caddy_install_mode` | `pkg` | Install method: `pkg` (DNF/COPR) or `custom` (pre-built binary) |
| `caddy_custom_binary_src` | `""` | **Required when `caddy_install_mode: custom`** — absolute path to a pre-built Caddy binary on the control node |

### Cloudflare integration

| Variable | Default | Description |
|---|---|---|
| `cloudflare_dns` | `false` | Enable Cloudflare DNS-01 challenge and Authenticated Origin Pull CA |
| `vault_cloudflare_acme_api_token` | *(required when `cloudflare_dns: true`)* | Cloudflare API token with DNS edit permissions. Store in Ansible Vault. |

### General

| Variable | Default | Description |
|---|---|---|
| `caddy_force_reload` | `true` | Force Caddy reload on config changes |
| `caddy_log_dir` | `/var/log/caddy` | Log directory |
| `caddy_logs` | `[]` | List of logging policy snippets to deploy (see templates/caddy_logging.caddyfile.j2) |
| `caddy_proxy_servers` | `[]` | List of reverse proxy server block definitions (see templates/caddy_server_block.caddyfile.j2) |

### Global Caddyfile config (`caddy_global`)

| Key | Default | Description |
|---|---|---|
| `email` | `""` | ACME registration email |
| `debug` | `false` | Enable Caddy debug logging |
| `persist_config` | `"off"` | Whether Caddy persists API-driven config changes |
| `admin` | `"localhost:2019"` | Admin API endpoint |
| `metrics` | `true` | Enable Prometheus metrics endpoint |
| `acme_dns.provider` | `cloudflare` | ACME DNS provider |
| `acme_dns.token_env` | `CF_API_TOKEN` | Environment variable name for the DNS token |
| `key_type` | `ed25519` | TLS key type |
| `ocsp_stapling` | `"off"` | OCSP stapling (`on`/`off`) |
| `grace_period` | `10s` | Graceful shutdown grace period |
| `shutdown_delay` | `5s` | Shutdown delay |
| `servers.trusted_proxy_networks` | Cloudflare + localhost ranges | List of trusted proxy CIDRs |
| `servers.trusted_proxies_strict` | `true` | Parse X-Forwarded-For right-to-left |
| `servers.disable_0rtt` | `true` | Disable QUIC 0-RTT |
| `servers.timeouts.*` | read_body: 10s, read_header: 10s, write: 30s, idle: 2m | Server timeout settings |
| `servers.max_header_size` | `32KiB` | Maximum request header size |
| `servers.protocols` | `[h1, h2, h3]` | Enabled HTTP protocols |

### TLS domains

```yaml
caddy_tls_domains:
  - name: "*.example.com"
    comment: "Wildcard — DNS-01 challenge"
```

### Common security snippet (`caddy_common_snippets`)

Defines a reusable Caddyfile snippet (`(common_config)`) with encoding and security headers.

| Variable | Default |
|---|---|
| `caddy_common_snippet_name` | `common_config` |
| `caddy_common_encode` | `[zstd, gzip]` |
| `caddy_common_header_directives` | HSTS, CSP, X-Frame-Options, etc. |

### Remote IP policy (`caddy_remote_ip`)

List of `remote_ip` matcher snippet definitions. Default includes a `cloudflare_only` snippet using the built-in `caddy_cloudflare_networks`.

### Cloudflare network CIDRs

`caddy_cloudflare_networks` — pre-populated with current Cloudflare published IP ranges.  
`caddy_localhost_networks` — `[127.0.0.1]`

---

## Example Playbook

### Package install (Fedora/Rocky)

```yaml
- hosts: webservers
  become: true
  vars:
    caddy_install_mode: pkg
    cloudflare_dns: true
    vault_cloudflare_acme_api_token: "{{ vault_cf_token }}"
    caddy_global:
      email: "admin@example.com"
      acme_dns:
        provider: cloudflare
        token_env: CLOUDFLARE_API_TOKEN
    caddy_tls_domains:
      - name: "*.example.com"
    caddy_proxy_servers:
      - name: myapp
        servers:
          - fqdn: app.example.com
            upstream: 192.168.1.10
            upstream_port: 8080
            routes:
              - port: 443
                comment: ["App reverse proxy"]
                imports:
                  - common_config
                  - cloudflare_only
  roles:
    - role: lachlanstone.caddy
```

### Custom binary install

```yaml
- hosts: webservers
  become: true
  vars:
    caddy_install_mode: custom
    caddy_custom_binary_src: "/path/to/caddy"  # pre-built with Cloudflare plugin
    cloudflare_dns: true
    vault_cloudflare_acme_api_token: "{{ vault_cf_token }}"
  roles:
    - role: lachlanstone.caddy
```

---

## Install Modes

### `pkg` (default)
Installs Caddy via DNF from the official [Caddy COPR](https://copr.fedorainfracloud.org/coprs/g/caddy/caddy/) and adds the Cloudflare DNS plugin using `caddy add-package`.

### `custom`
Copies a pre-built Caddy binary (with any plugins already embedded) from the control node. You are responsible for building the binary (e.g., via [xcaddy](https://github.com/caddyserver/xcaddy)) and setting `caddy_custom_binary_src`.

---

## Cloudflare Integration

When `cloudflare_dns: true`, the role:
1. Downloads the [Cloudflare Authenticated Origin Pull CA](https://developers.cloudflare.com/ssl/static/authenticated_origin_pull_ca.pem) to `/etc/caddy/cloudflare.crt`
2. Creates `/etc/caddy/cloudflare.env` with the API token (mode `0600`)
3. Adds a systemd override to load the environment file

The API token is read from `vault_cloudflare_acme_api_token` — store this in Ansible Vault.

---

## Tags

| Tag | Effect |
|---|---|
| `install` | Run install + configure tasks |
| `cloudflare` | Run only Cloudflare tasks |
| `configure` | Run only configuration tasks |

---

## License

MIT

---

## Author

[LachlanStone](https://github.com/LachlanStone)
