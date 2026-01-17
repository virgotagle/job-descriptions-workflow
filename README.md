# Job Descriptions Workflow

An n8n workflow that uses AI to extract structured job profiles from raw job listings for automated job matching.

## Purpose

This workflow processes job listings scraped by [jobs-scraper](https://github.com/virgotagle/jobs-scraper) and extracts structured `job_profile` data that can be used for:

- Filtering jobs by experience level, skills, or tools
- Building job recommendation systems
- AI-powered job matching
- Analytics and insights on job market trends

## How It Works

This workflow uses a local LLM (via Ollama) to transform unstructured job descriptions into consistent, queryable JSON.

### The Problem

Raw job descriptions are messy:

```
"We're looking for a passionate Data Engineer with 3+ years experience. 
You'll need strong Python skills and experience with AWS. SQL is essential. 
Docker experience is a plus but not required..."
```

Different companies write requirements differently — some use bullet points, some use paragraphs, some say "must have," others say "essential." This inconsistency makes automated matching difficult.

### The Solution

The LLM parses natural language and outputs normalized JSON:

```json
{
  "title": "Data Engineer",
  "level": "mid",
  "exp_years": [3, null],
  "exp_domains": ["data engineering"],
  "edu_min": null,
  "skills_req": [],
  "skills_pref": [],
  "tools_req": ["Python", "AWS", "SQL"],
  "tools_pref": ["Docker"],
  "certs_req": [],
  "certs_pref": []
}
```

This structured data can then be compared against your skills and experience for accurate job matching.

### Prompt Engineering

The LLM is given detailed instructions to ensure consistent output:

**Level Classification**

| Input Signals | Output |
|---------------|--------|
| "intermediate", "3-5 years" | `mid` |
| "senior", "5+ years" | `senior` |
| "recent graduate", "entry level" | `entry` |

**Experience Parsing**

| Input | Output |
|-------|--------|
| "3+ years" | `[3, null]` |
| "2-5 years" | `[2, 5]` |
| "extensive experience" | `[5, null]` |
| Not mentioned | `[null, null]` |

**Required vs Preferred Detection**

| Keywords | Classification |
|----------|----------------|
| "essential", "required", "must have" | `_req` fields |
| "desirable", "nice to have", "is a plus", "bonus" | `_pref` fields |

**Skills vs Tools Separation**

| Type | Examples |
|------|----------|
| Skills (abilities) | data modelling, troubleshooting, API design |
| Tools (software/languages) | Python, SQL, AWS, Docker, ServiceNow |

### Model Configuration

- **Model**: `gpt-oss:20b` (via Ollama)
- **Output format**: JSON mode enabled
- **Processing**: One job at a time (batched loop)

### Why Local LLM?

- **Privacy** — Job data stays on your machine
- **Cost** — No API fees for high-volume processing
- **Speed** — No network latency for each request
- **Control** — Swap models easily, fine-tune if needed

## Extracted Fields

| Field | Type | Description |
|-------|------|-------------|
| `title` | string | Job title |
| `level` | enum | entry, junior, mid, senior, lead, principal, exec |
| `exp_years` | [min, max] | Required experience range in years |
| `exp_domains` | string[] | Relevant work domains |
| `edu_min` | enum | Minimum education (HS, AS, BS, MS, PhD) |
| `skills_req` | string[] | Required technical skills |
| `skills_pref` | string[] | Preferred technical skills |
| `tools_req` | string[] | Required tools/languages/platforms |
| `tools_pref` | string[] | Preferred tools/languages/platforms |
| `certs_req` | string[] | Required certifications |
| `certs_pref` | string[] | Preferred certifications |

## Workflow

```
Manual Trigger
     ↓
Fetch unprocessed jobs (job_profile IS NULL)
     ↓
Loop over items (one at a time)
     ↓
Send to Ollama LLM → Extract job_profile JSON
     ↓
Validate response (is non-empty object?)
  ├─ Valid   → Save to database → Next item
  └─ Invalid → Skip → Next item
```

## Prerequisites

- [n8n](https://n8n.io/) (self-hosted or cloud)
- [PostgreSQL](https://www.postgresql.org/) with data from [jobs-scraper](https://github.com/virgotagle/jobs-scraper)
- [Ollama](https://ollama.ai/) running locally

## Setup

### 1. Install Ollama and Pull the Model

```bash
# Install Ollama (macOS)
brew install ollama

# Start Ollama
ollama serve

# Pull the model
ollama pull gpt-oss:20b
```

> **Note**: You can use any Ollama-compatible model. Update the `modelId` in the workflow if using a different model.

### 2. Import Workflow into n8n

1. Open your n8n instance
2. Go to **Workflows** → **Add Workflow** → **Import from File**
3. Select `Job Descriptions.json`

### 3. Configure Credentials

After importing, you'll need to set up two credentials:

#### PostgreSQL Credential

1. Click on **Credentials** in n8n sidebar
2. Click **Add Credential** → Search for **Postgres**
3. Fill in your database details:

| Field | Value |
|-------|-------|
| Host | `localhost` (or your DB host) |
| Database | Your database name |
| User | Your database user |
| Password | Your database password |
| Port | `5432` (default) |

4. Click **Save**
5. Open the workflow and update both Postgres nodes to use this credential

#### Ollama Credential

1. Click **Add Credential** → Search for **Ollama**
2. Fill in:

| Field | Value |
|-------|-------|
| Base URL | `http://localhost:11434` |

3. Click **Save**
4. Open the workflow and update the Ollama node to use this credential

### 4. Verify Database Schema

Ensure your database has the required tables from [jobs-scraper](https://github.com/virgotagle/jobs-scraper):

```sql
-- Check tables exist
SELECT * FROM job_listings LIMIT 1;
SELECT * FROM job_details LIMIT 1;

-- Check job_profile column exists
SELECT column_name FROM information_schema.columns 
WHERE table_name = 'job_details' AND column_name = 'job_profile';
```

If `job_profile` column doesn't exist:

```sql
ALTER TABLE job_details ADD COLUMN job_profile JSONB;
```

## Usage

### Run Manually

1. Open the workflow in n8n
2. Click **Execute Workflow**
3. Monitor progress in the execution panel

### Run on Schedule (Optional)

To process jobs automatically:

1. Delete the **Manual Trigger** node
2. Add a **Schedule Trigger** node
3. Configure your preferred schedule (e.g., every hour)
4. **Activate** the workflow

## Error Handling

- Invalid LLM responses are skipped (not saved)
- Failed jobs remain with `job_profile IS NULL`
- Re-running the workflow automatically retries failed jobs
- Self-healing: no manual intervention needed

## Customization

| Change | How |
|--------|-----|
| Switch LLM model | Ollama node → change `modelId` |
| Adjust extraction rules | Ollama node → edit prompt content |
| Add new fields | Update prompt + add column to `job_details` table |
| Change batch size | Loop Over Items node → adjust batch size |

## Troubleshooting

### "Ollama connection refused"

Ensure Ollama is running:

```bash
ollama serve
```

### "Credential not found"

Re-link credentials in each node after importing the workflow.

### "No items to process"

Check that your database has jobs with `job_profile IS NULL`:

```sql
SELECT COUNT(*) FROM job_details WHERE job_profile IS NULL;
```

### "Invalid JSON from LLM"

The IF node will skip these automatically. If it happens frequently, try a different model or adjust the prompt.

## Related Projects

- [jobs-scraper](https://github.com/virgotagle/jobs-scraper) — Scrapes job listings from Seek

## License

MIT
