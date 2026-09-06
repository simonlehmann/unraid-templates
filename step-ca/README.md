# step-ca

**Private Certificate Authority for Unraid** — trusted HTTPS certificates
for services on your LAN, plus an internal ACME server for automated
certificate issuance. Built on
[step-ca](https://github.com/smallstep/certificates), Smallstep's
open-source online certificate authority, via the official
[`smallstep/step-ca`](https://hub.docker.com/r/smallstep/step-ca) Docker
image.

> **Status:** this template is a working starting point but hasn't yet
> been through a full first-run → persistence → upgrade lifecycle test on
> real Unraid hardware, so it isn't submitted to Community Applications
> yet. See [Before you rely on this](#before-you-rely-on-this) below.

step-ca has no web dashboard; everything is driven by its HTTPS API and the
`step` CLI, either from another machine or via `docker exec` into the
container.

## Why a private CA instead of self-signed certificates?

A **self-signed certificate** is its own root of trust — each service you
run this way needs to be trusted individually, on every client:

```
gitea.home.arpa  → signed by itself → trust it on every client, one by one
kuma.home.arpa   → signed by itself → trust it on every client, one by one
```

A **private CA** signs certificates for all of your services, so clients
only need to trust the CA's root certificate once:

```
Home CA (root, trusted once)
  ├─ signs → gitea.home.arpa
  ├─ signs → kuma.home.arpa
  └─ signs → *.home.arpa (wildcard)
```

That's what step-ca gives you, and it's the recommended approach here
rather than generating independent self-signed certificates per service.

## Example architecture

The common pattern this template is built for: an internal DNS server
resolves your LAN hostnames, step-ca issues certificates for them via
ACME, and a reverse proxy terminates HTTPS and forwards to your actual
services (which can keep speaking plain HTTP internally):

```
                    LAN
                     │
        ┌────────────┴────────────┐
        │                         │
  internal DNS               step-ca
 (e.g. Technitium,               │
  Pi-hole, AdGuard)             ACME
  *.home.arpa → LAN IP           │
        │                        │
        └────────────┬───────────┘
                      │
              reverse proxy
                  :443
                      │
       ┌──────────────┼──────────────┐
       │              │              │
   service-a      service-b      service-c
   (HTTP only,    (HTTP only,    (HTTP only,
    internal)      internal)      internal)
```

- **DNS** resolves your internal hostnames (e.g. `*.home.arpa`) to your
  Unraid/reverse-proxy IP. Any internal DNS server works.
- **step-ca** issues certificates for those hostnames via ACME.
- **The reverse proxy** requests/renews certificates from step-ca and
  terminates HTTPS, so the backend services don't need TLS configured at
  all.

`home.arpa` is used above only as a worked example — it's a domain
[reserved for home networks](https://datatracker.ietf.org/doc/html/rfc8375),
which makes it a reasonable default if you don't already have an internal
naming scheme. This template does not require it; use whatever internal
namespace you like.

## First run

The container **initializes a brand-new CA the first time it starts**, as
long as its appdata directory is empty. Initialization is driven entirely
by environment variables (there's no interactive setup) — the entrypoint
script provided by the image runs `step ca init` non-interactively using
whatever you've filled in below.

Before starting the container for the first time, set:

- **CA Name** — a friendly name for your CA, e.g. `Home Lab CA`.
- **DNS Names** — comma-separated hostnames/IPs that clients will use to
  reach this CA, e.g. `ca.home.arpa,192.168.1.10`. Include every name/IP
  you expect to connect to the CA with, since the CA's own HTTPS
  certificate is only valid for these names.

If either is left blank, the container will start but `step-ca` itself
will fail to launch (it needs a config that doesn't exist yet) — check the
container log, fill in the missing value, and restart.

Everything else has a sensible default:

- **Enable ACME** defaults to `true`, so the CA comes up with a working
  ACME provisioner other services can use immediately.
- **CA Password** is optional. Leave it blank and step-ca generates a
  random password for you, printed **once** to the container log right
  after the first start — copy it somewhere safe immediately, since it's
  needed to unlock the CA's private keys (e.g. after a restore) and isn't
  shown again.
- The advanced options (SSH CA support, remote management, provisioner/
  admin names) are off/default and only matter on that first
  initialization — see
  [smallstep's Docker docs](https://github.com/smallstep/certificates/tree/master/docker)
  for what each one does.

None of the "only read during first-run initialization" variables have any
effect once `ca.json` exists in the appdata path — to reconfigure them
later, use the `step` CLI inside the container instead of changing these
fields and restarting.

## Trusting the root certificate

Clients need to trust your new root CA before they'll accept certificates
it issues. Fetch the root certificate from the running container, e.g.:

```bash
docker exec step-ca step ca root
```

or over HTTP once it's running:

```bash
curl https://<your-unraid-ip>:9000/roots.pem
```

then install it into your OS/browser trust store, or into whatever service
needs to validate certs issued by this CA.

## Using the ACME endpoint

With ACME enabled, other services can request certificates from:

```
https://<your-unraid-ip>:9000/acme/acme/directory
```

(`acme` here is the default provisioner name step-ca creates; adjust if
you changed it.) A wildcard certificate (e.g. `*.home.arpa`) covering all
your internal hostnames typically needs a DNS-01 challenge, since ACME's
HTTP-01 challenge can't validate a wildcard.

**Reverse proxy compatibility varies** — check before you commit to one:

- [Caddy](https://caddyserver.com/) and [Traefik](https://traefik.io/) both
  support pointing at a custom ACME directory URL natively (Caddy's global
  `acme_ca` option, Traefik's `caServer` setting), which makes them a
  straightforward fit for step-ca.
- **Nginx Proxy Manager** (jc21/NginxProxyManager) is certbot-based and, as
  of writing, doesn't expose a custom ACME server URL in its UI — it's
  built around Let's Encrypt. If you want to use step-ca with NPM, the
  practical options are to request certificates yourself (via the `step`
  CLI, or `acme.sh` pointed at step-ca's ACME directory with `--server`)
  and upload the result as a "Custom" certificate in NPM, or to automate
  that with a small renewal script/cron job. Double-check NPM's current
  release before assuming this — a native option may have landed since.

## Security considerations

Treat this CA as sensitive infrastructure, not just another container:

- The appdata volume (`/home/step` in the container) holds your root and
  intermediate CA private keys, the CA database, and its configuration.
  **Back it up**, and make sure it survives container recreation and image
  upgrades.
- Losing the private key means losing the ability to issue new certificates
  from this CA — you'd need to stand up a new root and re-trust it
  everywhere.
- Destroying or re-initializing the CA creates a **new** trust hierarchy;
  every certificate the old CA issued becomes unverifiable, and clients
  will need to trust the new root from scratch.
- Treat the CA password (and the provisioner password shown once on first
  init) as a secret, same as any private key material.
- This is meant for LAN-only use. There's no need to expose step-ca to the
  internet, and no reason to put it behind an internet-facing reverse
  proxy.

## Upgrading

Recreating the container with a newer `smallstep/step-ca` image tag
reuses the same appdata volume and should not touch your CA's identity —
step-ca only initializes when `ca.json` doesn't already exist. Still,
treat any upgrade of CA infrastructure with a bit of care: check the
[release notes](https://github.com/smallstep/certificates/releases) for
breaking changes, and don't delete or recreate the appdata path as part of
an upgrade.

## Before you rely on this

This template hasn't yet been validated end-to-end on a real Unraid host —
in particular: first-run initialization with the exact current
`smallstep/step-ca` image, that appdata correctly survives a container
stop/recreate without reinitializing the CA, and that an image upgrade
preserves the CA and its issued certificates. If you hit anything that
doesn't match what's documented here, please
[open an issue](https://github.com/simonlehmann/unraid-templates/issues).

## Links

- Project: https://github.com/smallstep/certificates
- Docs: https://smallstep.com/docs/step-ca
- Docker image: https://hub.docker.com/r/smallstep/step-ca
- Template issues: https://github.com/simonlehmann/unraid-templates/issues
