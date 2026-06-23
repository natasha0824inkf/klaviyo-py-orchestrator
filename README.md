# Klaviyo Py-Orchestrator

> **Turn static marketing CSVs into live, revenue-generating Klaviyo segments.**

A high-performance Python engine for marketing agencies that bridges the gap between **data analysis** and **email automation**. This tool ingests raw marketing data (Web Traffic, Campaigns, Email Metrics), applies custom KPI logic, and programmatically syncs high-value user segments to Klaviyo for instant Flow activation.

Built for the modern agency stack: **Python • Pandas • Klaviyo API • Supabase • Grafana**.

![Python](https://img.shields.io/badge/Python-3.9+-blue.svg)
![Klaviyo](https://img.shields.io/badge/Klaviyo-API-violet.svg)
![License](https://img.shields.io/badge/License-MIT-green.svg)

---

## 💡 Idea Behind This

Most agencies analyze data in silos, eg:
1.  Export CSVs from Ads/Website.
2.  Run Python scripts locally (Jupyter Notebooks).
3.  Manually copy User IDs into Klaviyo.

**The Orchestrator automates this loop.** It takes your `marketing_data.csv`, `email_open_rates.csv`, and `pages_visited.csv`, calculates a **Churn Risk Score** or **High-Value Customer** metric, and instantly updates a Klaviyo Segment definition via the API.

## 🏗️ Architecture

```mermaid
graph LR
    A[Raw CSVs] --> B(Python ETL Engine)
    B --> C{KPI Logic}
    C -->|Churn Risk| D[Klaviyo API]
    C -->|High AOV| D
    C -->|CPC Spike| E[Slack Alert]
    D --> F[Live Klaviyo Segments]
    F --> G[Automated Flows]
```

## 📦 Included Data Modules

This repo is pre-configured to handle the datasets from the Business Intelligence - Python for Marketing task:

| Module | Files | What it does |
|--------|-------|--------------|
| **Churn Detector** | `email_open_rates.csv`, `users.csv` | Calculates 30-day engagement decay, triggers "Win-back" segment |
| **Ad Efficiency** | `campaigns.csv`, `plot_cpc_data.ipynb` | Flags campaigns with CPC > threshold and Bounce Rate > threshold |
| **Behavioral Clusters** | `pages_visited.csv`, `pages_clicked.csv` | Groups users by "Bargain Hunter" vs "Premium Seeker" |
| **Time-Series Resample** | `resample_the_time_series.ipynb` | Smooths daily noise to predict next-week revenue |

## 🚀 Quick Start

**1. Installation**

```bash
git clone https://github.com/natasha0824inkf/klaviyo-py-orchestrator.git
cd klaviyo-py-orchestrator
pip install -r requirements.txt
```

**2. Configuration**

Create a `.env` file in the project root:

```env
KLAVIYO_API_KEY=your_secret_api_key
KLAVIYO_PRIVATE_API_KEY=your_private_key
DATA_DIR=./data
OUTPUT_DIR=./output
```

**3. Run the Engine**

```bash
# Analyze data and sync segments
python main.py --mode sync --segments churn,high_aov

# Generate a visual report only
python main.py --mode report --format pdf
```

## 🔧 Key Features

### 🧠 Dynamic Segment Creation
Leverages the Klaviyo Segments API to build complex condition groups automatically.

Example: "Users who visited /pricing in the last 7 days, have an email open rate < 10%, and are located in the EU."
Code: Uses profile-property, profile-metric, and profile-region condition types dynamically.

### 📊 Custom KPI Calculation

Integrates your custom Jupyter logic into a production script:

- `calculate_ctr.ipynb` → Real-time CTR alerts
- `handle_outliers.ipynb` → Bot traffic filtering before segment creation
- `create_a_rolling_average_plot.ipynb` → Trend smoothing for decision making

### 🤖 Automated Alerts

If a campaign's CPC spikes by >20% (detected via plot_cpc_data.ipynb logic), the tool triggers a webhook to Slack or sends a personalized alert email via send_personalized_emails.ipynb.

## 📂 Project Structure

```
klaviyo-py-orchestrator/
├── data/
│   ├── marketing_data.csv
│   ├── email_open_rates.csv
│   └── pages_visited.csv
├── src/
│   ├── etl.py              # Load & clean CSVs (clean_marketing_data.ipynb logic)
│   ├── kpi_engine.py       # Calculate CTR, Bounce, Churn (calculate_kpi.ipynb)
│   ├── klaviyo_connector.py # API wrapper for Segments API
│   └── alerts.py           # Slack/Email notifications
├── notebooks/
│   ├── analyze_trends.ipynb
│   └── visualize_heatmap.ipynb
├── tests/
├── .env.example
├── main.py
└── README.md
```

## 🧪 API Integration Details

This tool uses the Klaviyo Segments API to create dynamic definitions.

**Example: Creating a "High Risk Churn" Segment**

The tool constructs a JSON payload combining:
- `profile-metric`: "Fulfilled Order" count < 1 in last 90 days
- `profile-property`: "Email Open Rate" < 5%
- `profile-group-membership`: Not in "VIP List"

```python
# src/klaviyo_connector.py
segment_definition = {
    "type": "segment",
    "attributes": {
        "name": "Churn Risk - Q3 2026",
        "definition": {
            "condition_groups": [
                {
                    "conditions": [
                        { "type": "profile-metric", "field": "Fulfilled Order", "operator": "less-than", "value": 1 }
                    ],
                    "logic": "OR"
                },
                {
                    "conditions": [
                        { "type": "profile-property", "field": "Email Open Rate", "operator": "less-than", "value": 0.05 }
                    ],
                    "logic": "AND"
                }
            ]
        }
    }
}
```

## 🏢 Multi-Tenant Klaviyo Sync (Production)

Real multi-tenant setup for agencies. Each client has their own Klaviyo API key stored in Supabase. The sync function loops through all clients, fetches their metrics, and stores them with proper isolation.

### Where to Start

1. **Database & clients table** — run the migration once
2. **Clients config** — add your agency's clients to the `klaviyo_clients` table
3. **Edge Function** — deploy the sync function that loops through all clients
4. **Cron or GitHub Action** — schedule the sync to run daily

---

### 1. Database Schema (`supabase/migrations/001_init_klaviyo_tables.sql`)

Run this once in your Supabase SQL Editor.

```sql
-- Clients table: one row per agency client
CREATE TABLE IF NOT EXISTS klaviyo_clients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_id TEXT NOT NULL UNIQUE,
  client_name TEXT NOT NULL,
  api_key TEXT NOT NULL, -- Encrypted in production
  sync_status TEXT DEFAULT 'pending', -- pending, syncing, success, failed
  last_synced_at TIMESTAMPTZ,
  error_message TEXT,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Metrics: raw daily data per client
CREATE TABLE IF NOT EXISTS daily_client_metrics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_id TEXT NOT NULL REFERENCES klaviyo_clients(client_id) ON DELETE CASCADE,
  date DATE NOT NULL,
  
  -- Raw Klaviyo metrics
  revenue NUMERIC DEFAULT 0,
  orders_count INT DEFAULT 0,
  unsubscribes INT DEFAULT 0,
  opens INT DEFAULT 0,
  clicks INT DEFAULT 0,
  email_sent INT DEFAULT 0,
  bounces INT DEFAULT 0,
  
  -- Full payload for debugging
  raw_payload JSONB,
  
  synced_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(client_id, date)
);

-- Sync log for debugging
CREATE TABLE IF NOT EXISTS sync_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_id TEXT NOT NULL REFERENCES klaviyo_clients(client_id) ON DELETE CASCADE,
  sync_start TIMESTAMPTZ,
  sync_end TIMESTAMPTZ,
  status TEXT, -- success, failed, partial
  records_synced INT DEFAULT 0,
  error_details JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Grafana dashboard view: aggregated, clean
CREATE OR REPLACE VIEW grafana_daily_metrics AS
SELECT 
  date,
  client_id,
  (SELECT client_name FROM klaviyo_clients WHERE client_id = m.client_id) AS client_name,
  revenue,
  orders_count,
  ROUND((revenue / NULLIF(orders_count, 0))::NUMERIC, 2) AS avg_order_value,
  unsubscribes,
  opens,
  clicks,
  email_sent,
  bounces,
  ROUND((clicks::NUMERIC / NULLIF(opens, 0) * 100), 2) AS click_rate_pct,
  ROUND((opens::NUMERIC / NULLIF(email_sent, 0) * 100), 2) AS open_rate_pct,
  ROUND((bounces::NUMERIC / NULLIF(email_sent, 0) * 100), 2) AS bounce_rate_pct,
  ROUND((unsubscribes::NUMERIC / NULLIF(opens, 0) * 100), 2) AS unsub_rate_pct
FROM daily_client_metrics m
ORDER BY date DESC, client_id;

-- Indexes
CREATE INDEX IF NOT EXISTS idx_metrics_client_date ON daily_client_metrics(client_id, date);
CREATE INDEX IF NOT EXISTS idx_sync_log_client ON sync_log(client_id, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_clients_status ON klaviyo_clients(sync_status);
```

---

### 2. Add Your Clients

Insert via the Supabase Dashboard or SQL:

```sql
INSERT INTO klaviyo_clients (client_id, client_name, api_key) VALUES
('client_001', 'Acme Corp', 'pk_YourRealKlaviyoKeyHere'),
('client_002', 'Widget LLC', 'pk_AnotherClientKeyHere'),
('client_003', 'Premium Brand Co', 'pk_YetAnotherKeyHere');
```

**Production:** Encrypt the `api_key` column using Supabase Secrets or a key management service.

---

### 3. Edge Function (`supabase/functions/sync-klaviyo/index.ts`)

This function:
- Loops through all active clients
- Fetches metrics from Klaviyo for each (with retry logic)
- Stores results with isolation per client
- Logs sync status for debugging

```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const KLAVIYO_API_URL = "https://a.klaviyo.com/api/v1";
const API_VERSION = "2024-10-15";
const MAX_RETRIES = 3;
const RETRY_DELAY_MS = 1000;

interface Client {
  client_id: string;
  client_name: string;
  api_key: string;
}

async function fetchWithRetry(
  url: string,
  options: RequestInit,
  retries = 0
): Promise<Response> {
  try {
    const resp = await fetch(url, options);
    if (resp.status === 429) {
      if (retries < MAX_RETRIES) {
        const wait = RETRY_DELAY_MS * Math.pow(2, retries);
        await new Promise((r) => setTimeout(r, wait));
        return fetchWithRetry(url, options, retries + 1);
      }
    }
    return resp;
  } catch (e) {
    if (retries < MAX_RETRIES) {
      await new Promise((r) => setTimeout(r, RETRY_DELAY_MS * Math.pow(2, retries)));
      return fetchWithRetry(url, options, retries + 1);
    }
    throw e;
  }
}

async function syncClientMetrics(
  client: Client,
  supabase: any,
  yesterday: string
): Promise<{ success: boolean; records: number; error?: string }> {
  try {
    // Fetch from Klaviyo's metrics endpoint
    // Note: adjust endpoint based on your actual Klaviyo API structure
    const metricsUrl = `${KLAVIYO_API_URL}/metrics?since=${yesterday}&until=${yesterday}`;
    
    const resp = await fetchWithRetry(metricsUrl, {
      headers: {
        "Authorization": `Klaviyo-API-Key ${client.api_key}`,
        "revision": API_VERSION,
      },
    });

    if (!resp.ok) {
      const error = await resp.text();
      throw new Error(`Klaviyo API ${resp.status}: ${error}`);
    }

    const data = await resp.json();
    if (!data.data || data.data.length === 0) {
      return { success: true, records: 0 };
    }

    // Transform and batch insert
    const records = data.data.map((item: any) => ({
      client_id: client.client_id,
      date: yesterday,
      revenue: parseFloat(item.revenue) || 0,
      orders_count: parseInt(item.orders_count) || 0,
      unsubscribes: parseInt(item.unsubscribes) || 0,
      opens: parseInt(item.opens) || 0,
      clicks: parseInt(item.clicks) || 0,
      email_sent: parseInt(item.email_sent) || 0,
      bounces: parseInt(item.bounces) || 0,
      raw_payload: item,
    }));

    // Upsert with conflict handling
    const { error: upsertError } = await supabase
      .from("daily_client_metrics")
      .upsert(records, { onConflict: "client_id,date" });

    if (upsertError) throw upsertError;

    return { success: true, records: records.length };
  } catch (error) {
    return {
      success: false,
      records: 0,
      error: error instanceof Error ? error.message : String(error),
    };
  }
}

serve(async (req) => {
  const supabaseUrl = Deno.env.get("SUPABASE_URL")!;
  const supabaseKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
  const supabase = createClient(supabaseUrl, supabaseKey);

  const syncStart = new Date();
  const yesterday = new Date();
  yesterday.setDate(yesterday.getDate() - 1);
  const dateStr = yesterday.toISOString().split("T")[0];

  try {
    // Fetch all active clients
    const { data: clients, error: clientsError } = await supabase
      .from("klaviyo_clients")
      .select("client_id,client_name,api_key")
      .eq("sync_status", "pending");

    if (clientsError) throw clientsError;
    if (!clients || clients.length === 0) {
      return new Response(
        JSON.stringify({ message: "No clients to sync", total: 0 }),
        { headers: { "Content-Type": "application/json" } }
      );
    }

    // Sync each client
    const results: any[] = [];
    for (const client of clients as Client[]) {
      const result = await syncClientMetrics(client, supabase, dateStr);
      results.push({ client_id: client.client_id, ...result });

      // Log to sync_log table
      const syncEnd = new Date();
      await supabase.from("sync_log").insert({
        client_id: client.client_id,
        sync_start: syncStart,
        sync_end: syncEnd,
        status: result.success ? "success" : "failed",
        records_synced: result.records,
        error_details: result.error ? { message: result.error } : null,
      });

      // Update client status
      await supabase
        .from("klaviyo_clients")
        .update({
          sync_status: result.success ? "success" : "failed",
          last_synced_at: new Date(),
          error_message: result.error || null,
        })
        .eq("client_id", client.client_id);
    }

    const totalRecords = results.reduce((sum, r) => sum + r.records, 0);
    return new Response(JSON.stringify({ success: true, results, totalRecords }), {
      headers: { "Content-Type": "application/json" },
    });
  } catch (error) {
    console.error("Sync failed:", error);
    return new Response(
      JSON.stringify({
        success: false,
        error: error instanceof Error ? error.message : String(error),
      }),
      { status: 500, headers: { "Content-Type": "application/json" } }
    );
  }
});
```

---

### 4. Cron Trigger (GitHub Action)

Create `.github/workflows/sync-klaviyo.yml`:

```yaml
name: Daily Klaviyo Sync

on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM UTC daily
  workflow_dispatch:      # Manual trigger

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Supabase sync function
        run: |
          curl -X POST \
            https://${{ secrets.SUPABASE_PROJECT_ID }}.supabase.co/functions/v1/sync-klaviyo \
            -H "Authorization: Bearer ${{ secrets.SUPABASE_ANON_KEY }}" \
            -H "Content-Type: application/json" \
            -d '{}' \
            --fail-with-body
```

Add secrets to GitHub: `SUPABASE_PROJECT_ID` and `SUPABASE_ANON_KEY`.

---

### 5. Grafana Setup

1. **PostgreSQL Data Source:**
   - Host: `db.your-project-id.supabase.co:5432`
   - Database: `postgres`
   - User: `postgres` (or read-only user)
   - SSL: Require

2. **Sample Grafana Query:**
   ```sql
   SELECT 
     date,
     client_name,
     revenue,
     avg_order_value,
     click_rate_pct,
     open_rate_pct
   FROM grafana_daily_metrics
   WHERE date >= now()::date - 30
   ORDER BY date DESC
   ```

3. **Panels to build:** Revenue trend, AOV by client, engagement rates, unsubscribe tracking.

**Performance note:** The `grafana_daily_metrics` view uses a subquery for `client_name`. This is fine for <100 clients. For larger scale, optimize the view to JOIN `klaviyo_clients` instead:

```sql
CREATE OR REPLACE VIEW grafana_daily_metrics AS
SELECT 
  m.date, m.client_id, c.client_name,
  m.revenue, m.orders_count,
  ROUND((m.revenue / NULLIF(m.orders_count, 0))::NUMERIC, 2) AS avg_order_value,
  ... (rest of metrics)
FROM daily_client_metrics m
JOIN klaviyo_clients c USING (client_id)
ORDER BY m.date DESC, m.client_id;
```

---

### Deploy

```bash
npx supabase init
npx supabase link --project-ref your-project-id
npx supabase db push
npx supabase functions deploy sync-klaviyo
npx supabase secrets set KLAVIYO_API_VERSION=2024-10-15
```

Test manually in the Supabase Dashboard under **Functions** → **sync-klaviyo** → **Invoke**.

---

## 💡 Alternative: Drop Supabase, Use Neon + Python

If you already have data in Supabase and want to keep it, ignore this. But if you're starting fresh or frustrated with Supabase overhead — this stack is simpler for the Klaviyo → Grafana flow.

**The idea:** Supabase is overkill here. You're not using auth, realtime, or storage — just Postgres + a cron job. Neon gives you serverless Postgres with a plain connection string and nothing else to manage.

```
Klaviyo API → Python script (GitHub Action) → Neon Postgres → Grafana
```

**Why it's cleaner:**
- No SDK, no edge functions, no Supabase dashboard to fight
- Same schema, same Grafana queries — just a different `DATABASE_URL`
- Python script replaces the TypeScript Edge Function (you already have the logic in this repo)
- GitHub Action handles the cron (same as the Supabase approach anyway)

**To migrate:** Change `DATABASE_URL` in your `.env` to your Neon connection string. The schema and Python sync script work as-is against any Postgres.

> **Keep Supabase if:** you already have clients or users data there and don't want to migrate. The tables in the Multi-Tenant section above were designed to coexist with existing Supabase tables.

---

## 📈 Roadmap

- v1.1: Add Supabase integration for long-term storage of analytics_data_updated.csv.
- v1.2: Integrate seo-gtm-brain for content gap analysis.
- v1.3: Real-time Grafana dashboard auto-generation.

## 🤝 Contributing

Pull requests are welcome! If you have a new KPI metric or a better way to handle handle_missing_data.ipynb logic, open an issue or submit a PR.

## 🗄️ Data Architecture Plan

### 1. Database Schema (Supabase/PostgreSQL)

```sql
CREATE TABLE users (
  user_id TEXT PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  first_name TEXT,
  last_name TEXT,
  country TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  total_orders INT DEFAULT 0,
  total_revenue NUMERIC DEFAULT 0,
  avg_order_value NUMERIC,
  last_open_date DATE,
  last_click_date DATE,
  churn_risk_score FLOAT DEFAULT 0.0,
  segment_tags TEXT[]
);

CREATE TABLE campaigns (
  campaign_id TEXT PRIMARY KEY,
  name TEXT,
  channel TEXT,
  status TEXT,
  budget NUMERIC,
  spent NUMERIC,
  impressions INT,
  clicks INT,
  ctr FLOAT DEFAULT 0.0,
  cpc NUMERIC DEFAULT 0.0,
  last_updated TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE user_events (
  event_id BIGSERIAL PRIMARY KEY,
  user_id TEXT REFERENCES users(user_id),
  event_type TEXT,
  event_data JSONB,
  occurred_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_user_events_time ON user_events (user_id, occurred_at);

CREATE TABLE klaviyo_sync_log (
  log_id BIGSERIAL PRIMARY KEY,
  segment_name TEXT,
  sync_status TEXT,
  records_synced INT,
  error_message TEXT,
  synced_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE VIEW v_churn_risk AS
SELECT 
  user_id,
  CASE 
    WHEN last_open_date < NOW() - INTERVAL '30 days' THEN 0.9
    WHEN last_click_date < NOW() - INTERVAL '14 days' THEN 0.7
    WHEN avg_order_value < 10 THEN 0.5
    ELSE 0.1
  END as risk_score
FROM users;
```

### 2. Python Orchestration Script

```python
import pandas as pd
import psycopg2
import requests
import os
from datetime import datetime

SUPABASE_URL = os.getenv("SUPABASE_URL")
SUPABASE_KEY = os.getenv("SUPABASE_KEY")
KLAVIYO_API_KEY = os.getenv("KLAVIYO_API_KEY")
KLAVIYO_API_URL = "https://a.klaviyo.com/api/v2"

def load_data_to_supabase(df, table_name, conn):
    cursor = conn.cursor()
    for _, row in df.iterrows():
        columns = ', '.join(row.index)
        placeholders = ', '.join(['%s'] * len(row))
        values = list(row)
        sql = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders}) ON CONFLICT (user_id) DO UPDATE SET {', '.join([f'{k}=EXCLUDED.{k}' for k in row.index if k != 'user_id'])}"
        try:
            cursor.execute(sql, values)
        except Exception as e:
            print(f"Error inserting row: {e}")
    conn.commit()
    cursor.close()

def sync_churn_segment(conn):
    cursor = conn.cursor()
    cursor.execute("SELECT user_id FROM v_churn_risk WHERE risk_score > 0.7")
    users = [row[0] for row in cursor.fetchall()]
    cursor.close()

    if not users:
        return

    payload = {
        "data": {
            "type": "segment",
            "attributes": {
                "name": f"High Churn Risk - {datetime.now().strftime('%Y-%m-%d')}",
                "definition": {
                    "condition_groups": [
                        {
                            "conditions": [
                                {"type": "profile-property", "field": "user_id", "operator": "in", "value": users}
                            ],
                            "logic": "OR"
                        }
                    ]
                }
            }
        }
    }

    headers = {
        "Authorization": f"Klaviyo-API-Key {KLAVIYO_API_KEY}",
        "Content-Type": "application/json",
        "Klaviyo-Version": "2024-08-15"
    }

    response = requests.post(f"{KLAVIYO_API_URL}/segment", json=payload, headers=headers)
    
    log_entry = {
        "segment_name": "High Churn Risk",
        "sync_status": "success" if response.status_code == 201 else "failed",
        "records_synced": len(users),
        "error_message": response.text if response.status_code != 201 else None
    }

    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO klaviyo_sync_log (segment_name, sync_status, records_synced, error_message) VALUES (%s, %s, %s, %s)",
        (log_entry["segment_name"], log_entry["sync_status"], log_entry["records_synced"], log_entry["error_message"])
    )
    conn.commit()
    cursor.close()

def main():
    conn = psycopg2.connect(os.getenv("DATABASE_URL"))
    
    # Load Users
    df_users = pd.read_csv("data/email_open_rates.csv")
    load_data_to_supabase(df_users, "users", conn)

    # Load Campaigns
    df_campaigns = pd.read_csv("data/campaigns.csv")
    load_data_to_supabase(df_campaigns, "campaigns", conn)

    # Sync Segments
    sync_churn_segment(conn)
    
    conn.close()

if __name__ == "__main__":
    main()
```

### 3. Grafana Dashboard Queries

**Panel 1: Churn Risk Count**
```sql
SELECT count(*) as churn_count
FROM users
WHERE churn_risk_score > 0.7;
```

**Panel 2: CPC Trend (7-Day Rolling Average)**
```sql
SELECT 
  DATE_TRUNC('day', last_updated) as day,
  AVG(cpc) OVER (ORDER BY DATE_TRUNC('day', last_updated) ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as rolling_cpc
FROM campaigns
WHERE channel = 'facebook';
```

**Panel 3: Email Open Rate by Day**
```sql
SELECT 
  DATE_TRUNC('day', occurred_at) as day,
  count(*) as opens
FROM user_events
WHERE event_type = 'email_open'
GROUP BY day
ORDER BY day;
```

### 4. Environment Variables (.env)

```env
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-anon-key
DATABASE_URL=postgresql://postgres:password@db.your-project.supabase.co:5432/postgres
KLAVIYO_API_KEY=your-klaviyo-private-api-key
```

See CONTRIBUTING.md for setup instructions.
