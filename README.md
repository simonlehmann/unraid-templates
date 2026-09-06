# unraid-templates

A collection of [Unraid](https://unraid.net/) Docker application templates,
maintained by Simon Lehmann and published to the
[Community Applications](https://forums.unraid.net/topic/38582-plug-in-community-applications/)
marketplace.

## Adding this repository to Unraid

1. In Unraid, go to **Apps → Settings** (or **Settings → Community Applications**).
2. Under **Template repositories**, add:
   ```
   https://github.com/simonlehmann/unraid-templates
   ```
3. Save. Templates from this repo will now show up under **Apps → Previous
   Apps / My Templates** (or search results, once published to the store).

## Templates

| App | Description |
| --- | --- |
| _none yet_ | |

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
