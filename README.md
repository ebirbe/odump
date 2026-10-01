# odump

Universal Odoo database backup CLI. One command to back up any Odoo instance — local, inside Docker/Podman, or remote — producing the exact same file that Odoo's built-in database manager produces: `.dump` (PostgreSQL custom format) or `.zip` (database + filestore).

## Features

- **Works everywhere**: local installs, Docker/Podman containers, remote servers (URL or `host:port`).
- **Zero-config by default**: auto-detects the target (port mapping, container IP, `odoo.conf`), the master password and the database name.
- **Container fallback**: if the HTTP endpoint is blocked (`list_db = False`, disabled database manager), it runs the dump natively inside the container via `odoo.service.db.dump_db` — no restart, no config edits.
- **Resilient database discovery**: falls back to PostgreSQL's catalog through the Odoo process when `/web/database/list` is locked.
- **Readable errors**: extracts the actual reason from Odoo HTML error pages instead of dumping raw HTML.
- **Faithful output**: server-side it is the same `pg_dump --no-owner --format=c` that Odoo's `/web/database/backup` runs, so restores behave identically.

## Requirements

- `bash` and `curl`
- `docker` or `podman` (only for container targets)

## Install

```bash
curl -fsSL -o odump https://raw.githubusercontent.com/ebirbe/odump/main/odump
chmod +x odump
sudo mv odump /usr/local/bin/
```

Or just clone the repo and run `./odump`.

## Usage

```
odump [target] [options]
```

**Target** (optional): a container name/ID, a URL (`https://odoo.example.com`) or `host:port`. Defaults to `http://localhost:8069`.

| Option | Description |
| --- | --- |
| `-d, --database <name>` | Database to back up (auto-detected if only one exists or set in `odoo.conf`) |
| `-p, --password <pwd>` | Odoo master password (`admin_passwd`); or env `ODOO_MASTER_PASSWD` |
| `-c, --config <path>` | Path to `odoo.conf` (local or inside the container) |
| `-o, --output <file>` | Custom output path |
| `--zip` | Full backup with filestore (`.zip`). Default is SQL-only `.dump` |
| `-k, --insecure` | Skip HTTPS certificate verification |
| `-h, --help` | Show help |

### Examples

```bash
# 1. Local instance on localhost:8069
odump -d my_db

# 2. Docker or Podman container
odump my-odoo-container --zip

# 3. Remote server (no console or file access needed)
odump https://odoo.example.com -d prod_db -p 'SuperSecretPwd' --zip

# 4. CI/CD or cron
ODOO_MASTER_PASSWD="secret" odump :8069 -d test_db
```

## Security notes

- The master password is resolved in this order: `-p` flag, `ODOO_MASTER_PASSWD` environment variable, `admin_passwd` from `odoo.conf`, or an interactive hidden prompt.
- Prefer the environment variable or the interactive prompt over `-p` in shared machines: command-line flags can leak through shell history and process listings.
- The password travels in the POST body to `/web/database/backup` — the same endpoint and mechanism Odoo's own database manager uses from your browser.
- Use `--insecure` (`-k`) only against servers with self-signed certificates you trust.

## How it works

1. **HTTP first**: POSTs to `/web/database/backup` with the master password, database name and format — identical to Odoo's database manager.
2. **Integrity check**: validates the HTTP status and that the response is not an HTML error page.
3. **Container fallback**: if the endpoint is blocked and the target is a container, it spawns an in-container Python process that loads the same Odoo config, enables `list_db` in memory only, and streams `odoo.service.db.dump_db` straight to the output file.

The generated dump is restored with `pg_restore` exactly like Odoo's `/web/database/manager` does — no surprises on restore.

## License

[MIT](LICENSE) © Erick Birbe
