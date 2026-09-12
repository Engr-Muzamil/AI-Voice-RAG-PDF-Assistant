# Setup Guide

Follow these in order. Ingestion first, agent second. If you activate the agent before any vectors exist, it will return empty results and look broken.

## 1. Create the Pinecone index

1. Log in to Pinecone and create a new index.
2. Name it exactly `projects`. The workflows reference this name directly.
3. Set the dimension to match the Mistral embedding model's output dimension. Check the current value in Mistral's embeddings documentation before creating the index. If the dimension is wrong, inserts will fail and the error message will not be obvious.
4. Use cosine as the metric.

## 2. Prepare the Google Drive folder

1. Create a folder in Google Drive and put your source documents in it.
2. Open the folder and copy the ID from the URL. It is the part after `/folders/`.
3. Keep this ID handy for step 5.

The ingestion workflow reads every file in this folder with no filter. Only put documents you actually want embedded in there.

## 3. Add credentials in n8n

Create these three credentials under Settings, Credentials:

| Credential type | What you need |
|---|---|
| Mistral Cloud API | Your Mistral API key |
| Pinecone API | Your Pinecone API key |
| Google Drive OAuth2 API | Google Cloud OAuth client ID and secret, then authorize |

For Google Drive OAuth2 you need a Google Cloud project with the Drive API enabled and an OAuth consent screen configured. n8n's own documentation covers this step in detail.

## 4. Import the workflows

1. In n8n, choose Import from File.
2. Import `workflows/02_knowledge_ingestion.json`.
3. Import `workflows/01_rag_agent.json`.

Both will import with placeholder values. They will not run yet.

## 5. Wire up the ingestion workflow

Open `knowledge_ingestion` and fix these nodes:

- **Search files and folders**: select your Google Drive credential, then pick your folder from the list (this replaces `REPLACE_WITH_YOUR_DRIVE_FOLDER_ID`).
- **Download file**: select the same Google Drive credential.
- **Embeddings Mistral Cloud**: select your Mistral credential.
- **Pinecone Vector Store**: select your Pinecone credential and confirm the index is `projects` with mode set to `insert`.

Optionally open **Recursive Character Text Splitter** and set an explicit chunk size and overlap. Starting points are 1000 characters with 200 overlap, but tune them against your own documents.

## 6. Run ingestion

Click Execute Workflow. Watch the Loop Over Items node process each file.

Verify in the Pinecone console that the `projects` index now has a non-zero vector count. Do not skip this check. If the count is zero, something failed silently and the agent will be useless.

## 7. Wire up the agent workflow

Open `rag_agent` and fix these nodes:

- **Webhook**: n8n generates a fresh path on import. Note the production URL.
- **Mistral Cloud Chat Model** and **Mistral Cloud Chat Model1**: select your Mistral credential on both. There are two chat model nodes, one for the agent and one for the vector store tool. Both need credentials.
- **Embeddings Mistral Cloud**: select your Mistral credential. This must be the same model used during ingestion.
- **Pinecone Vector Store**: select your Pinecone credential, index `projects`.

## 8. Recommended fixes before you rely on it

These are not required to make it run, but the agent is not production-ready without them. See the Known Limitations section in the README.

**Fix the input expression.** Open the AI Agent node. The text field reads `{{ $json.body }}`. Change it to point at a specific field, for example `{{ $json.body.query }}`, and document which field your clients must send.

**Add a system prompt.** In the AI Agent node options, add a system message. Something along these lines:

```
You answer questions about the projects in the knowledge base.

Always search the Projects tool before answering.
If the tool returns nothing relevant, say you do not have that
information. Do not answer from general knowledge.
Keep answers under four sentences unless asked for detail.
```

**Add authentication.** Open the Webhook node and set Authentication to Header Auth. Create a header auth credential. Without this, the URL is the only thing protecting your API spend.

## 9. Activate and test

Toggle the agent workflow to Active, then:

```bash
curl -X POST https://your-n8n-host/webhook/<your-path> \
  -H "Content-Type: application/json" \
  -d '{"query": "What projects are in the knowledge base?"}'
```

## Troubleshooting

**Empty or irrelevant answers.** Check the Pinecone vector count first. If it is zero, ingestion did not work. If it is non-zero, the embedding model used at query time probably differs from the one used at ingestion time.

**Pinecone insert fails.** Almost always a dimension mismatch between the index and the embedding model.

**The agent ignores the knowledge base.** With no system prompt, the model decides on its own whether to call the tool. Adding an explicit instruction to always search first fixes this.

**Duplicate results after re-running ingestion.** Expected. The workflow has no deduplication. Clear the index before re-running, or add a delete step.

**Webhook returns 404.** The workflow is not active, or you are using the test URL instead of the production URL.
