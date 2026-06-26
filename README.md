# Ansible Role: Forgejo

Installs [Forgejo](https://forgejo.org/) (a self-hosted Git forge) as a systemd service with optional Caddy reverse proxy built via [xcaddy](https://github.com/caddyserver/xcaddy) for automatic HTTPS using Cloudflare DNS challenges.

Based on: https://puvvadi.net/posts/self-hosting-github-alternative-forgejo-with-ssh/

## What This Role Does

1. Creates a `git` system user and group
2. Downloads the Forgejo binary from Codeberg releases
3. Configures `app.ini` (database, server, SSH)
4. Deploys a systemd service for Forgejo
5. (Optional) Installs Go, builds Caddy with xcaddy + Cloudflare DNS plugin, and configures it as a reverse proxy

## Requirements

- Debian/Ubuntu target host
- Ansible >= 2.14
- `community.general` collection (for Caddy capability setting)

## Role Variables

### Forgejo

| Variable | Default | Description |
|----------|---------|-------------|
| `forgejo_version` | `12.0.0` | Forgejo release version to install |
| `forgejo_user` | `git` | System user that runs Forgejo |
| `forgejo_group` | `git` | System group for Forgejo |
| `forgejo_home` | `/home/git` | Home directory for the Forgejo user |
| `forgejo_work_dir` | `/var/lib/forgejo` | Forgejo working directory |
| `forgejo_config_dir` | `/etc/forgejo` | Directory for `app.ini` |
| `forgejo_domain` | `localhost` | Public domain name |
| `forgejo_http_port` | `3000` | HTTP listen port |
| `forgejo_root_url` | `http://localhost:3000` | Full public URL |
| `forgejo_ssh_port` | `22` | SSH port for git operations |
| `forgejo_db_type` | `sqlite3` | Database type (`sqlite3`, `postgres`, `mysql`) |
| `forgejo_db_path` | `/var/lib/forgejo/forgejo.db` | SQLite database path |
| `forgejo_db_host` | `` | DB host (for postgres/mysql) |
| `forgejo_db_name` | `` | DB name |
| `forgejo_db_user` | `` | DB user |
| `forgejo_db_password` | `` | DB password |
| `forgejo_disable_registration` | `false` | Disable public user registration |

### Caddy (optional)

| Variable | Default | Description |
|----------|---------|-------------|
| `forgejo_caddy_enabled` | `true` | Whether to install and configure Caddy |
| `forgejo_caddy_xcaddy_plugins` | `[github.com/caddy-dns/cloudflare]` | xcaddy plugins to compile into Caddy |
| `forgejo_caddy_email` | `` | Email for ACME/TLS certificate |
| `forgejo_caddy_cloudflare_api_token` | `` | Cloudflare API token for DNS challenge |
| `forgejo_caddy_cloudflare_zone_api_token` | `` | Optional Cloudflare token with `Zone:Read` scope (recommended for restricted API tokens) |
| `forgejo_caddy_dns_resolvers` | `['1.1.1.1', '8.8.8.8']` | Recursive resolvers used for DNS challenge checks |
| `forgejo_caddy_propagation_delay` | `30s` | Wait before DNS propagation checks begin |
| `forgejo_caddy_propagation_timeout` | `2m` | Max time to wait for DNS propagation |
| `forgejo_go_version` | `1.23.4` | Go version used to build xcaddy/Caddy |

## Example Playbook

```yaml
- hosts: forgejo
  become: true
  roles:
    - role: ansible-role-forgejo
      forgejo_version: "12.0.0"
      forgejo_domain: git.example.com
      forgejo_root_url: "https://git.example.com"
      forgejo_caddy_email: admin@example.com
      forgejo_caddy_cloudflare_api_token: "{{ vault_cloudflare_token }}"
      forgejo_caddy_cloudflare_zone_api_token: "{{ vault_cloudflare_zone_read_token }}"
```

Cloudflare token requirements for DNS-01:

- `forgejo_caddy_cloudflare_api_token`: `Zone:DNS:Edit` for the zone that contains your domain (for example, `jershlabs.xyz`).
- `forgejo_caddy_cloudflare_zone_api_token` (optional but recommended): `Zone:Read` for the same zone.

If you see errors like `expected 1 zone, got 0 for xyz`, Cloudflare could not match a managed zone for your challenge domain. This is usually caused by one of these:

- The token does not include the correct zone.
- `Zone:Read` permission is missing.
- The FQDN is not under a Cloudflare-managed zone.

To skip the Caddy setup (e.g. if you already have a reverse proxy):

```yaml
- hosts: forgejo
  become: true
  roles:
    - role: ansible-role-forgejo
      forgejo_caddy_enabled: false
```

## SSH Passthrough

The role creates the `git` user with `/bin/bash` as its shell. Forgejo manages `~git/.ssh/authorized_keys` automatically, so SSH clone/push works out of the box on port 22:

```
git clone git@git.example.com:user/repo.git
```

## License

MIT
