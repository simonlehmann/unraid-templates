# unraid-templates

A collection of [Unraid](https://unraid.net/) Docker application templates,
maintained by Simon Lehmann and published to the
[Community Applications](https://forums.unraid.net/topic/38582-plug-in-community-applications/)
marketplace.

## Getting these templates

Templates here are submitted to the official
[Community Applications](https://ca.unraid.net/submit) feed. Once a
template is accepted, it's discoverable directly from the **Apps** tab for
every Unraid user — no manual repository setup required.

Manually adding this repo is only needed if you want a template *before*
it's been submitted/accepted (e.g. to test one in progress), or if you'd
rather track this repo directly instead of relying on the aggregated feed:

1. In Unraid, go to **Apps → Settings** (or **Settings → Community Applications**).
2. Under **Template repositories**, add:
   ```
   https://github.com/simonlehmann/unraid-templates
   ```
3. Save. Templates from this repo will now show up under **Apps → Previous
   Apps / My Templates**.

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
