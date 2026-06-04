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
| `Example.csproj` | **Bug case** — `Newtonsoft.Json` pinned to a **bare** version `13.0.1`. |
| `Control.csproj` | **Control case** — same package/target, but a **bracketed exact range** `[13.0.1]`. Must update on *both* versions, isolating the bug to bare versions. |
| `renovate.json` | Minimal config (`config:recommended`, onboarding off). |
| `.github/workflows/renovate-repro.yml` | Runs Renovate in **dry-run** against this repo with `43.207.4` and `43.211.0` side-by-side (matrix). |

Both project files use the **default `nuget` manager** and the **public nuget.org
feed** — no custom managers, no private registry.

## How to run

1. Create a new **public** GitHub repo and push these files to the default branch.
2. Create a **Personal Access Token** and add it as an Actions secret named
   `RENOVATE_TOKEN` (**Settings → Secrets and variables → Actions → New repository
   secret**). The default `GITHUB_TOKEN` does **not** work — Renovate's repo-init
   GraphQL query fails with `FORBIDDEN: Resource not accessible by integration`.
   - **Classic PAT:** scope `repo` (or just `public_repo` for a public repo).
   - **Fine-grained PAT:** grant this repo **Contents: Read** and
     **Metadata: Read** (add Issues/Pull requests: Read if you switch to live PRs).
3. Go to **Actions → "Renovate NuGet regression repro" → Run workflow**.
4. Open the run and compare the two matrix jobs' logs.

## What to look for

Each version job evaluates **both** project files, so you get a 2×2 result.
Expected outcome:

| Renovate version | `Example.csproj` (bare `13.0.1`) | `Control.csproj` (bracketed `[13.0.1]`) |
|------------------|----------------------------------|------------------------------------------|
| `43.207.4` (good)     | ✅ update found | ✅ update found |
| `43.211.0` (regressed)| ❌ **no update** | ✅ update found |

The decisive cell is the bottom-left: the bracketed form still updates on
`43.211.0`, but the bare form does not — so the regression is specific to bare
NuGet versions, not to the package or the feed.

In the **`43.211.0`** job, the bare dependency shows the pinned value as
`currentValue` but the *latest* release as `currentVersion`, with an empty
`updates` array:

```json
{
  "depName": "Newtonsoft.Json",
  "currentValue": "13.0.1",
  "packageFile": "Example.csproj",
  "datasource": "nuget",
  "versioning": "nuget",
  "updates": [],
  "currentVersion": "13.0.3"
}
```

while the bracketed control still resolves correctly and yields an update:

```json
{
  "depName": "Newtonsoft.Json",
  "currentValue": "[13.0.1]",
  "packageFile": "Control.csproj",
  "updates": [{ "newValue": "[13.0.3]", "newVersion": "13.0.3" }]
}
```

The `43.207.4` job instead logs `flattened update(s) found` for **both** files
(and, with dry-run, `DRY-RUN: Would create PR: Update dependency Newtonsoft.Json`).

(Exact version numbers depend on the latest Newtonsoft.Json on nuget.org at run
time; the point is the bare case has `currentVersion` ≠ pinned `13.0.1` with
`updates: []`, while the bracketed case still updates.)

## Optional: prove it with real PRs instead of dry-run

If you'd rather see an actual PR appear on the good version and none on the bad
one:

- remove the `RENOVATE_DRY_RUN: "full"` line from
  `.github/workflows/renovate-repro.yml`, and
- make sure `RENOVATE_TOKEN` can write: a **classic** PAT with `repo` already
  can; a **fine-grained** PAT additionally needs **Contents: Write** and
  **Pull requests: Write**.

The `43.207.4` job will open a `Update dependency Newtonsoft.Json` PR; the
`43.211.0` job will open none. (Run the two versions in separate dispatches if
you do this, so they don't race on the same branch.)
