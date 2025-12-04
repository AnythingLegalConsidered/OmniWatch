# OmniWatch

Automated Semantic Intelligence Infrastructure - On-Premise

## 📋 Technical Stack

| Service | Role | Port |
|---------|------|------|
| **Ollama** | Local AI Inference Engine | 11434 |
| **Elasticsearch** | Vector & Hybrid Search Database | 9200 |
| **n8n** | ETL Orchestration | 5678 |
| **Dify** | RAG Engine & Interface | 3000 |

## 🚀 Quick Start

### 1. Prerequisites

- Docker & Docker Compose v2+
- Minimum 4 GB RAM available (Elasticsearch 1 GB + Ollama + models)
- (Optional) NVIDIA Container Toolkit for GPU

### 2. Configuration

```bash
# Copy environment file
cp .env.example .env

# Edit variables (passwords, keys, etc.)
nano .env
```

> ⚠️ **IMPORTANT** : Define `ELASTIC_PASSWORD` with a strong password!

### 3. Create Docker Network

```bash
docker network create --driver bridge ai-network
```

### 4. Launch Services

```bash
docker compose up -d
```

---

## 🔍 Elasticsearch - Configuration

### Check Cluster Status

```bash
# Check cluster health (should return green or yellow)
curl -u elastic:${ELASTIC_PASSWORD} http://localhost:9200/_cluster/health?pretty

# Via Docker
docker exec -it elasticsearch curl -s -u elastic:changeme_elastic_password http://localhost:9200/_cluster/health?pretty
```

### Cluster Information

```bash
curl -u elastic:${ELASTIC_PASSWORD} http://localhost:9200
```

### Memory Configuration

| Parameter | Value | Description |
|-----------|--------|-------------|
| `ES_JAVA_OPTS` | `-Xms1g -Xmx1g` | JVM Heap fixed at 1 GB |
| `discovery.type` | `single-node` | Single-node mode |
| `xpack.security.enabled` | `true` | Authentication enabled |

> 💡 **Note** : JVM heap should be ≤ 50% of available RAM. Adjust `ES_JAVA_OPTS` in `.env` if necessary.

---

## ⚙️ n8n - ETL Orchestration

### Access Interface

Open http://localhost:5678 in your browser.

### Internal URLs for Workflows

In your n8n workflows, use these URLs to communicate with other services:

| Service | Internal URL | Usage |
|---------|-------------|-------|
| **Ollama** | `http://ollama:11434` | LLM API calls |
| **Dify API** | `http://dify-api:5001` | Document sending, chat |
| **Elasticsearch** | `http://elasticsearch:9200` | Direct ES queries |

### Example: Ollama Call from n8n

```json
{
  "method": "POST",
  "url": "http://ollama:11434/api/generate",
  "body": {
    "model": "llama3",
    "prompt": "Summarize this text..."
  }
}
```

### Generate Encryption Key

```bash
# Linux/Mac
openssl rand -hex 32

# PowerShell
-join ((1..32) | ForEach-Object { "{0:x2}" -f (Get-Random -Maximum 256) })
```

> 💡 **Tip** : Define `N8N_ENCRYPTION_KEY` in `.env` to secure stored credentials.

---

## 🤖 Ollama - Model Configuration

Once the Ollama container is launched, you need to download AI models.

### Download Required Models

```bash
# Embedding model (document vectorization)
docker exec -it ollama ollama pull nomic-embed-text

# Chat model (response generation)
docker exec -it ollama ollama pull llama3

# Alternative: Mistral (lighter)
docker exec -it ollama ollama pull mistral
```

### Check Installed Models

```bash
docker exec -it ollama ollama list
```

### Test a Model

```bash
# Quick test of chat model
docker exec -it ollama ollama run llama3 "Hello, respond in one sentence."
```

### Approximate Model Sizes

| Model | Size | Usage |
|--------|--------|-------|
| `nomic-embed-text` | ~275 MB | Embedding / Vectorization |
| `llama3` (8B) | ~4.7 GB | Chat / Generation |
| `mistral` (7B) | ~4.1 GB | Chat / Generation (alternative) |

> 💡 **Tip** : Models are persisted in `./volumes/ollama/`. They won't be re-downloaded after restart.

---

## 🔧 GPU Configuration (Optional)

To enable NVIDIA GPU acceleration:

### 1. Install nvidia-container-toolkit

```bash
# Ubuntu/Debian
sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

### 2. Enable in docker-compose.yml

Uncomment the `deploy` section of the `ollama` service:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all
          capabilities: [gpu]
```

### 3. Restart Service

```bash
docker compose up -d ollama
```

---

## 📁 Volume Structure

```
volumes/
├── ollama/           # Downloaded LLM models
├── elasticsearch/
│   └── data/         # Index and data
├── n8n/              # Workflows and credentials
└── dify/
    ├── db/           # PostgreSQL
    ├── redis/        # Cache
    └── storage/      # Uploaded files
```

---

## 🔒 Security

- Never commit the `.env` file containing secrets
- Use `.env.example` as template
- All passwords must be defined via environment variables

---

## 🔄 n8n Workflow - Automated RSS Intelligence

This workflow automatically collects RSS articles and indexes them in Dify for RAG search.

### Workflow Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   CRON      │────▶│  RSS Feed   │────▶│  Loop Items │────▶│ HTTP POST   │
│  8AM/daily  │     │  Reader     │     │             │     │ Dify API    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

### Prerequisites in Dify

#### 1. Get Dify API Key

1. Open Dify: http://localhost
2. Click your **profile** (top right corner)
3. Go to **Settings** → **API Keys**
4. Click **Create New API Key**
5. Copy the generated key (format: `dataset-xxxxxx...`)

#### 2. Create Dataset and Get ID

1. In Dify, go to **Knowledge** (left menu)
2. Click **Create Knowledge**
3. Give a name: `AI Intelligence` (or other)
4. Configure embedding model: **Ollama → nomic-embed-text**
5. Once created, the page URL contains the Dataset ID:
   ```
   http://localhost/datasets/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/...
                              └──────────── Dataset ID ────────────┘
   ```

### n8n Workflow - Complete JSON

Import this JSON into n8n (Menu → Import from File):

```json
{
  "name": "OmniWatch - RSS Intelligence to Dify",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "triggerAtHour": 8
            }
          ]
        }
      },
      "id": "cron-trigger",
      "name": "Every day at 8 AM",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.1,
      "position": [240, 300]
    },
    {
      "parameters": {
        "url": "https://news.google.com/rss/search?q=intelligence+artificielle&hl=fr&gl=FR&ceid=FR:fr",
        "options": {}
      },
      "id": "rss-reader",
      "name": "Google News RSS AI",
      "type": "n8n-nodes-base.rssFeedRead",
      "typeVersion": 1,
      "position": [460, 300]
    },
    {
      "parameters": {
        "batchSize": 1,
        "options": {}
      },
      "id": "loop-items",
      "name": "Process each article",
      "type": "n8n-nodes-base.splitInBatches",
      "typeVersion": 3,
      "position": [680, 300]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "http://dify-api:5001/v1/datasets/{{$credentials.dify_dataset_id}}/document/create_by_text",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Content-Type",
              "value": "application/json"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"name\": \"{{ $json.title.substring(0, 100) }}\",\n  \"text\": \"# {{ $json.title }}\\n\\nSource: {{ $json.link }}\\nDate: {{ $json.pubDate }}\\n\\n{{ $json.contentSnippet || $json.content || $json.description }}\",\n  \"indexing_technique\": \"high_quality\",\n  \"process_rule\": {\n    \"mode\": \"automatic\"\n  }\n}",
        "options": {
          "timeout": 30000
        }
      },
      "id": "http-dify",
      "name": "Send to Dify",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [900, 300],
      "credentials": {
        "httpHeaderAuth": {
          "id": "dify-api-key",
          "name": "Dify API Key"
        }
      }
    },
    {
      "parameters": {
        "content": "## OmniWatch Workflow\n\n### Required Configuration:\n1. Create a \"HTTP Header Auth\" credential named **Dify API Key**\n   - Header Name: `Authorization`\n   - Header Value: `Bearer dataset-YOUR_API_KEY`\n\n2. Replace `{{$credentials.dify_dataset_id}}` with your real Dataset ID in the HTTP node\n\n3. Adjust the RSS feed URL according to your needs",
        "height": 340,
        "width": 340
      },
      "id": "sticky-note",
      "name": "Instructions",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [240, 480]
    }
  ],
  "connections": {
    "Every day at 8 AM": {
      "main": [
        [
          {
            "node": "Google News RSS AI",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google News RSS AI": {
      "main": [
        [
          {
            "node": "Process each article",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Process each article": {
      "main": [
        [
          {
            "node": "Send to Dify",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Process each article",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "settings": {
    "executionOrder": "v1"
  }
}
```

### Configure Credentials in n8n

1. Open n8n: http://localhost:5678
2. Go to **Settings** → **Credentials**
3. Click **Add Credential** → **Header Auth**
4. Configure:
   - **Name**: `Dify API Key`
   - **Header Name**: `Authorization`
   - **Header Value**: `Bearer dataset-YOUR_DIFY_API_KEY`

### Adjust the Workflow

#### Change RSS Feed URL

Feed examples:

```
# Google News - AI
https://news.google.com/rss/search?q=intelligence+artificielle&hl=fr&gl=FR&ceid=FR:fr

# Google News - Cybersecurity
https://news.google.com/rss/search?q=cybersécurité&hl=fr&gl=FR&ceid=FR:fr

# Hacker News
https://hnrss.org/frontpage

# TechCrunch
https://techcrunch.com/feed/
```

#### Change Dataset ID

In the **"Send to Dify"** node, replace the URL:
```
http://dify-api:5001/v1/datasets/YOUR_DATASET_ID/document/create_by_text
```

### Structure of Document Sent to Dify

```json
{
  "name": "Article title (max 100 chars)",
  "text": "# Title\n\nSource: URL\nDate: Publication date\n\nArticle content...",
  "indexing_technique": "high_quality",
  "process_rule": {
    "mode": "automatic"
  }
}
```

### Test the Workflow

1. In n8n, open the workflow
2. Click **Execute Workflow** (green button)
3. Check results of each node
4. In case of error, check logs:
   ```bash
   docker logs n8n
   docker logs dify-api
   ```

### Verify Indexing in Dify

1. Go to Dify → **Knowledge** → Your dataset
2. Imported documents appear in the list
3. Test with chat: ask questions about indexed articles

---

## 📖 Documentation

- [Ollama](https://ollama.ai/)
- [Dify](https://dify.ai/)
- [n8n](https://n8n.io/)
- [Elasticsearch](https://www.elastic.co/)
