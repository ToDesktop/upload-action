# ToDesktop upload

Uploads the apps a GitHub Actions job built to ToDesktop, which signs,
notarizes, packages, smoke tests and releases them. Each job of a workflow run
adds to one build, so a job per platform or per arch works.

```yaml
permissions:
  id-token: write # lets ToDesktop check the job is yours
  contents: read

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - run: ./build.sh # your build, unsigned: ToDesktop signs
      - uses: ToDesktop/upload-action@v1
        with:
          # A tag (v1.2.3) names the version; other runs use todesktop.ts's.
          version: ${{ github.ref_type == 'tag' && github.ref_name || '' }}
```

The action reads `todesktop.ts` (or `todesktop.json`) from the working
directory, checks each app it lists that is on this runner, runs the config's
`prepareApp` hook on a copy, and uploads it. It authenticates with the job's
GitHub OIDC token, so there are no secrets to add: connect the repository to
your app in the ToDesktop dashboard and allow the branches or tags that release.

## Inputs

| Input     | Default                         | What                                                                    |
| --------- | ------------------------------- | ----------------------------------------------------------------------- |
| `version` | the config's, or the Mac app's  | The version to release; a leading `v` is dropped.                       |
| `config`  | `todesktop.ts` in the directory | Path to the config file.                                                |
| `url`     | `https://updates.todesktop.com` | Where to upload. Leave it unset unless ToDesktop gives you another URL. |

`v1` follows the latest 1.x release; pin a full tag (`v1.2.3`) or a commit SHA
to stay on one.
