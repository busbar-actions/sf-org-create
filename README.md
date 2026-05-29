# busbar-actions/sf-org-create

Create a Salesforce scratch org via direct API — no `sf` CLI, no monolith binary, no stored credentials.

## What it does

1. Authenticates to the DevHub via [`busbar-auth`](../../busbar-extensions/crates/busbar-auth) — `session_from_env()` uses GitHub OIDC token-exchange when running in Actions (`id-token: write`), or `SF_ACCESS_TOKEN`/`SF_INSTANCE_URL` for local dev.
2. Inserts `ScratchOrgInfo` against the DevHub's Data API.
3. Polls until `Status = Active`.
4. Redeems the one-time `AuthCode` via the `authorization_code` grant against the new org.
5. Writes the resulting credential context (`access_token` + `instance_url`) to a JSON file for downstream steps to consume.

The leg-1 (DevHub auth) and leg-2 (new-org token) clients are decoupled — the `ConnectedAppConsumerKey` field on the record names the leg-2 client independent of how you authed to the DevHub. This is the direct-API path that works where `sf org create scratch` couples both legs to one client.

## Inputs

| Input | Default | Description |
|---|---|---|
| `org-name` (required) | — | Org name for the new scratch. |
| `admin-email` (required) | — | Admin email. |
| `edition` | `Developer` | Org edition. |
| `duration-days` | `7` | Lifetime in days. |
| `namespace` | `` | Optional namespace. |
| `features` | `` | Comma-separated features list. |
| `description` | `` | Optional description. |
| `snapshot` | `` | Snapshot name or id — create-from-snapshot. |
| `poll-timeout-secs` | `600` | Seconds to wait for `Active`. |
| `credentials-output` | `.busbar/scratch-credentials.json` | Where to write the new org's credentials. |
| `version` | `latest` | `sf-org-create` release tag. |
| `binary-repo` | `busbar-actions/actions-dist` | Where to fetch the binary. |

## Outputs

| Output | Description |
|---|---|
| `scratch-org-info-id` | DevHub `ScratchOrgInfo` record id (use this to delete later). |
| `scratch-org-id` | New scratch org's 15-char id. |
| `username` | Admin username. |
| `credentials-path` | Path to the JSON file with the new org's `access_token` + `instance_url`. |

The credentials JSON is intentionally **not** exposed as a step output (which gets logged) — downstream steps read the file directly.

## Env vars (from the Environment — Variables, not Secrets, per the busbar convention)

The action consumes the universal busbar-auth config:

- `SF_INSTANCE_URL` — the DevHub.
- `BUSBAR_ECA_CLIENT_ID` — the OAuth client used for the leg-2 AuthCode redemption.
- `BUSBAR_ECA_CALLBACK_URL` — its matching callback (default `http://localhost:1717/OauthRedirect`).
- For OIDC: `BUSBAR_TOKEN_HANDLER` (defaults to `BBGitHubTokenExchangeHandler`), `BUSBAR_OIDC_AUDIENCE`. The runner injects `ACTIONS_ID_TOKEN_REQUEST_*` automatically when the workflow has `id-token: write`.
- Local-dev fallback: set `SF_ACCESS_TOKEN` + `SF_INSTANCE_URL` directly.

## Example

```yaml
permissions:
  contents: read
  id-token: write   # for OIDC exchange

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
        env:
          SF_INSTANCE_URL: ${{ fromJSON(steps.org.outputs.credentials-path) }}
        run: |
          export SF_ACCESS_TOKEN=$(jq -r .credentials.access_token "${{ steps.org.outputs.credentials-path }}")
          export SF_INSTANCE_URL=$(jq -r .credentials.instance_url "${{ steps.org.outputs.credentials-path }}")
          # … run subsequent busbar actions / sf calls …
```

## Cleanup

Pair with [`busbar-actions/sf-org-delete`](https://github.com/busbar-actions/sf-org-delete) for tear-down. Save `scratch-org-info-id` from this action's output and pass it to delete on the next step / job / cleanup workflow.
