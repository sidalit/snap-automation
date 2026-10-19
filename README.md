# snap-automation

Build and publish amd64 and arm64 in parallel. Each architecture publishes
independently when its build succeeds. Only HEAD builds need source preparation.

## Use it

The repository needs `snap/snapcraft.yaml` configured for amd64 and arm64.
Add `.github/workflows/snap-build.yaml`:

```yaml
name: Snap build
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:
    inputs:
      build-from:
        type: choice
        options: [tag, head]
        default: tag
jobs:
  snap:
    uses: sidalit/snap-automation/.github/workflows/snap-build.yaml@v1.0.2
    with:
      build-from: ${{ inputs.build-from || 'tag' }}
      channel: beta
    secrets:
      SNAPCRAFT_STORE_CREDENTIALS: ${{ secrets.SNAPCRAFT_STORE_CREDENTIALS }}
```

Add `SNAPCRAFT_STORE_CREDENTIALS` in the repository's Actions secrets to publish.
Publishing requires a channel and a push, schedule, or manual run on the default
branch. PRs only build. The example above publishes tagged builds to beta;
manual runs can select HEAD instead.

| Input | Default | Purpose |
|---|---|---|
| `build-from` | `tag` | Use the recipe as written, or `head` for the latest upstream commit. |
| `channel` | empty | Store channel, such as `beta` or `edge`. Empty means build only. |
| `test-command` | empty | Optional command after installing the snap. Nothing runs by default. |

HEAD builds expect one Git part with `source-tag`; each architecture resolves
HEAD independently and sets `grade: devel`. Versions use `<version>-<short-commit>`,
including when HEAD is a release tag. Recipes using `adopt-info` need one standalone
`craftctl set version=...` command in an override step. Tag builds skip preparation.
The recipe is printed before building; artifacts use `<snap-name>-<arch>`, such
as `openspec-amd64` and `openspec-arm64`. A failed build or optional command
prevents that architecture from publishing.

The manual caller offers a dropdown; reusable workflows require a string input.
For weekly HEAD builds, see the [OpenSpec caller](https://github.com/sidalit/openspec-snap/blob/main/.github/workflows/snap-build.yaml).
Runs of the same workflow/ref are serialized: an active run finishes, while newer
runs can replace an older pending run.

For a specific check, pass a command such as:

```yaml
      test-command: snap run openspec --version
```

## Renovate

Add `renovate.json`:

```json
{
  "extends": ["github>sidalit/snap-automation#v1.0.2"],
  "assignees": ["your-github-username"]
}
```

Mark the upstream version in `snap/snapcraft.yaml`:

```yaml
    # renovate: depName=Fission-AI/OpenSpec
    source-tag: 'v1.14.2'
```

The preset disables the Dependency Dashboard and uses GitHub automerge for
upstream release updates. Enable Allow auto-merge and require both build jobs
in branch protection. Workflow and preset updates need manual review.
Each repository defines its own assignees.

## Releases

After committing changes, publish a new, unused version tag. Renovate opens PRs
to update pinned workflow and preset versions in repositories using this preset.

## License

[MIT](LICENSE), covering this repository's automation, configuration, and documentation.
