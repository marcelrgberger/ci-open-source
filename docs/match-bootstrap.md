# Bootstrapping fastlane match for the open-source app fleet

One-time setup to get `apple-certificates` populated and every Apple project under the `marcelrgberger` account talking to the same match store.

## Prerequisites

- A Mac with Xcode 26+ and Ruby installed (Homebrew's `ruby` is fine; the system Ruby works for fastlane too).
- An App Store Connect API key with **Admin** role (key ID, issuer ID, `.p8` file).
- The private GitHub repository `marcelrgberger/apple-certificates` already created (empty is fine; this guide fills it).
- Your Developer ID Application certificate accessible — either already in your login keychain or in a `.p12` file. If you don't have it locally, generate a fresh one through Xcode's Manage Certificates UI before starting.

## Step 1 — install fastlane

```bash
gem install fastlane --no-document
fastlane --version  # expect 2.220+
```

## Step 2 — pick a passphrase

Generate a strong random passphrase. **Save it in your password manager immediately** — losing this passphrase means the match repository becomes unreadable and you start over from scratch.

```bash
openssl rand -base64 48
```

## Step 3 — nuke any stale state

If `apple-certificates` already has old entries (or you're rotating after a leak), wipe them first. **`nuke` deletes the corresponding certs on Apple's side too** — only run this if you're sure.

```bash
fastlane match nuke developer_id \
  --git_url "git@github.com:marcelrgberger/apple-certificates.git" \
  --team_id <YOUR_TEAM_ID> \
  --api_key_path ~/Developer/aso/keys/AuthKey_<KEY_ID>.p8 \
  --api_key_id <KEY_ID> \
  --api_issuer_id <ISSUER_ID>
```

Skip this step on a fresh setup.

## Step 4 — provision certificates and profiles

Run match in **read-write** mode. This is the only invocation that creates new artefacts; CI runs `--readonly`.

```bash
export MATCH_PASSWORD='<the passphrase from step 2>'

fastlane match developer_id \
  --git_url "git@github.com:marcelrgberger/apple-certificates.git" \
  --app_identifier "za.co.digitalfreedom.AutoBrew,za.co.digitalfreedom.AutoBrew.WidgetExtension,za.co.digitalfreedom.AutoBrew.cli" \
  --team_id <YOUR_TEAM_ID> \
  --api_key_path ~/Developer/aso/keys/AuthKey_<KEY_ID>.p8 \
  --api_key_id <KEY_ID> \
  --api_issuer_id <ISSUER_ID>
```

After this, the private `apple-certificates` repo contains AES-encrypted blobs and the local login keychain has the newly issued cert. The profiles end up under `~/Library/MobileDevice/Provisioning Profiles`.

Re-run the same command (without `nuke`) every time you add a new bundle ID to an app — match is incremental, it only creates what's missing.

## Step 5 — push and verify

```bash
cd $(fastlane match dir)  # match prints this; usually ~/Library/.../fastlane/match
git status                # confirm the new files are present
git push                  # push to apple-certificates
```

Match auto-commits and auto-pushes when you give it a writable git URL — verify the push landed on GitHub by opening the private repo's commit list.

## Step 6 — configure consumer secrets

In each Apple app repository that consumes `ci-open-source`, set these secrets:

| Secret | Value |
| --- | --- |
| `MATCH_PASSWORD` | The passphrase from step 2. |
| `MATCH_GIT_TOKEN` | A fine-grained PAT with **Contents: Read** scoped to `marcelrgberger/apple-certificates`. |
| `APPLE_CERT_P12` | The Developer ID `.p12` exported from the keychain, base64-encoded. |
| `APPLE_CERT_PASSWORD` | The `.p12` export password. |
| `KEYCHAIN_PASSWORD` | Any random string — used to lock the transient CI keychain. |
| `ASC_API_KEY` | The `.p8` file content, base64-encoded. |
| `ASC_API_KEY_ID` | The ASC API Key ID. |
| `ASC_ISSUER_ID` | The ASC API Issuer ID. |

The cert + ASC key secrets stay needed because match-managed profiles still reference your real Developer ID identity — the `.p12` carries the private key that match cannot store on Apple's side.

## Step 7 — call the reusable workflow

See [onboarding.md](onboarding.md) for the consumer-side workflow template.

## Rotation playbook

When your Developer ID certificate is approaching expiry (Xcode shows it in the Signing UI ~30 days ahead):

1. From your developer Mac, run `fastlane match nuke developer_id` followed by `fastlane match developer_id …` (same command as step 4).
2. Re-export the new `.p12` from the keychain and refresh the `APPLE_CERT_P12` + `APPLE_CERT_PASSWORD` secrets in every consumer repository.
3. CI runs continue to work because `--readonly` picks up the new cert + profiles from the next match repo pull automatically.

No CI workflow changes needed during rotation — only the per-consumer cert secret rotates.
