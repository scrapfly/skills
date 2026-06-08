---
name: scrapfly-rpa
description: >-
  Use the Scrapfly RPA API to run typed workflow graphs against managed
  cloud browsers, retrieve produced artifacts (HTML snapshots, downloads,
  HTTP response bodies, JS return values), and route those artifacts into
  Scrapfly's managed bucket or your own customer GCS or S3 bucket. Use
  when the user asks to "run an RPA workflow", "fetch artifacts from a
  run", "configure BYOS storage", "set up customer GCS or S3 for RPA",
  "test a storage destination", or any task involving the
  /v2/rpa/* endpoints.
---

# Scrapfly RPA

Scrapfly RPA (Robotic Process Automation) is a typed-graph workflow runtime
on top of Scrapfly Cloud Browser. A workflow is a directed graph of nodes
(navigate, click, type, evaluate, snapshot, branch, for_each, http,
webhook, AI step, etc). Runs produce artifacts (bytes plus indexed
metadata) that can land in Scrapfly's managed bucket or in a customer
owned bucket via the Bring-Your-Own-Storage (BYOS) flow.

## When to use

- Triggering a stored RPA workflow and waiting for the run to complete.
- Listing or downloading artifacts produced by a run.
- Configuring a customer GCS or S3 bucket as the storage destination
  for a workflow's artifacts (BYOS, for HIPAA workloads or data residency).
- Testing a storage destination before saving it ("does my bucket plus
  credentials configuration actually work?").
- Inspecting which destination an existing artifact landed in
  (managed vs gcs vs s3).

## Setup

API key via environment variable, used as a query string parameter on
every request:

```bash
export SCRAPFLY_API_KEY="scp-live-..."
```

All endpoints live under `https://api.scrapfly.io/v2/rpa/`.

## Run a workflow

```bash
curl -sX POST "https://api.scrapfly.io/v2/rpa/workflows/$WORKFLOW_ID/run?key=$SCRAPFLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input": { "customer_id": 42 }
  }'
# Response: { "run": { "id": "run_01k...", "status": "queued", ... } }
```

Poll until completion:

```bash
RUN_ID=run_01k...
while :; do
  status=$(curl -s "https://api.scrapfly.io/v2/rpa/runs/$RUN_ID?key=$SCRAPFLY_API_KEY" | jq -r .run.status)
  case "$status" in completed|failed|canceled) break ;; esac
  sleep 2
done
echo "Run finished with status: $status"
```

## List artifacts from a run

```bash
curl -s "https://api.scrapfly.io/v2/rpa/runs/$RUN_ID/artifacts?key=$SCRAPFLY_API_KEY" \
  | jq '.artifacts[] | {id, kind, bytes, content_type, destination_kind}'
```

Each artifact's `destination_kind` is one of:
- `managed`: bytes in Scrapfly's bucket
- `gcs`: bytes in your customer GCS bucket
- `s3`: bytes in your customer S3 bucket

## Download an artifact

```bash
curl -s "https://api.scrapfly.io/v2/rpa/artifacts/$ART_ID?key=$SCRAPFLY_API_KEY" \
  | jq -r .artifact.download_url \
  | xargs curl -L -o output.html
```

The signed URL expires in 5 minutes. Mint a fresh one if you need to
retry.

## Storage destination (managed default)

Workflows save their preferred destination in `definition.context.storage_destination`.
When that field is empty or set to `kind: "managed"`, artifacts land in
Scrapfly's managed bucket. Zero configuration required.

## Storage destination (BYOS GCS)

Prerequisite vault item shape: create a Scrapfly Vault item of type
`blob`. Put the full GCP service account JSON in the `data` field as
a single string. Bind that vault item to the run under the alias you
set in `credentials_vault_ref`.

Legacy compatibility: items of type `password` with the JSON payload
in the `password` field are still accepted so existing BYOS
customers do not have to re-bind. New integrations should use the
typed `blob` item.

Point a workflow at your own GCS bucket:

```json
{
  "definition": {
    "nodes": [ ... ],
    "edges": [ ... ],
    "context": {
      "storage_destination": {
        "kind": "gcs",
        "bucket": "my-rpa-archive",
        "prefix": "rpa/runs",
        "credentials_vault_ref": "vault_01k.../sa_json"
      }
    }
  }
}
```

`credentials_vault_ref` points at a Scrapfly Vault item holding a GCP
service account JSON. The credentials never leave your Vault to disk
or to a workflow definition; they get resolved per call and zeroized
immediately after use. The Vault itself is end-to-end encrypted with
a key Scrapfly does not hold, which makes the BYOS credential path
HIPAA compliant by construction. How that vault key gets shared
across your organization is your responsibility.

## Storage destination (BYOS S3)

Same shape with AWS:

```json
{
  "storage_destination": {
    "kind": "s3",
    "bucket": "my-rpa-archive",
    "region": "us-east-1",
    "prefix": "rpa/runs",
    "credentials_vault_ref": "vault_01k.../aws_creds"
  }
}
```

The vault item holds an AWS access key triple
(`access_key_id`, `secret_access_key`, optional `session_token`).

## Test a destination

Before saving a BYOS configuration, round-trip a probe object through
the destination so misconfigurations surface immediately. Pass the
same `vault_bindings` map you would send on a workflow run so the
probe can resolve the customer credentials with the customer-held
vault key:

```bash
curl -sX POST "https://api.scrapfly.io/v2/rpa/storage/probe?key=$SCRAPFLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "storage_destination": {
      "kind": "gcs",
      "bucket": "my-rpa-archive",
      "credentials_vault_ref": "acme_gcs"
    },
    "vault_bindings": {
      "acme_gcs": {
        "vault_id": "vlt_01k...",
        "item_id":  "vlti_01k...",
        "key":      "BASE64-OF-CUSTOMER-VAULT-KEY"
      }
    }
  }'
```

Successful probe:

```json
{
  "ok": true,
  "destination_kind": "gcs",
  "bucket": "my-rpa-archive",
  "storage_path": "gs://my-rpa-archive/.scrapfly-rpa-probe/01k...",
  "write_ms": 142,
  "sign_ms": 18
}
```

Common failure codes:
- `storage_destination_invalid`: required field missing (bucket, region
  for S3, credentials_vault_ref).
- `storage_probe_write_failed`: credentials resolved but the bucket
  rejected the write. Check IAM permissions on the service account
  or AWS access key.

## What it doesn't do

- Authoring workflows from scratch via API. Use the dashboard editor
  for that; this skill is for run + artifact + storage operations.
- Replacing the managed bucket on existing artifacts. Editing
  `storage_destination` only affects future runs; existing rows stay
  where they were written.
- Retention. The managed bucket evicts after 90 days. BYOS buckets are
  governed by your own lifecycle policy.

## Related skills

- `scrapfly-browser`: the lower-level Cloud Browser API the RPA runtime
  drives. Use it when you need direct CDP control instead of typed
  workflow steps.
- `scrapfly-cli`: shell access to all Scrapfly products. There is no
  `scrapfly rpa` subcommand - drive RPA workflows through the REST API or
  the SDKs; the CLI's `exit-peer` group installs the BYOP connector an RPA
  `byop` profile routes through.
