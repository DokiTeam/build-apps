# Quick Guide — simplest steps

Recommended (fastest): fork → enable Actions → Actions tab → run `build_release.yml` → Releases → download APK → install on your phone (no PC needed).

> [!TIP]
> Download the APK from the repository's Releases page on your phone and tap it to install.

> [!IMPORTANT]
> `check.yml` may trigger builds automatically when upstream changes are detected — check Releases or Actions artifacts for new APKs.

> [!CAUTION]
> You may need to allow "Install unknown apps" for your browser or file manager in your device settings before installing.

If you prefer a local build (developers):

```powershell
.\gradlew assembleRelease
```

> [!WARNING]
> Signed CI builds require secrets: `KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`.
> Maintainers: add these in your fork's Settings → Secrets and variables → Actions.

Questions? Ask for one small expansion (enable Actions steps, screenshots, or exact artifact names).

