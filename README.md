# DocGenie

An internal multi-agent document chatbot platform for Moneyboxx Finance.

Drop a folder of documents onto the dashboard. The platform reads them, indexes
them, and writes the agent's own name, description, persona prompt and starter
questions. No forms are filled in. Answers are produced by a multi-stage
retrieval pipeline and always carry verifiable citations back to the exact page
or timestamp they came from.

---

## Contents

1. [Quick start](#quick-start)
2. [How it works](#how-it-works)
3. [Vector database choice](#vector-database-choice)
4. [The retrieval pipeline](#the-retrieval-pipeline)
5. [Figures: answering with images](#figures-answering-with-images)
6. [Zoho Desk: learning from resolved tickets](#zoho-desk-learning-from-resolved-tickets)
7. [Hindi and other languages](#hindi-and-other-languages)
8. [Screens](#screens)
9. [Configuration](#configuration)
10. [Running without Docker](#running-without-docker)
11. [Offline development mode](#offline-development-mode)
12. [Verifying an installation](#verifying-an-installation)
13. [Design decisions and deviations](#design-decisions-and-deviations)
14. [Adding authentication later](#adding-authentication-later)
15. [Troubleshooting](#troubleshooting)

---

## Quick start

```bash
cp .env.example .env
```

Put your key in `OPENAI_API_KEY`, then:

```bash
docker compose up --build
```

Open <http://localhost:8080>.

The app boots and renders with no key configured. A banner explains that
indexing and chat will fail until one is set, and everything else stays usable.

Three sample documents live in `sample-documents/`. Drag the three topic files
onto the drop zone on the Chat Agents page to see the whole flow.

---

## How it works

```
                     ┌──────────────┐
   browser ──────────│   frontend   │   React, Vite, nginx
                     │   port 8080  │   serves the app, proxies /api
                     └──────┬───────┘
                            │  /api/v1
                     ┌──────┴───────┐
                     │     api      │   FastAPI, uvicorn, ffmpeg
                     │   port 8000  │   ingestion queue lives in process
                     └──┬────────┬──┘
                        │        │
              ┌─────────┘        └──────────┐
     ┌────────┴────────┐          ┌─────────┴────────┐
     │  app-data vol   │          │    vectordb      │
     │  uploads +      │          │    Qdrant        │
     │  SQLite         │          │  one collection  │
     └─────────────────┘          │  per knowledge   │
                                  │  base            │
                                  └──────────────────┘
```

Ownership runs `KnowledgeBase -> Source -> Document`, with agents attached to
knowledge bases many to many, and `Agent -> Conversation -> Message` for chat.

**Ingestion.** An upload writes the file to disk, creates a `Document` row in
`queued`, and hands the id to an in-process asyncio queue with a bounded worker
pool (three at a time by default). Each worker parses, chunks, summarises,
embeds in batches and upserts. A failure is caught per document: its status
becomes `failed` with the error text persisted, and the other files in the same
batch carry on. Documents interrupted by a restart are requeued on boot.

**Chunking.** A recursive token-aware splitter targets 700 tokens with 120
overlap. Every chunk is embedded with a contextual header built from its own
metadata:

```
Document: RemoteKYCLink.md | Section: Verification > Retry rules
```

The header is stored with the chunk and shown to the model with it, so a
retrieved passage always says what it is and where it came from. Building it
from metadata costs nothing. Setting `CONTEXTUAL_CHUNKS_LLM=true` additionally
asks the utility model for a one or two sentence situating note per chunk. That
code path is complete and off by default, because it costs one call per chunk.

**Isolation.** Each knowledge base owns one Qdrant collection. Retrieval for a
chat only ever queries the collections of the knowledge bases attached to that
agent, so one agent cannot read another's documents.

---

## Vector database choice

**Qdrant**, running as a container in the compose stack.

Weighed against the criteria in the specification:

1. **Self-hosted with no external dependency.** Qdrant is one container with one
   volume. The same `qdrant-client` also runs the store embedded on local disk
   when `QDRANT_URL` is empty, which is what makes `Running without Docker`
   below possible with no second process.
2. **Native per-collection isolation.** Creating or dropping a collection is one
   cheap API call, so one collection per knowledge base is practical rather than
   theoretical. Isolation is physical rather than a metadata filter that a
   future query could forget to apply.
3. **Comfortable at low millions of vectors on one server.** HNSW with optional
   scalar quantization, and a memory-mapped storage path, on a Rust core.
4. **Fetch by id and filter on payload.** Neighbour expansion needs exactly
   this: given a chunk, pull chunk indices `±1` within the same document. That
   is a `scroll` with a `document_id` match and a `chunk_index` `MatchAny`.
5. **Simple, actively maintained Python client.** Typed models, no ceremony.

Rejected alternatives:

- **Chroma.** Genuinely simpler to start, but weaker filtering and a less
  convincing story at multi-million scale on a corpus meant to grow company
  wide.
- **pgvector.** A good database, but it offers no native per-collection
  isolation. Every agent boundary would become a `WHERE` clause that must never
  be omitted, which is exactly the kind of invariant worth pushing down into the
  storage engine. It would also force Postgres as the application database for a
  workload SQLite handles comfortably.

Swapping the store means writing one class with the same surface as
`QdrantStore` in `backend/app/retrieval/vectorstore.py`.

---

## The retrieval pipeline

Every user message runs the same chain. All parameters are configurable.

| Stage | What happens | Calls |
|---|---|---|
| 1. Rewrite | One structured call turns the message plus recent turns and the rolling summary into a standalone query with pronouns resolved, plus two differently phrased variants. | 1 |
| 2. Search | All three queries embedded in one batched call, then searched in parallel across the agent's collections, top 12 each. | 1 embed |
| 3. Fuse | Reciprocal rank fusion at k=60, deduplicated by chunk id, capped at 20 candidates. | 0 |
| 4. Rerank | One structured call scores every candidate 0 to 10. Top 6 above a threshold of 3 are kept. Invalid output falls back to fusion order. | 1 |
| 5. Expand | Adjacent chunks are pulled from the store and merged where contiguous, so an answer does not die at a chunk boundary. | 0 |
| 6. Assemble | Token-budgeted context in priority order: platform rules, agent prompt, corpus overview, numbered passages, conversation memory, the user's original wording. | 0 |
| 7. Answer | Streamed over server-sent events with inline `[n]` citations. | 1 |
| 8. Follow-ups | Three questions answerable from the same passages. Optional. | 1 |

Three calls per message at the defaults, four with follow-ups on, plus a
periodic memory fold. Every stage records its token usage against the agent, and
those totals appear in the Overview tab and the Usage panel.

**Why the rewrite matters.** Ask "how is the login fee calculated?", then "and
when is that refunded?". The second question embeds to nothing useful on its
own. The rewrite resolves it against the conversation before any search runs.
The user's original wording, not the rewrite, is what the answering model sees.

**Conversation memory.** Every eight turns an older stretch of the conversation
is folded into a rolling summary stored on the conversation row. Context is
always the summary plus the most recent raw turns, so a long chat stays inside
budget without losing the thread.

**When nothing is found.** In `strict` grounding the assistant says it cannot
find this in its documents and, using the corpus overview, names what it can
help with. It does not guess. In `flexible` grounding it may answer from general
knowledge, prefixed with "Not from your documents:" so the reader can tell.

**Extension point.** Stages 1 to 5 are a fixed chain by design. An iterative
agentic loop would replace `run_retrieval()` in
`backend/app/retrieval/pipeline.py` and nothing else.

---

## Figures: answering with images

A guide that says "tap **Collect Fee**" is far more useful next to the
screenshot of that screen. During ingestion every image embedded in a PDF or
Word file is extracted, captioned from the text around it, and written to disk
with a record of where it came from: a page number for PDF, a heading path for
Word.

That locator is the whole mechanism. A figure is only offered to the answering
model when a passage that was **actually retrieved** sits on the same page, or
under the same heading, as the figure. A screenshot cannot be attached to an
answer it has nothing to do with.

The model is given a numbered list of the figures available for this question
with their captions, and told to place `[[figure:2]]` after the step it
illustrates, only where seeing it genuinely helps. Then two things happen
server-side:

- Any marker the model invented, or repeated, is **stripped before anything
  downstream sees it**. A model can produce `[[figure:9]]` when only three were
  offered; the reader must never see a dead marker, and the stored answer must
  match the streamed one.
- Validated figures are returned as a separate event and stored on the message,
  so reloading a chat renders the same images.

Three filters keep decorative images out: anything below
`FIGURE_MIN_WIDTH`/`FIGURE_MIN_HEIGHT`, anything with a strip-like aspect ratio,
and anything repeated on more than two pages, which is how running headers and
footer logos are recognised.

Images are streamed from `/api/v1/figures/{id}` rather than inlined, because a
single screenshot is routinely larger than the whole answer. The widget has its
own figure route which checks that the figure belongs to a knowledge base the
agent can actually read, so a public key cannot be used to enumerate every
image on the installation.

There is no OCR in this version. Figures are extracted as images and cited by
locator and caption; their contents are not read into the index.

---

## Zoho Desk: learning from resolved tickets

The answers your support team already gave are the best documentation you have,
and they are sitting in a ticketing system nobody searches. The connector reads
resolved and closed tickets and turns them into indexed troubleshooting notes.
It is strictly read-only: nothing is ever written back to Zoho.

```
fetch resolved tickets -> redact -> distil -> write a note -> normal ingestion
```

**Redaction runs first, before anything else.** A ticket is written by and about
a real customer, and the useful part is the problem and the fix, never the
person. Mobile numbers, emails, PAN, Aadhaar, IFSC, GSTIN and bank account
numbers are replaced with labelled placeholders before the text reaches a model
and before it reaches the index. That means identifiers are not sent to your
provider and are not sitting in the vector store waiting to be retrieved into an
answer. Rupee amounts and ticket numbers are deliberately left alone.

**Distillation is what makes this worth doing.** A raw ticket thread is mostly
logistics: greetings, "any update?", internal handoffs. Indexing the whole
thread pollutes retrieval, because the noise embeds just as strongly as the fix.
One model call turns each ticket into a question and an answer stated generally
rather than about one customer, and it is told to **reject** tickets that teach
nothing: no real resolution, duplicates, test tickets, callback requests, and
anything that applied only to a single customer's record. Skipped tickets are
listed on the Integrations page with the reason, so the judgement is auditable
rather than silent.

The resulting note is written to disk and pushed through the **normal ingestion
queue**. That is deliberate: a ticket learning is chunked, summarised, embedded
and cited by exactly the same code as an uploaded PDF, so there is one indexing
path to reason about rather than two.

**Re-syncing** is cheap. Each ticket is fingerprinted, so an unchanged ticket
costs nothing on the next run: no model call, no re-index. An edited ticket
replaces its old note. A ticket that stops being useful has its note deleted
rather than left to rot in the index.

**Authentication** uses the self-client refresh-token grant. You generate the
refresh token once in the Zoho API console and put it in the environment, which
avoids running a redirect flow for a server with no public callback URL and
keeps the token out of the application database and therefore out of every
backup of it.

**With no credentials configured** the connector reads five built-in sample
tickets instead of your desk, so redaction, distillation, indexing and retrieval
can all be exercised offline. Like the mock model provider it is never a
fallback: if credentials are present and Zoho rejects them, the error surfaces
rather than quietly serving sample data.

One thing to watch: the learnings land in their own knowledge base, and a
knowledge base nothing reads from is indexed but never retrieved. The
Integrations page says so explicitly and links to the agent that reads it.

---

## Hindi and other languages

Two separate controls, because they solve different problems.

**Answer in Hindi.** A toggle above the composer sets the language for the next
question. The answer is generated in Hindi directly rather than translated
afterwards, which reads better, and the choice is remembered. The same control
exists per agent under Settings for the default.

**Translate this answer.** A button under any answer translates it after the
fact, which is what you want when a colleague needs to forward an answer to
someone else. Translations are cached on the message, so asking twice is not
billed twice.

Two details matter more than they look:

- **Interface text stays in English.** An RM searching for a button labelled
  **Collect Fee** cannot find it if the answer renamed it. Screen names, button
  labels, field names and acronyms like KYC, PAN, IFSC and OTP are kept in
  English with the meaning in brackets on first use. Set
  `TRANSLATION_KEEP_UI_TERMS=false` to translate everything.
- **Citations and figures must survive.** A translation that drops `[1]` or
  `[[figure:2]]` breaks the reader's ability to check the answer. The markers
  are compared before and after, and a translation that lost any of them is
  **rejected** and the English kept, with an explanation, rather than returning
  an answer whose sources no longer resolve.

Add languages with `TRANSLATION_LANGUAGES=hi:Hindi,mr:Marathi,gu:Gujarati`.

---

## Screens

**Chat Agents** is the dashboard: a drop zone and a card grid. Dropping files
creates the agent, and the card shows live build progress through uploading,
reading, indexing, configuring, ready.

**Agent workspace** has five tabs.

- **Overview.** Document, chunk and token counts, chat and message counts,
  ratings, the generated starter questions, and token spend broken down by
  pipeline stage.
- **Chat.** Conversation list with auto-generated titles, streamed markdown
  answers, a status line that reflects the real pipeline stage, a Sources row of
  document chips with their retrieved-passage counts, expandable passages with
  page numbers or timestamps, a Continue exploring row, copy, and thumbs.
- **Knowledge.** Attached knowledge bases, which are append-only, the list of
  ones that can be added, and a drop zone with per-document status, errors and
  summaries.
- **Widget.** Publish the agent as a chat bubble for an internal site, with its
  own colour, greeting, position, starter questions and allowed domains, a live
  preview, and the embed snippet.
- **Settings.** Identity, system prompt, grounding mode, starter questions,
  known issues, answer behaviour, and the danger zone. Every field can be
  regenerated from the documents individually.

**Knowledge Base** holds sources, which are named groups of documents, and the
upload surface for each.

**Audio Insights** lists every media file indexed, with its timestamped
transcript.

**Integrations** holds the Zoho Desk connector: connection state, which
knowledge base the learnings land in, sync controls, and the full list of
tickets read with what was done to each and why.

**Templates**, **Organisation**, **User Management**, **My Account** and
**Help** round out the sidebar.

---

## Configuration

Everything lives in `.env`. `.env.example` documents every variable. The ones
worth knowing:

| Variable | Default | Notes |
|---|---|---|
| `LLM_PROVIDER` | `openai` | `mock` runs offline, see below |
| `LLM_MODEL` | `gpt-5.6-luna` | answering |
| `LLM_MODEL_UTILITY` | `gpt-5.6-luna` | rewrite, rerank, summaries, auto-config |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | changing this needs a reindex |
| `TRANSCRIPTION_MODEL` | `whisper-1` | |
| `QDRANT_URL` | empty | empty means embedded on local disk |
| `INGEST_CONCURRENCY` | `3` | documents processed at once |
| `CHUNK_TARGET_TOKENS` | `700` | with `CHUNK_OVERLAP_TOKENS=120` |
| `RERANK_KEEP` | `6` | passages kept after reranking |
| `RERANK_SCORE_THRESHOLD` | `3.0` | below this, nothing is grounded |
| `CONTEXT_TOKEN_BUDGET` | `12000` | assembled prompt budget |
| `USAGE_MONTHLY_MESSAGE_LIMIT` | `10000` | drives the meter in the chat sidebar |
| `EXTRACT_FIGURES` | `true` | pull images out of PDF and Word files |
| `FIGURE_MAX_PER_ANSWER` | `4` | most figures one answer may show |
| `TRANSLATION_LANGUAGES` | `hi:Hindi` | comma separated `code:name` pairs |
| `TRANSLATION_KEEP_UI_TERMS` | `true` | leave screen and button names in English |
| `ZOHO_DATA_CENTER` | `in` | `in`, `com`, `eu`, `au` or `jp` |
| `ZOHO_SYNC_INTERVAL_MINUTES` | `30` | `0` disables the scheduler |
| `ZOHO_REDACT_PII` | `true` | strip identifiers before a ticket is indexed |

Changing a model is an `.env` edit and a restart, with no code change. If the
provider rejects a model the error is surfaced exactly as it came back, and no
substitute is ever chosen quietly.

Optional request parameters that a particular model does not accept, such as
`temperature` or `max_tokens`, are detected once from the provider's own 400
response, remembered for that model, and dropped from later calls. That keeps
one code path working across model generations without a hardcoded compatibility
table.

---

## Running without Docker

Useful when Docker is not available. The vector store runs embedded, so there is
no second process to start.

**API**

```bash
python -m venv backend/.venv
backend/.venv/Scripts/python -m pip install -r backend/requirements.txt
backend/.venv/Scripts/python -m uvicorn app.main:app --app-dir backend --port 8000
```

On Linux or macOS use `backend/.venv/bin/python` instead.

**Frontend**

```bash
cd frontend && npm install && npm run dev
```

Open <http://localhost:5173>. Set `VITE_API_BASE_URL=http://localhost:8000` in
`.env` so the dev server knows where the API is.

`ffmpeg` is not bundled outside Docker. Without it, video files with no matching
caption file fail with a clear message rather than silently. Upload an `.srt` or
`.vtt` alongside the video and no transcription is needed at all.

---

## Offline development mode

Set `LLM_PROVIDER=mock` and no key is needed, and no request leaves the machine.

The mock provider is a development aid, never a fallback. The OpenAI adapter
never degrades into it, so a bad model name or a missing key still fails loudly.

Its embeddings are a signed hashing-trick bag of words over unigrams and
bigrams, which gives real lexical similarity. Retrieval, fusion, reranking,
neighbour expansion, citation mapping and the whole user interface can therefore
be exercised end to end. Answers are assembled from the retrieved passages
rather than written by a model, so they read mechanically. That is the one thing
offline mode cannot show you.

---

## Verifying an installation

```bash
backend/.venv/Scripts/python backend/scripts/smoke_test.py
```

It runs against a temporary data directory, so nothing you have is touched. It
walks the acceptance criteria end to end: the no-code build flow, grounded
answers with citations, multi-turn pronoun resolution, a question spanning two
documents, agent isolation, strict-mode refusal, settings edits surviving new
uploads, widget gating, and deletion cascades.

It also covers the connected features: figures extracted from a PDF it builds on
the fly and served over HTTP, invented figure markers being stripped, Hindi
translation preserving citations and being cached, redaction removing every
class of identifier while leaving amounts alone, and the full Zoho sync
including the skip decision, the no-op re-sync, and a resolved ticket coming
back as a cited source in a chat answer.

It prints a pass and fail count and exits non-zero on any failure.

It defaults to `LLM_PROVIDER=mock`. Point it at `openai` to exercise the real
models:

```bash
LLM_PROVIDER=openai OPENAI_API_KEY=sk-... backend/.venv/bin/python backend/scripts/smoke_test.py
```

---

## Design decisions and deviations

Two places where this build departs from the written specification, both
deliberate.

**Knowledge bases sit between agents and collections.** The specification asks
for one collection per agent. The product screens show knowledge bases that
several agents can attach to, which is the more useful model: one indexed corpus
of product documentation should serve the RM agent, the COPS agent and the BDO
agent without being uploaded three times. So a **knowledge base** owns the
collection, and an agent may read only from the knowledge bases attached to it.
The isolation guarantee is unchanged. Dropping files on the dashboard still
produces a working agent in one action: it creates the knowledge base and the
agent together.

Attachment is append-only, matching the product rule shown on the Knowledge tab.
Deleting an agent removes any knowledge base created just for it, including
vectors and uploaded files, unless another agent still reads from it.

**The widget is built.** The specification lists an embeddable widget as a
non-goal. The screens include a Widget tab, and the brief was to match them, so
it is implemented: public endpoints keyed by the agent's public key, refused
unless the widget is enabled, an allowed-domain check, an iframe loader that
cannot read the host page, and widget chats recorded on their own channel.

**Interface language.** Flat surfaces, hairline rules, and the Moneyboxx palette
(`#0056AE` blue, `#00458A` deep blue, `#AC2C1F` maroon, with the Excel green,
yellow and red for status). Text is a dark navy rather than black. There are no
gradients anywhere.

---

## Adding authentication later

There is none in v1. This is meant for the internal network.

Every route already depends on `get_current_user` in `backend/app/deps.py`,
which returns a fixed stub. Turning authentication on means implementing that
one function and adding a login screen. No route signature changes. The user
directory and its roles are already recorded under User Management, ready to be
enforced.

---

## Troubleshooting

**The banner says no API key is configured.** Set `OPENAI_API_KEY` in `.env` and
restart the API. To work offline instead, set `LLM_PROVIDER=mock`.

**A document failed to index.** Open the knowledge base and expand the row. The
exact error is stored against the document. The usual cause is a scanned PDF:
there is no OCR in this version, so a page with no extractable text is skipped
and recorded as a warning.

**A knowledge base says it needs reindexing.** The embedding model in `.env` no
longer matches the one the collection was built with, so its vectors are not
comparable. Search is disabled for it until it is rebuilt. Use `Reindex now` on
the knowledge base page.

**Video files fail outside Docker.** `ffmpeg` is not installed. Either run in
Docker, install `ffmpeg`, or upload a matching `.srt` or `.vtt` caption file,
which skips transcription entirely.

**An agent refuses a question you expected it to answer.** Check the Knowledge
tab: the document may have failed, or it may not be in a knowledge base attached
to that agent. If retrieval is finding the passage but the answer is refused,
lower `RERANK_SCORE_THRESHOLD` or switch the agent to `flexible` grounding.

**Answers arrive all at once instead of streaming.** Something between the
browser and the API is buffering. The bundled nginx config sets
`proxy_buffering off` for `/api/`; any proxy in front of it needs the same.

**No figures appear in answers.** Check the document actually has extractable
images: open its knowledge base and look at the Figures count, or call
`/api/v1/documents/{id}/figures`. A PDF whose "screenshots" are drawn as vector
graphics rather than embedded images has nothing to extract. Images below
`FIGURE_MIN_WIDTH` or `FIGURE_MIN_HEIGHT`, or repeated on more than two pages,
are filtered as decoration.

**The Zoho sync says it fetched nothing.** The connector reads only tickets
whose status is Closed or Resolved, and only ones modified since the last run.
Use **Full resync** to ignore the watermark, or **Clear sync history** to make
the next run re-read everything from the start.

**Ticket learnings are indexed but never cited.** They land in their own
knowledge base, and an agent can only read from knowledge bases attached to it.
Open the agent's Knowledge tab and attach it. The Integrations page warns about
this and links straight there.
