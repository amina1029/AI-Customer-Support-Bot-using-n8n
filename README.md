# AI Customer Support Bot - RAG-powered automation (n8n)

An automated customer support system that answers customer queries using a company knowledge base, and intelligently decides when to resolve a query itself versus escalating it to a human team. Built entirely in n8n.

The workflow has two independent parts: a **knowledge base ingestion pipeline** and a **live query-handling pipeline**.


---

## Part 1: Knowledge Base Setup

Whenever new information needs to be added to the knowledge base, it is uploaded to Google Drive, which triggers this part of the workflow.

**Flow:**
1. A file is uploaded to a watched Google Drive folder (trigger)
2. The file is downloaded
3. The document is split into chunks using a Recursive Character Text Splitter
4. Each chunk is embedded using Cohere
5. The embeddings are stored in Pinecone as the vector database

This pipeline runs independently of live customer queries: update it whenever company policy or support docs change.

---

## Part 2: Live Query Handling

**Trigger:** A customer submits a query through a Google Form. The response lands in a linked Google Sheet, which triggers this part of the workflow.

### Step 1 — Email Validation
The customer's email is checked for a valid format (regex). If invalid, the row is flagged and logged back to the sheet as "Invalid email"; no further processing occurs.

### Step 2 — Query Validity Check
The AI Agent checks whether the incoming query is coherent or gibberish.
- **Gibberish/nonsensical** → logged to the Google Sheet as an invalid query, and the workflow stops here.
- **Valid query** → passed to the AI Agent for a full response.

### Step 3 — AI Agent Response (RAG)
The AI Agent searches the Pinecone knowledge base to attempt an answer, using a Structured Output Parser to return consistent, parseable output rather than free-form text.

### Step 4 — Routing Logic
Based on the AI Agent's confidence, the response is routed one of two ways:

| **Successful reply** | AI Agent can confidently answer using the knowledge base | Reply sent to the customer via Gmail |
| **Acknowledgment + escalation** | AI Agent cannot confidently answer from the knowledge base | Customer is emailed an acknowledgment that their complaint has been registered and escalated to the team. A Slack alert is also sent to the team with the case details so a human can take over. |

### Step 5 — Logging
Every outcome (processed, error, invalid, in-process) is written back to the same Google Sheet, so it doubles as a live status log of all customer queries.

### Error Handling
Error handling is built into the workflow; failures during processing are caught and logged back to the Google Sheet (rather than failing silently), so no customer query is lost without a trace.

---

## Stack
- **n8n**  workflow orchestration
- **Google Drive**  knowledge base source documents
- **Google Forms + Google Sheets**  customer query intake and logging
- **Pinecone**  vector database
- **Cohere**  embeddings
- **OpenRouter**  LLM for the AI Agent
- **Gmail**  customer email replies
- **Slack**  team escalation alerts

