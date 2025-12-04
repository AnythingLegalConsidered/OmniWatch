# OmniWatch 👁️
### Enterprise-Grade Semantic Intelligence Pipeline

![Docker](https://img.shields.io/badge/Docker-Containerized-blue?logo=docker) ![Elasticsearch](https://img.shields.io/badge/Vector_DB-Elasticsearch_8.x-yellow?logo=elasticsearch) ![Ollama](https://img.shields.io/badge/AI_Inference-Ollama-black) ![n8n](https://img.shields.io/badge/Orchestration-n8n-orange)

**OmniWatch** is a self-hosted, offline-first intelligence infrastructure designed to automate the collection, analysis, and vectorization of high-value information. Unlike traditional keyword monitoring, OmniWatch leverages **Hybrid Search (Vector + Keyword)** to understand context and nuance.

---

## 🏗️ Architecture

The pipeline is built on a micro-services architecture, fully containerized:

| Component | Service | Role | Port |
|-----------|---------|------|------|
| **Ingestion** | **n8n** | ETL & Orchestration (RSS/Web Scraping) | `5678` |
| **Brain** | **Dify** | RAG Engine, Context Management & UI | `80/5001` |
| **Memory** | **Elasticsearch** | Hybrid Vector Store (kNN + BM25) | `9200` |
| **Inference** | **Ollama** | Local LLM Runner (GPU Accelerated) | `11434` |

---

## ⚡ Hardware Requirements

**Warning:** AI and Vector Search are resource-intensive. Running this on a Raspberry Pi is not recommended.

### Minimum (CPU Only - High Latency)
*   **RAM:** 16 GB (Strict minimum to avoid OOM Kills).
*   **CPU:** 4+ Cores (AVX2 support required).
*   **Storage:** 50 GB SSD (NVMe recommended for Vector Indexing).

### Recommended (Production / GPU)
*   **RAM:** 32 GB System RAM.
*   **GPU:** NVIDIA RTX 3060 (12GB VRAM) or better.
*   **VRAM Requirement:**
    *   ~6 GB for `Qwen2.5-7B` (4-bit quantized).
    *   ~14 GB for `Mistral-Nemo-12B` or `Llama-3.1-8B` (FP16).

---

## 🚀 Deployment

### 1. Clone & Configure
```bash
git clone https://github.com/your-username/omniwatch.git
cd omniwatch

# Setup environment variables
cp .env.example .env
# ⚠️ CRITICAL: Set ELASTIC_PASSWORD and N8N_ENCRYPTION_KEY in .env
```

### 2. Networking & Volumes
```bash
# Create the dedicated bridge network
docker network create --driver bridge ai-network

# Create volume directories (ensure permissions)
mkdir -p volumes/{ollama,elastic,n8n,dify}
```

### 3. Launch Stack
```bash
docker compose up -d
```

---

## 🧠 Model Strategy (2025 Standards)

To ensure maximum performance vs resource usage, we recommend the following model stack via Ollama.

### 1. Vector Embeddings (The Archivist)
- gemma3-270m-embed
- stella-400-en
- qwen3-0.6B-embed
- colnomic-embed-multimodal-7b
- nomic-embed-text-v1.5
**Model:** `nomic-embed-text-v1.5`
*   **Role:** Converts text into mathematical vectors.
*   **Why:** Best open-source performance for RAG (Retrieval Augmented Generation), supports long context (8192 tokens).
```bash
docker exec -it ollama ollama pull nomic-embed-text
```

### 2. Large Language Model (The Analyst)
**Option A: Performance/Speed Ratio (Recommended)**
**Model:** `qwen2.5:7b`
*   **Why:** Qwen 2.5 currently beats Llama 3.1 and Mistral in reasoning benchmarks while being faster. Excellent multilingual support.
```bash
docker exec -it ollama ollama pull qwen2.5:7b
```

**Option B: The Standard**
**Model:** `llama3.1:8b`
*   **Why:** The industry standard, very stable, but slightly more VRAM hungry than Qwen for similar results.
```bash
docker exec -it ollama ollama pull llama3.1
```

---

## ⚙️ Workflow Setup (n8n)

We do not hardcode workflows; we import them.

1.  Access **n8n** at `http://localhost:5678`.
2.  Go to **Workflows** > **Import from File**.
3.  Select the file located in `./workflows/rss-to-dify-rag.json` from this repository.

### Workflow Logic
`Cron (8:00 AM)` ➔ `RSS Fetcher` ➔ `HTML Cleaner` ➔ `Dify API (Indexing)`

> **Note for Dify Connection:**
> Inside n8n Docker container, Dify is accessible via `http://dify-api:5001`, NOT localhost.

---

## 🔧 Maintenance & Ops

### Monitoring Elasticsearch Health
Vector indexing can be heavy. Check your cluster health:
```bash
curl -u elastic:${ELASTIC_PASSWORD} http://localhost:9200/_cluster/health?pretty
```
*Status should be `green` or `yellow`. If `red`, check Java Heap usage.*

### GPU Passthrough Check
Verify that Ollama sees your NVIDIA GPU:
```bash
docker logs ollama | grep "GPU"
```

---

## 🔒 Security Best Practices

1.  **Network:** Never expose ports `9200` (Elastic) or `5678` (n8n) to the public internet without a Reverse Proxy (Nginx/Traefik) and SSL.
2.  **Secrets:** Use `.env` file. Never commit keys to Git.
3.  **Updates:** Watch out for breaking changes in Dify updates (check their release notes before pulling new images).
