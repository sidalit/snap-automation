# snap-automation

Build and publish amd64 and arm64 in parallel. Each source/architecture runs
fetch, prepare, build, optional test, and publish steps in one job. Only HEAD
builds need source preparation.

## Use it

The repository needs `snap/snapcraft.yaml` configured for amd64 and arm64.
Add `.github/workflows/snap-build.yaml`:

```yaml
name: Snap build
on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '17 3 * * 1'
  workflow_dispatch:
    inputs:
      build-from:
        type: choice
        options: [tag, head]
        default: tag
      publish:
        type: boolean
        default: false
jobs:
  snap:
    uses: sidalit/snap-automation/.github/workflows/snap-build.yaml@v1.1.0
    with:
      build-from: ${{ github.event_name == 'schedule' && 'head' || inputs.build-from || 'tag' }}
      publish: ${{ github.event_name == 'push' || github.event_name == 'schedule' || inputs.publish == true }}
    secrets:
      SNAPCRAFT_STORE_CREDENTIALS: ${{ secrets.SNAPCRAFT_STORE_CREDENTIALS }}
```

Add `SNAPCRAFT_STORE_CREDENTIALS` in the repository's Actions secrets to publish.
Pushes publish the release tag to `beta`; weekly runs publish `HEAD` to `edge`.

| Input | Default | Purpose |
|---|---|---|
| `build-from` | `tag` | Use the recipe tag or upstream `head`. PRs always build both. |
| `publish` | `false` | Publish to `edge` for HEAD or `beta` for the release tag. |
| `test-command` | empty | Optional command after installing the snap. Nothing runs by default. |

Declare the release version once and reuse it in the upstream tag:

```yaml
version: '0.40.5'
parts:
  pocketbase:
    source: https://github.com/pocketbase/pocketbase.git
    source-tag: v$SNAPCRAFT_PROJECT_VERSION
```

HEAD builds replace its tag with the full upstream commit and set
`version: git-<7-character-sha>` and `grade: devel`.

Artifacts use `<snap-name>-<version>-<arch>`, such as `pocketbase-git-6f370ea-amd64`.

For a specific check, pass a command such as:

```yaml
      test-command: snap run openspec --version
```

## Renovate

Add `renovate.json`:

```json
{
  "extends": ["github>sidalit/snap-automation#v1.1.0"],
  "assignees": ["your-github-username"]
}
```

Renovate reads the upstream GitHub URL from the part using `SNAPCRAFT_PROJECT_VERSION`
and updates `version` from releases using `semver`.

## Releases

After committing changes, publish a new, unused version tag. Renovate opens PRs
to update pinned workflow and preset versions in repositories using this preset.

## License

[MIT](LICENSE), covering this repository's automation, configuration, and documentation.
