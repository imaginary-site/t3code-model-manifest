# T3(ReImagined) model manifest

`model-manifest.json` is the provider model metadata that T3(ReImagined) reads at runtime. Editing it here changes the model list and the provider CLI compatibility checks without an app release.

## How the app reads it

- Every release bundles a copy and runs on it offline.
- A running server fetches `https://raw.githubusercontent.com/imaginary-site/t3code-model-manifest/main/model-manifest.json` at most once an hour, and again after a failed attempt no sooner than 5 minutes later.
- The Refresh control in provider settings, and every provider CLI update, fetch at once.
- The server keeps the last valid fetch on disk. A failed fetch or an invalid file never breaks the model list: the server keeps its last usable copy.
- `updatedAt` decides between copies. A fetched or cached file whose `updatedAt` is older than the bundled copy's is ignored, so bump it on every edit.
- Turning off provider update checks in settings stops the fetch.
- `T3CODE_MODEL_MANIFEST_URL` points a server at another copy.
- `raw.githubusercontent.com` caches files for a few minutes, so an edit can take that long to reach a server.

## Schema, version 1

- `version`: always `1`. Fields are only added. Servers ignore keys they do not know and reject versions they do not understand.
- `updatedAt`: ISO 8601 time of the last edit.
- `currentModels`: per driver, the slugs that count as current. Drivers use it to decide which models fold into the picker's legacy section; a catalog entry's `status` takes precedence.
- `providers.<driver>`: a static model catalog. Claude (`claudeAgent`) takes its whole built-in model list from it.
  - `models[]`: `slug`, `name`, optional `shortName`, `subProvider` and `aliases`, `status` (`current` or `legacy`), `profile` (a key of `profiles`), and a provider-specific `adapter`. For Claude: `{ "claudeCode": { "minVersion": "2.1.257" } }`, with an optional `maxVersionExclusive`. A model outside that Claude Code range is hidden.
  - `profiles`: shared `capabilities` (the picker's option descriptors) and an `adapter`. For Claude: `effortMap` (picker effort to `--effort` value, `null` omits the flag), `modelSuffixes` (for example `[1m]` for the 1M context window), `contextWindowTokens` and `fixedContextWindowTokens`.
  - `defaults.chat`: optional default model slug.
- `compatibility[]`: one policy per driver for its CLI.
  - `driver`, and `t3CodeRange`: the app versions the policy applies to.
  - Optional `recommendedRange` and `recommendedVersion`.
  - `ranges[]`: `{ "range": ">=2.1.257", "status": "supported" }`. Statuses are `supported`, `graceful` (limited support), `unsupported`, `broken` and `unknown`. The first match wins. A version no range covers that falls outside `recommendedRange` counts as `graceful`.
  - The app warns in provider settings on `graceful`, `unsupported` and `broken`, and never offers an update to a latest version marked `unsupported` or `broken`.

Ranges use space-separated comparators (`>=`, `>`, `<=`, `<`, `=`, `^`) and `||` between groups.

Driver names: `codex`, `claudeAgent`, `cursor`, `grok`, `opencode`, `acpRegistry`.

The server rejects the whole file when a slug repeats within a catalog, a `profile` or `defaults.chat` points nowhere, a Claude adapter does not decode, or a `recommendedVersion` falls outside a supported range.

## Adding a Claude model

1. Add an entry to `providers.claudeAgent.models`. Reuse a profile when the capabilities match; add a profile for a new combination.
2. Set `adapter.claudeCode.minVersion` if the model needs a newer Claude Code.
3. If the model is current, add its slug to `currentModels.claudeAgent` too, so the list stays in step with the catalog.
4. Bump `updatedAt`.
5. Check that the file is valid JSON before pushing.

## License

MIT, see `LICENSE`.
