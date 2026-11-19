# MailRules template — pre-submission validation checklist

Tracks manual validation of this template against the real
`ghcr.io/turtlebyte-in/mailrules` image on actual Unraid hardware, before
submitting it to Community Applications. Not part of the published app
documentation.

## Basic installation

- [ ] With no appdata folder created beforehand, note who owns the
      folder Unraid/Docker creates (`ls -ld /mnt/user/appdata/mailrules`)
      and whether the container starts as 99:100 without a manual `chown`.
      If not, drop the "if" from the Requires note and README and make
      the `chown -R 99:100` a required pre-install step.
- [ ] If it fails, note the exact log message for the README's
      troubleshooting section, and confirm `chown -R 99:100` fixes it.
- [ ] `--user 99:100`, `--read-only`, `--tmpfs /tmp`, `--cap-drop=ALL` and
      `no-new-privileges` in Extra Parameters cause no errors.
- [ ] WebUI button opens `http://[IP]:8080/`.
- [ ] `/healthz` and `/readyz` respond.
- [ ] Log shows the expected `listening` line and version.

## First run

- [ ] **Set up MailRules** creates the admin account.
- [ ] Sign-in over plain HTTP works with Secure Cookie = `false`.
- [ ] Dry-run is on by default.
- [ ] A mailbox connects with an app password.
- [ ] Dry-run decisions appear in Activity; nothing changes in the mailbox.

## Ollama

- [ ] Ollama URL set in the template shows as **Set by environment** in
      Settings.
- [ ] Decision model `ollama` with a pulled model makes decisions.
- [ ] Rule composer `ollama:<model>` works for **Describe it**.
- [ ] Without an Anthropic key, the "fallback model is not active"
      warning appears and nothing else breaks.

## Reverse proxy (Nginx Proxy Manager)

- [ ] Proxy host to `[unraid-ip]:8080` serves the UI over HTTPS.
- [ ] Live updates (`/api/events`) keep working past 60 seconds.
- [ ] Secure Cookie = `true`: sign-in works over HTTPS and is refused
      over plain HTTP.
- [ ] Trusted Proxies value from the README matches the address
      MailRules actually sees for NPM on bridge, custom network and br0.
- [ ] `/metrics` returns 404 through the proxy.

## Persistence

- [ ] Stop/start: account, mailboxes, rules and keys retained.
- [ ] Remove and recreate the container (same appdata): nothing lost,
      no re-setup prompt.
- [ ] `master.key` exists in appdata owned by 99:100, mode 600.

## Backup and restore

- [ ] Appdata Backup plugin stops the container and backs it up.
- [ ] Restore into a fresh folder plus `chown -R 99:100`: works.
- [ ] Restore without `master.key`: MailRules refuses to start with the
      documented message.
- [ ] `MAILRULES_NEW_MASTER_KEY=true` for one start recovers, as documented.

## Upgrade

- [ ] Pinned tag (e.g. `0.3.0`) → newer tag: database migrates, data kept.
- [ ] `latest` update through the Unraid Docker tab works.

## Misc

- [ ] `docker exec -it mailrules /mailrules users reset-password` works.
- [ ] Icon renders in the Docker tab and in CA.
- [ ] Works on an arm64 host, if one is available.
