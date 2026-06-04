# busbar-actions/sf-org-create

Create a Salesforce scratch org via direct API — no `sf` CLI, no monolith binary, no stored credentials.

## What it does

The `sf-org-create` binary owns all logic and UX. The action only installs it (via `busbar-actions/setup`) and passes inputs + auth/config env through. The binary:

1. Authenticates to the **DevHub** via [`busbar-auth`](https://github.com/busbar-extensions) — `session_from_env()` uses GitHub OIDC token-exchange when running in Actions (requires `id-token: write`), or `SF_ACCESS_TOKEN`/`SF_INSTANCE_URL` for local dev.
2. Inserts `ScratchOrgInfo` against the DevHub's Data API (`v60.0`).
3. Polls until `Status = Active` (or `Error`), up to `poll-timeout-secs`.
4. Redeems the one-time `AuthCode` via the `authorization_code` grant against the **new org**.
5. Writes the resulting credential context (`access_token` + `instance_url`) to a JSON file for downstream steps to consume, masks the new token, and emits step outputs, a job summary, and a notice annotation.
6. Disposes the DevHub session — revoking it only if it was OIDC-minted (ephemeral + job-scoped), and zeroizing its secrets on drop.

The leg-1 (DevHub auth) and leg-2 (new-org token) clients are fully decoupled: leg-1 uses the busbar OIDC ECA (`BUSBAR_ECA_CLIENT_ID`); the `ConnectedAppConsumerKey` field on the record names the leg-2 client (`BUSBAR_SCRATCH_CONSUMER_KEY`, default `PlatformCLI`) independent of how you authed to the DevHub. This is the direct-API path that works where `sf org create scratch` couples both legs to one client.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `org-name` | yes | — | Org name for the new scratch. |
| `admin-email` | yes | — | Admin email for the scratch. |
| `edition` | no | `Developer` | Org edition (Developer, Enterprise, Group, Professional, Partner Developer, …). |
| `duration-days` | no | `7` | Lifetime in days (1–30 per Salesforce policy). |
| `namespace` | no | `` | Optional namespace. |
| `features` | no | `` | Optional comma-separated features list. |
| `description` | no | `` | Optional description. |
| `snapshot` | no | `` | Snapshot name or id — create-from-snapshot. |
| `poll-timeout-secs` | no | `600` | Seconds to wait for the org to reach `Active`. |
| `credentials-output` | no | `.busbar/scratch-credentials.json` | Where to write the new org's credentials JSON. |
| `version` | no | `latest` | `sf-org-create` release tag. `latest` resolves the most recent release. |
| `binary-repo` | no | `busbar-actions/actions-dist` | GitHub repo hosting the prebuilt binary releases. |

## Outputs

| Output | Description |
|---|---|
| `scratch-org-info-id` | DevHub `ScratchOrgInfo` record id (use this to delete later). |
| `scratch-org-id` | New scratch org's 15-char id. |
| `username` | New scratch org's admin username. |
| `login-url` | Login URL for the created scratch org. |
| `credentials-path` | Path to the JSON file with the new org's `access_token` + `instance_url`. |

The new org's `access_token` is intentionally **not** exposed as a step output (outputs get logged). It is masked, written only to the credentials file, and handed off to downstream steps via that file.

## Auth & permissions model

The action consumes the universal `busbar-auth` config from the **GitHub Environment as Variables** (not Secrets — per the config-from-environments convention; nothing here is secret in steady state):

- `SF_INSTANCE_URL` — the DevHub.
- `BUSBAR_ECA_CLIENT_ID` — **leg-1 only**: the busbar OIDC ECA consumer key used to exchange the GitHub OIDC token for a DevHub session. **Required** for OIDC auth.
- `BUSBAR_SCRATCH_CONSUMER_KEY` — **leg-2**: the Connected App consumer key stamped onto the scratch's `ConnectedAppConsumerKey` and used for the `AuthCode` redemption. Defaults to `PlatformCLI` (Salesforce's platform-global CLI app, present in every Dev Hub). Override only if a Dev Hub needs a different scratch-signup app. The busbar ECA must **not** be used here — it's Tier-2 global OAuth pinned to the publisher org and only resolves as a `ConnectedAppConsumerKey` there.
- `BUSBAR_SCRATCH_CALLBACK_URL` — the leg-2 callback (default `http://localhost:1717/OauthRedirect`); must match what the consumer key's app registers.
- For OIDC DevHub auth: `BUSBAR_TOKEN_HANDLER`, `BUSBAR_OIDC_AUDIENCE`. The runner injects `ACTIONS_ID_TOKEN_REQUEST_*` automatically when the workflow grants `id-token: write`.
- Local-dev fallback: set `SF_ACCESS_TOKEN` + `SF_INSTANCE_URL` directly (skips OIDC).

The calling workflow must grant:

```yaml
permissions:
  contents: read
  id-token: write   # for the OIDC token-exchange DevHub auth
```

## Example

```yaml
permissions:
  contents: read
  id-token: write

jobs:
  build-scratch:
    runs-on: ubuntu-latest
    environment: devhub-pbo-scratch    # vars: SF_INSTANCE_URL, BUSBAR_ECA_CLIENT_ID, ...
    steps:
      - uses: busbar-actions/sf-org-create@v1
        id: org
        with:
          org-name: "PR-${{ github.event.number }}"
          admin-email: ci@example.com
          duration-days: 1
          snapshot: golden-busbar         # create-from-snapshot

      - name: Use the new org
        run: |
          export SF_ACCESS_TOKEN=$(jq -r .credentials.access_token "${{ steps.org.outputs.credentials-path }}")
          export SF_INSTANCE_URL=$(jq -r .credentials.instance_url "${{ steps.org.outputs.credentials-path }}")
          # … run subsequent busbar actions / sf calls …
```

## Observability

The binary emits a `$GITHUB_STEP_SUMMARY` table (ids, username, login URL, edition, duration), step outputs, and a notice annotation on success. On failure it emits a `::error` workflow command and exits non-zero. Note: failures are surfaced via the legacy `fail()` path, so they reach the log/annotation but are **not** guaranteed into the job summary (the binary does not yet use `run_outcome` + `RecordingReporter`).

## Cleanup

This is a **composite** action, so it has no `post:` hook. The DevHub (leg-1) session is disposed in-process — revoked if OIDC-minted, always zeroized. The **new org's** session token is handed off via the credentials file and is **not** revoked by this action; tear-down is the job's responsibility.

Pair with [`busbar-actions/sf-org-delete`](https://github.com/busbar-actions/sf-org-delete) for tear-down: save `scratch-org-info-id` from this action's output and pass it to delete on a later step / job / cleanup workflow.
