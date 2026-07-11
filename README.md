> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/sf-org-create

Create a Salesforce scratch org via direct API — no `sf` CLI, no monolith binary, no stored credentials.

## What it does

The `sf-org-create` binary owns all logic and UX. The action only installs it (via `busbar-actions/setup`) and passes inputs + auth/config env through. The binary:

1. Authenticates to the **DevHub** via [`busbar-auth`](https://github.com/busbar-extensions) — **self-minting via GitHub OIDC by default**: `session_from_env()` exchanges the runner's OIDC id-token for a short-lived DevHub session **in-process** (requires `target-instance` set + `id-token: write` granted). No Salesforce token is handed to a script, written to `GITHUB_ENV`, or passed as an input/output. A pre-obtained `SF_ACCESS_TOKEN`/`SF_INSTANCE_URL` (via the `sf-access-token`/`sf-instance-url` inputs) is an **optional local-dev override** only.
2. Inserts `ScratchOrgInfo` against the DevHub's Data API (`v60.0`).
3. Polls until `Status = Active` (or `Error`), up to `poll-timeout-secs`.
4. Redeems the one-time `AuthCode` via the `authorization_code` grant against the **new org**.
5. Writes the resulting credential context (`access_token` + `instance_url`) to a JSON file for downstream steps to consume, masks the new token, and emits step outputs, a job summary, and a notice annotation.
6. Disposes the DevHub session — revoking it only if it was OIDC-minted (ephemeral + job-scoped), and zeroizing its secrets on drop.

The leg-1 (DevHub auth) and leg-2 (new-org token) clients are fully decoupled: leg-1 uses the busbar OIDC ECA (`BUSBAR_ECA_CLIENT_ID`); the `ConnectedAppConsumerKey` field on the record names the leg-2 client (`BUSBAR_SCRATCH_CONSUMER_KEY`, default `PlatformCLI`) independent of how you authed to the DevHub. This is the direct-API path that works where `sf org create scratch` couples both legs to one client.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `target-instance` | no¹ | `` | **PRIMARY OIDC path.** Instance URL of the Busbar-equipped DevHub (e.g. `busbar-pilot-demo2`) to self-mint a short-lived token against. Maps to `SF_INSTANCE_URL`. |
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
| `busbar-package` | no | `` | busbar (core) package version (04t id or alias). Presence triggers the in-process Busbar setup sequence (packages, ECA policy, permset, handler, trust rules) right after AuthCode redemption; empty skips it (create-only). |
| `github-package` | no | `` | busbar-github (adapter) package version (04t id or alias). Required if `busbar-package` is set. |
| `trust-specs` | no | `` | Path to a trust-rule spec file/directory (relative to the checked-out repo). Only used when `busbar-package` is set; empty skips the trust phase. |
| `hub-instance-url` | no | `` | Instance URL of a Busbar Hub org to register the new org with. Only used when `busbar-package` is set; empty skips Hub registration. |
| `eca-client-id` | no | `` | Optional OIDC tuning → `ECA_CLIENT_ID`. Baked default; override only on a PBO consumer rotation. |
| `token-handler` | no | `` | Optional OIDC tuning → `TOKEN_HANDLER_APEX`. Defaults to `GitHubTokenExchangeHandler`. |
| `oidc-audience` | no | `` | Optional OIDC tuning → `OIDC_AUDIENCE`. Defaults to the target instance URL. |
| `sf-instance-url` | no | `` | **Optional local-dev/advanced override** of the DevHub instance URL; wins over `target-instance`. |
| `sf-access-token` | no | `` | **Optional local-dev/advanced override only.** A pre-obtained DevHub token; when set the binary skips OIDC self-minting. Leave empty in CI. |
| `version` | no | `latest` | `sf-org-create` release tag. `latest` resolves the most recent release. |
| `binary-repo` | no | `busbar-actions/actions-dist` | GitHub repo hosting the prebuilt binary releases. |

¹ `target-instance` is required for the default OIDC path (or supply the `sf-instance-url`/`sf-access-token` local-dev override). One of the two auth paths must be configured.

## Outputs

| Output | Description |
|---|---|
| `scratch-org-info-id` | DevHub `ScratchOrgInfo` record id (use this to delete later). |
| `scratch-org-id` | New scratch org's 15-char id. |
| `username` | New scratch org's admin username. |
| `login-url` | Login URL for the created scratch org. |
| `credentials-path` | Path to the JSON file with the new org's `access_token` + `instance_url`. |

The new org's `access_token` is intentionally **not** exposed as a step output (outputs get logged). It is masked, written only to the credentials file, and handed off to downstream steps via that file.

## Auth & permissions model — OIDC self-mint (default)

> [!NOTE]
> **DevHub prerequisite.** In-process OIDC → DevHub requires a DevHub that has the
> **Busbar managed package installed and a trust rule** for the calling repo's
> workflow. The pilot DevHub `busbar-pilot-demo2` is set up this way — point
> `target-instance` at it (supplied via the workflow Environment's
> `SF_INSTANCE_URL` variable). You can also exercise the action locally via the
> `sf-instance-url` + `sf-access-token` override inputs (the local-dev fast path).

This action **self-mints** its DevHub token. The binary exchanges the runner's
GitHub OIDC id-token for a short-lived Salesforce session **in-process**, uses
it, then revokes + zeroizes it at exit (`SalesforceSession::dispose`). **No
Salesforce access token is ever handed to a script, written to `GITHUB_ENV`, or
passed as an action input/output.** There is **no** `org-auth` handoff step.

To self-mint, set **`target-instance`** (the Busbar-equipped DevHub URL, e.g. `busbar-pilot-demo2`) and grant `id-token: write`:

```yaml
permissions:
  contents: read
  id-token: write   # REQUIRED — lets the runner mint the OIDC id-token

jobs:
  build-scratch:
    runs-on: ubuntu-latest
    environment: devhub-pbo-scratch    # vars: BUSBAR_ECA_CLIENT_ID, BUSBAR_SCRATCH_*, ...
    steps:
      - uses: busbar-actions/sf-org-create@v1
        id: org
        with:
          target-instance: ${{ vars.SF_INSTANCE_URL }}   # Busbar-equipped DevHub (busbar-pilot-demo2)
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

(The `SF_ACCESS_TOKEN` exported above is the **new scratch org's** token from the
credentials file, not the DevHub token — the DevHub token never leaves the
binary.)

Supporting config still comes from the **GitHub Environment as Variables** (not Secrets — per the config-from-environments convention; nothing here is secret in steady state):

- `BUSBAR_ECA_CLIENT_ID` — **leg-1**: the busbar OIDC ECA consumer key used to exchange the GitHub OIDC token for a DevHub session (or override per-run with the `eca-client-id` input). Has a baked default.
- `BUSBAR_SCRATCH_CONSUMER_KEY` — **leg-2**: the Connected App consumer key stamped onto the scratch's `ConnectedAppConsumerKey` and used for the `AuthCode` redemption. Defaults to `PlatformCLI` (Salesforce's platform-global CLI app, present in every Dev Hub). Override only if a Dev Hub needs a different scratch-signup app. The busbar ECA must **not** be used here — it's Tier-2 global OAuth pinned to the publisher org and only resolves as a `ConnectedAppConsumerKey` there.
- `BUSBAR_SCRATCH_CALLBACK_URL` — the leg-2 callback (default `http://localhost:1717/OauthRedirect`); must match what the consumer key's app registers.
- OIDC tuning: `BUSBAR_TOKEN_HANDLER`/`token-handler`, `BUSBAR_OIDC_AUDIENCE`/`oidc-audience`. The runner injects `ACTIONS_ID_TOKEN_REQUEST_*` automatically when the workflow grants `id-token: write`.

### Local-dev / advanced override

For local runs (or where a DevHub token already exists), set the `sf-instance-url`
+ `sf-access-token` inputs. When `sf-access-token` is non-empty the binary uses
that token directly and **skips OIDC self-minting**. This path does **not** revoke
the handed-in token (it isn't OIDC-minted) — it only zeroizes it.

## Observability

The binary emits a `$GITHUB_STEP_SUMMARY` table (ids, username, login URL, edition, duration), step outputs, and a notice annotation on success. On failure it emits a `::error` workflow command and exits non-zero. Note: failures are surfaced via the legacy `fail()` path, so they reach the log/annotation but are **not** guaranteed into the job summary (the binary does not yet use `run_outcome` + `RecordingReporter`).

## Cleanup

This is a **composite** action, so it has no `post:` hook. The DevHub (leg-1) session is disposed in-process — revoked if OIDC-minted, always zeroized. The **new org's** session token is handed off via the credentials file and is **not** revoked by this action; tear-down is the job's responsibility.

Pair with [`busbar-actions/sf-org-delete`](https://github.com/busbar-actions/sf-org-delete) for tear-down: save `scratch-org-info-id` from this action's output and pass it to delete on a later step / job / cleanup workflow.
