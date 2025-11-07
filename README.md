# Personal-AI-Agent

## 🎯 Project Goal

The objective of this project is to build a complete, self-hosted Personal AI Assistant. This agent is designed to be a long-term companion that can learn, remember, and securely access personal data to provide truly personalized and context-aware interactions.

The core goals include:
* **🧠 Memory:** Implementing long-term and short-term memory using a database (e.g., SQL, VectorDB).
* **📚 Personal Data:** Securely connecting to personal documents, notes, and other data sources (RAG).
* **🛠️ Tool Use (MCP):** Implementing tool use via MCP (Model Context Protocol) and Resource Prompts, allowing the agent to perform actions (e.g., web search, file management).
* **🌱 Continuous Learning:** Developing a feedback loop for the agent to learn from conversations and new information.

---

## 🏛️ System Architecture

This project is built as a modular, containerized stack. The current infrastructure serves as the secure gateway and the "brain" (LLM) of the agent.

The planned data flow is:
1.  **User** sends a request.
2.  **Nginx Gateway** (`app-server-stack`) authenticates the request using an API key.
3.  **Agent Service** (Future Module) receives the request. It processes logic, fetches/stores data from the **Memory Database**, and queries the **vLLM Backend**.
4.  **vLLM Backend** (GPU Server) generates the response.
5.  The response is returned to the user.

---

## 🚀 Current Status: Infrastructure Gateway

The foundation of the agent is complete. This consists of a secure Nginx reverse proxy that authenticates requests and forwards them to a backend vLLM server.

### Current Features
* **Nginx Reverse Proxy:** Forwards requests from `/llm_api/` to the backend vLLM.
* **API Key Authentication:** Secures the endpoint by requiring a `Authorization: Bearer <token>`.
* **Containerized:** Deployed via `docker-compose` for easy management.
* **Environment-based Config:** All sensitive values (IPs, API keys) are safely stored in a `.env` file.

### Project Structure
```text
.
├── docker-compose.yml     # Main Docker Compose file
├── .env.example           # Example environment file
├── devlog/
│   └── DEVLOG.md          # Project development log
└── nginx/
    ├── Dockerfile             # Nginx Dockerfile
    ├── index.html             # Static test page
    └── nginx.conf.template    # Nginx configuration template
```
---
## 🛠️ Setup & Deployment (Current Infrastructure)

### 1. Configure Environment

Create a `.env` file from the example.
```sh
cp .env.example .env
```
Now, edit .env and add your server IP and a secret API key.
```sh
# .env
GPU_SERVER_IP=your_vllm_server_ip_here
LLM_API_KEY=your_secret_api_key_here
```
(You can generate a secure key using: openssl rand -hex 32)

### 2. Build and Run
Use Docker Compose to build and start the Nginx container in the background.

```sh
docker-compose up --build -d
```
---
### 📨 How to Use (Test Endpoint)
You can test the Nginx gateway and vLLM backend using curl.

```sh
curl -X POST "http://<your-nginx-server-ip>/llm_api/chat/completions" \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer <your_secret_api_key_here>" \
     -d '{
         "model": "unsloth/gpt-oss-20b",
         "messages": [
             {
                 "role": "user",
                 "content": "What is the capital of France?"
             }
         ]
     }'
```
- A correct key will forward the request to your vLLM.
- An incorrect key will return a 401 Unauthorized error.
---
### 🗺️ Roadmap: Building the Agent
The next steps involve building the agent's "consciousness" that sits between Nginx and the vLLM.

- [ ] Module 2: Agent Service (FastAPI)
  - Create a new FastAPI service in docker-compose.yml.
  - Update Nginx to proxy requests to this new service instead of directly to the vLLM.
  - The FastAPI service will be responsible for handling all logic.

- [ ] Module 3: Memory
  - Integrate a database (e.g., PostgreSQL or SQLite) for structured memory (user preferences, facts).
  - Integrate a Vector Database (e.g., LanceDB, Chroma) for conversational memory and RAG.

- [ ] Module 4: Document Integration (RAG)
  - Build an ingestion pipeline to add personal documents (.md, .pdf) to the vector store.
  - Update the agent service to retrieve relevant context before calling the LLM.

- [ ] Module 5: Tool Use (MCP)
  - Integrate the MCP (Model Context Protocol) framework for defining tools.
  - Create "Resource Prompts" that allow the agent to request and use tools like web search or file access.

- [ ] Module 6: Continuous Learning
  - Implement a feedback mechanism (e.g., "save this" or "you were wrong") to update the agent's knowledge base.
