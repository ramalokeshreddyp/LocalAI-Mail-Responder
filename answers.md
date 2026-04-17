# Questionnaire Answers

## 1) Design choices for the n8n workflow (prompt construction and loop prevention)
I designed the workflow as a minimal linear pipeline with one guard branch:

1. `Email Read (IMAP)` trigger (`n8n-nodes-base.emailReadImap`) polls `INBOX` every minute.
2. `Loop Prevention` (`n8n-nodes-base.if`) filters out messages that could create auto-reply loops.
3. `Generate Reply with Ollama` (`n8n-nodes-base.httpRequest`) calls `http://ollama:11434/api/generate`.
4. `Send Reply Email` (`n8n-nodes-base.emailSend`) sends the model output back to the original sender.
5. `Stop - Loop Detected` (`n8n-nodes-base.noOp`) handles the false branch.

Why these nodes:
- IMAP + SMTP nodes map directly to inbound/outbound email requirements.
- IF node is the simplest reliable loop gate before expensive LLM inference.
- HTTP Request node keeps Ollama integration generic and transparent.

Prompt engineering choices in `workflow.json`:
- The prompt is dynamic and includes sender, subject, and body:
  - `{{ $('Email Read (IMAP)').item.json.from.address }}`
  - `{{ $('Email Read (IMAP)').item.json.subject }}`
  - `{{ $('Email Read (IMAP)').item.json.text }}`
- It sets clear behavior constraints:
  - Persona: "professional and helpful email assistant"
  - Output control: "Provide ONLY the reply body text"
  - Style control: concise, relevant, polite
- `stream=false` simplifies parsing because Ollama returns a single JSON object with `response`.

Loop prevention logic:
- Condition 1: sender must not equal `SMTP_USER` (`notEquals`).
- Condition 2: subject must not start with `Re: AI Auto-Reply:` (`notStartsWith`).
- Conditions use `AND`, so only safe emails proceed to Ollama.

## 2) Handling potential failure points
What I implemented now:
- Docker-level resilience:
  - `restart: unless-stopped` for long-running services.
  - Health checks for both `n8n` and `ollama`.
  - `n8n` depends on healthy `ollama` before startup.
  - `ollama-init` waits for Ollama and pulls `llama3:8b` automatically.
- Workflow-level timeout:
  - HTTP Request timeout is set to `120000` ms.

What happens on failure in current workflow:
- If Ollama is unavailable or times out, the HTTP node fails and execution stops (visible in n8n Executions).
- If IMAP/SMTP rejects auth/connection, the respective email node fails and execution is logged as failed.

What is not yet implemented in this workflow:
- No explicit retry/backoff node chain.
- No alternate fallback response path.
- No dedicated error workflow (`settings.errorWorkflow` is empty).

## 3) Scaling to thousands of emails per hour: bottlenecks and mitigations
Primary bottlenecks:
1. LLM inference throughput in Ollama (largest bottleneck).
2. n8n execution concurrency/queue under burst load.
3. IMAP polling frequency and SMTP provider rate limits.
4. Model context size and prompt/response token latency.

How I would address them:
- Inference layer:
  - Use a faster/smaller model for first-pass triage.
  - Run multiple Ollama instances behind a routing layer.
  - Use GPU acceleration and quantized models.
- Workflow layer:
  - Run n8n in queue mode with Redis and multiple workers.
  - Separate ingest, classify, and reply into decoupled workflows.
  - Add idempotency keys based on Message-ID to prevent duplicate sends.
- Email layer:
  - Batch outbound sends and enforce provider-aware throttling.
  - Add retry with exponential backoff for transient SMTP/IMAP errors.
  - Move from strict every-minute polling to event-driven/webhook where possible.

## 4) Trade-offs when choosing a different local model
Using a smaller model (for example Phi-3 Mini) vs `llama3:8b`:

- Speed/latency:
  - Smaller model: faster inference, lower CPU/GPU and memory usage.
  - Llama 3 8B: slower but generally better reasoning and instruction-following.
- Quality:
  - Smaller model: higher risk of shallow, generic, or slightly off-tone replies.
  - Llama 3 8B: better relevance, tone control, and nuanced responses.
- Cost/resource:
  - Smaller model: lower hardware requirements, better for high-volume workloads.
  - Llama 3 8B: higher RAM/VRAM footprint and power usage.
- Operational strategy:
  - For volume-heavy systems: use small model by default, route complex emails to 8B model.
  - For quality-first systems: keep 8B as default and accept lower throughput.

Why I used `llama3:8b` here:
- Better response quality for professional email drafting while still practical on local hardware.

## 5) Data privacy advantages of local-first architecture
Compared with cloud API usage, this architecture keeps sensitive content under local control.

Where data resides:
1. Email arrives from provider (IMAP) to n8n runtime only.
2. n8n sends prompt to Ollama over internal Docker network (`ai_responder_net`).
3. Inference runs inside local Ollama container; no third-party LLM endpoint receives email content.
4. Reply is sent out via SMTP from your configured account.
5. Workflow state and credentials stay in local n8n volume (`n8n_data`), models in local Ollama volume (`ollama_data`).

Why this is significant:
- No external AI vendor data transfer for prompt/response payloads.
- Lower exposure to third-party logging, retention, and model-training policies.
- Better compliance posture for sensitive or regulated email content.
- Predictable cost and control because inference is self-hosted.

Residual privacy caveat:
- Email still transits your email provider (IMAP/SMTP), so local-first here means local AI inference, not provider-independent email transport.
