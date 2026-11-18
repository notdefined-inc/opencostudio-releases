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
| Secret | `SOURCE_REPO_SSH_KEY` | Read-only deploy key for the private source (deploy key "opencostudio-releases CI (read-only)" on that repository; replaced the expired `SOURCE_REPO_TOKEN` on 2026-11-17) |
| Secret | `OPENCOSTUDIO_RELEASE_PRIVATE_KEY` | Machine manifest signer, Ed25519 PKCS#8 PEM; signing step only |
| Secret | `OPENCOSTUDIO_RELEASE_KEY_ID` | Embedded public-root ID (`release-1`) |

macOS builds are ad-hoc signed but not notarized, so macOS shows an unidentified-developer prompt on
first launch (the README explains how to open it). Notarization needs an Apple Developer ID.

After a release, update the download links in `README.md` — asset names carry the version.

## Machine inputs and native Linux candidates

**Machine Release** (`machine.yml`) is manual and runs before desktop packaging.
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

### Second pass: opt-in signed machine / native Linux releases

Commit/push the reviewed catalogue in the source repository, with `apps/server/package.json`
matching the intended version. Dispatch against that exact new source SHA and the first run ID:

```sh
gh workflow run machine.yml --repo notdefined-inc/opencostudio-releases --ref main \
  -f version=0.6.2-rc.1 -f ref=<full-reviewed-source-sha> \
  -f publish_machine=true -f deepseek_run_id="$DEEPSEEK_RUN_ID"
```

`deepseek` is skipped. Both `machine` jobs retrieve the reviewed archives/records, refuse
catalogue/lock/tag/version mismatches and build the native payload with frozen inputs. Native
arm64 then runs `machine/kernel/build.sh arm64` and `scripts/build-machine-image.ts` from the
payload that job just built; x64 remains payload-only. Each target signs exactly once using
the secrets **only in the signing step**, and run native `release verify` with compiled roots
during signing and independently before collecting assets. The verifier is built from source in
the payload build. Only these jobs request `contents: write` (plus `actions: read` for downloads).
All source checkouts disable credential persistence.

`machine_target=all` (default), `linux-arm64`, or `linux-x64` selects publication jobs; the
DeepSeek input pass always builds both architectures. Each selected job attaches
`opencostudio-machine-<version>-linux-arm64-image.tar.gz` (complete machine) or
`opencostudio-machine-<version>-linux-x64.tar.gz` (native payload), its JSON
size/digest/commit report and the exact standalone DeepSeek archive. Inside each release tar,
`<target>/` contains its manifest/signature, payload, notices, signed provenance with component
accounting, `sizes.json` and frozen input/build reports. The arm64 unit also includes `kernel`,
`initrd`, `system.raw.gz`, `data.template` and `kernel-source.txt` (GPL text, full config and
corresponding-source references). No logical `data.raw` is shipped. Target manifests cannot overwrite each
other. DeepSeek stays outside the offline payload and release tar; signed provenance records its
digest. Additional size/input reports are evidence, not separate signed admission inputs.

The tag `v<version>` must be a **draft prerelease**: the workflow creates one if needed and
refuses other existing releases. It never promotes, marks latest or clobbers machine assets. An already attached standalone
DeepSeek archive is reused only after downloading and comparing its exact bytes. A failed
matrix can leave partial draft assets; resolve those explicitly or choose a new reviewed candidate
version before rerunning. Avoid running Desktop Release on that tag during assembly.

The existing **v0.7.0-rc.1** draft already has payload-only arm64/x64 assets (machine run
44142568982). The full image uses a distinct `-linux-arm64-image` filename so those remain
immutable. After the source changes and workflow changes are committed/pushed by the owner,
run from the source checkout:

```sh
SOURCE_SHA="$(git rev-parse HEAD)" # full pushed source SHA containing the DMG change
gh workflow run machine.yml --repo notdefined-inc/opencostudio-releases --ref main \
  -f version=0.7.0-rc.1 -f ref="$SOURCE_SHA" -f publish_machine=true \
  -f deepseek_run_id=44110290201 -f machine_target=linux-arm64
```

This does not rerun x64 or overwrite the prior arm64 payload. The reviewed input run must still
be retained; if expired, follow the source's reviewed-input recovery process with exact retained
bytes. A retry that encounters an existing image/report refuses replacement.

## Mac DMG with the machine inside

The macOS arm64 build in `release.yml` requires the same-version draft prerelease,
downloads the `-linux-arm64-image.tar.gz`, confines extraction to fixed regular files, and builds
its Mac verifier from source. Rust checks compiled trust roots, signature, every artifact, payload
archive and sparse template; the admitted release must match the desktop version and exact source
commit. The downloaded payload supplies no host verifier or runtime. After success, the job sets
`OPENCOSTUDIO_MACHINE_RELEASE_DIR`; packaging verifies source/staged copies again. The job runs
the existing pinned patched-vfkit build and retains its digest gate. Python's standard-library
tar reader and existing Node/Rust/tar tooling add no package or lockfile change. The arm64 job has
90 minutes; other platform build paths/timeouts are unchanged.

After the machine pass succeeds, using the **same SOURCE_SHA**:

```sh
gh workflow run release.yml --repo notdefined-inc/opencostudio-releases --ref main \
  -f version=0.7.0-rc.1 -f ref="$SOURCE_SHA" -f mac_arm64_draft=true
```

This runs lint, typecheck and tests on Apple Silicon after building the Rust helper. The build
attaches the installer and size report to the existing draft prerelease. Release publication,
CLI publication and updater manifests are disabled in this mode. With the default input, the
workflow uses Linux preflight and the full platform matrix. Test the installed DMG on a fresh
Apple Silicon profile before promoting the draft.

The app retains existing ad-hoc signing and is **not notarized**: no Developer ID exists. Expect
macOS's unidentified-developer prompt; use System Settings → Privacy & Security → Open Anyway.
No new Apple certificate/notarization requirement was added.

The final Mac step records actual **DMG bytes and SHA-256** in its job summary and attaches
`<installer>.dmg.size.json`. The same report records source/release identity and first-launch
app-data baseline. No new DMG was built locally, so its size is **unmeasured**. Component budgets
are payload ≤750 MiB compressed, kernel + compressed system ≤1 GiB and expanded system ≤3 GiB;
template/notices/desktop shell are additional bytes, so these are not a DMG size estimate.

First launch keeps a private copy of all signed machine inputs (M), expands one system disk (S),
and restores allocated XFS extents (D): expect approximately **M + S + D** of app data, plus
profile/logs/work/caches. The report gives M + S. The installed app and downloaded DMG are
additional space; updates retain previous releases. Data is **2 TiB logical, sparse**, with only
allocated blocks consuming disk. Keep **2 GiB host free** beyond installation/activation needs.
Measure allocated use with `du -sk`, not the raw disk's apparent length.

On first launch Electron finds unpacked resources, bundled vfkit/workers and its own Electron
runtime. Both crypto workers use Electron's Node mode. Working directories live owner-only in
app data (runtime sockets may use owner-only `/private/tmp/ocs-*`); system/data never enter the
app bundle. Packaged builds ignore the development image override; existing data survives
relaunch. The existing launch shell shows starting/connecting and plain recovery actions.

Check the downloaded draft DMG on a fresh Apple Silicon profile: drag to Applications, open via
Gatekeeper, launch without terminal/settings, reach ready + connected, use pinned Codex/Claude,
create work, quit and reopen. Inspect unpacked signed image files/workers/vfkit, nested signature
and virtualization entitlement, owner-only app-data modes, sparse allocated data/system use and
unchanged bundle contents. Record DMG bytes/hash and before/after `du -sk`. Exercise low-space
recovery/Try again only with a disposable profile.

Spec 01 tasks 2/4 stay unchecked until actual CI artifacts, clean rebuild/boot, notices and exact
corresponding-source audit, installer/disk measurements and promotion acceptance are observed.
No Docker/vfkit, production signing, upload, push or dispatch ran during this implementation.
The source guide and spec Results record narrow tests and external acceptance limits.
