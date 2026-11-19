# MailRules

**Self-hosted email sorting for Unraid.** MailRules watches your mailboxes
over IMAP and files new email in real time, using rules you write in plain
English ("receipts and invoices go to Receipts") or as structured
conditions. An AI model decides which plain-English rule an email matches,
and it can be your own [Ollama](https://ollama.com) server, so no mail
leaves your network. Built on
[TurtleByte-IN/mailrules](https://github.com/TurtleByte-IN/mailrules), via
the official `ghcr.io/turtlebyte-in/mailrules` image (amd64 and arm64).

> **Status:** this template is a working starting point but hasn't yet
> been through a full install → persistence → upgrade test on real Unraid
> hardware, so it isn't submitted to Community Applications yet. See
> [Before you rely on this](#before-you-rely-on-this) below.

MailRules starts safe: **dry-run is on**, so it records what it *would*
do and changes no mailbox until you switch dry-run off. It never
permanently deletes mail, fetches without marking mail read, and every
action can be undone for 30 days.

## What works and what doesn't

- **Works:** any IMAP mailbox that signs in with a password or app
  password: iCloud, Gmail/Google Workspace, Fastmail, Yahoo, Zoho, or your
  own mail server (including one on your LAN).
- **Doesn't work:** Outlook.com and Microsoft 365, which only accept
  Microsoft's own sign-in. The self-hosted build doesn't include it.
- **Proton Mail:** upstream says Proton Mail Bridge doesn't work from
  Docker, because Bridge listens on `127.0.0.1` and inside a container
  that's the container itself. A Bridge running in its own container and
  reachable on a LAN or Docker-network address *might* work, but this
  hasn't been tested.
- **Not included:** "Suggest from my mail" is a feature of the hosted
  version at [mailrules.app](https://mailrules.app/) only. The free build
  hides it.

## Example architecture

The setup this template is built for: MailRules runs on Unraid, uses an
Ollama server you already have, and is reached in the browser through
an HTTPS reverse proxy such as Nginx Proxy Manager. It's never exposed to
the internet.

```
        Browser
           │ HTTPS  https://mailrules.home.arpa
           ▼
  Nginx Proxy Manager ──── internal DNS: mailrules.home.arpa → Unraid IP
           │ HTTP  [unraid-ip]:8080
           ▼
       MailRules  ──── IMAP over TLS ────▶  your mail provider(s)
           │
           │ HTTP  [ollama-ip]:11434
           ▼
        Ollama (local model)
```

None of it is required: MailRules works on its own at
`http://[unraid-ip]:8080`, with a cloud model instead of Ollama, or with
condition-only rules that need no model at all.

## Installation

### 1. Prepare the appdata folder

The image is distroless and runs as uid `65532`, not root, and the folder
Docker creates for a new bind mount belongs to root. MailRules then can't
create its database and won't start. Before the first start, open
**Terminal** in the Unraid web UI and run:

```bash
mkdir -p /mnt/user/appdata/mailrules
chown 65532:65532 /mnt/user/appdata/mailrules
chmod 700 /mnt/user/appdata/mailrules
```

(Change the path if you change **Appdata** in the template.) Do this again
whenever you restore a backup or move the folder.

### 2. Install the template

Install from **Apps** once the template is in Community Applications, or
manually before then (see the [repository README](../README.md#getting-these-templates)).

The settings that matter on first install:

| Setting | Default | Notes |
| --- | --- | --- |
| WebUI Port | `8080` | Change it if another container already uses 8080. |
| Appdata | `/mnt/user/appdata/mailrules` | Must be owned by uid 65532 (step 1). |
| Secure Cookie | `false` | Leave it `false` until you have an HTTPS proxy in front, see [HTTPS](#https-with-nginx-proxy-manager). |
| Ollama URL | empty | Optional, see [Using Ollama](#using-ollama). |

Everything under **Show more settings** is optional. Model choice,
thresholds and API keys are easier to set in MailRules' own **Settings**
page, which overrides the environment variables anyway.

### 3. First run

1. Start the container and open the **WebUI** right away. The first
   visit shows **Set up MailRules** and creates the only admin account,
   so don't leave a fresh install sitting unclaimed on a shared network.
   The password must be at least 12 characters.
2. Choose a **decision model** and give it a key, or skip that. Rules
   built only from conditions, and sender rules, need no model.
3. Connect a mailbox with an **app password** (not your normal password,
   for providers that have them), and pick a few starter rules.
4. Watch **Activity** while dry-run is on. When the decisions look right,
   switch dry-run off.

Upstream's [First run](https://github.com/TurtleByte-IN/mailrules/blob/main/docs/guide/first-run.md)
guide covers each step in detail.

## Using Ollama

MailRules talks to Ollama directly. It needs the server URL and a model
name, and Ollama costs nothing per email.

1. Make sure Ollama listens on more than loopback. For the Unraid Ollama
   container this is already the case. For a standalone install, set
   `OLLAMA_HOST=0.0.0.0`.
2. Set **Ollama URL** in the template to the server's LAN address, e.g.
   `http://192.168.1.10:11434`. Don't use `localhost` or `127.0.0.1`:
   inside the MailRules container that's the container itself. If both
   containers share a custom Docker network, the container name works
   too, e.g. `http://ollama:11434`.
3. In MailRules, go to **Settings → Decision model**, choose **Ollama on
   my server** and enter a model name you've pulled (e.g. `llama3.2`). To
   do the same through the template instead, set **Decision Model** to
   `ollama` and **Decision Model Name** to the model.
4. To have Ollama write rules from your plain-English descriptions too,
   set **Rule composer model** to `ollama:<model>`, e.g. `ollama:llama3.2`.
   Its default is Claude Haiku, which needs an Anthropic key, and without
   a working composer **Describe it** / **Rewrite with AI** don't work.
   **Build with conditions** and **Templates** still do.

Without an Anthropic API key, the **fallback model** (a Claude second
opinion when the decision model is unsure) is inactive and MailRules
logs a warning saying so. That's expected for an all-local setup:
decisions then rest on the local model alone, and any below the **Act at**
confidence threshold wait in **Needs review**. Upstream's
[Decision models and keys](https://github.com/TurtleByte-IN/mailrules/blob/main/docs/guide/models.md)
guide covers the other providers, thresholds and costs.

## HTTPS with Nginx Proxy Manager

To use MailRules from other devices, put a TLS reverse proxy in front and
reach it only through that. Don't forward the port to the internet:
without TLS your password and session cookie travel unencrypted.

**Proxy host in NPM:**

- **Domain:** `mailrules.home.arpa` (or your own local name), with a DNS
  record pointing at the proxy.
- **Forward:** `http` → your Unraid IP → `8080` (the WebUI Port).
- **SSL:** your internal certificate, e.g. one issued by
  [step-ca](../step-ca/) via ACME. Turn on *Force SSL*.
- **Websockets:** not needed. MailRules uses server-sent events on
  `/api/events` instead, and asks nginx not to buffer them. nginx's
  default 60s read timeout is enough, because MailRules sends a keep-alive
  every 25 seconds.
- **Advanced tab:** keep the Prometheus metrics endpoint, which needs no
  sign-in, off the proxy:

  ```nginx
  location = /metrics { return 404; }
  ```

**Then update the template** and apply:

| Setting | Value |
| --- | --- |
| Secure Cookie | `true`. Signing in at `http://[unraid-ip]:8080` stops working, which is what you want. |
| Public URL | `https://mailrules.home.arpa`, so links in the summary email work. |
| Trusted Proxies | The address MailRules sees requests from: see below. |

**Trusted Proxies.** Failed sign-ins are rate-limited per client address.
Behind a proxy every request comes from the proxy, so one person
mistyping their password would lock everyone out for a minute. Listing
the proxy lets MailRules read the real client address from
`X-Forwarded-For`, which NPM sets. What to list depends on how NPM is
networked:

- NPM on Docker's default **bridge** network: `172.17.0.0/16`.
- NPM on a **custom Docker network** shared with MailRules: that
  network's subnet (`docker network inspect <name>`).
- NPM on **br0** (its own LAN IP) or on another machine: that single IP,
  e.g. `192.168.1.20`.

Never list a range your LAN clients can reach MailRules from directly,
e.g. your whole LAN subnet: anyone in it could then forge their address
and get around the limit. MailRules refuses `0.0.0.0/0`. Even without this
setting, a separate per-account limit still stops password guessing.

## Backup and restore

**Back up the whole appdata folder, never just the database.** It holds:

- `mailrules.db` (plus `-wal` and `-shm` while running): rules, settings,
  activity, undo history, and your mailbox passwords and API keys in
  encrypted form.
- `master.key`: the key that decrypts those passwords and keys. **If it's
  lost, every stored mailbox password and API key is gone for good.**
  Rules, settings and history survive, but every app password and key has
  to be entered again.

The database is written continuously, so copy it with the container
**stopped**. The Appdata Backup plugin does this by default. Treat the
backup as sensitive as the passwords themselves: database and key
together unlock them.

**Upgrades** migrate the database on first start, and there's no way to
undo that, so back up before every upgrade. The template uses `latest`,
which upstream only moves after a release has passed its install tests. To
upgrade only when you choose, change **Repository** to a pinned version
such as `ghcr.io/turtlebyte-in/mailrules:0.3.0`.

**Restore:** stop the container, replace the folder's contents with the
backup, *including* `master.key`, fix ownership, then start:

```bash
chown -R 65532:65532 /mnt/user/appdata/mailrules
```

If `master.key` is missing and the database holds stored passwords,
MailRules deliberately refuses to start rather than create a new key, and
says so in the log. Put the key back and start again. If the key is truly
gone, start **once** with an extra variable `MAILRULES_NEW_MASTER_KEY=true`
(**Add another Path, Port, Variable…** in the template), then remove it
and re-enter your passwords and keys. Details are in upstream's
[Backup and the master key](https://github.com/TurtleByte-IN/mailrules/blob/main/docs/guide/backup.md).

## Security considerations

The template applies the same hardening as upstream's Docker Compose file:
`--read-only` root filesystem with a `/tmp` tmpfs, `--cap-drop=ALL`, and
`no-new-privileges`, with the container running as non-root uid 65532.

- **Unraid publishes the port on all interfaces.** Upstream's own setup
  publishes it to `127.0.0.1` only, which doesn't suit a NAS reached from
  other machines. Keep it LAN-only, and once you use a proxy, set
  **Secure Cookie** to `true` so the plain-HTTP port can't be used to sign
  in.
- **`Secure Cookie = false`** is only appropriate while you sign in over
  plain HTTP on a network you trust. It's the template default so the
  WebUI button works out of the box.
- **API keys** entered as template variables are visible to anyone who
  can run `docker inspect`. Entering them in MailRules' **Settings** page
  instead stores them encrypted with `master.key`.
- `/healthz`, `/readyz` and `/metrics` answer without sign-in. They hold
  no mail or account data.

## Troubleshooting

- **Container stops right after starting with a permission error in
  the log:** the appdata folder isn't owned by uid 65532. Re-run the
  commands in [step 1](#1-prepare-the-appdata-folder).
- **Can't stay signed in / login loops:** **Secure Cookie** is `true` (or
  `auto`) but you're using plain HTTP. Use the HTTPS proxy address, or set
  it to `false` for direct LAN access.
- **"the master key is missing":** see [Backup and restore](#backup-and-restore).
- **Ollama model unreachable:** check **Ollama URL** isn't `localhost`, and
  that the Unraid host can reach it:
  `curl http://<ollama-ip>:11434/api/tags`.
- **Forgot the admin password:** there's no email reset. From the Unraid
  terminal (the container has no shell, so the Console button won't work):

  ```bash
  docker exec -it mailrules /mailrules users reset-password
  ```

- **Logs:** the container log (Docker tab → mailrules icon → Logs) is JSON
  lines, and never contains passwords, keys or email text. Set **Log
  Level** to `debug` for more detail.

Upstream's
[Troubleshooting](https://github.com/TurtleByte-IN/mailrules/blob/main/docs/guide/troubleshooting.md)
guide covers mailbox-specific errors (iCloud, Gmail, Reconnect needed,
Certificate changed, …).

## Before you rely on this

This template hasn't yet been validated end-to-end on a real Unraid host,
in particular: first start with the appdata ownership step, sign-in over
plain HTTP and through Nginx Proxy Manager, the Ollama connection, and
that the database and `master.key` survive container recreation and an
image upgrade. [TESTING.md](TESTING.md) tracks that. If you hit anything
that doesn't match what's documented here, please
[open an issue](https://github.com/simonlehmann/unraid-templates/issues).

## Links

- Website and hosted version: <https://mailrules.app/>
- Source: <https://github.com/TurtleByte-IN/mailrules> (AGPL-3.0)
- User guide: <https://github.com/TurtleByte-IN/mailrules/tree/main/docs/guide>
- Settings reference: <https://github.com/TurtleByte-IN/mailrules/blob/main/docs/guide/settings.md>
- Image: <https://github.com/TurtleByte-IN/mailrules/pkgs/container/mailrules>
- Template issues: <https://github.com/simonlehmann/unraid-templates/issues>
