# kinesis-to-opensearch-lambda

AWS Lambda function that consumes events from a Kinesis data stream and forwards them to two destinations:

- **OpenSearch** — a filtered subset of fields, minimizing index size
- **Splunk** — full event payloads via HTTP Event Collector (HEC)

This is specifically used for `quay.io`.

## Architecture

```
Kinesis Data Stream
        │
        ▼
   Lambda Handler
        │
        ├──► OpenSearch  (filtered: 9 allowed fields, daily indices, bulk API)
        │
        └──► Splunk HEC  (full payload, batched in groups of 500)
```

## Configuration

The function reads its configuration from environment variables and AWS Secrets Manager.

### Environment Variables

| Variable       | Description                                      | Example              |
|----------------|--------------------------------------------------|----------------------|
| `AWS_REGION`   | AWS region                                       | `us-east-1`          |
| `es_endpoint`  | OpenSearch domain endpoint (host only, port 443) | `search-xxx.es.amazonaws.com` |
| `secret_name`  | Secrets Manager secret name (see below)          | `prod/opensearch`    |
| `index_prefix` | Prefix for daily OpenSearch indices              | `logs-`              |

### Secrets Manager Fields

| Key                  | Required | Description                              |
|----------------------|----------|------------------------------------------|
| `master_user_name`   | No*      | OpenSearch username                      |
| `master_user_password` | No*   | OpenSearch password                      |
| `splunk_hec_url`     | Yes      | Splunk HEC endpoint URL                  |
| `splunk_hec_token`   | Yes      | Splunk HEC authentication token          |
| `splunk_index`       | Yes      | Destination Splunk index                 |
| `splunk_disabled`    | No       | Set to `"true"` to skip Splunk delivery  |

\* If `secret_name` is empty, the function authenticates to OpenSearch using IAM (AWSV4SignerAuth) instead.

## OpenSearch Field Filtering

Only these fields are sent to OpenSearch:

```
random_id, kind_id, account_id, performer_id,
repository_id, ip, metadata, datetime, @timestamp
```

Splunk receives the full unfiltered event.

## Development

### Prerequisites

- Python 3.x
- pip

### Install Dependencies

```bash
pip install -r requirements-dev.txt
```

### Run Tests

```bash
pytest
```

## Releasing

Releases are triggered by pushing a Git tag. This kicks off a GitHub Actions job that runs `build_tag.sh`, which creates a GitHub release and uploads the Lambda deployment package as a zip artifact.

```bash
git tag 1.0.0
git push origin 1.0.0
```
