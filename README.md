# certwarden-to-npm-sync

A small Python tool (standard library only) that pushes certificates from
[Cert Warden](https://www.certwarden.com/) to your nginx servers and keeps them updated when Cert Warden renews
them. It is meant to run daily from a systemd timer. Two kinds of target are supported:

- **`npm`**: [Nginx Proxy Manager](https://nginxproxymanager.com/) (NPM), as **custom certificates**, through
  NPM's API.
- **`nginx-ssh`**: plain nginx hosts (no NPM). The tool writes the certificate and key files over SSH, tests
  the nginx config, reloads nginx, and puts the old files back if the test or reload fails.

Why: NPM keeps its certificates in its own database, so an external certificate manager cannot just drop files
into a folder; pushing through NPM's API solves that. Plain nginx reads files, so those are copied over SSH.
Either way the proxy host needs no internet access and no ACME/DNS credentials: only the Cert Warden host
talks to the certificate authority, and it pushes out to the proxies. This suits proxies in a DMZ, and one
Cert Warden serving many proxies.

Tested with NPM 2.15.1, Cert Warden 0.30, and plain nginx 1.24 on Ubuntu 24.04 (`nginx-ssh`).

## How it works

For each target in the config file, first download the full chain and the private key from Cert Warden's API,
then:

### `type = "npm"` (the default)

1. Log in to NPM's API.
2. If no NPM custom certificate with the target's name exists, create one. Otherwise upload over the **same**
   certificate, so proxy hosts that use it keep working with no change.
3. Remember a SHA-256 of the chain, so an unchanged certificate is skipped on the next run.

### `type = "nginx-ssh"`

For each host in the target's `hosts` list:

1. Connect with `ssh` (key authentication, no prompts) and run a short shell script fed on stdin, so the private
   key never appears in a command line.
2. The script writes the new certificate (full chain) and key next to the old files, keeps the existing owner and
   mode of the old files, and swaps them in. New key files are created with mode 600.
3. It runs the **test command** (default `nginx -t`) and then the **reload command** (default
   `nginx -s reload`). If either fails, the old files are put back (or removed, if they did not exist before),
   and the host is reported as failed.
4. On success the SHA-256 is remembered **per host**, so a host that failed is retried on the next run while the
   others are skipped. One failing host does not stop the rest.

Proxy hosts and nginx configs are never touched; only the two certificate files. Cert Warden renews a certificate when about a third of its lifetime is left,
and the daily timer then delivers it within about a day.

To move many **existing** hosts onto the pushed certificate in one go, see
[npm-swap-cert](https://github.com/homelab-zk/npm-swap-cert).

## Requirements

- Python 3.11 or newer (uses `tomllib`), no extra packages.
- Cert Warden with the certificate already issued, plus an API key for each certificate and for its private
  key (Cert Warden: edit the certificate / private key, API key).
- For `nginx-ssh` targets: `ssh` on the machine that runs the tool, and key access to each nginx host (see
  [Plain nginx hosts](#plain-nginx-hosts-nginx-ssh)).
- For `npm` targets, an NPM user for the script. NPM identifies users by **e-mail address**. Give it only the
  *certificates: manage* permission; it does not need to be an admin. NPM shows a non-admin user only the
  items that user owns, so it cannot see or change your proxy hosts. That is intentional.
- The machine that runs the script must reach Cert Warden, and each target: NPM's admin/API port, or SSH (port
  22 by default) on the nginx hosts.

## Security notes

- The tool handles the **private keys** of your certificates and an NPM login. Keep the env file readable by
  root only (`chmod 600`) and run the tool on a host you trust.
- Secrets are never printed. The env file is parsed as plain text, not executed by a shell, so special
  characters in passwords need no escaping.
- For `nginx-ssh`, the tool can write certificate files and reload nginx on every listed host. Use a dedicated
  SSH key (`ssh_key`) and a dedicated user per host, give that user only what it needs (see below), and
  restrict the key with `from=` in `authorized_keys`. Host keys are checked: connect once by hand as the user
  that runs the tool (`ssh -i <key> user@host`) so the host is in `known_hosts`.
- The two Cert Warden API keys per certificate are separate (one for the certificate, one for its key).
  Use a separate key set per certificate.

## Setup

1. Install the script and create the directories:
   ```bash
   sudo install -m 750 certwarden-to-npm-sync /usr/local/bin/certwarden-to-npm-sync
   sudo mkdir -p /etc/certwarden-to-npm-sync /var/lib/certwarden-to-npm-sync
   sudo chmod 700 /var/lib/certwarden-to-npm-sync
   ```
2. Create `/etc/certwarden-to-npm-sync/targets.toml` from
   [`examples/targets.example.toml`](examples/targets.example.toml): one `[[target]]` per certificate.
3. Create `/etc/certwarden-to-npm-sync/env` from [`examples/env.example`](examples/env.example), fill in the
   Cert Warden API keys for each target (and the NPM login for `npm` targets), then `sudo chmod 600 /etc/certwarden-to-npm-sync/env`.
   For a plain nginx target use `type = "nginx-ssh"` (see below). Both kinds can be mixed in one file.
4. Test without changing anything, then run it for real:
   ```bash
   sudo certwarden-to-npm-sync --dry-run
   sudo certwarden-to-npm-sync
   ```
   The first real run creates a custom certificate in NPM per target. Assign it to your proxy hosts once (SSL
   tab in NPM, or in bulk with `npm-swap-cert`). Later runs print `unchanged` until Cert Warden has renewed
   the certificate.
5. Install the timer:
   ```bash
   sudo cp examples/certwarden-to-npm-sync.service examples/certwarden-to-npm-sync.timer /etc/systemd/system/
   sudo systemctl daemon-reload && sudo systemctl enable --now certwarden-to-npm-sync.timer
   journalctl -u certwarden-to-npm-sync.service      # logs of each run
   ```

## Plain nginx hosts (`nginx-ssh`)

Example target (more in [`examples/targets.example.toml`](examples/targets.example.toml)):

```toml
[[target]]
name       = "web"
type       = "nginx-ssh"
cw_cert    = "example-wildcard"
ssh_user   = "certpush"
ssh_key    = "/etc/certwarden-to-npm-sync/id_ed25519"
cert_path  = "/etc/ssl/wildcard/fullchain.pem"
key_path   = "/etc/ssl/wildcard/privkey.pem"
test_cmd   = "sudo -n nginx -t"
reload_cmd = "sudo -n systemctl reload nginx"
hosts = [
  "proxy1.example.lan",
  { host = "proxy2.example.lan", cert_path = "/etc/nginx/ssl/fullchain.pem", key_path = "/etc/nginx/ssl/privkey.pem" },
]
```

Settings can be given on the target (the default for every host) or on a single host. A host is either a
string (the host name) or a table.

| Setting | Meaning |
|---|---|
| `host` | Host name or address (required). |
| `cert_path`, `key_path` | Absolute paths on the host (required). The certificate file receives the **full chain**. |
| `ssh_user`, `ssh_port`, `ssh_key` | SSH login. Left out: the `ssh` defaults for that host. |
| `test_cmd` | Run after the files are in place. Default `nginx -t`. |
| `reload_cmd` | Run if the test passed. Default `nginx -s reload`. Use `systemctl reload nginx`, or `docker exec <container> nginx -s reload` for nginx in a container. |

Preparing a host:

1. Create a user (for example `certpush`) and put the tool's public key in its `authorized_keys`.
2. Give that user write access to the **directory** holding the certificate and key (the tool writes
   `<file>.cwnew` and `<file>.cwbak` there), for example through group ownership of that one directory.
3. Let it run the test and reload commands, for example with a `sudoers` rule for exactly those commands, and
   put `sudo -n` in front of them in the config (`-n` makes sudo fail instead of waiting for a password).
4. Check that your nginx config already points at `cert_path` and `key_path`. The tool never edits nginx
   configs.

Notes: a reload is graceful, so a renewed certificate is served a moment after the tool reports success
(check with `openssl s_client -connect <host>:443 -servername <name>` after a few seconds, not instantly). If
your nginx wants the certificate and key in one file, or a separate chain file, this tool does not
do that yet. Nginx in a container works when the certificate directory is a volume and `reload_cmd` runs
`docker exec`.

## Options

| Option | Meaning |
|---|---|
| `--only NAME` | Only this target. |
| `--force` | Push even if the certificate has not changed. |
| `--dry-run` | Show what would happen, change nothing. |
| `--config FILE`, `--env FILE`, `--state-dir DIR` | Override the default paths under `/etc/certwarden-to-npm-sync` and `/var/lib/certwarden-to-npm-sync`. |

## Monitoring

The tool does not send alerts. Monitor the certificate your proxy actually serves (for example with Zabbix or
Uptime Kuma, alerting at 14 and 7 days left) to catch a failure anywhere in the chain: Cert Warden, the timer,
the upload, or the reload. For NPM targets, NPM may need to reload nginx before a renewed certificate is
served; check this at your first renewal.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `NPM login failed (HTTP 400)` | NPM logs in with the user's **e-mail address**, not a username. Check the password and that 2FA is off for the script user. |
| `Cert Warden certificate download failed (HTTP 401/404)` | Wrong API key, or `cw_cert` in `targets.toml` does not match the certificate name in Cert Warden. |
| `no private key found in Cert Warden response` | The private-key API key is wrong, or the certificate has no key attached. |
| `ssh exit 255: ...` | SSH itself failed: host unreachable, key rejected, or host key not yet in `known_hosts` (connect once by hand). |
| `CW_FAIL: nginx test or reload failed` | The test or reload command failed on that host, so the old files were restored. Run the same commands there by hand to see why (often a missing `sudo` rule or a wrong path). |
| `no target named ...` | The name given to `--only` is not in `targets.toml`. |
| Config or env file not found | Create them, or pass `--config` and `--env`. |

## Details

Cert Warden endpoints used: `/certwarden/api/v1/download/certificates/<name>` (full chain, header
`X-API-Key: <certificate key>`) and `/certwarden/api/v1/download/privatecerts/<name>` (key, header
`X-API-Key: <certificate key>.<private key key>`). The `nginx-ssh` push runs `ssh -o BatchMode=yes ... sh -s`
and feeds the script on stdin. NPM endpoints: `/api/tokens`, `/api/nginx/certificates`,
and `/api/nginx/certificates/<id>/upload` with the `certificate` and `certificate_key` fields.

## License

MIT, see [LICENSE](LICENSE).
