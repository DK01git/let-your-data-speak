# 🏎️ Let Your Data Speak with Microsoft Azure
### A Hands-On Session using F1 Race Data, Azure AI Foundry & Microsoft Fabric

![Azure](https://img.shields.io/badge/Azure-AI%20Foundry-0078D4?style=flat&logo=microsoftazure)
![Fabric](https://img.shields.io/badge/Microsoft-Fabric-F5B800?style=flat&logo=microsoft)
![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=flat&logo=python)
![License](https://img.shields.io/badge/License-MIT-green)

---

## 📌 Session Overview

In this session, you will build an end-to-end AI-powered data pipeline that extracts live F1 race data, stores it in Azure Blob Storage, and lets you query it in natural language using an Azure AI Foundry Agent — all orchestrated through Microsoft Fabric.

**By the end of this session, you will be able to:**
- Extract live F1 data from the OpenF1 API using a Fabric Notebook
- Store structured parquet files in Azure Blob Storage
- Create an AI Foundry Agent grounded on your real data
- Ask natural language questions and generate Python plots on the fly using VS Code

---

## 🏗️ Architecture

```
OpenF1 API (live F1 data)
        │
        ▼
Microsoft Fabric Notebook
(PySpark / Pandas — weekly scheduled pipeline)
        │
        ▼
Azure Blob Storage
(drivers.parquet / laps.parquet / race_results.parquet)
        │
        ├──── Fabric Shortcut (read raw data back into Fabric)
        │
        ▼
Azure AI Foundry
  ├── Project
  ├── Deployed LLM (e.g. GPT-4o)
  └── Agent (grounded on Blob parquet files)
        │
        ▼
VS Code + GitHub Copilot (Agent Mode)
(natural language → insights → Python scripts → plots)
```

---

## 📋 Prerequisites

Before starting, make sure you have the following:

| Requirement | Details |
|---|---|
| Azure Subscription | Free trial works: [https://azure.microsoft.com/free](https://azure.microsoft.com/free) |
| Microsoft Fabric Trial | Enable at [https://fabric.microsoft.com](https://fabric.microsoft.com) |
| VS Code | Latest version: [https://code.visualstudio.com](https://code.visualstudio.com) |
| Python | 3.10 or later |
| GitHub Copilot | Active subscription or trial |
| VS Code Extensions | Azure Resources, Azure AI Foundry, Azure MCP Server (install in Step 3) |

---

## 📁 Repository Structure

```
📦 let-your-data-speak/
├── 📂 notebooks/
│   └── F1data.ipynb              # Fabric notebook — extracts F1 data and uploads to Blob
├── 📂 scripts/
│   └── query_agent.py            # Sample Python script to query the agent programmatically
├── 📂 sample_prompts/
│   └── prompts.md                # Curated prompts to try with the agent
├── 📂 architecture/
│   └── architecture_diagram.png  # End-to-end architecture diagram
├── .env.example                  # Environment variable template
├── requirements.txt              # Python dependencies
└── README.md                     # This file
```

---

## 🔑 Step 1 — Create Azure Resources

### 1.1 Create a Resource Group

1. Go to [https://portal.azure.com](https://portal.azure.com)
2. Search for **Resource groups** → Click **+ Create**
3. Fill in:
   - **Subscription:** Your active subscription
   - **Resource group name:** `rg-f1-session` (or any name you prefer)
   - **Region:** `East US` (recommended for AI Foundry availability)
4. Click **Review + Create** → **Create**

---

### 1.2 Create an Azure Key Vault

You will store sensitive credentials (storage connection string, container name) in Key Vault so they never appear as plain text in your notebook.

1. In the Azure Portal, search for **Key vaults** → Click **+ Create**
2. Fill in:
   - **Resource group:** `rg-f1-session`
   - **Key vault name:** `kv-f1-session` (must be globally unique)
   - **Region:** Same as your resource group
   - **Pricing tier:** Standard
3. Click **Review + Create** → **Create**

---

### 1.3 Create an Azure Storage Account

1. Search for **Storage accounts** → Click **+ Create**
2. Fill in:
   - **Resource group:** `rg-f1-session`
   - **Storage account name:** `champdemo5954940607` (lowercase, no hyphens, globally unique)
   - **Region:** Same as your resource group
   - **Performance:** Standard
   - **Redundancy:** LRS (Locally Redundant Storage)
3. Click **Review + Create** → **Create**
4. Once created, go to the storage account → **Containers** → **+ Container**
   - **Name:** `f1data`
   - **Public access level:** Private

---

### 1.4 Store Secrets in Key Vault

You need two secrets: the storage **connection string** and the **container name**.

**Grant yourself permission to create secrets (do this first):**

Azure Key Vault uses RBAC by default. Even if you created the Key Vault, you cannot create or read secrets unless you are explicitly assigned a role.

1. Go to your Key Vault → **Access control (IAM)** (left sidebar)
2. Click **+ Add** → **Add role assignment**
3. Under **Role** tab → search for **Key Vault Administrator** → select it → click **Next**
4. Under **Members** tab:
   - **Assign access to:** User, group, or service principal
   - Click **+ Select members** → search for your own Azure account (email) → select it
5. Click **Review + assign** → **Review + assign**

> ✅ Wait 1–2 minutes for the role to propagate before proceeding to create secrets.

---

**Get the Storage Connection String:**
1. Go to your Storage Account → **Security + networking** → **Access keys**
2. Click **Show** next to `key1` → Copy the **Connection string**

**Add secrets to Key Vault:**
1. Go to your Key Vault → **Objects** → **Secrets** → **+ Generate/Import**
2. Create the first secret:
   - **Name:** `storageKey`
   - **Value:** *(paste the connection string you copied)*
   - Click **Create**
3. Create the second secret:
   - **Name:** `container`
   - **Value:** `f1data`
   - Click **Create**

**Grant Fabric access to Key Vault:**
1. In Key Vault → **Access control (IAM)** → **+ Add role assignment**
2. Role: **Key Vault Secrets User**
3. Assign access to: **Managed Identity**
4. Select your Fabric workspace's managed identity
5. Click **Review + assign**

> ⚠️ Without this step, `mssparkutils.credentials.getSecret()` in the notebook will fail with a 403 error.

---

## 🧱 Step 2 — Set Up Microsoft Fabric

### 2.1 Create a Fabric Workspace

1. Go to [https://fabric.microsoft.com](https://fabric.microsoft.com)
2. Click **Workspaces** (left sidebar) → **+ New workspace**
3. Name it: `F1-Analytics`
4. Under **Advanced** → set **License mode** to Trial or Fabric capacity
5. Click **Apply**

---

### 2.2 Create a Lakehouse

1. Inside your workspace → Click **+ New item**
2. Select **Lakehouse**
3. Name it: `F1_Lakehouse`
4. Click **Create**

---

### 2.3 Create a Shortcut to Azure Blob Storage

This links your Blob Storage parquet files directly into your Fabric Lakehouse without copying data.

1. Inside `F1_Lakehouse` → **Files** section → click the **...** menu → **New shortcut**
2. Select **Azure Data Lake Storage Gen2**
3. Fill in:
   - **URL:** `https://champdemo5954940607.dfs.core.windows.net` *(replace with your storage account name)*
   - **Authentication:** Account Key *(paste your storage access key)*
4. Click **Next** → select the `f1data` container → click **Next** → **Create**

> You should now see the shortcut appear under **Files** in your Lakehouse.

---

### 2.4 Upload and Configure the Notebook

1. Inside your workspace → Click **+ New item** → **Notebook**
2. Rename it: `F1_Data_Extraction`
3. Copy and paste the code from `notebooks/F1data.ipynb` (available in this repo) into the notebook cells

The notebook does the following:

| Cell | What it does |
|---|---|
| Cell 1 | Installs required libraries (commented out — uncomment if needed) |
| Cell 2 | Imports `requests`, `pandas`, `BlobServiceClient`, `BytesIO` |
| Cell 3 | Reads `storageKey` and `container` from Azure Key Vault using `mssparkutils` |
| Cell 4 | Connects to Blob Storage and creates the container if it doesn't exist |
| Cell 5 | Defines `fetch_data()` — calls OpenF1 API with `session_key=latest` |
| Cell 6 | Defines `upload_dataframe()` — converts DataFrame to snappy-compressed parquet and uploads |
| Cell 7 | Fetches `drivers`, `laps`, and `session_result` endpoints |
| Cell 8 | Cleans `gap_to_leader` column in results (handles `"LAP"` string values) |
| Cell 9 | Uploads all three DataFrames as timestamped parquet files to Blob |

> ⚠️ **Known limitation:** The notebook uses `session_key=latest`, meaning each run captures only the most recent F1 session. Over time, weekly runs accumulate files across sessions. For cross-race queries, ensure you have enough scheduled runs or manually run it after each race weekend.

---

### 2.5 Schedule the Pipeline

1. Inside your workspace → Click **+ New item** → **Data pipeline**
2. Name it: `F1_Weekly_Refresh`
3. Click **+ Add pipeline activity** → **Notebook**
4. Configure:
   - **Notebook:** Select `F1_Data_Extraction`
   - **Workspace:** `F1-Analytics`
5. Click **Schedule** (top toolbar):
   - **Repeat:** Weekly
   - **Day:** Monday (day after most race weekends)
   - **Time:** 06:00 AM (Sri Lanka time = UTC+5:30)
6. Click **Apply** → **Save**

---

## 🤖 Step 3 — Set Up Azure AI Foundry

### 3.1 Install VS Code Extensions

Open VS Code → Extensions panel (`Ctrl+Shift+X`) → Search and install:

- **Azure Resources** (by Microsoft)
- **Azure AI Foundry** (by Microsoft)
- **Azure MCP Server** (by Microsoft) — search `ms-azuretools.vscode-azure-mcp-server`
- **GitHub Copilot for Azure** (by Microsoft) — recommended; includes and uses the Azure MCP Server, and streamlines Azure workflows directly in Copilot Chat

**Create the workspace `mcp.json` — this step is mandatory:**

Inside your project folder, create `.vscode/mcp.json` with this exact content:

```json
{
  "servers": {
    "Azure MCP Server": {
      "command": "npx",
      "args": [
        "-y",
        "@azure/mcp@latest",
        "server",
        "start"
      ]
    }
  }
}
```

> ⚠️ **Without this file, Copilot will not use Azure MCP tools inline.** It will fall back to generating terminal scripts and cannot read the output back — giving you responses like *"check the terminal for results"* instead of answering in chat.

> ⚠️ **Duplicate server warning:** If you have a global `mcp.json` at `C:\Users\<you>\AppData\Roaming\Code\User\mcp.json` that also defines an Azure MCP entry (e.g. using `dnx Azure.Mcp@2.0.0-beta.34`), remove that entry. Having two Azure MCP server definitions causes conflicting tool behavior. Keep only the workspace `.vscode/mcp.json` version above.

**Verify Azure MCP Server is active:**
1. Open GitHub Copilot Chat (`Ctrl+Shift+I`) → switch to **Agent Mode**
2. Click the 🔧 tools icon → search `storage`
3. You should see **one** `storage` entry — *"Storage operations - Commands for creating, listing, getting, and managing Azure Storage..."*
4. If you see it listed twice, you have a duplicate MCP config — follow the warning above to fix it

> ⚠️ Azure MCP Server requires you to be signed in to Azure in VS Code. If the tools don't appear, press `Ctrl+Shift+P` → **Azure: Sign In** and authenticate first, then press `Ctrl+Shift+P` → **MCP: Restart All MCP Servers**.

---

### 3.2 Create an AI Foundry Hub and Project

1. Go to [https://ai.azure.com](https://ai.azure.com)
2. Click **+ New hub**:
   - **Hub name:** `f1-ai-hub`
   - **Resource group:** `rg-f1-session`
   - **Region:** `East US`
   - **Connect Azure AI Services:** Create new or use existing
3. Click **Create**
4. Inside the hub → Click **+ New project**:
   - **Project name:** `f1-data-analyst`
5. Click **Create project**

---

### 3.3 Deploy a Model

1. Inside your project → **My assets** → **Models + endpoints** → **+ Deploy model**
2. Select **GPT-4o** (recommended) or **GPT-4o-mini** (lower cost)
3. Deployment settings:
   - **Deployment name:** `gpt4o-f1`
   - **Tokens per minute:** 10K (sufficient for demo)
4. Click **Deploy**
5. Wait until status shows **Succeeded**

---

### 3.4 Create an Azure AI Search Index (Grounding Layer)

The Foundry Agent portal does **not** support direct Azure Blob Storage as a knowledge source. The available options are **Files** (manual upload), **Azure AI Search**, and **Grounding with Bing**. The correct path to ground your agent on the parquet files in Blob Storage is through an **Azure AI Search** index.

**Create an Azure AI Search service:**
1. In the Azure Portal → search **Azure AI Search** → Click **+ Create**
2. Fill in:
   - **Resource group:** `rg-f1-session`
   - **Service name:** `srch-f1-session` (globally unique)
   - **Region:** Same as your resource group
   - **Pricing tier:** Basic (minimum required for semantic ranker)
3. Click **Review + Create** → **Create**

**Create an index from your Blob Storage:**
1. Once deployed, go to your Search service → **Overview** → Click **Import data**
2. **Data source:** Select **Azure Blob Storage**
3. Connect to your storage account `champdemo5954940607` → select container `f1data`
4. Click **Next: Add cognitive skills** → skip → Click **Next: Customize target index**
5. Set **Index name:** `f1-index`
6. Ensure key fields (`driver_number`, `lap_duration`, `position`) are marked as **Retrievable** and **Searchable**
7. Click **Next: Create an indexer** → set schedule to **Hourly** (picks up new parquet files automatically)
8. Click **Submit**

> ✅ Azure AI Search will now index your parquet files from Blob Storage and keep the index updated.

---

### 3.5 Create and Configure the Agent

1. Inside your Foundry project → **Build & customize** → **Agents** → **+ New agent**
2. Set the agent name: `F1 Data Analyst`
3. Select your deployed model: `gpt4o-f1`
4. Paste the following **System Prompt:**

```
You are an expert F1 race data analyst. You have access to three datasets:

1. drivers — driver number, name, team, country, headshot URL
2. laps — lap number, lap duration, driver number, session key, date/time
3. race_results — finishing position, driver number, points, gap to leader, session key

Your job is to answer questions about F1 race performance in a clear, analytical way.
When asked to visualize data, generate clean Python code using matplotlib or seaborn.
When asked to compare drivers, always reference specific numbers and lap times.
Always be precise. If data is unavailable, say so clearly — do not guess.
```

5. Under **Knowledge** → **+ Add** → Select **Azure AI Search**:
   - Select your search service `srch-f1-session`
   - Select index `f1-index`
   - Click **Add**

6. Click **Save**

> ✅ Your agent is now grounded on F1 data via Azure AI Search, which indexes your Blob Storage parquet files.

---

## 💻 Step 4 — Connect VS Code and Run Demos

### 4.1 Sign In to Azure in VS Code

1. Open VS Code
2. Press `Ctrl+Shift+P` → type **Azure: Sign In**
3. Follow browser authentication flow
4. In the Azure Resources panel, confirm your subscription is visible

---

### 4.2 Open the AI Foundry Extension

1. Click the AI Foundry icon in the VS Code sidebar
2. Select your hub `f1-ai-hub` → project `f1-data-analyst`
3. You should see your agent `F1 Data Analyst` listed (created in Step 3.5)

---

### 4.3 Grant Yourself Blob Storage Access for Azure MCP Server

Azure MCP Server uses your signed-in Azure identity to access resources. You must have the right role on the storage account or MCP will silently return empty results.

1. Go to your Storage Account `champdemo5954940607` → **Access control (IAM)** → **+ Add role assignment**
2. Role: **Storage Blob Data Reader** (read-only, sufficient for demos)
3. Assign to: your own Azure account
4. Click **Review + assign**

> ✅ Without this, Azure MCP Server can list container names but cannot read blob content or metadata.

---

### 4.4 Open GitHub Copilot in Agent Mode

1. Press `Ctrl+Shift+I` to open Copilot Chat
2. Click the model selector dropdown → select **Agent mode**
3. Click the 🔧 tools icon → confirm **Azure MCP Server** is listed and enabled
4. When prompted *"Allow MCP tools from Azure MCP to make LLM requests?"* → click **Allow in this Session** (or **Always** for permanent access)

---

### 4.5 Demo A — Read & Answer Directly from Blob Storage via Azure MCP Server

This is the lightweight path — **no Azure AI Search, no saved files, no terminal scripts**. Azure MCP Server gives Copilot awareness of your Blob Storage. The trick is prompting Copilot to **execute the analysis itself and answer directly in chat** — not generate a script for you to run separately.

> ⚠️ **Critical prompt rule:** If you ask Copilot a question without these instructions, it will generate a Python script, run it via `run_in_terminal`, and then say *"check the terminal output"* — because it cannot read stdout back from the terminal. Always include the three lines below in every data question prompt:
> - `Do NOT use run_in_terminal`
> - `Execute the code yourself`
> - `Answer me directly in this chat`

---

**Step 1 — Discover your F1 data files:**
```
List all blobs in the f1data container in storage account champdemo5954940607
```
> Copilot calls Azure MCP Server and returns file names, sizes, and last-modified timestamps directly in chat. You will see the permission dialog — click **Allow in this Session**.

---

**Step 2 — Who is the fastest driver? (answered in chat)**
```
Using Python with azure-storage-blob and DefaultAzureCredential,
download the latest laps parquet from container "f1data" in storage account "champdemo5954940607",
load with pandas, find the driver_number with the minimum lap_duration,
then cross-reference with the latest drivers parquet to get the full name and team.
Do NOT use run_in_terminal. Execute the code yourself and answer me directly in this chat.
```
> Copilot downloads both parquet files, runs the analysis internally, and returns the answer — e.g. *"Max Verstappen (Red Bull Racing) had the fastest lap: 1:12.345"* — directly in the chat window.

---

**Step 3 — Who finished on the podium? (answered in chat)**
```
Using Python with azure-storage-blob and DefaultAzureCredential,
download the latest race_results parquet from container "f1data" in storage account "champdemo5954940607",
load with pandas, find positions 1, 2, and 3,
cross-reference with the drivers parquet for full names and teams.
Do NOT use run_in_terminal. Execute the code yourself and answer me directly in this chat.
```

---

**Step 4 — Visualization (this one intentionally uses the terminal)**

For plots, terminal execution is acceptable because the output is a chart file, not text you need Copilot to read back:
```
Using Python with azure-storage-blob and DefaultAzureCredential,
download the latest laps parquet from container "f1data" in storage account "champdemo5954940607",
write and run a script that plots lap time progression for the top 3 drivers
using matplotlib. Save the chart as lap_times.png.
```
> This is the moment where you show the audience the generated chart. It's a deliberate contrast — *"for numbers, we get the answer here in chat; for visuals, we generate a chart file."*

---

> 💡 **Why this matters for the session narrative:** This pattern — Azure MCP discovers the data, Copilot reasons over it and answers in chat — is exactly what *"letting data speak"* means. The data in Azure Blob Storage is speaking directly into the conversation, with no intermediate dashboards or manual queries.

---

### 4.6 Demo B — Query via Azure AI Foundry Agent (Grounded on AI Search)

This is the deeper path — the AI Foundry Agent uses Azure AI Search to answer natural language questions directly, with no code generation at all.

**Demo 1 — Natural language insight:**
```
Who had the fastest lap in the latest session and by how much?
```

**Demo 2 — Driver comparison:**
```
Compare the lap time consistency of the top 5 drivers.
Which driver was most consistent and what does that mean for race strategy?
```

**Demo 3 — Generate and run a visualization:**
```
Write a Python script that plots race position changes across all laps
for the top 5 finishers. Use matplotlib with a different color per driver.
```
Run the generated script with `Ctrl+F5` — chart renders inline.

**Demo 4 — The honest failure (intentional):**
```
Tell me the tyre strategy for each driver in the last race.
```
> The agent should say this data is unavailable — tyre data is not in the OpenF1 endpoints you extracted. **This is a feature, not a bug.** Show the audience that a well-instructed agent admits what it doesn't know rather than hallucinating. This moment builds trust.

---

## 💬 Sample Prompts to Try

These are ready-to-use prompts organized by approach — Azure MCP Server (Blob discovery path) and Azure AI Foundry Agent (grounded search path).

---

### 🔵 Azure MCP Server Prompts (Inline Answer Pattern)

Use these in GitHub Copilot Agent Mode. Always end with the three enforcement lines to get answers directly in chat.

**Template structure — copy and adapt:**
```
Using Python with azure-storage-blob and DefaultAzureCredential,
download the latest [DATASET] parquet from container "f1data" in storage account "champdemo5954940607",
load with pandas, [YOUR QUESTION / ANALYSIS].
Do NOT use run_in_terminal. Execute the code yourself and answer me directly in this chat.
```

**Fastest lap:**
```
Using Python with azure-storage-blob and DefaultAzureCredential,
download the latest laps parquet from container "f1data" in storage account "champdemo5954940607",
load with pandas, find the driver_number with the minimum lap_duration,
cross-reference with the drivers parquet for the full name and team.
Do NOT use run_in_terminal. Execute the code yourself and answer me directly in this chat.
```

**Podium results:**
```
Using Python with azure-storage-blob and DefaultAzureCredential,
download the latest race_results parquet from container "f1data" in storage account "champdemo5954940607",
find positions 1, 2 and 3, cross-reference drivers parquet for names and teams.
Do NOT use run_in_terminal. Execute the code yourself and answer me directly in this chat.
```

**Lap consistency ranking:**
```
Using Python with azure-storage-blob and DefaultAzureCredential,
download the latest laps parquet from container "f1data" in storage account "champdemo5954940607",
calculate standard deviation of lap_duration per driver_number, rank all drivers,
cross-reference drivers parquet for names. Show the full ranked table.
Do NOT use run_in_terminal. Execute the code yourself and answer me directly in this chat.
```

**Points gap:**
```
Using Python with azure-storage-blob and DefaultAzureCredential,
download the latest race_results parquet from container "f1data" in storage account "champdemo5954940607",
calculate the points gap between P1 and every other driver,
cross-reference drivers parquet for names.
Do NOT use run_in_terminal. Execute the code yourself and answer me directly in this chat.
```

---

### 🟠 Azure AI Foundry Agent Prompts (Grounded Search Path)

Use these in the Azure AI Foundry Agent playground or via the AI Foundry VS Code extension.

### Insight Prompts
```
Who finished on the podium in the latest race session?
```
```
What was the gap between P1 and P2 at the end of the race?
```
```
Which team had the most combined points in the latest session?
```

### Analysis Prompts
```
Compare the lap time consistency of the top 5 drivers.
Use standard deviation. Which driver was most consistent?
```
```
How many pit stop laps are visible in the laps data?
Identify them by unusually long lap times (> 40 seconds).
```

### Visualization Prompts
```
Write a Python script to create a bar chart of final race positions
with driver names on the x-axis and position on the y-axis.
```
```
Plot the lap time distribution for each driver as a box plot.
Use seaborn. Save it as lap_times.png.
```

### Edge Case Prompts (good for demos — shows honest agent behavior)
```
Tell me the tyre strategy for each driver in the last race.
```
> *(The agent should say this data is not available — OpenF1's `session_result` endpoint does not include tyre data. This is the correct, honest response.)*

---

## 🛠️ Local Setup (Optional — for attendees who want to run scripts locally)

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/let-your-data-speak.git
cd let-your-data-speak

# Create virtual environment
python -m venv venv
source venv/bin/activate        # Linux/Mac
venv\Scripts\activate           # Windows

# Install dependencies
pip install -r requirements.txt
```

**requirements.txt contents:**
```
requests
pandas
pyarrow
azure-storage-blob
matplotlib
seaborn
python-dotenv
```

**Create your `.env` file from the template:**
```bash
cp .env.example .env
```

Edit `.env`:
```env
AZURE_STORAGE_CONNECTION_STRING=your_connection_string_here
AZURE_CONTAINER_NAME=f1data
AZURE_AI_FOUNDRY_ENDPOINT=https://your-foundry-endpoint.openai.azure.com/
AZURE_AI_FOUNDRY_KEY=your_key_here
AGENT_ID=your_agent_id_here
```

---

## ❗ Common Issues and Fixes

| Issue | Cause | Fix |
|---|---|---|
| `403 Forbidden` on Key Vault | Fabric workspace not granted access | Add **Key Vault Secrets User** role to Fabric managed identity |
| `No data returned for laps` | No active F1 session right now | Run notebook during or just after a race weekend |
| `ResourceExistsError` on container | Container already exists | Safe to ignore — the notebook handles this silently |
| Agent returns generic answers | Blob data not indexed yet | Wait 2–5 minutes after adding data source in Foundry, then retry |
| Parquet file not found in shortcut | Shortcut URL mismatch | Ensure you used `.dfs.core.windows.net` (not `.blob.core.windows.net`) for ADLS Gen2 |

---

## 📚 Resources

- [OpenF1 API Documentation](https://openf1.org)
- [Azure AI Foundry Documentation](https://learn.microsoft.com/azure/ai-foundry)
- [Microsoft Fabric Documentation](https://learn.microsoft.com/fabric)
- [Azure Blob Storage Documentation](https://learn.microsoft.com/azure/storage/blobs)
- [Azure Key Vault Documentation](https://learn.microsoft.com/azure/key-vault)

---

## 👤 Author

**Dilussa** — Associate Data Engineer @ Bistec Global Services | Microsoft Learn Student Ambassador 🇱🇰

---

## 📄 License

This project is licensed under the MIT License.
