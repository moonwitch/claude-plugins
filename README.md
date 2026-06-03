# claude-plugins

A Claude Code / Cowork plugin marketplace. Add it once, then install the plugins below.

## Add the marketplace

```
/plugin marketplace add moonwitch/claude-plugins
```

## Available plugins

| Plugin | Version | Description |
| --- | --- | --- |
| `gcp-terraform-scaffold` | 0.4.2 | Scaffolds a GCP Terraform project in Loop's house style — one repo per project, environments as git branches, file-per-service, impersonation-based GCS backend, branch-aware CI/CD. |

### Install

```
/plugin install gcp-terraform-scaffold@claude-plugins
/reload-plugins
```

Update later with:

```
/plugin marketplace update claude-plugins
/plugin update gcp-terraform-scaffold
```

## Repo layout

```
.
├── .claude-plugin/
│   └── marketplace.json        # lists the plugins (relative source paths)
├── gcp-terraform-scaffold/     # plugin source
│   ├── .claude-plugin/plugin.json
│   ├── skills/
│   └── README.md
└── README.md
```

## Releasing a new version

1. Edit the plugin and bump `version` in its `.claude-plugin/plugin.json` (semver).
2. Update the same `version` in `.claude-plugin/marketplace.json`.
3. Commit, then tag using the `<plugin-name>--v<version>` convention so one repo can host
   several plugins on independent version lines:

   ```
   git tag gcp-terraform-scaffold--v0.4.3
   git push origin main --tags
   ```

Note: if you ever drop the `version` field, every commit becomes a new version (git SHA).
Keeping an explicit `version` means teammates only get updates when you bump it.
