# OpenCoStudio — releases

Installers for [OpenCoStudio](https://github.com/notdefined-inc/opencostudio-work). Downloads live
here so that getting a build never requires access to the source.

## Why the builds run here

Actions minutes are free and unmetered on a public repository, and metered on a private one — where
macOS runners bill at ten times the wall-clock minute. A release is four platform builds, two of
them macOS, so running them against a private repository's quota is what exhausted it. Nothing is
built in the source repository any more.

## Cutting a release

Releases are manual, never automatic. Run the **Release Desktop** workflow from the Actions tab and
give it a version (`0.2.0`) and the source ref to build (`main`).

Configuration lives here, not in the source repository:

| Kind | Name | Purpose |
|---|---|---|
| Variable | `SOURCE_REPO` | `notdefined-inc/opencostudio-work` |
| Secret | `SOURCE_REPO_TOKEN` | Read access to the private source |

macOS builds are ad-hoc signed but not notarized, so macOS shows an
unidentified-developer prompt on first launch. Notarization needs an Apple Developer ID.
