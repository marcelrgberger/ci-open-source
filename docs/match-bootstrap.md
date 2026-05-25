# Bootstrapping fastlane match

One-time setup to get `apple-certificates` populated. After this, every Apple project under the `marcelrgberger` account can fetch its Developer-ID material with a single read-only match call from CI.

There are two ways to run the bootstrap: through GitHub Actions (recommended, no Mac required) or locally on a developer machine (fallback when ASC connectivity is misbehaving). The Actions path is the primary one.

## Path A — via GitHub Actions (recommended)

### Step 1 — set the admin secrets on `apple-certificates`

The bootstrap workflow lives inside the private `apple-certificates` repository and needs five pieces of state. Set them via the Settings UI or `gh secret set`:

| Secret | Value | Notes |
| --- | --- | --- |
| `ASC_API_KEY_ADMIN` | Base64-encoded `.p8` file content. | ASC API key with **Admin** role. Required to create certs and profiles. |
| `ASC_API_KEY_ID` | The ASC API Key ID. | Plain string. |
| `ASC_ISSUER_ID` | The ASC API Issuer ID. | UUID. |
| `MATCH_PASSWORD` | A strong random passphrase. | Save in 1Password — losing this means re-bootstrapping from scratch. Generate with `openssl rand -base64 48`. |

Also set the repository variable:

| Variable | Value |
| --- | --- |
| `APPLE_TEAM_ID` | Your Apple Developer Team ID (not a secret — Team IDs are public). |

### Step 2 — run the workflow

Navigate to `apple-certificates` → **Actions** → **Bootstrap / Rotate match** → **Run workflow**:

- **app-identifiers**: comma-separated list, e.g. `za.co.digitalfreedom.AutoBrew,za.co.digitalfreedom.AutoBrew.WidgetExtension,za.co.digitalfreedom.AutoBrew.cli`.
- **type**: leave on `developer_id` for direct-distribution apps.
- **nuke**: leave unchecked on the first run. Only enable when intentionally rotating — `nuke` revokes the existing cert and profiles on Apple's side as well.

The workflow runs `fastlane match`, creates the cert + provisioning profiles via the ASC API, encrypts the artefacts with `MATCH_PASSWORD`, and commits + pushes back into the repo.

When you later add a new bundle ID, re-run the same workflow with the full bundle-ID list — match is incremental and only creates what's missing.

### Step 3 — configure consumer repositories

Match access from CI uses an **SSH deploy key** scoped per-consumer-repo and read-only on `apple-certificates`. Generating + uploading the key is one command:

```bash
ssh-keygen -t ed25519 -C "match deploy key for <consumer-repo>" -f /tmp/match_key -N ""
gh repo deploy-key add /tmp/match_key.pub \
  --repo marcelrgberger/apple-certificates \
  --title "<consumer-repo> CI match read-access"
gh secret set MATCH_SSH_KEY --repo marcelrgberger/<consumer-repo> < /tmp/match_key
rm /tmp/match_key /tmp/match_key.pub
```

The pubkey lands on `apple-certificates` as a **read-only** deploy key; the private key lands as the `MATCH_SSH_KEY` secret on the consumer repo. Deploy keys are scoped to a single repository — even if a consumer's PAT pool gets compromised, the worst case is read access to one private cert repo.

Then set these additional secrets on each consumer:

| Secret | Value |
| --- | --- |
| `MATCH_SSH_KEY` | The ed25519 private key from above. |
| `MATCH_PASSWORD` | Same passphrase as on `apple-certificates`. |
| `ASC_API_KEY` | The same `.p8` content (base64). A non-admin role is sufficient here — read-only match + notarytool. |
| `ASC_API_KEY_ID` | Plain string. |
| `ASC_ISSUER_ID` | UUID. |
| `KEYCHAIN_PASSWORD` | Any random string — used to lock the transient CI keychain. |
| `SPARKLE_PRIVATE_KEY` | Per-app EdDSA private key, base64. Only when the app ships Sparkle updates. |

Notice that **no `APPLE_CERT_P12` is required on consumer repos** — match handles cert installation on the runner.

### Rotation playbook

When the Developer ID Application certificate is approaching expiry (Xcode shows a warning ~30 days ahead):

1. Open `apple-certificates` → Actions → Bootstrap / Rotate match.
2. Run workflow with the full bundle-ID list **and `nuke: true`**.
3. Wait for the run to finish. Apple-side revocation is immediate; the new cert is now in the match store.
4. CI builds continue working without intervention — the next `match --readonly` from a consumer pipeline picks up the new cert and profiles automatically.

No CI workflow changes needed during rotation.

## Path B — local fallback

Use this when the Actions path can't reach App Store Connect (rare; usually a transient ASC outage) or when you need to debug a match issue interactively.

### Prerequisites

- A Mac with Xcode 26+ and Ruby (system Ruby is fine).
- The ASC API `.p8` key at a known path on disk.
- The `MATCH_PASSWORD` passphrase in your password manager.

### Commands

```bash
gem install fastlane --no-document
export MATCH_PASSWORD='<your passphrase>'

# Provision incrementally — same as Path A step 2.
fastlane match developer_id \
  --git_url "git@github.com:marcelrgberger/apple-certificates.git" \
  --app_identifier "za.co.digitalfreedom.AutoBrew,za.co.digitalfreedom.AutoBrew.WidgetExtension,za.co.digitalfreedom.AutoBrew.cli" \
  --team_id <YOUR_TEAM_ID> \
  --api_key_path ~/Developer/aso/keys/AuthKey_<KEY_ID>.p8 \
  --api_key_id <KEY_ID> \
  --api_issuer_id <ISSUER_ID>

# Rotate — same as Path A step 2 with nuke:
fastlane match nuke developer_id \
  --git_url "git@github.com:marcelrgberger/apple-certificates.git" \
  --team_id <YOUR_TEAM_ID> \
  --api_key_path ~/Developer/aso/keys/AuthKey_<KEY_ID>.p8 \
  --api_key_id <KEY_ID> \
  --api_issuer_id <ISSUER_ID>
# then repeat the provision command above.
```

Match auto-commits and auto-pushes to the match repo when the git URL is writable, so no extra `git push` step is needed.
