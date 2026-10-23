# V_share Release Guide

This guide explains how to validate, build, test, and publish **V_share Desktop for Windows** and the **V_share Android APK** from this pnpm monorepo.

## 1. Release outputs

| Artifact | Build method | Intended use |
|---|---|---|
| `V_share-0.1.0-x64.exe` | Electron Builder `nsis` target | Windows installer for end users |
| `V_share-0.1.0-x64-portable.exe` | Electron Builder `portable` target | Windows executable that runs without installation |
| Android `.apk` | EAS Build `preview` profile | Install directly on Android phones for testing or direct distribution |
| Android `.aab` | EAS Build `production` profile | Upload to Google Play Console; it is not directly installable like an APK |

The version in the filenames changes when the desktop package version changes. Exact filenames may differ if the version is updated.

## 2. Prerequisites

- Windows 10/11 64-bit for producing and testing the Windows installer.
- Node.js 22.13 or newer and pnpm 10.28.0, as specified by the repository.
- Git, network access, and an Expo account for cloud Android builds.
- An Android phone for final install/transfer testing. Android Platform Tools (`adb`) are optional if installing over USB.
- A Google Play Console app and EAS Android signing credentials only if publishing on Google Play.

Run commands in PowerShell from the repository root unless a section explicitly says to change directory.

## 3. Pre-release checks

Start from the release branch and make sure there are no uncommitted changes you do not intend to release:

```powershell
git status --short
git branch --show-current
pnpm install --frozen-lockfile
pnpm test
pnpm typecheck
pnpm --filter @vshare/mobile export:android
```

Fix every failing test, typecheck, or Android bundle error before distributing the build. Then test the application manually on a real Windows PC and Android phone on the same Wi-Fi network.

### Manual release test checklist

- [ ] App starts and closes cleanly; desktop tray behavior is correct.
- [ ] Pair using the QR code and using a manual IP address/port.
- [ ] Check that both devices show the same verification code and the receiver must approve a transfer.
- [ ] Send a small file Windows → Android and Android → Windows; verify contents and size.
- [ ] Test a larger file and a batch of files; watch for stalls or crashes.
- [ ] Pause, resume, cancel, decline, and allow an offer to time out.
- [ ] Confirm cancellation does not leave a partial file.
- [ ] Toggle receiving off/on during an active transfer and verify existing transfers behave as intended.
- [ ] Try changing the listening port to an unused port, then to a port already in use; confirm the UI reports failure accurately.
- [ ] Confirm selected files/folders, history, download locations, and light/dark/system themes.
- [ ] Test the Windows installer on a normal user account and confirm uninstall works.
- [ ] Test the Android APK on a device without Metro/Expo development server running.

## 4. Build the Windows desktop release

The repository already defines Electron Builder targets for an NSIS installer and portable executable.

From the repository root:

```powershell
pnpm build:protocol
pnpm build:desktop
pnpm build:win
```

`pnpm build:protocol` is listed explicitly so the shared workspace protocol is compiled before Electron packages it.

### Expected output

Look in:

```text
apps/desktop/dist/
```

Expected release files include:

- `V_share-0.1.0-x64.exe` — installer.
- `V_share-0.1.0-x64-portable.exe` — portable build.
- `win-unpacked/` — unpacked application directory, useful for testing but not the normal download for end users.

Confirm that the `pnpm build:win` command finishes and returns to the prompt without an error. A successful `electron-vite build` alone does not prove that the installer and portable package finished successfully.

### Test the desktop output

Run the packaged executable from the unpacked output:

```powershell
.\apps\desktop\dist\win-unpacked\V_share.exe
```

Then install the installer on a test machine/account:

```powershell
.\apps\desktop\dist\V_share-0.1.0-x64.exe
```

Follow the installer prompts and test sending/receiving. Windows Firewall should permit V_share on **Private networks** if Windows asks. Do not recommend allowing it on public networks by default.

### Windows signing note

The current Electron Builder configuration sets `signAndEditExecutable: false`. Unless the release is separately signed with a publisher code-signing certificate, Windows may show SmartScreen or unknown-publisher warnings. Do not put a certificate or its password in the Git repository. Signing is recommended for a public release but is separate from building the app.

## 5. Configure Android EAS Build

The mobile app lives in `apps/mobile`. Run EAS commands from that directory, as recommended for an app inside a monorepo.

Install EAS CLI if needed and sign in:

```powershell
npm install --global eas-cli
cd apps/mobile
eas login
eas whoami
eas project:info
```

### Link the Expo project

The Android configuration currently uses package ID `com.vshare.app`, app slug `vshare`, and app version `0.1.0` in `app.config.ts`. Keep the package ID stable after publishing; changing it makes Android treat the result as a different application.

Before the first EAS build, verify that `app.config.ts` has the correct `extra.eas.projectId` for the V_share project. The checked-in configuration should contain this field because it uses a dynamic `app.config.ts` rather than a static `app.json`. If this is the EAS project previously created for V_share, the project ID recorded for it is `60e7e6a7-f73d-44d5-a715-5fe60ad13922`. Add/merge the following property into the returned Expo config object (do not overwrite other `extra` fields):

```typescript
extra: {
  ...(config.extra ?? {}),
  eas: {
    projectId: "60e7e6a7-f73d-44d5-a715-5fe60ad13922",
  },
},
```

If you created a different EAS project, use that project's ID instead. Confirm the result using `eas project:info` before building.

## 6. Build an installable Android APK

The repository's existing profiles mean:

- `development`: internal APK that includes the development client and needs a development server for its normal developer workflow. Do not use this as the ordinary public release APK.
- `preview`: internal, directly installable APK suitable for device testing and direct distribution.
- `production`: Android App Bundle (`.aab`) for Google Play, because `android.buildType` is `app-bundle`.

From `apps/mobile`:

```powershell
eas build --platform android --profile preview
```

At the first Android build, follow EAS prompts to create or select Android signing credentials. Keep the signing key under controlled access; losing it may prevent publishing updates signed with the same key.

EAS builds in the cloud. Follow the build URL printed in the terminal or open the build in the Expo dashboard. When the build completes, download the `.apk` artifact and save a release copy outside `node_modules` and build caches, for example:

```text
release-assets/V_share-0.1.0-android.apk
```

The exact EAS-downloaded filename may be different; rename the downloaded copy for clarity only after the build has completed. Keep the original artifact available for traceability.

### Install and test the APK

**Option A — install on the phone:** transfer the APK to the phone or open its EAS download link on the phone. Approve Android's prompt to allow installation from that source, then install and launch V_share.

**Option B — install with USB and ADB:** enable USB debugging, connect the phone, and run this from the directory containing the downloaded APK:

```powershell
adb devices
adb install -r .\V_share-0.1.0-android.apk
```

If Android reports a signature conflict when updating an existing install, verify that the APK was signed with the same EAS keystore. Uninstalling an older differently signed build may remove that app's local data, so back up anything important first.

**Important:** the production profile currently produces an `.aab`, not an `.apk`. For a directly installable APK, use `--profile preview`. Expo's docs explain the distinction between APK and AAB: https://docs.expo.dev/build-reference/apk/.

## 7. Optional: prepare a Google Play release

For Google Play, build the production app bundle:

```powershell
cd apps/mobile
eas build --platform android --profile production
```

Download the resulting `.aab` and upload it to Google Play Console (start with internal testing). A Play Store submission also needs the Play Console app set up and EAS submission credentials configured; those steps are separate from producing the bundle. New Play Store releases should use the `.aab` workflow rather than distributing the preview APK as the store upload.

Reference: https://docs.expo.dev/submit/android/.

## 8. Prepare the GitHub release

Do not commit generated installers or APKs into source folders. Create a GitHub Release in the repository and attach the release binaries as assets.

1. Finish testing both applications and verify the target version.
2. Create a version tag, for example `v0.1.0`, on the exact commit being released.
3. Open the repository on GitHub → **Releases** → **Draft a new release**.
4. Choose the tag and add a title such as `V_share v0.1.0`.
5. Attach only the finished installer, portable EXE, and tested standalone APK (if distributing one):
   - `apps/desktop/dist/V_share-0.1.0-x64.exe`
   - `apps/desktop/dist/V_share-0.1.0-x64-portable.exe`
   - `release-assets/V_share-0.1.0-android.apk`
6. Add release notes, verify the assets and tag, then publish the release.

Do not upload `win-unpacked/`, source maps, development-client APKs, or temporary build output as normal end-user downloads.

### Generate SHA-256 checksums

From the repository root, use PowerShell to generate checksums for the files you plan to publish:

```powershell
Get-FileHash .\apps\desktop\dist\V_share-0.1.0-x64.exe -Algorithm SHA256
Get-FileHash .\apps\desktop\dist\V_share-0.1.0-x64-portable.exe -Algorithm SHA256
Get-FileHash .\release-assets\V_share-0.1.0-android.apk -Algorithm SHA256
```

Publish the resulting values in the release notes or as a `SHA256SUMS.txt` release asset. Update paths and version numbers to match the actual artifacts.

## 9. Versioning for later releases

Before creating the next release:

1. Update the desktop version in `apps/desktop/package.json`.
2. Update `VERSION` in `apps/mobile/app.config.ts` so the user-facing Android version matches the release.
3. Keep the Android package ID `com.vshare.app` unchanged for updates to the same installed app.
4. `eas.json` uses `appVersionSource: "remote"` and the production profile has `autoIncrement: true`; EAS manages the Android build number for production builds. Check the version/build number shown in EAS before publishing. Preview builds do not use the same production auto-increment setting.
5. Create a corresponding Git tag (for example, `v0.1.1`) and GitHub Release.

Do not assume a build succeeded just because the JavaScript bundle compiled. Verify final artifact files, test them, and retain the release assets associated with each tag.

## 10. Troubleshooting

| Problem | Check |
|---|---|
| EAS says the project is not configured or linked | Verify `extra.eas.projectId`, then run `eas project:info` from `apps/mobile`. |
| A build gives an AAB but you need an APK | Run `eas build --platform android --profile preview` from `apps/mobile`. |
| APK installs but the app cannot connect to the PC | Keep both devices on the same Wi-Fi/LAN, confirm receiving is enabled, use the current PC IP/port, and allow Windows Firewall on Private networks. |
| Android cannot update the installed build | Confirm the package ID and signing key are unchanged; a different signing key cannot update the existing installation. |
| Windows warns about the publisher | The executable is unsigned unless you configure code signing with a trusted certificate. |
| `pnpm build:win` ends with an error | Read the final error lines and verify the two expected EXE files exist before publishing. |

## Official references

- EAS Build — APK configuration: https://docs.expo.dev/build-reference/apk/
- EAS internal distribution: https://docs.expo.dev/build/internal-distribution/
- Expo monorepos: https://docs.expo.dev/build-reference/build-with-monorepos/
- EAS Android submission: https://docs.expo.dev/submit/android/
- Electron Builder: https://www.electron.build/docs/
