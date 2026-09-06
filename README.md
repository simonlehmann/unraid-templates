# unraid-templates

A collection of [Unraid](https://unraid.net/) Docker application templates,
maintained by Simon Lehmann and published to the
[Community Applications](https://forums.unraid.net/topic/38582-plug-in-community-applications/)
marketplace.

## Getting these templates

Templates here are submitted to the official
[Community Applications](https://ca.unraid.net/submit) feed. Once a
template is accepted, it's discoverable directly from the **Apps** tab for
every Unraid user — no manual setup required.

There's no way to point Community Applications at a custom repository URL
like this one — that "Template repositories" settings field was removed in
Unraid 6.10.0 and isn't coming back, precisely because the official feed
already covers almost everything and open repo URLs were an unnecessary
risk. (See this
[forum thread](https://forums.unraid.net/topic/112170-allow-template-repositories-to-be-hosted-from-other-sources/)
for the background.)

If you want to run a template from this repo *before* it's been
submitted/accepted, install it manually by copying its XML onto the
Unraid flash drive, into one of:

- `/boot/config/plugins/dockerMan/templates-user/` — the container then
  appears in Docker's **Add Container** template dropdown.
- `/boot/config/plugins/community.applications/private/<your-name>/` — the
  container appears under a **Private** category in Community Applications
  instead, and can still be managed from within CA.

e.g. for step-ca:

```bash
mkdir -p /boot/config/plugins/dockerMan/templates-user/
cp step-ca/step-ca.xml /boot/config/plugins/dockerMan/templates-user/
```

then in Unraid, go to **Docker → Add Container** and pick it from the
template dropdown (or **Apps → Private** if you used the CA folder
instead).

## Templates

| App | Description |
| --- | --- |
| [step-ca](step-ca/) | Private online certificate authority ([smallstep/step-ca](https://github.com/smallstep/certificates)) — issue TLS/SSH certs and run your own ACME server. |

## Repository structure

Each template lives in its own directory:

```
unraid-templates/
├── README.md
├── LICENSE
├── ca_profile.xml       # Repo metadata shown by Community Applications
│
├── <app-name>/
│   ├── <app-name>.xml   # Unraid Docker template
│   ├── README.md        # App-specific documentation
│   └── icon.svg
│
└── ...
```

## License

[MIT](LICENSE)
