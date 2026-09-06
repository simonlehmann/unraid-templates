# step-ca template — pre-submission validation checklist

Tracks manual validation of this template against the real
`smallstep/step-ca` image on actual Unraid hardware, before submitting it
to Community Applications. Not part of the published app documentation.

## Basic installation

- [ ] Install using the Unraid template.
- [ ] Container starts.
- [ ] `/mnt/user/appdata/step-ca` is created.
- [ ] Port 9000 is reachable.
- [ ] `/health` reports healthy status.

## First-run configuration

- [ ] CA Name is honored.
- [ ] DNS Names is honored.
- [ ] Enable ACME creates a working ACME provisioner.
- [ ] CA Password (set) is honored; leaving it blank auto-generates one
      and prints it to the log exactly once.
- [ ] Generated `ca.json` matches the supplied configuration.
- [ ] Root certificate is generated and retrievable.

## Persistence

- [ ] Stop/start the container — CA identity retained.
- [ ] Recreate the container (same image) — CA is *not* reinitialized.
- [ ] Keys, database, and issued-certificate history are retained.
- [ ] First-run variables have no effect once `ca.json` exists.

## Certificate issuance

- [ ] Retrieve the CA root certificate.
- [ ] Trust the root on a test client.
- [ ] Issue a certificate for an internal hostname.
- [ ] Certificate validates and is trusted by the client.

## ACME

- [ ] ACME directory endpoint is reachable.
- [ ] Issue a certificate through ACME via at least one client
      (e.g. Caddy or Traefik pointed at the custom ACME directory).
- [ ] Automatic renewal works.
- [ ] Wildcard certificate issuance works (requires DNS-01).
- [ ] Confirm current Nginx Proxy Manager support (or lack of it) for a
      custom ACME server URL, and update the README if that's changed.

## Upgrade

- [ ] Recreate the container against a newer image tag.
- [ ] CA state survives; no reinitialization occurs.
- [ ] Existing certificates remain valid.
- [ ] CA database remains intact.

## Before submitting to Community Applications

- [ ] All of the above pass.
- [ ] Run the repo through the CA submission tooling at
      [ca.unraid.net/submit](https://ca.unraid.net/submit) (repository
      validation, template scan, duplicate-app check).
- [ ] Address any moderator feedback.
