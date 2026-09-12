# n8n RAG Agent (Mistral + Pinecone)

A two-workflow Retrieval-Augmented Generation system built in n8n. One workflow ingests documents from Google Drive into a Pinecone vector index. The other exposes an AI agent over an HTTP webhook that answers questions using that index as a tool.

No code to deploy. Import two JSON files into n8n, connect four credentials, and it runs.

## What it does

**Ingestion:** Reads every file in a Google Drive folder, downloads them, splits them into chunks, embeds the chunks with Mistral, and writes them to a Pinecone index called `projects`.

**Retrieval:** Accepts a POST request, passes the question to an AI agent, and the agent decides whether to query the Pinecone index before answering.

## Architecture

```
INGESTION  (run manually, on demand)

  Manual Trigger
        |
  Google Drive: list files in folder
        |
  Google Drive: download file
        |
  Loop Over Items  <---------------+
        |                          |
  Pinecone Vector Store (insert) --+
        ^
        |
  Default Data Loader  <--  Recursive Character Text Splitter
        ^
        |
  Mistral Embeddings


RETRIEVAL  (always on, webhook)

  POST /webhook/<path>
        |
    AI Agent  <--- Mistral Chat Model (magistral-small-latest)
        |
        +--- tool: "Projects" (Vector Store QA)
        |            |
        |            +-- Pinecone Vector Store (retrieve)
        |            |         ^
        |            |         +-- Mistral Embeddings
        |            |
        |            +-- Mistral Chat Model (answer synthesis)
        |
  Respond to Webhook
```

Both workflows share the same Pinecone index and the same embedding model. That is not optional. If the ingestion workflow and the retrieval workflow use different embedding models, the vectors will not match and retrieval returns nonsense.

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Orchestration | n8n | Visual workflows, no deployment step |
| LLM | Mistral `magistral-small-latest` | Cheap, fast enough for Q&A |
| Embeddings | Mistral Cloud embeddings | Same provider, one API key |
| Vector store | Pinecone (index: `projects`) | Managed, no infra to run |
| Document source | Google Drive | Non-technical users can drop files in a folder |
| Interface | n8n Webhook | Any client can POST to it |

## Requirements

- n8n (self-hosted or cloud) with the LangChain nodes available
- A Mistral Cloud API key
- A Pinecone account with an index named `projects`
- A Google account with a Drive folder holding your documents

Your Pinecone index dimension must match the Mistral embedding model's output dimension. Create the index with the correct dimension before running ingestion, otherwise the insert fails.

## Setup

See [docs/SETUP.md](docs/SETUP.md) for the full walkthrough. Short version:

1. Import `workflows/02_knowledge_ingestion.json` and `workflows/01_rag_agent.json` into n8n.
2. Create the four credentials in n8n: Mistral Cloud, Pinecone, Google Drive OAuth2.
3. Open each node marked `REPLACE_WITH_YOUR_...` and select your own credential and folder.
4. Run the ingestion workflow once, manually. Confirm vectors appear in Pinecone.
5. Activate the agent workflow. Copy the production webhook URL.
6. Send a test request.

## Usage

```bash
curl -X POST https://your-n8n-host/webhook/<your-path> \
  -H "Content-Type: application/json" \
  -d '{"query": "What projects are in the knowledge base?"}'
```

## Known limitations

These are real and unfixed. Listed here so nobody is surprised.

- **There is no system prompt.** The agent node receives the user input with no instructions about its role, its scope, or when to refuse. It will answer confidently from its own training data when retrieval returns nothing. Adding a system message is the single highest-value change to this repo.
- **The input expression is fragile.** The agent reads `{{ $json.body }}`, which is the entire request body object rather than a string. It should read a specific field, for example `{{ $json.body.query }}`. As written, the input the model sees depends on how n8n stringifies the object.
- **No conversation memory.** Every request is independent. Follow-up questions like "and what about the second one?" will not work. Add a memory node to fix this.
- **No error handling.** If Pinecone times out or Mistral rate-limits, the workflow fails and the caller gets a raw n8n error. There is no retry, no fallback, and no friendly error response.
- **No authentication on the webhook.** Anyone who has the URL can query the knowledge base and spend your API credits. Add header auth before putting this anywhere public.
- **Text splitter uses defaults.** Chunk size and overlap are not configured, so n8n's defaults apply. These have not been tuned against the document set, which directly affects retrieval quality.
- **The response is raw.** `Respond to Webhook` is set to `allIncomingItems`, so the caller gets the agent's full item structure rather than a clean `{"answer": "..."}` payload.
- **Ingestion is not incremental.** Re-running it re-embeds and re-inserts every file in the folder. There is no deduplication and no delete-before-insert, so repeated runs create duplicate vectors.
- **This is not a voice assistant.** There is no speech-to-text or text-to-speech in either workflow. It is a text interface over HTTP.

## Roadmap

- [ ] Add a system prompt with scope and refusal rules
- [ ] Fix the input expression to read a named field
- [ ] Add a memory node for multi-turn conversations
- [ ] Add webhook header authentication
- [ ] Return a clean JSON response shape
- [ ] Tune chunk size and overlap, and measure retrieval quality
- [ ] Make ingestion incremental (track processed file IDs)
- [ ] Add error branches with a user-facing fallback message
- [ ] Optional: add STT and TTS nodes for an actual voice interface

## Repo layout

```
.
├── README.md
├── LICENSE
├── .gitignore
├── .env.example
├── docs/
│   └── SETUP.md
└── workflows/
    ├── 01_rag_agent.json
    └── 02_knowledge_ingestion.json
```

## A note on the JSON files

The workflow exports in this repo have been scrubbed. Credential IDs, the n8n instance ID, the webhook ID and path, and the Google Drive folder ID were replaced with placeholders. n8n exports do not contain API keys themselves, but they do contain identifiers that are specific to one instance, and those do not belong in a public repository.

## License

MIT. See [LICENSE](LICENSE).
