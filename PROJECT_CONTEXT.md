# Project Name: OmniWatch (Automated Semantic Intelligence Infrastructure)

## 1. Objective
Set up a complete "On-Premise" infrastructure to automate intelligence gathering on any subject. The system must collect web feeds, filter relevant information via AI, and allow natural language querying (RAG) based on hybrid search (Keywords + Vector).

## 2. Technical Stack (Strictly defined)
- **Infrastructure:** Docker & Docker Compose (Shared bridge network).
- **Orchestration (ETL):** n8n (Latest stable version).
- **RAG Engine & Interface:** Dify (Open Source).
- **Vector & Text Database:** Elasticsearch 8.x (For hybrid search).
- **AI Inference Engine:** Ollama (Local).
    - Embedding Model: `nomic-embed-text`
    - Chat Model: `llama3` or `mistral`

## 3. Data Architecture
1. **Input:** RSS Feeds / Web Scraping (via n8n).
2. **Process:** HTML Cleaning → Send to Dify API.
3. **Storage:** Dify vectorizes (via Ollama) and indexes in Elasticsearch.
4. **Output:** Dify Chat interface queries Elasticsearch.

## 4. System Administrator Constraints
- **Persistence:** All data (DB, Models, Configs) must be stored in local Docker volumes (`./volumes/...`).
- **Resources:** Elasticsearch must be limited in RAM (JVM Heap).
- **Network:** All containers must communicate on a Docker network named `ai-network`.
- **Security:** No hardcoded passwords in code, use `.env` file.