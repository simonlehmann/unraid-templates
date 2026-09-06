# step-ca

[step-ca](https://github.com/smallstep/certificates) is
[Smallstep](https://smallstep.com)'s open-source online certificate
authority. This template runs the official
[`smallstep/step-ca`](https://hub.docker.com/r/smallstep/step-ca) Docker
image so you can operate your own private CA on Unraid — issuing
short-lived TLS certificates to internal services, and/or acting as an ACME
server that other Docker containers (reverse proxies, apps, etc.) can pull
certificates from automatically.

step-ca has no web dashboard; everything is driven by its HTTPS API and the
`step` CLI, either from another machine or via `docker exec` into the
container.

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

## Appdata / backups

The appdata volume (`/home/step` in the container) holds your root and
intermediate CA private keys, the CA database, and its configuration.
**Back this up.** If you lose it, every certificate this CA has ever issued
becomes unverifiable, and you'll need to re-initialize a new CA (and
re-trust it everywhere) from scratch.

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
you changed it). Point your reverse proxy or ACME client (e.g. Caddy,
Traefik, acme.sh) at that directory URL instead of Let's Encrypt.

## Links

- Project: https://github.com/smallstep/certificates
- Docs: https://smallstep.com/docs/step-ca
- Docker image: https://hub.docker.com/r/smallstep/step-ca
- Template issues: https://github.com/simonlehmann/unraid-templates/issues
