# Releasing OpenCoStudio

Installers for OpenCoStudio are built and published here, so getting a build never requires access
to the source (which is private).

## Why the builds run here

Actions minutes are free and unmetered on a public repository, and metered on a private one — where
macOS runners bill at ten times the wall-clock minute. A release is four platform builds, two of
them macOS, so running them against a private repository's quota is what exhausted it. Nothing is
built in the source repository any more.

## Cutting a release

Releases are manual, never automatic. Run the **Release Desktop** workflow from the Actions tab and
give it a version (`0.3.0`) and the source ref to build — a branch name, or a **full 40-character
commit SHA** (the checkout step cannot resolve a short one).

Configuration lives here, not in the source repository:

| Kind | Name | Purpose |
|---|---|---|
| Variable | `SOURCE_REPO` | The private source repository |
| Secret | `SOURCE_REPO_TOKEN` | Read access to the private source |
| Secret | `OPENCOSTUDIO_RELEASE_PRIVATE_KEY` | Machine manifest signer, Ed25519 PKCS#8 PEM; signing step only |
| Secret | `OPENCOSTUDIO_RELEASE_KEY_ID` | Embedded public-root ID (`release-1`) |

macOS builds are ad-hoc signed but not notarized, so macOS shows an unidentified-developer prompt on
first launch (the README explains how to open it). Notarization needs an Apple Developer ID.

After a release, update the download links in `README.md` — asset names carry the version.

## Machine inputs and native Linux candidates

**Machine Release** (`machine.yml`) is manual and separate from `release.yml`, which is unchanged.
The source ref is resolved once to a commit. Native x64 runs on `ubuntu-24.04`; native arm64 runs
on `ubuntu-24.04-arm`, with host and Node architecture checks. Actions use the same version pins
as the desktop workflow. No workflow commits or pushes source.

### First pass: build DeepSeek inputs only

Push reviewed source to `notdefined-inc/opencostudio-work` and this workflow to this repository's
default branch, then run:

```sh
gh workflow run machine.yml --repo notdefined-inc/opencostudio-releases --ref main \
  -f version=0.6.2-rc.1 -f ref=machine/linux-vm -f publish_machine=false
gh run list --repo notdefined-inc/opencostudio-releases --workflow machine.yml --limit 5
```

`version` is the OpenCoStudio publication tag (optional leading `v`), **not** DeepSeek's version;
`ref` selects the private source branch or full SHA. `--ref main` selects this repository's
workflow. The preflight/deepseek jobs are read-only. They build `@deepseek-ai/dsh@0.2.2-rc.3`
with the frozen lock in the exact digest-pinned Node builder from each target's source inputs.
Nothing is uploaded to a release and no signing secrets enter these jobs.

Retain both `deepseek-linux-arm64` and `deepseek-linux-x64` workflow artifacts from one successful
run (they expire after 30 days). Each contains one archive and `<target>.json`. From a source
checkout with an empty `machine/payload/out/deepseek`:

```sh
DEEPSEEK_RUN_ID=<successful-run-id>
for target in linux-arm64 linux-x64; do
  gh run download "$DEEPSEEK_RUN_ID" --repo notdefined-inc/opencostudio-releases \
    --name "deepseek-$target" --dir machine/payload/out/deepseek
done
node machine/payload/prepare-deepseek.ts catalogue
```

Before committing the generated source catalogue, check archive SHA-256 and length, frozen-lock
digest, architecture, release URL and extracted `NOTICES.txt`/`AUDIT.json` hashes; review audit
results (high/critical must pass), notices and native/offline execution. Catalogue generation
requires **both** architectures' archives and records. Keep permanent copies outside Actions
storage. Rebuilding an archive may change audit bytes, so publication consumes this exact run.

The new x64 inputs reuse the source's frozen multi-platform builder index digests and share its
architecture-independent dependency graph. Local Docker Hub access failed with `ENOTFOUND`, so
x64 registry pull/platform execution remains unverified until CI; Docker checks the digest on
pull and Node asserts its native architecture before building. No new dependency is resolved.

### Second pass: opt-in signed native Linux payloads

Commit/push the reviewed catalogue in the source repository, with `apps/server/package.json`
matching the intended version. Dispatch against that exact new source SHA and the first run ID:

```sh
gh workflow run machine.yml --repo notdefined-inc/opencostudio-releases --ref main \
  -f version=0.6.2-rc.1 -f ref=<full-reviewed-source-sha> \
  -f publish_machine=true -f deepseek_run_id="$DEEPSEEK_RUN_ID"
```

`deepseek` is skipped. Both `machine` jobs retrieve the reviewed archives/records, refuse
catalogue/lock/tag/version mismatches, build the native payload with frozen inputs, sign using
the secrets **only in the signing step**, and run native `release verify` with compiled roots
during signing and independently before collecting assets. The verifier is built from source in
the payload build. Only these jobs request `contents: write` (plus `actions: read` for downloads).
All source checkouts disable credential persistence.

Each job attaches a target-specific `opencostudio-machine-<version>-<target>.tar.gz`, its JSON
size/digest/commit report and the exact standalone DeepSeek archive. Inside each release tar,
`<target>/` contains its manifest/signature, payload, notices, signed provenance with component
accounting, `sizes.json` and frozen input/build reports. Target manifests cannot overwrite each
other. DeepSeek stays outside the offline payload and release tar; signed provenance records its
digest. Additional size/input reports are evidence, not separate signed admission inputs.

The tag `v<version>` must be a **draft prerelease**: the workflow creates one if needed and
refuses other existing releases. It never promotes, marks latest or clobbers assets. A failed
matrix can leave partial draft assets; resolve those explicitly or choose a new reviewed candidate
version before rerunning. Avoid running Desktop Release on that tag during assembly.

This is **partial spec 01 task 2**: native shared payload assembly, separate signed manifests,
component reports and generated notices are wired. The arm64 kernel/system/data-template image,
kernel corresponding source/config retention, signed/notarized Mac packaging, final DMG/ZIP
measurement, notice/source audit, clean rebuild and boot/promotion acceptance are not completed.
The workflow has not been dispatched or production-signed locally. Source `docs/RELEASING.md`
contains the full acceptance checklist; spec 01 tasks 2/4 remain unchecked.
