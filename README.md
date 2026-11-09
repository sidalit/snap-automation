# snap-automation

Build and publish amd64 and arm64 in parallel by default. Each source/architecture runs
fetch, prepare, build, optional test, and publish steps in one job. Only HEAD
builds need source preparation.

## Use it

The repository needs `snap/snapcraft.yaml` configured for the selected architectures.
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
        options: [tag, head, both]
        default: both
      publish:
        type: boolean
        default: false
jobs:
  snap:
    uses: sidalit/snap-automation/.github/workflows/snap-build.yaml@v1.3.0
    secrets:
      SNAPCRAFT_STORE_CREDENTIALS: ${{ secrets.SNAPCRAFT_STORE_CREDENTIALS }}
```

Add `SNAPCRAFT_STORE_CREDENTIALS` in the repository's Actions secrets to publish.
By default, the shared workflow builds both sources on PRs and pushes, HEAD on
weekly runs, and the source selected in the caller's manual dropdown. An explicit
`with.build-from` overrides these defaults for every event. Tag builds publish to
`beta`; HEAD builds publish to `edge`.

| Event | Publishing |
|---|---|
| Pull request | Never. |
| Push to the default branch | Automatic. |
| Schedule | Automatic. |
| Manual run | Only when `publish` is selected, on the default branch. |

Other branches and events never publish. Publishing runs only after successful
build and test steps.

Snap components are saved and uploaded with their snap; the Snap Store must
enable components for the snap first.
If an upload loses its connection to the Snap Store before the store starts
processing it, publishing retries up to twice. Store errors, such as permission
or review failures, fail at once.

| Input | Default | Purpose |
|---|---|---|
| `architectures` | `["amd64", "arm64"]` | JSON array of architectures to build: `amd64`, `arm64`, or both. |
| `build-from` | Manual selection, otherwise `head` for schedules or `both` | `tag` builds only the recipe tag, `head` builds only upstream HEAD, and `both` builds them in parallel. |
| `publish` | Manual checkbox, otherwise `false` | Publish manual runs. Pushes and schedules publish automatically; PRs never publish. |
| `test-command` | empty | Optional command after installing the snap. Nothing runs by default. |

For AMD64 only, add `architectures: '["amd64"]'` under `with`.

For release-only packages, set `build-from: tag` under `with`. This also applies
to PRs and scheduled runs and does not require HEAD source preparation.

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
  "extends": ["github>sidalit/snap-automation#v1.3.0"],
  "assignees": ["your-github-username"],
  "packageRules": [
    {
      "matchDepNames": ["upstream"],
      "overridePackageName": "pocketbase/pocketbase"
    }
  ]
}
```

Set `overridePackageName` to the upstream GitHub `owner/repo`.
Renovate updates `version` in `snapcraft.yaml` from releases using `semver`.
For custom tags such as `spec-v4.4.26`, add `"extractVersion": "^spec-v(?<version>.+)$"`
to the same rule.
Repository rules extend the shared defaults.

## Releases

After committing changes, publish a new, unused version tag. Renovate opens PRs
to update pinned workflow and preset versions in repositories using this preset.

## License

[MIT](LICENSE), covering this repository's automation, configuration, and documentation.
