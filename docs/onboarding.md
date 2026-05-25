# Onboarding a new Apple project

How to wire a fresh macOS (or iOS) project under the `marcelrgberger` account into `ci-open-source`.

## Prerequisites

- The match store has already been bootstrapped — if not, follow [match-bootstrap.md](match-bootstrap.md) first.
- Your project compiles locally via `xcodebuild build -scheme <Scheme>`.
- Bundle IDs for every signed target are registered in App Store Connect.

## Step 1 — register bundle IDs with match

From your developer Mac, run match once with the new bundle IDs added:

```bash
export MATCH_PASSWORD='<your passphrase>'

fastlane match developer_id \
  --git_url "git@github.com:marcelrgberger/apple-certificates.git" \
  --app_identifier "com.example.MyApp,com.example.MyApp.WidgetExtension" \
  --team_id <YOUR_TEAM_ID> \
  --api_key_path ~/Developer/aso/keys/AuthKey_<KEY_ID>.p8 \
  --api_key_id <KEY_ID> \
  --api_issuer_id <ISSUER_ID>
```

Match is incremental — it only creates what's missing, so this is safe to re-run.

## Step 2 — set repository secrets

Add the secrets listed in [match-bootstrap.md § Step 3](match-bootstrap.md#step-3--configure-consumer-repositories) to the consumer repo settings.

## Step 3 — minimum project.yml

`ci-open-source` expects the project to be XcodeGen-driven (the default; you can opt out with `run-xcodegen: false`). Minimum settings:

```yaml
options:
  bundleIdPrefix: com.example
  xcodeVersion: "26"

settings:
  base:
    DEVELOPMENT_TEAM: <YOUR_TEAM_ID>
    CODE_SIGN_STYLE: Manual
    CODE_SIGN_IDENTITY: Developer ID Application
    ENABLE_HARDENED_RUNTIME: true
```

Do **not** set `PROVISIONING_PROFILE_SPECIFIER` per target — the workflow injects those via the export step.

## Step 4 — call the reusable workflow

Create `.github/workflows/release.yml` in your project:

```yaml
name: Release

on:
  workflow_dispatch:

jobs:
  build:
    uses: marcelrgberger/ci-open-source/.github/workflows/release-macos-app.yml@v1
    with:
      project: MyApp.xcodeproj
      scheme: MyApp
      app-identifiers: "com.example.MyApp,com.example.MyApp.WidgetExtension"
      team-id: ${{ vars.APPLE_TEAM_ID }}
      match-git-url-template: "https://x:{TOKEN}@github.com/marcelrgberger/apple-certificates.git"
      provisioning-profiles: |
        com.example.MyApp=match Direct com.example.MyApp
        com.example.MyApp.WidgetExtension=match Direct com.example.MyApp.WidgetExtension
    secrets:
      keychain-password:    ${{ secrets.KEYCHAIN_PASSWORD }}
      asc-api-key:          ${{ secrets.ASC_API_KEY }}
      asc-api-key-id:       ${{ secrets.ASC_API_KEY_ID }}
      asc-issuer-id:        ${{ secrets.ASC_ISSUER_ID }}
      match-git-token:      ${{ secrets.MATCH_GIT_TOKEN }}
      match-password:       ${{ secrets.MATCH_PASSWORD }}
```

The job uploads the notarized `.app` as the `macos-app` artifact. Add follow-up jobs that pick the artifact up and do the project-specific packaging (DMG, ZIP, Sparkle appcast, GitHub Release, Homebrew tap bump, …).

## Step 5 — packaging follow-up

For DMG packaging + Sparkle EdDSA signing + GitHub Release + appcast bump, use the à-la-carte composite actions:

```yaml
  release:
    needs: build
    runs-on: macos-26
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact-name }}
          path: build/export

      # ... build .dmg, then:
      - uses: marcelrgberger/ci-open-source/actions/macos-codesign-dmg@v1
        with:
          dmg-path: build/MyApp.dmg

      - uses: marcelrgberger/ci-open-source/actions/macos-notarize@v1
        with:
          bundle-path: build/MyApp.dmg
          api-key-path: ${{ env.ASC_KEY_PATH }}
          api-key-id: ${{ secrets.ASC_API_KEY_ID }}
          api-issuer-id: ${{ secrets.ASC_ISSUER_ID }}

      - uses: marcelrgberger/ci-open-source/actions/sparkle-sign-appcast@v1
        id: sparkle
        with:
          private-key-base64: ${{ secrets.SPARKLE_PRIVATE_KEY }}
          artifact-path: build/MyApp.zip
```

The signature and length end up in `steps.sparkle.outputs.signature` / `.length` — splice them into your `appcast.xml` and push.

## Verifying the setup

A dry run helps surface secret typos before you tag a real release:

```bash
gh workflow run release.yml --ref development
gh run watch
```

If the build job reaches "Notarize the .app" without errors, your secrets are correct.
