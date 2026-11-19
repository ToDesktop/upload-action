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
          # A tag (v1.2.3) names the version; other runs use todesktop.config.ts's.
          version: ${{ github.ref_type == 'tag' && github.ref_name || '' }}
```

The action reads `todesktop.config.ts` (or `todesktop.config.json`) from the
working directory, checks each app it lists that is on this runner, runs the
config's `prepareApp` hook on a copy, and uploads it. It authenticates with the
job's GitHub OIDC token, so there are no secrets to add: connect the repository
to your app in the ToDesktop dashboard and allow the branches or tags that
release.

## Waiting for the result

The upload ends when the apps are in: ToDesktop signs, packages and smoke-tests
the build afterwards. To have the workflow run itself say how that went, add a
last job that waits. It uploads nothing and needs no checkout, only the job's
own identity:

```yaml
report: # waits for ToDesktop's result of a test build, and fails with the reason
  needs: release # every job that uploads
  if: ${{ github.ref == 'refs/heads/todesktop-test' }}
  runs-on: ubuntu-latest
  permissions:
    id-token: write
  steps:
    - uses: ToDesktop/upload-action@v1
      with:
        wait: only
        app: your-app-id
```

The `if` keeps it to test builds (`test: true`), here the pushes to the
`todesktop-test` branch. Without it the job waits for every run's build.

The job prints a line when something changes (uploads, each platform's build,
each smoke-test machine), then the result, which is also the job's summary:

| Result                                         | The job                                       |
| ---------------------------------------------- | --------------------------------------------- |
| The build passed its smoke test                | passes                                        |
| The build failed, or its smoke test did        | fails, with the reason as an error on the run |
| It was built, and no smoke test tested it      | passes, with a warning that says why          |
| No result after `wait-timeout` minutes         | fails; not a verdict, re-run the job to wait  |
| ToDesktop can't say (a server without results) | passes, with a warning                        |

Anyone who can read the run can read the result, so a coding agent with access
to the repository needs no ToDesktop sign-in to learn why a test build failed.

A failed smoke test's report has what the test machine saw: the step that
failed, whose code was running, and the end of the app's output. That output is
your app's own, and a public repository's logs are public, so by default it is
printed only when the repository is private (`details`).

`wait: true` does the same at the end of an upload step, in a workflow whose one
job uploads everything.

## Inputs

| Input          | Default                                | What                                                                                                                                                        |
| -------------- | -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `version`      | the config's, or the Mac app's         | The version to release; a leading `v` is dropped.                                                                                                           |
| `test`         | `false`                                | `true` makes a test build: built, signed and smoke-tested, never released. Every job of the run passes the same value. Without a version it's `0.0.0-test`. |
| `config`       | `todesktop.config.ts` in the directory | Path to the config file.                                                                                                                                    |
| `wait`         | `false`                                | `true` uploads, then waits for the build's result. `only` uploads nothing and waits for the result of this workflow run's build.                            |
| `wait-timeout` | `90`                                   | Minutes to wait. Running out fails the job without a verdict; re-running it waits again.                                                                    |
| `app`          | read from the config                   | The app's ID, for `wait: only` in a job with no checkout.                                                                                                   |
| `details`      | `auto`                                 | Whether a failed smoke test prints the end of the app's output: `auto` (only in a private repository), `true` or `false`.                                   |
| `url`          | `https://updates.todesktop.com`        | Where to upload. Leave it unset unless ToDesktop gives you another URL.                                                                                     |

`v1` follows the latest 1.x release; pin a full tag (`v1.2.3`) or a commit SHA
to stay on one.
