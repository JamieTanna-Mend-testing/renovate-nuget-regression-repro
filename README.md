# Renovate NuGet regression repro

Minimal reproduction for a regression where **NuGet dependencies pinned to a bare
version stop receiving updates**. Renovate resolves the dependency's
`currentVersion` to the *latest* available release instead of the pinned
`currentValue`, so it reports `0 flattened updates found` and never opens a PR.

- **Last known good:** `43.207.4`
- **Regressed:** `43.208.2` and later (confirmed on `43.211.0`)
- **Suspected cause:** [renovatebot/renovate#43737](https://github.com/renovatebot/renovate/pull/43737)
  added a `versioningApi.isSingleVersion(compareValue)` guard in
  `lib/workers/repository/process/lookup/index.ts`. For the `nuget` versioning,
  `isSingleVersion()` only returns `true` for bracketed exact ranges (`[1.0.0]`);
  a bare version like `13.0.1` is treated as a minimum (`>= 13.0.1`), so the guard
  is `false`, `currentVersion` falls through to the latest-satisfying version, and
  no update is produced.

## What's in here

| File | Purpose |
|------|---------|
| `Example.csproj` | One `PackageReference` pinned to `Newtonsoft.Json` `13.0.1` (older than the latest stable on nuget.org). Uses the **default `nuget` manager** and the **public nuget.org feed** — no custom managers, no private registry. |
| `renovate.json` | Minimal config (`config:recommended`, onboarding off). |
| `.github/workflows/renovate-repro.yml` | Runs Renovate in **dry-run** against this repo with `43.207.4` and `43.211.0` side-by-side (matrix). |

## How to run

1. Create a new **public** GitHub repo and push these files to the default branch.
2. Go to **Actions → "Renovate NuGet regression repro" → Run workflow**.
   (Default `GITHUB_TOKEN` is enough — dry-run only reads the repo; the version
   lookup hits nuget.org and needs no token.)
3. Open the run and compare the two matrix jobs' logs.

## What to look for

Search each job's log for `Newtonsoft.Json` and `flattened update`.

**`renovate 43.207.4` (working) — expect an update is found:**

```
DEBUG: Dependency Newtonsoft.Json has currentVersion 13.0.1
DEBUG: 1 flattened update(s) found: Newtonsoft.Json
INFO:  DRY-RUN: Would create PR: Update dependency Newtonsoft.Json to 13.0.3
```

**`renovate 43.211.0` (regressed) — expect NO update:**

```
DEBUG: 0 flattened updates found:
```

and in the package-file dump the dependency shows the pinned value as
`currentValue` but the *latest* release as `currentVersion`, with an empty
`updates` array:

```json
{
  "depName": "Newtonsoft.Json",
  "currentValue": "13.0.1",
  "datasource": "nuget",
  "versioning": "nuget",
  "updates": [],
  "currentVersion": "13.0.3"
}
```

(Exact version numbers depend on the latest Newtonsoft.Json on nuget.org at run
time; the point is `currentVersion` ≠ the pinned `13.0.1` and `updates: []`.)

## Optional: prove it with real PRs instead of dry-run

If you'd rather see an actual PR appear on the good version and none on the bad
one, in `.github/workflows/renovate-repro.yml`:

- change `permissions:` to `contents: write` and `pull-requests: write`, and
- remove the `RENOVATE_DRY_RUN: "full"` line.

The `43.207.4` job will open a `Update dependency Newtonsoft.Json` PR; the
`43.211.0` job will open none. (Run the two versions in separate dispatches if
you do this, so they don't race on the same branch.)
