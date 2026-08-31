# transglot-action

**Ship every language on every push.** The official GitHub Action for
[transglot](https://transglot.ai): it sends your source strings for translation,
waits for the batch, and commits the finished locale files back to your branch
before the build is over.

[![npm](https://img.shields.io/npm/v/transglot?label=transglot%20CLI)](https://www.npmjs.com/package/transglot)
[![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

```yaml
- uses: transglot/transglot-action@v1
  with:
    token: ${{ secrets.TRANSGLOT_TOKEN }}
```

That is the whole integration. No dashboard to wire up, no translation files to
shuttle by hand, no separate release step for localization.

## Why teams wire this in

- **Translation stops being a release blocker.** New strings are picked up on the
  same push that introduced them, so `main` is never waiting on a language.
- **Powered by Glossa**, our own localization engine. Glossa Flux drafts everyday
  copy fast; Quality Mode switches to Glossa Deep with a corrective retry pass for
  higher-stakes strings. Placeholders, plurals and brand terms survive both.
- **You never pay to translate the same sentence twice.** Translation memory
  reuses what you have already approved, and your glossary is enforced server
  side so product names and do-not-translate terms come back untouched.
- **20 file formats**, so it fits the stack you already have rather than the other
  way round.
- **Your pipeline learns the truth.** The action fails the job with a real exit
  code and exposes `batch_id` and `files_changed` as outputs, so a quota wall or a
  failed batch shows up as a red build instead of a silent no-op.

<details>
<summary><strong>All 20 supported formats</strong></summary>

JSON (flat and nested), Android XML, iOS `.strings`, `.stringsdict` and
`.xcstrings`, Flutter ARB, Laravel PHP and JSON, XLIFF 1.2 and 2.0, gettext PO,
YAML, CSV, XLSX, Java properties, RESX, Markdown, Unity CSV, Unreal PO.

</details>

## Quick start

**1. Get a project access token.** Create one (`tgl_...`) in your project settings
in the web UI. Token creation is a paid-plan feature, so the Free tier cannot
drive this action.

**2. Describe your files, once.**

```bash
export TRANSGLOT_TOKEN=tgl_your_token_here
npx transglot init
```

`init` uses the token to confirm the project, then asks for your server URL and
file format. It writes a `transglot.json` holding that URL plus the `sources` and
`pull` path globs. The token itself never goes in the file.

**3. Add the token as the repository secret `TRANSGLOT_TOKEN`, then the workflow:**

```yaml
name: Sync translations

on:
  push:
    branches: [main]

concurrency:
  group: transglot-${{ github.ref }}

jobs:
  translate:
    runs-on: ubuntu-latest
    permissions:
      contents: write # required: commit "branch" pushes translated files back
    steps:
      - uses: actions/checkout@v4
      - uses: transglot/transglot-action@v1
        with:
          token: ${{ secrets.TRANSGLOT_TOKEN }}
          url: https://api.transglot.ai # optional when transglot.json has "url"
          wait: 'true'   # block until the batch is terminal, then pull
          commit: branch # or: artifact | none
```

The `contents: write` permission is not optional while `commit` is at its
`branch` default. The action pushes translated files back, and a repository on
the default read-only `GITHUB_TOKEN` fails there with a 403.

## What happens on each run

1. **Push.** Your source strings go up. Anything genuinely new is queued for
   translation; anything already in translation memory is reused.
2. **Wait** (when `wait: 'true'`). The action blocks until the batch reaches a
   terminal state rather than guessing at a delay.
3. **Pull.** Finished locale files are written back to the paths in your
   `transglot.json`.
4. **Commit.** Depending on `commit`, the pulled files are committed and pushed to
   your branch, uploaded as a build artifact, or simply left in the workspace.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `token` | yes | | Project access token (`tgl_...`). Store it as a repository secret, never in the workflow file. |
| `url` | no | `''` | Server base URL. Optional when `transglot.json` already carries `url`. |
| `wait` | no | `'true'` | `'true'` waits for the batch after push, `'false'` fires and forgets. |
| `commit` | no | `'branch'` | What to do with pulled files: `branch` (commit and push), `artifact` (upload), or `none`. |
| `cli-version` | no | `'0.2.0'` | `transglot` CLI version on npm. A value starting with `.`, `/` or `file:` is treated as a local tarball path. |

## Outputs

| Output | Description |
| --- | --- |
| `batch_id` | Id of the translation batch queued by push. Empty when none was queued. |
| `files_changed` | Number of files that `pull` actually rewrote. |

Both outputs are populated even when the sync fails, for every exit code the CLI
itself returns (1 to 6), so you can branch on them in later steps:

```yaml
      - id: sync
        uses: transglot/transglot-action@v1
        with:
          token: ${{ secrets.TRANSGLOT_TOKEN }}
      - if: ${{ steps.sync.outputs.files_changed != '0' }}
        run: echo "Batch ${{ steps.sync.outputs.batch_id }} changed ${{ steps.sync.outputs.files_changed }} files"
```

## Exit codes

The action fails the job with the CLI's exit code, so your pipeline reflects what
actually happened:

| Code | Meaning |
| --- | --- |
| `0` | Success. |
| `1` | Config or usage error. |
| `2` | Authentication failed. |
| `3` | Quota exceeded, batch suppressed. Source files were still synced. |
| `4` | Batch finished failed, completed with errors, or was cancelled. |
| `5` | `wait: 'true'` timed out before the batch became terminal. |
| `6` | Server or network error. |

Codes `3` and `4` are informational rather than fatal to the sync itself. The
action still writes both outputs and still commits whatever translated files did
land before surfacing the code, so a partially failed batch never costs you the
work that succeeded.

## Versioning

Pin to the moving major tag `@v1` to pick up compatible updates, or to an exact
release such as `@v0.2.0` for a frozen build. The `cli-version` default tracks the
`transglot` CLI release each tag was cut against.

## Other CI systems

Not on GitHub Actions? The same flow runs anywhere Node does:

```bash
npx --yes transglot push --wait
npx --yes transglot pull
```

## Support

Docs, pricing and contact: [transglot.ai](https://transglot.ai).

## License

MIT. See [LICENSE](LICENSE).
