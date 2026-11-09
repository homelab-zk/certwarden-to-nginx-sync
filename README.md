# certwarden-to-npm-sync

A small Python tool (standard library only) that pushes certificates from
[Cert Warden](https://www.certwarden.com/) to [Nginx Proxy Manager](https://nginxproxymanager.com/) (NPM) as
**custom certificates**, and keeps them updated when Cert Warden renews them. It is meant to run daily from a
systemd timer.

Why: NPM keeps its certificates in its own database, so an external certificate manager cannot just drop files
into a folder. Pushing through NPM's API means the NPM host does not need internet access or its own
ACME/DNS credentials: only the Cert Warden host talks to the certificate authority. This suits an NPM in a
DMZ, and one Cert Warden serving several NPM instances.

Tested with NPM 2.15.1 and Cert Warden 0.30.

## How it works

For each target in the config file:

1. Download the full chain and the private key from Cert Warden's API.
2. Log in to NPM's API.
3. If no NPM custom certificate with the target's name exists, create one. Otherwise upload over the **same**
   certificate, so proxy hosts that use it keep working with no change.
4. Remember a SHA-256 of the chain, so an unchanged certificate is skipped on the next run.

Proxy hosts are never touched. Cert Warden renews a certificate when about a third of its lifetime is left,
and the daily timer then delivers it within about a day.

To move many **existing** hosts onto the pushed certificate in one go, see
[npm-swap-cert](https://github.com/homelab-zk/npm-swap-cert).

## Requirements

- Python 3.11 or newer (uses `tomllib`), no extra packages.
- Cert Warden with the certificate already issued, plus an API key for each certificate and for its private
  key (Cert Warden: edit the certificate / private key, API key).
- An NPM user for the script. NPM identifies users by **e-mail address**. Give it only the
  *certificates: manage* permission; it does not need to be an admin. NPM shows a non-admin user only the
  items that user owns, so it cannot see or change your proxy hosts. That is intentional.
- The machine that runs the script must reach both Cert Warden and NPM's admin/API port.

## Security notes

- The tool handles the **private keys** of your certificates and an NPM login. Keep the env file readable by
  root only (`chmod 600`) and run the tool on a host you trust.
- Secrets are never printed. The env file is parsed as plain text, not executed by a shell, so special
  characters in passwords need no escaping.
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
3. Create `/etc/certwarden-to-npm-sync/env` from [`examples/env.example`](examples/env.example), fill in the NPM
   login and the Cert Warden API keys for each target, then `sudo chmod 600 /etc/certwarden-to-npm-sync/env`.
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

## Options

| Option | Meaning |
|---|---|
| `--only NAME` | Only this target. |
| `--force` | Upload even if the certificate has not changed. |
| `--dry-run` | Show what would happen, change nothing. |
| `--config FILE`, `--env FILE`, `--state-dir DIR` | Override the default paths under `/etc/certwarden-to-npm-sync` and `/var/lib/certwarden-to-npm-sync`. |

## Monitoring

The tool does not send alerts. Monitor the certificate your proxy actually serves (for example with Zabbix or
Uptime Kuma, alerting at 14 and 7 days left) to catch a failure anywhere in the chain: Cert Warden, the timer,
the upload, or NPM. NPM may also need to reload nginx before a renewed certificate is served; check this at
your first renewal.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `NPM login failed (HTTP 400)` | NPM logs in with the user's **e-mail address**, not a username. Check the password and that 2FA is off for the script user. |
| `Cert Warden certificate download failed (HTTP 401/404)` | Wrong API key, or `cw_cert` in `targets.toml` does not match the certificate name in Cert Warden. |
| `no private key found in Cert Warden response` | The private-key API key is wrong, or the certificate has no key attached. |
| `no target named ...` | The name given to `--only` is not in `targets.toml`. |
| Config or env file not found | Create them, or pass `--config` and `--env`. |

## Details

Cert Warden endpoints used: `/certwarden/api/v1/download/certificates/<name>` (full chain, header
`X-API-Key: <certificate key>`) and `/certwarden/api/v1/download/privatecerts/<name>` (key, header
`X-API-Key: <certificate key>.<private key key>`). NPM endpoints: `/api/tokens`, `/api/nginx/certificates`,
and `/api/nginx/certificates/<id>/upload` with the `certificate` and `certificate_key` fields.

## License

MIT, see [LICENSE](LICENSE).
