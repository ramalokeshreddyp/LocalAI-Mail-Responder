# 🤖 Local AI Email Auto-Responder

> A fully private, cost-free AI email auto-responder powered by **n8n** and **Ollama** — no cloud APIs, no data leaks, no subscriptions.

[![Docker](https://img.shields.io/badge/Docker-Compose-blue?logo=docker)](https://www.docker.com/)
[![n8n](https://img.shields.io/badge/n8n-Workflow%20Automation-orange?logo=n8n)](https://n8n.io/)
[![Ollama](https://img.shields.io/badge/Ollama-Local%20LLM-green)](https://ollama.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 📖 Overview

This project builds an intelligent email auto-responder that runs **entirely on your local machine**. When a new email arrives in your inbox, the system:

1. **Detects** the incoming email via IMAP polling
2. **Prevents loops** — skips emails from itself or already auto-replied threads
3. **Generates** a contextually relevant reply using a local LLM (Llama 3 8B via Ollama)
4. **Sends** the reply via SMTP, correctly threaded using the original `Message-ID`

```
┌─────────────────────────────────────────────────────────────────┐
│                         Docker Network                          │
│                                                                 │
│  ┌──────────────────────────┐    ┌──────────────────────────┐   │
│  │       n8n Container      │    │     Ollama Container     │   │
│  │                          │    │                          │   │
│  │  [IMAP Trigger]          │    │  llama3:8b model         │   │
│  │       ↓                  │    │                          │   │
│  │  [Loop Prevention IF]    │    │  REST API:               │   │
│  │       ↓ (pass)           │───▶│  POST /api/generate      │   │
│  │  [HTTP Request]          │◀───│                          │   │
│  │       ↓                  │    └──────────────────────────┘   │
│  │  [Send Email SMTP]       │                                   │
│  └──────────────────────────┘                                   │
└─────────────────────────────────────────────────────────────────┘
        ↑ read emails                         ↓ send reply
   ┌────────────────────────────────────────────────────┐
   │                   Email Server                      │
   │            (Gmail / Outlook / Custom)               │
   └────────────────────────────────────────────────────┘
```

### Why Local AI?

| Feature | This Project | Cloud API |
|---|---|---|
| 🔒 Privacy | ✅ Data never leaves your machine | ❌ Sent to third-party servers |
| 💰 Cost | ✅ Free (hardware only) | ❌ Pay per token |
| 🌐 Internet | ✅ Works offline | ❌ Requires connectivity |
| 🎛️ Control | ✅ Full model & prompt control | ❌ Subject to provider limits |

---

## 🏗️ Architecture

### Services

| Service | Image | Port | Purpose |
|---|---|---|---|
| `n8n` | `n8nio/n8n:latest` | `5678` | Workflow automation engine |
| `ollama` | `ollama/ollama:latest` | *(internal)* | Local LLM inference server |
| `ollama-init` | `ollama/ollama:latest` | — | One-time model pull on startup |

### n8n Workflow Nodes

```
Email Read (IMAP)
      │
      ▼
Loop Prevention (IF)
   ├─ TRUE → Generate Reply with Ollama (HTTP POST)
   │               │
   │               ▼
   │         Send Reply Email (SMTP)
   │
   └─ FALSE → Stop - Loop Detected (NoOp)
```

---

## 🚀 Getting Started

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (includes Docker Compose)
- An email account with IMAP/SMTP access enabled
- At least **8 GB RAM** available for Docker (16 GB recommended for llama3:8b)
- At least **5 GB free disk space** for the LLM model

> **Gmail Users**: Enable "2-Step Verification" and generate an [App Password](https://myaccount.google.com/apppasswords) to use instead of your regular password. Set IMAP access to "Enabled" in Gmail settings.

---

### Step 1: Clone the Repository

```bash
git clone https://github.com/ramalokeshreddyp/LocalAI-Mail-Responder.git
cd LocalAI-Mail-Responder
```

### Step 2: Configure Environment Variables

Copy the example environment file and fill in your credentials:

```bash
cp .env.example .env
```

Open `.env` in your editor and set the following values:

```env
# n8n Settings
N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http
WEBHOOK_URL=http://localhost:5678/

# Incoming Email (IMAP)
IMAP_HOST=imap.gmail.com        # Your IMAP server
IMAP_PORT=993                   # 993 for SSL
IMAP_USER=you@gmail.com         # Your email address
IMAP_PASS=your-app-password     # Gmail App Password (not your regular password)

# Outgoing Email (SMTP)
SMTP_HOST=smtp.gmail.com        # Your SMTP server
SMTP_PORT=587                   # 587 for STARTTLS
SMTP_USER=you@gmail.com         # Your email address (same as IMAP_USER)
SMTP_PASS=your-app-password     # Same App Password as above
```

> ⚠️ **Security**: Add `.env` to your `.gitignore` file. Never commit real credentials.

### Step 3: Start the Services

```bash
docker-compose up -d
```

This command will:
- Pull the `n8nio/n8n` and `ollama/ollama` Docker images
- Start both containers in the background
- Automatically pull the **llama3:8b** model via the `ollama-init` service

Check that all containers are running and healthy:

```bash
docker-compose ps
```

Wait ~2-3 minutes for the model to download (it's ~4.7 GB). Monitor progress:

```bash
docker-compose logs -f ollama-init
```

### Step 4: Verify the LLM Model

Confirm the model is available:

```bash
docker exec ollama_responder ollama list
```

You should see `llama3:8b` listed. You can also test the API directly:

```bash
curl http://localhost:11434/api/tags
```

### Step 5: Import the Workflow into n8n

1. Open your browser and navigate to **http://localhost:5678**
2. Complete the n8n first-run setup (create an owner account)
3. Go to **Workflows** → Click **Import from File**
4. Select the `workflow.json` file from the project root
5. The workflow will open in the editor

### Step 6: Configure Credentials in n8n

After importing, you need to link the email credentials:

**IMAP Credential:**
1. Click on the **Email Read (IMAP)** node
2. Click on the **Credential** selector → **Create New**
3. Fill in Host, Port (993), User, Password — check **SSL/TLS**
4. Click **Save**

**SMTP Credential:**
1. Click on the **Send Reply Email** node
2. Click on the **Credential** selector → **Create New**
3. Fill in Host, Port (587), User, Password — check **STARTTLS**
4. Click **Save**

### Step 7: Activate the Workflow

1. In the workflow editor, toggle the **Active** switch (top right) to **ON**
2. The workflow is now live and will poll your INBOX every minute

---

## 🧪 Testing

### End-to-End Test

1. From a **different email account**, send an email to your configured IMAP address:
   - Subject: `Test - Please advise on scheduling a meeting`
   - Body: `Hi, I'd like to schedule a 30-minute meeting next week. What times work for you?`

2. In n8n, navigate to **Executions** (left sidebar) — you should see a new execution appear within ~1 minute.

3. Click the execution to inspect the data flow through each node.

4. Check the sending account's inbox — you should receive a threaded AI-generated reply.

### Loop Prevention Test

Send an email **from your auto-responder's own address** to itself. Confirm that:
- The workflow triggers
- Execution stops at the **Loop Prevention** node (false branch → Stop)
- No reply is sent

### Verify Email Threading

The reply should appear in the **same conversation thread** as your original email in your email client (Gmail, Outlook, etc.). This works because the workflow sets the `In-Reply-To` header using the original `Message-ID`.

---

## 🔧 Troubleshooting

### n8n can't connect to Ollama

**Symptom**: HTTP Request node fails with "Connection refused" or "Network Error"

**Fix**: Ensure both containers are on the same Docker network. The URL must be `http://ollama:11434/api/generate` (using the service name `ollama`, not `localhost`).

```bash
docker network inspect ai_responder_net
# Both n8n_responder and ollama_responder should appear in "Containers"
```

### Model not found error

**Symptom**: Ollama API returns `{"error":"model 'llama3:8b' not found"}`

**Fix**: Manually pull the model:

```bash
docker exec -it ollama_responder ollama pull llama3:8b
docker exec ollama_responder ollama list   # Verify it appears
```

### IMAP/SMTP connection fails

**Symptom**: Email nodes show authentication or SSL errors

**Fix**:
- For Gmail: Use an [App Password](https://myaccount.google.com/apppasswords), not your regular password
- Verify IMAP is enabled in Gmail: Settings → See all settings → Forwarding and POP/IMAP → Enable IMAP
- Double-check port numbers: IMAP=993 (SSL), SMTP=587 (STARTTLS) or 465 (SSL)

### Response quality is poor

**Fix**: The prompt in the HTTP Request node drives response quality. Edit the `Generate Reply with Ollama` node's body to refine the prompt. Use clear persona instructions ("You are a professional assistant"), context, and output format constraints.

### Low memory / Container crashes

**Fix**: llama3:8b requires ~8 GB RAM. You can try a smaller model:

```bash
docker exec -it ollama_responder ollama pull phi3:mini
```

Then update the `"model"` field in the HTTP Request node body from `"llama3:8b"` to `"phi3:mini"`.

---

## 📁 Project Structure

```
LocalAI-Mail-Responder/
├── docker-compose.yml    # Service definitions (n8n, ollama, ollama-init)
├── .env.example          # Template for environment variables
├── .env                  # Your local credentials (DO NOT COMMIT)
├── workflow.json         # Exported n8n workflow
├── submission.json       # Test credentials schema for evaluation
└── README.md             # This file
```

---

## 🔐 Security Notes

- The `.env` file contains sensitive credentials — **never commit it** to version control
- Add `.gitignore` entry: `echo ".env" >> .gitignore`
- For production use, consider using Docker secrets or a dedicated secrets manager
- The Ollama service is not exposed publicly (no host port mapping) — it's only accessible within the Docker network
- n8n credential storage is encrypted at rest

---

## 🛠️ Stopping the Services

```bash
# Stop containers (preserves data)
docker-compose stop

# Stop and remove containers (preserves volumes/data)
docker-compose down

# Full cleanup including named volumes (WARNING: deletes all n8n workflows and models)
docker-compose down -v
```

---

## 📚 Resources

- [n8n Documentation](https://docs.n8n.io/)
- [Ollama Model Library](https://ollama.com/library)
- [Llama 3 on Ollama](https://ollama.com/library/llama3)
- [Docker Compose Reference](https://docs.docker.com/compose/)
- [n8n Email Read (IMAP) Node](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.emailreadimap/)

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

---

*Built as a hands-on demonstration of privacy-preserving AI automation using open-source tools.*
