# n8n + BigQuery Marketing Metrics

This repository contains n8n workflows to automate:
- **Ingestion (Drive → GCS → BigQuery)**  
- **HTTP API endpoint** to fetch rows by date interval  
- **Monthly CAC/ROAS report** (email)  
- **Tool workflow** to compare CAC/ROAS between two months (for agents/automations)

> **Note**: Workflows reference Google Cloud resources but **do not** include any credentials. You must configure credentials in n8n after import.

---

## Workflows in this repo

- `Ingestion___Drive.json` — Watches a Google Drive folder, uploads CSVs to Google Cloud Storage (GCS) and loads them to BigQuery with schema autodetect (via jobs API).  
- `API_sql_endpoint.json` — Exposes `POST /metrics` to return table rows between `from_date` and `to_date`.  
- `Monthly_Query.json` — Computes CAC & ROAS for two calendar months and emails an HTML table.  
- `BigQuery_Tool.json` — Helper workflow to compute CAC/ROAS for two months (YYYY-MM).  
- `AI_Query.json` — Optional: an AI agent that calls the tool workflow and formats results.

Data table expected columns:
```
date, platform, account, campaign, country, device, spend, clicks, impressions, conversions
```

---

## Prerequisites

- n8n (self-hosted or Desktop)
- Google Cloud project with:
  - **BigQuery** enabled and a dataset (e.g., `ads_spend`)
  - **GCS bucket** in the same multi‑region/region as your dataset (e.g., `US`)
- n8n Credentials:
  - **Google BigQuery** (OAuth2 or Service Account)
  - **Google Drive** (for ingestion), optional if not using ingestion
  - **Gmail** (for monthly email), optional

Minimum permissions:
- BigQuery: *BigQuery Job User* + *BigQuery Data Editor* (dataset-level)
- GCS: *Storage Object Viewer* (and *Object Creator* if writing)
- Drive: read access to the watched folder

---

## Quick Start

1. **Clone** this repository.
2. Copy environment sample and set your values:
   ```bash
   cp .env.sample .env
   ```
3. **Start n8n** (ensure it can read your `.env` — for Docker, pass `--env-file .env` or map envs).
4. In n8n, **import** each workflow JSON via *Workflows → Import from file*:
   - `Ingestion___Drive.json`
   - `API_sql_endpoint.json`
   - `Monthly_Query.json`
   - `BigQuery_Tool.json`
   - `AI_Query.json` (optional)
5. Re‑assign **Credentials** in each node to your n8n credential entries.
6. Update node expressions to reference env vars, for example:
   - `{{$env.N8N_GCP_PROJECT_ID}}`
   - `{{$env.N8N_GCS_BUCKET}}`
   - `{{$env.N8N_DRIVE_FOLDER_ID}}`
   - `{{$env.N8N_BQ_DATASET}}`
   - `{{$env.N8N_BQ_TABLE}}`

> Tip: If you prefer to hardcode values, you can, but environment variables make the repo portable and safer.

---

## Configuration

Set these environment variables (see `.env.sample`):

```
N8N_GCP_PROJECT_ID=your-gcp-project-id
N8N_GCS_BUCKET=your-gcs-bucket
N8N_DRIVE_FOLDER_ID=your-drive-folder-id
N8N_BQ_DATASET=ads_spend
N8N_BQ_TABLE=ads_spend-20250828
N8N_REGION=US
N8N_EMAIL_TO=you@example.com
```

### Regional consistency
Ensure your **BigQuery dataset** and **GCS bucket** are in the same region/multi‑region (e.g., `US`).

---

## Using the API workflow

Endpoint (after activating the workflow):
```
POST /webhook/metrics
Content-Type: application/json
{
  "from_date": "YYYY-MM-DD",
  "to_date":   "YYYY-MM-DD"
}
```

Example:
```bash
curl -X POST https://<your-n8n-host>/webhook/metrics   -H 'Content-Type: application/json'   -d '{ "from_date": "2025-06-01", "to_date": "2025-06-30" }'
```

Response: JSON rows from BigQuery within the date range, including the expected columns.

> If you must send `DD-MM-YYYY`, adjust the SQL to use `PARSE_DATE('%d-%m-%Y', ...)` in the `DECLARE` statements.

---

## Monthly report

- The **Monthly_Query** workflow runs a BigQuery query comparing **current month vs previous month** (CAC & ROAS), builds an HTML table, and sends it via Gmail.
- Configure recipients with `N8N_EMAIL_TO` or inside the Gmail node.
- Use the Cron node to schedule (e.g., day 1 at 08:00, timezone of your choice).

---

## Security notes

- Project ID and resource names are **not secrets**; do **not** publish credentials (OAuth tokens, service account JSON).
- Keep your GCS bucket **private** unless you need public access.
- Prefer least-privilege IAM roles.
- Enable **Webhook authentication** (Basic auth, header token, or IP allow‑list) for production endpoints.

---

## Troubleshooting

- **`Not found: URI gs://...`** — verify the exact object name written to GCS; don’t append `.csv` if your object key doesn’t have it. Ensure n8n identity has `storage.objects.get` on the bucket.
- **`Access Denied: File gs://...`** — missing *Storage Object Viewer* on the bucket for the account running the BigQuery job.
- **Empty current month** — data may not exist for that month; try a date interval query or a different month.
- **Date parse errors** — ensure `YYYY-MM-DD` inputs, or change to `PARSE_DATE('%d-%m-%Y', ...)`.

---

## License

MIT (or choose your preferred license)
