# klaviyo-py-orchestrator
The Concept 

**A Python CLI tool that reads your marketing CSVs, applies your custom KPI logic (Churn Risk, CPC Efficiency), and automatically creates or updates Klaviyo Segments via the API. It essentially turns your .ipynb analysis into a production-grade automation engine**

---
# Klaviyo Py-Orchestrator

> **Turn static marketing CSVs into live, revenue-generating Klaviyo segments.**

A high-performance Python engine for marketing agencies that bridges the gap between **data analysis** and **email automation**. This tool ingests raw marketing data (Web Traffic, Campaigns, Email Metrics), applies custom KPI logic, and programmatically syncs high-value user segments to Klaviyo for instant Flow activation.

Built for the modern agency stack: **Python • Pandas • Klaviyo API • Supabase • Grafana**.

![Python](https://img.shields.io/badge/Python-3.9+-blue.svg)
![Klaviyo](https://img.shields.io/badge/Klaviyo-API-violet.svg)
![License](https://img.shields.io/badge/License-MIT-green.svg)

---

##  Idea behind this

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

Churn Detector
email_open_rates.csv, users.csv
Calculates 30-day engagement decay & triggers "Win-back" segment.
Ad Efficiency
campaigns.csv, plot_cpc_data.ipynb
Flags campaigns with CPC > $X and Bounce Rate > Y%.
Behavioral Clusters
pages_visited.csv, pages_clicked.csv
Groups users by "Bargain Hunter" vs. "Premium Seeker".
Time-Series Resample
resample_the_time_series.ipynb
Smooths daily noise to predict next-week revenue.

## 🚀 Quick Start

1. Installation
bash

Copy
git clone https://github.com/natasha0824inkf/klaviyo-py-orchestrator.git
cd klaviyo-py-orchestrator
pip install -r requirements.txt
2. Configuration
Set your environment variables in a .env file:

env

Copy
KLAVIYO_API_KEY=your_secret_api_key
KLAVIYO_PRIVATE_API_KEY=your_private_key
DATA_DIR=./data
OUTPUT_DIR=./output
3. Run the Engine
Execute the main pipeline:

bash

Copy
# Analyze data and sync segments
python main.py --mode sync --segments churn,high_aov

# Generate a visual report only
python main.py --mode report --format pdf

## 🔧 Key Features

### 🧠 Dynamic Segment Creation
Leverages the Klaviyo Segments API to build complex condition groups automatically.

Example: "Users who visited /pricing in the last 7 days, have an email open rate < 10%, and are located in the EU."
Code: Uses profile-property, profile-metric, and profile-region condition types dynamically.

### 📊 Custom KPI Calculation

Integrates your custom Jupyter logic into a production script:

calculate_ctr.ipynb → Real-time CTR alerts.
handle_outliers.ipynb → Bot traffic filtering before segment creation.
create_a_rolling_average_plot.ipynb → Trend smoothing for decision making.

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

Example: Creating a "High Risk Churn" Segment The tool constructs a JSON definition combining:

profile-metric: "Fulfilled Order" count < 1 in last 90 days.
profile-property: "Email Open Rate" < 5%.
profile-group-membership: Not in "VIP List".
python

Copy
# Pseudo-code from src/klaviyo_connector.py
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

## 🏢 Multi-Tenant Klaviyo Setup

This section covers deploying the orchestrator for agencies managing multiple Klaviyo clients via Supabase Edge Functions and Grafana.

### Where to Start

Follow these steps in order to get from zero to a live multi-client dashboard.

---

### 1. Environment Variables (`.env`)

Create a file named `.env` in your project root with your actual credentials:

```env
# .env
SUPABASE_URL="https://your-project-id.supabase.co"
SUPABASE_SERVICE_ROLE_KEY="your-super-secret-service-role-key"

KLAVIYO_API_KEY="your-klaviyo-api-key"
KLAVIYO_API_VERSION="2024-10-15" # Check Klaviyo docs for latest version

# Cron schedule (Supabase uses standard cron syntax)
# Runs every day at 2:00 AM UTC
CRON_SCHEDULE="0 2 * * *"
```

---

### 2. Database Schema (`supabase/migrations/001_init_klaviyo_tables.sql`)

Run this in your Supabase SQL Editor. It creates the raw storage table and the optimized view for Grafana.

```sql
-- 1. Create the raw metrics table
CREATE TABLE IF NOT EXISTS daily_client_metrics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  client_id TEXT NOT NULL,
  client_name TEXT,
  date DATE NOT NULL,
  metrics JSONB NOT NULL, -- Stores the full normalized JSON payload
  revenue NUMERIC DEFAULT 0,
  orders_count INT DEFAULT 0,
  unsubscribes INT DEFAULT 0,
  opens INT DEFAULT 0,
  clicks INT DEFAULT 0,
  UNIQUE(client_id, date)
);

-- 2. Create an index for fast Grafana filtering
CREATE INDEX IF NOT EXISTS idx_metrics_date_client 
ON daily_client_metrics(date, client_id);

-- 3. Create a SQL View for Grafana (Aggregated & Cleaned)
CREATE OR REPLACE VIEW grafana_klaviyo_dashboard AS
SELECT 
  date,
  client_id,
  client_name,
  revenue,
  orders_count,
  (revenue / NULLIF(orders_count, 0)) AS avg_order_value,
  unsubscribes,
  opens,
  clicks,
  (clicks::float / NULLIF(opens, 0)) AS click_through_rate,
  (unsubscribes::float / NULLIF(opens, 0)) AS unsubscribe_rate
FROM daily_client_metrics
ORDER BY date DESC, client_id;

-- 4. Enable Row Level Security (Optional but recommended)
ALTER TABLE daily_client_metrics ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Allow service role full access" ON daily_client_metrics
  FOR ALL USING (auth.uid() IS NOT NULL)
  WITH CHECK (auth.uid() IS NOT NULL);
```

---

### 3. Supabase Edge Function (`supabase/functions/sync-klaviyo/index.ts`)

Create this file via the Supabase CLI or Dashboard. It handles the ETL logic.

```typescript
// supabase/functions/sync-klaviyo/index.ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

serve(async (req) => {
  const supabaseUrl = Deno.env.get("SUPABASE_URL")!;
  const supabaseKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
  const supabase = createClient(supabaseUrl, supabaseKey);

  const apiKey = Deno.env.get("KLAVIYO_API_KEY");
  if (!apiKey) return new Response("Missing API Key", { status: 500 });

  try {
    const response = await fetch(
      `https://a.klaviyo.com/api/metrics?filter[client_id]=your-client-id`,
      {
        headers: {
          "Authorization": `Klaviyo-API-Key ${apiKey}`,
          "revision": Deno.env.get("KLAVIYO_API_VERSION") || "2024-10-15"
        }
      }
    );

    if (!response.ok) throw new Error(`Klaviyo API Error: ${response.statusText}`);

    const data = await response.json();

    const records = data.data.map((item: any) => ({
      client_id: item.attributes.client_id,
      client_name: item.attributes.name,
      date: new Date().toISOString().split('T')[0],
      metrics: item.attributes,
      revenue: item.attributes.revenue || 0,
      orders_count: item.attributes.orders_count || 0,
      unsubscribes: item.attributes.unsubscribes || 0,
      opens: item.attributes.opens || 0,
      clicks: item.attributes.clicks || 0,
    }));

    const { error } = await supabase
      .from('daily_client_metrics')
      .upsert(records, { onConflict: 'client_id,date' });

    if (error) throw error;

    return new Response(JSON.stringify({ success: true, records: records.length }), {
      headers: { "Content-Type": "application/json" },
    });

  } catch (error) {
    console.error(error);
    return new Response(JSON.stringify({ error: error.message }), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    });
  }
});
```

---

### 4. Cron Job Configuration (`supabase/config.toml`)

```toml
[functions.sync-klaviyo]
verify_jwt = false
```

> **Note:** For production scheduling, use a GitHub Action that hits your Edge Function URL once a day, or enable the `pg_cron` extension in your Supabase Dashboard under **Database Extensions**.

---

### 5. Grafana Data Source Setup

1. **Add Data Source**: Select **PostgreSQL**.
2. **Connection Details:**
   - Host: `db.your-project-id.supabase.co`
   - Port: `5432`
   - Database: `postgres`
   - User: `postgres`
   - Password: Your Supabase DB password (not the API key).
3. **Query:** Select `grafana_klaviyo_dashboard` from the table dropdown. Use filters for `client_id` and `date`.

---

### How to Deploy

```bash
# 1. Initialize
npx supabase init

# 2. Link to your project
npx supabase link --project-ref your-project-id

# 3. Run migrations
npx supabase db push

# 4. Deploy the Edge Function
npx supabase functions deploy sync-klaviyo --env-file .env

# 5. Test — trigger manually via the Supabase Dashboard to verify data flow
```

Once data is flowing, build your Grafana dashboards using the `grafana_klaviyo_dashboard` view.

---

## 📈 Roadmap

- v1.1: Add Supabase integration for long-term storage of analytics_data_updated.csv.
- v1.2: Integrate seo-gtm-brain for content gap analysis.
- v1.3: Real-time Grafana dashboard auto-generation.

## 🤝 Contributing

Pull requests are welcome! If you have a new KPI metric or a better way to handle handle_missing_data.ipynb logic, open an issue or submit a PR.

Data Architecture Plan

Here is the complete data architecture plan.

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
