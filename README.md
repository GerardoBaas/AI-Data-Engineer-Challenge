# n8n + BigQuery Marketing Metrics

This repository contains n8n workflows to automate marketing data ingestion, analysis, and reporting using Google BigQuery.

> **Note**: These workflows reference Google Cloud resources but **do not** include credentials. Configure them in n8n after import.

---

## Workflows

### 1. Ingestion – Drive (`Ingestion___Drive.json`)
**Purpose**  
- Watches a Google Drive folder for new CSV files.  
- Downloads the file, uploads it to a Google Cloud Storage (GCS) bucket.  
- Creates/ensures a BigQuery dataset exists.  
- Runs a BigQuery `LOAD DATA` job with `autodetect` enabled.  

**Setup**  
- Update the Drive folder ID to your own folder.  
- Update the GCS bucket name (`N8N_GCS_BUCKET`).  
- Ensure the dataset and table logic matches your requirements.  

**Expected columns in the CSV**  
```
date, platform, account, campaign, country, device, spend, clicks, impressions, conversions
```

---

### 2. API SQL Endpoint (`API_sql_endpoint.json`)
**Purpose**  
- Exposes a Webhook endpoint:  
  ```
  POST /webhook/metrics
  ```
- Accepts JSON body:  
  ```json
  { "from_date": "YYYY-MM-DD", "to_date": "YYYY-MM-DD" }
  ```
- Runs a BigQuery query returning all rows between the given dates.  

**Setup**  
- Import workflow, re-assign BigQuery credentials.  
- (Optional) Enable Basic Auth or a secret header for production security.  

**Test Example**  
```bash
curl -X POST https://<your-n8n>/webhook/metrics   -H 'Content-Type: application/json'   -d '{ "from_date": "2025-06-01", "to_date": "2025-06-30" }'
```

---

### 3. Monthly Query (`Monthly_Query.json`)
**Purpose**  
- Scheduled workflow (via Cron).  
- Runs a BigQuery query comparing **current month vs previous month**.  
- Computes **CAC** (spend/conversions) and **ROAS** (revenue/spend).  
- Formats results into an HTML table.  
- Sends the table via Gmail.  

**Setup**  
- Set up Gmail credentials.  
- Update recipient (`N8N_EMAIL_TO`).  
- Adjust Cron node to match desired schedule (e.g., first day of the month).  

**Output**  
- HTML email with table comparing CAC & ROAS between two months.  

---

### 4. BigQuery Tool (`BigQuery_Tool.json`)
**Purpose**  
- Provides a callable sub-workflow for other flows or agents.  
- Accepts two months (`YYYY-MM`) as input (e.g., `"2025-05"`, `"2025-06"`).  
- Returns CAC and ROAS values for those two months, with deltas.  

**Usage**  
- Call this workflow from another n8n workflow using *Execute Workflow*.  
- Or connect to an external AI agent.  

---

### 5. AI Query (`AI_Query.json`)
**Purpose**  
- Demonstrates how an AI agent can use the BigQuery Tool to answer questions like:  
  - "Compare CAC between May and June 2025"  
  - "What is the ROAS last month vs the previous one?"  
- Uses a system prompt that maps natural language to tool calls.  

**Setup**  
- Provide your own OpenAI/LLM credentials in n8n.  
- Adjust the system prompt to match your requirements.  

**Output**  
- JSON with CAC and ROAS comparisons, optionally formatted into text/table by the agent.  

---

## Prerequisites

- n8n (self-hosted or Desktop)
- Google Cloud project with BigQuery and GCS enabled
- Credentials in n8n for:
  - BigQuery (OAuth2 or Service Account)
  - Google Drive (for ingestion)
  - Gmail (for monthly email)

---

## Environment Configuration

Use the `.env.sample` file and fill values:
```
N8N_GCP_PROJECT_ID=your-project-id
N8N_GCS_BUCKET=your-gcs-bucket
N8N_DRIVE_FOLDER_ID=your-drive-folder-id
N8N_BQ_DATASET=ads_spend
N8N_BQ_TABLE=ads_spend-20250828
N8N_REGION=US
N8N_EMAIL_TO=you@example.com
```

Reference env vars inside workflow nodes as:  
```
{{$env.N8N_GCP_PROJECT_ID}}
{{$env.N8N_GCS_BUCKET}}
{{$env.N8N_DRIVE_FOLDER_ID}}
```

---

## Security Notes

- Project ID, bucket names, and dataset names are safe to share; credentials are not.  
- Do **not** commit exported n8n credentials.  
- Restrict access to GCS buckets and BigQuery datasets (least privilege).  
- Protect Webhook endpoints with Basic Auth, header tokens, or IP allow-lists.  

---

## License

MIT (or your preferred license)
