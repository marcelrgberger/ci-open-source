# ci-open-source

Reusable GitHub Actions and workflows for open-source Apple-platform projects maintained by Marcel R. G. Berger.

This repository is the shared CI toolbox behind apps like **[AutoBrew](https://github.com/marcelrgberger/auto-brew)** and any future open-source macOS/iOS project under the [`marcelrgberger`](https://github.com/marcelrgberger) account. Each component is a small, composable building block — combine them via the top-level reusable workflow, or call them à la carte from your own workflow.

## What's in here

### Composite actions — `actions/`

Drop into any job step with `uses: marcelrgberger/ci-open-source/actions/<name>@v1`.

| Action | What it does |
| --- | --- |
| `apple-create-keychain` | Creates a fresh, unlocked temporary keychain on the runner. The recommended companion for `apple-match` — match installs the cert + profiles into this keychain. |
| `apple-import-cert` | Fallback alternative when not using match: imports a base64-encoded Developer ID `.p12` certificate into a fresh temporary keychain on the runner. |
| `apple-asc-key` | Materializes a base64-encoded App Store Connect API key (`AuthKey_*.p8`) on disk and exports its path + ID as outputs for downstream actions. |
| `apple-match` | Runs `fastlane match` (read-only by default) against a private certificate repository, fetching the Developer-ID cert and provisioning profiles for every bundle ID. Set `readonly: false` for the bootstrap/rotation flow. |
| `macos-archive` | Runs `xcodebuild archive` + `xcodebuild -exportArchive` with manual signing and an auto-generated `ExportOptions.plist`. |
| `macos-codesign-dmg` | Code-signs a built `.dmg` with the Developer ID Application identity (hardened runtime + secure timestamp). Required so the DMG passes Gatekeeper outside of brew/cask install paths. |
| `macos-notarize` | Submits an `.app` or `.dmg` to Apple's notary service via `notarytool` and staples the resulting ticket. |
| `sparkle-sign-appcast` | Signs a Sparkle update payload with an EdDSA private key and emits the signature + length for embedding in `appcast.xml`. |

### Reusable workflow — `.github/workflows/`

`release-macos-app.yml` orchestrates the above into a complete signed, notarized, Sparkle-ready macOS release pipeline. Call it with `uses: marcelrgberger/ci-open-source/.github/workflows/release-macos-app.yml@v1`.

### Docs — `docs/`

| Doc | Topic |
| --- | --- |
| `match-bootstrap.md` | One-time setup: create the private `apple-certificates` repo, run `fastlane match nuke developer_id`, then `match developer_id` to populate it. |
| `onboarding.md` | Adding a new Apple project to ci-open-source: required secrets, minimum `project.yml` shape, how to call the reusable workflow. |

## Versioning

This repository follows semantic versioning. Pin consumers to the **major version branch** rather than a specific tag — that way you receive forward-compatible patches automatically but never a breaking change.

- `@v1` — current major. Tracks the latest `v1.x.y` tag. Recommended for production use.
- `@v1.0.0` — exact tag. Use if you need byte-for-byte reproducibility.
- `@main` — bleeding edge. Don't use this for releases.

When a breaking change lands, a `v2` branch and `v2.0.0` tag are cut; `v1` continues to receive non-breaking fixes for at least one release cycle.

## License

MIT — see [LICENSE](LICENSE).

## Related repositories

- [marcelrgberger/auto-brew](https://github.com/marcelrgberger/auto-brew) — first consumer; macOS menu-bar Homebrew companion.
- `marcelrgberger/apple-certificates` (private) — the fastlane match store this toolbox reads from. Not browsable; see `docs/match-bootstrap.md` for setup.
