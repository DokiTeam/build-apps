# Getting the APK ready for use

This guide explains how to obtain ready-to-install APKs for the Doki apps from this repository. You can either let the included GitHub Actions workflows produce APK artifacts automatically, trigger a workflow manually from the GitHub UI, or build locally on your machine. The instructions below explain what to expect, how to trigger builds, how to download and verify artifacts, and how to install the APK on an Android device.

> [!NOTE]
> This project includes GitHub Actions workflows that build APKs for convenience. They are intended for users who need signed or unsigned builds without setting up an Android toolchain locally.

## Table of contents

- [Quick Tutorial](#quick-tutorial)
- [Quick status](#quick-status)
- [What the workflows do (brief)](#what-the-workflows-do-brief)
- [Workflow files explained](#workflow-files-explained)
- [How to trigger builds on GitHub](#how-to-trigger-builds-on-github)
- [Where to find and download APK artifacts](#where-to-find-and-download-apk-artifacts)
- [Signing and repository secrets](#signing-and-repository-secrets)
- [Downloading and installing an APK locally](#downloading-and-installing-an-apk-locally)
- [Building locally (if you prefer to build yourself)](#building-locally-if-you-prefer-to-build-yourself)
- [Troubleshooting common issues](#troubleshooting-common-issues)
- [Recommended workflow for maintainers](#recommended-workflow-for-maintainers)
- [Contributing](#contributing)
- [License](#license)

## Quick Tutorial

Follow these quick steps if you just want a working APK with minimal fuss:

1. Create a GitHub account (if you don't have one) and fork this repository to your account.
2. Open your fork on GitHub, go to the **Actions** tab and enable Actions for the repository if GitHub prompts you to (this is required to run workflows on forks).
3. In the **Actions** tab choose `build_release.yml` (or `build_legacy.yml`) and click **Run workflow** to trigger a manual build. Wait for the run to complete.
4. Open the workflow run, download the artifact (ZIP) from the **Artifacts** section, extract it, and install the APK on your device using `adb install -r <path-to-apk>`.

> [!NOTE]
> - If you need a signed release APK from CI, configure signing secrets in your fork (see the "Signing and repository secrets" section below).
> - If you prefer not to use Actions, build locally using the Gradle wrapper: `./gradlew assembleRelease` and sign with `apksigner` as needed.
> - If you want both release and legacy builds, run both `build_release.yml` and `build_legacy.yml` (there's a `trigger_builds.yml` helper that can run both).

> [!TIP]
> For most users the fastest path is: fork → enable Actions → run `build_release.yml` → download artifact → install via `adb`.

## Quick status

- Workflows included: `.github/workflows/build_legacy.yml`, `.github/workflows/build_release.yml`, and a workflow to run checks.
- Artifacts: the Actions workflows upload APK artifacts (debug/release) to the Actions run; you can download them from the GitHub Actions web UI.

## What the workflows do (brief)

- Build the Android app using Gradle on a GitHub-hosted runner.
- Produce APK(s) and upload them as workflow artifacts.
 - Optionally sign the release APK if signing secrets are configured in the repository (see "Signing and secrets" below).

> [!TIP]
> Use the Actions artifacts when you want quick, repeatable builds without installing the Android SDK locally. They're great for testing and distribution to trusted users.

## Workflow files explained

This repository contains four GitHub Actions workflows under `.github/workflows/`. Below are short, practical explanations of what each file does and important details you should know when using or modifying them.

- `build_legacy.yml` — Build Legacy Artifact

	- Purpose: Builds the "legacy" variant of the app from the `legacy` branch of `DokiTeam/Doki` and uploads a `legacy-apk` artifact. It then creates a GitHub Release with the generated APK.
	- Key points:
		- Uses `actions/checkout@v4` with `repository: DokiTeam/Doki` and `ref: legacy` to pull upstream source before building.
		- Sets `APK_PATH` to `app/build/outputs/apk/legacy/app-legacy.apk` and performs up to three build attempts (`./gradlew assembleLegacy`). The first two attempts run with `continue-on-error: true` and the third must succeed (or the job fails).
		- The Gradle invocation passes `-Dtg_backup_bot_token=$(curl -s https://dragonx943.github.io/waydroid_x86_64/value.json)` and `-Dgithub_updates_repo=${{ github.repository_owner }}` as system properties (the curl call reads a value from the network at runtime).
		- On success the job uploads the APK artifact named `legacy-apk` with `retention-days: 1` and then the `release` job moves the APK to `legacy-release.apk` and creates a GitHub Release using `softprops/action-gh-release@v2`.
	- Caveats & recommendations:
		- Artifact retention is short (1 day) — download artifacts promptly.
		- The build pulls from `DokiTeam/Doki`. If you are running the workflow in a fork or without appropriate token/perms, the checkout or release steps may behave differently.

- `build_release.yml` — Build Release Artifact

	- Purpose: Builds the `release` variant from the `base` branch of `DokiTeam/Doki`, uploads a `release-apk` artifact, and creates a GitHub Release containing the APK.
	- Key points:
		- Uses `actions/checkout@v4` with `repository: DokiTeam/Doki` and `ref: base`.
		- `APK_PATH` is `app/build/outputs/apk/release/app-release.apk` and the workflow runs up to three `./gradlew assembleRelease` attempts, similar to the legacy build.
		- After building it uploads `release-apk` (retention 1 day) and the `release` job renames to `release.apk` and creates a GitHub Release.
	- Caveats & recommendations:
		- Like `build_legacy.yml`, this workflow uses a curl call to fetch a runtime property and assumes the runner has network access.
		- If you want signed release APKs from CI, ensure signing secrets are configured in repository settings (see "Signing and repository secrets" above).

- `check.yml` — Check for Updates (Daily)

	- Purpose: Runs on a daily schedule (and via manual dispatch) to check whether upstream repositories have new commits in the last 24 hours. If changes exist it triggers the build workflows automatically.
	- Key points:
		- Uses the GitHub REST API to query commit counts on `DokiTeam/Doki` and `DokiTeam/doki-exts` since 24 hours ago. It uses `jq` to parse JSON and sets an output `has_changes=true/false`.
		- If changes are detected the `trigger_builds` job runs `gh workflow run build_release.yml` and `gh workflow run build_legacy.yml` to dispatch the build workflows.
	- Caveats & recommendations:
		- The job relies on `jq` and the `gh` CLI being available on the runner (current runners typically include common utilities, but if you modify the job, ensure the needed tools are present).
		- The check uses GitHub API rate limits; for high-frequency checks adapt the schedule or use a token with appropriate rate limits.

- `trigger_builds.yml` — Trigger Doki builds

	- Purpose: Simple manual dispatch workflow that triggers both `build_release.yml` and `build_legacy.yml` using the `gh` CLI. It's useful when you want to manually kick off both builds from the Actions UI.
	- Key points:
		- On manual dispatch it checks out the repo and runs `gh workflow run build_release.yml` and `gh workflow run build_legacy.yml` with `GITHUB_TOKEN` in the environment.
	- Caveats & recommendations:
		- The runner must have the `gh` CLI installed and authenticated (the workflow sets `GITHUB_TOKEN` in the environment; the GH runner supports the CLI but behavior in forks or restricted permission contexts may vary).

> [!NOTE]
> 
> **General notes about these workflows**
> 
> - Artifact retention is configured as `retention-days: 1` in the build workflows — artifacts are ephemeral by default; download them quickly or extend retention in the workflow if you maintain the repo.
> - The build workflows pull code from `DokiTeam/Doki` explicitly. If you plan to run builds in your fork, check that the checkout step behaves as you expect and that permissions for `GITHUB_TOKEN` allow necessary operations (creating releases, uploading artifacts).
> - The builds pass a dynamic property to Gradle via a remote `curl` call; this means a network dependency exists at build time. If that resource is unavailable the Gradle invocation may fail.
> - Release creation uses `softprops/action-gh-release@v2` which requires permission to create releases in the repository; ensure `GITHUB_TOKEN` or a PAT has appropriate scopes for release uploads.

> [!IMPORTANT]
> Some workflows may also run automatically on `push` or `pull_request` events. Check the top of the workflow file in `.github/workflows/` to see the configured triggers.

## Where to find and download APK artifacts

1. After a workflow completes, open the specific run under the Actions tab.
2. On the run page, scroll to the "Artifacts" section (usually at the bottom-right or near the job summary).
3. Click the artifact name to download a ZIP that contains the APK(s).

Artifact names and contents vary by workflow, but you can generally expect:

- `app-debug.apk` — unsigned debug build, useful for quick testing.
- `app-release-unsigned.apk` — release build but unsigned (needs signing before Play Store distribution).
- `app-release.apk` (or similar) — release build signed if the repository has signing secrets configured.

> [!CAUTION]
> Do NOT install APKs from untrusted sources. Only install artifacts from this repository or builds you trust.

## Signing and repository secrets

If the workflows are configured to produce a signed release APK, the signing step requires repository secrets (for the keystore and passwords). Typical secrets used by Android signing workflows:

- `KEYSTORE_BASE64` (or `KEYSTORE`): base64-encoded keystore file
- `KEYSTORE_PASSWORD`
- `KEY_ALIAS`
- `KEY_PASSWORD`

If those secrets are missing, the workflow will usually still produce an unsigned release APK (or it will fail the sign step). If you maintain a fork and want signed releases, add the required secrets to your fork's Settings → Secrets and variables → Actions.

> [!TIP]
> If you don't want to store secrets in GitHub, you can produce a release APK locally and sign it using your private keystore (instructions below).

## Downloading and installing an APK locally

After you download and extract the artifact ZIP, install the APK on a device or emulator.

On Windows PowerShell (example):

```powershell
# Install to a connected device (replace path with actual APK path)
adb install -r C:\path\to\app-release.apk

# If you see signature conflicts, uninstall the existing app first (data will be lost):
adb uninstall com.example.app
adb install C:\path\to\app-release.apk
```

On emulators or for testing, `app-debug.apk` is convenient because it doesn't require matching signatures.

## Building locally (if you prefer to build yourself)

Requirements:

- Java JDK (11+ recommended)
- Android SDK / command-line tools
- Gradle (this repo includes a Gradle wrapper so you don't need a global Gradle install)

To build a release APK locally with the Gradle wrapper (PowerShell):

```powershell
cd path\to\repo\(where build.gradle is)
.# Windows: run the wrapper
.\gradlew assembleRelease

# The produced APKs will appear in the module's build/outputs/apk/ directory
```

If you need to sign the APK locally, create or use an existing keystore and then either configure the `signingConfigs` in your `build.gradle` or sign the unsigned APK with `apksigner`:

```powershell
# Sign with apksigner (bundled with Android build tools)
apksigner sign --ks path\to\keystore.jks --ks-key-alias KEY_ALIAS --out app-release-signed.apk app-release-unsigned.apk
```

## Troubleshooting common issues

- Workflow failed with "missing secret": check repository Settings → Secrets and variables → Actions and add the required secrets (only repository administrators can do this).
- APK won't install because of a different signature: uninstall the existing app (`adb uninstall <package>`) or install a debug build.
- Artifact not present in Actions run: check the workflow logs for the upload-artifact step and ensure the job runs to completion.

> [!WARNING]
> Signing and distributing release APKs have security implications. Keep keystores and passwords private. Do not commit keystore files to the repository.

## Recommended workflow for maintainers

1. Push a tag like `v1.2.3` or a release branch (depending on how `build_release.yml` is configured). The release workflow will build and upload artifacts.
2. If you want a one-off build, use the Actions tab and the "Run workflow" button to manually dispatch a workflow.
3. For a production distribution, sign locally with your secure keystore and upload to the Play Console (or use your CI with protected secrets).



## Contributing

Contributions welcome. If you change the build workflows or artifact names, please update this GUIDE.md so users know what to expect.

## License

See the project `LICENSE`.
