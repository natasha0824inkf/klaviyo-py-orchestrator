# klaviyo-py-orchestrator
The Concept 

A Python CLI tool that reads your marketing CSVs, applies your custom KPI logic (Churn Risk, CPC Efficiency), and automatically creates or updates Klaviyo Segments via the API. It essentially turns your .ipynb analysis into a production-grade automation engine
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
---
📦 Included Data Modules
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
🚀 Quick Start
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
🔧 Key Features
🧠 Dynamic Segment Creation
Leverages the Klaviyo Segments API to build complex condition groups automatically.

Example: "Users who visited /pricing in the last 7 days, have an email open rate < 10%, and are located in the EU."
Code: Uses profile-property, profile-metric, and profile-region condition types dynamically.
📊 Custom KPI Calculation
Integrates your custom Jupyter logic into a production script:

calculate_ctr.ipynb → Real-time CTR alerts.
handle_outliers.ipynb → Bot traffic filtering before segment creation.
create_a_rolling_average_plot.ipynb → Trend smoothing for decision making.
🤖 Automated Alerts
If a campaign's CPC spikes by >20% (detected via plot_cpc_data.ipynb logic), the tool triggers a webhook to Slack or sends a personalized alert email via send_personalized_emails.ipynb.

📂 Project Structure
text

Copy
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
🧪 API Integration Details
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
📈 Roadmap
v1.1: Add Supabase integration for long-term storage of analytics_data_updated.csv.
v1.2: Integrate seo-gtm-brain for content gap analysis.
v1.3: Real-time Grafana dashboard auto-generation.
🤝 Contributing
Pull requests are welcome! If you have a new KPI metric or a better way to handle handle_missing_data.ipynb logic, open an issue or submit a PR.

See CONTRIBUTING.md for setup instructions.
