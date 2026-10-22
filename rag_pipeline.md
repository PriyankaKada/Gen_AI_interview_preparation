# AI Engineer preparation — questions and answers
Concrete answers drawn from [docs/ai-engineer-linkedin-article.md](ai-engineer-linkedin-article.md). Each answer already includes the corrections and suggestions from that note, so you can use it as something to say, not as a list of gaps.
Employer and product names are left out. Where the note describes your effort-estimation platform, say only the parts you have actually built, plus one decision, one failure, and one number you measured.
## Contents
- [Role and how you prepare](#role-and-how-you-prepare)
- [Project](#project)
- [RAG pipeline and bad retrieval](#rag-pipeline-and-bad-retrieval)
- [Exact constraints](#exact-constraints)
- [Evaluation](#evaluation)
- [Embedding model changes](#embedding-model-changes)
- [Cost and duplicate uploads](#cost-and-duplicate-uploads)
- [Agents](#agents)
- [Python backend](#python-backend)
- [Coding agents and security](#coding-agents-and-security)
- [System design](#system-design)
- [What to build and how to present yourself](#what-to-build-and-how-to-present-yourself)
---
## Role and how you prepare
### Question 1. What does this role actually involve?
**Answer.** This is a backend AI engineer role. The work is the system around the model, not writing a prompt and calling an API. The product is an AI workspace: people upload documents, ask questions, search the web, and run agents that use tools and remember a long conversation. I would build that platform and take it to production.
Five areas show up repeatedly. Retrieval is first: I should walk a document from upload to answer, and also explain why a correct fact in a PDF can still produce a wrong answer. Agents are second: tool calling, planning, state, retries, checkpoints, memory, and a coding agent that can read a repository, edit files, and run commands without becoming dangerous. Python backend is third: Django/DRF or FastAPI, async I/O, REST, authentication, PostgreSQL, Redis, queues, caching, and background workers. Those services also call hosted or open-source models, web search, document processing, OCR, speech, and multimodal models, and they have to be observable with logs, metrics, Docker, and a cloud environment I can debug in production. Production behavior is fourth: streaming, observability, latency, token cost, rate limits, retries, and deployment. The differentiator is developer tooling: index a codebase, retrieve the right symbols, apply a patch, and verify it.
### Question 2. How are you preparing, and how do you judge whether you understand a component?
**Answer.** I am preparing in six modules: Python backend, LLM fundamentals, production RAG, agentic workflows, coding agents, and system design. For every component I should be able to say why I chose it, how I implemented it, what fails in production, and how I would measure it.
---
## Project
### Question 3. Walk me through a project you would use in this interview.
**Answer.** My concrete example is a GenAI effort-estimation platform. It takes software requirements and recommends a stack, an effort estimate, a timeline, and a cost.
The reference shape has a Python API in front, retrieval over embeddings with a vector store and relational metadata, and a workflow that analyzes, retrieves, estimates, and validates, ending in a structured result. I treat that as a reference architecture and I only claim the pieces I have actually built.
I prepare three follow-ups in advance: one technical decision, one failure, and one number I measured.
---
## RAG pipeline and bad retrieval
### Question 4. Explain the complete RAG pipeline from document upload to the final answer.
**Answer.** A document is parsed, cleaned, split into chunks, and stored with metadata. Each chunk is embedded and written to a vector store. At query time I transform the question if needed, embed it with the same model that indexed the documents, retrieve candidates, optionally rerank them, assemble a context that fits the token budget, generate the answer, and evaluate both retrieval and the answer. If I cannot name where a failure sits on that path, I am not ready to change the prompt.
### Question 5. Users say answers are wrong even though the correct fact is in the uploaded PDF. How do you investigate and improve retrieval?
**Answer.** I do not start by changing the prompt. I trace the request so I can see which stage ran and which chunks came back.
The model usually does not retrieve anything. The path is query, query embedding, initial retrieval, optional reranking, context construction, then generation. If the right chunk is in the first result set and missing from the final context, the bug is reranking or context selection. If it never appears in the first result set, reranking cannot save it. I look at ingestion, chunking, embeddings, filters, and the search strategy first.
When the file is stored and the top five chunks are still wrong, I look at chunk boundaries first. Fixed-size token chunks, semantic chunks, and model-proposed chunks all fail when one fact is split across two chunks. I fix that with overlap, structure-aware splits, and parent-child chunks, so a small matched piece can still pull in the section it came from.
Exact strings fail for a different reason. A question like "projects that took exactly 10 man-days and used Java" needs the tokens `10` and `Java`, not only a nearby meaning. BM25 ranks by term frequency, inverse document frequency, and document length. Vector search ranks by meaning. Hybrid search sends the same query to both and fuses the lists, usually with reciprocal rank fusion. I do not split the user question into two queries to do that.
Metadata filters are the next check. A filter on tenant, document, project, date, or type can drop the right chunk before ranking starts. I use filters to scope the search, and I confirm they are not excluding the evidence.
Then I widen the candidate set, often to 20–50 chunks, and rerank with a cross-encoder. I pick the final context from relevance, token budget, and diversity. Before and after each change I measure Recall@K, MRR, or nDCG on a set of questions with known relevant chunks.
---
## Exact constraints
### Question 6. The user asks for projects executed in exactly 10 man-days with a Java backend. Hybrid search returns projects of 8, 10, and 12 man-days. What do you change?
**Answer.** Similarity is the wrong tool for an equality constraint. Hybrid search can still surface 8, 10, and 12 because those descriptions are semantically close.
If the fields live in a relational database, I filter there first, and I put both constraints in the query, not only the number of days:
```sql
SELECT project_id, project_name, backend, effort_days
FROM projects
WHERE effort_days = 10
  AND backend = 'Java';
```
The model may summarize those rows. It must not decide whether a project meets the constraint. If the fields exist only inside PDFs, I extract them into validated records at ingestion time, keep the source span, and flag uncertain extractions for review. Semantic search is for the description and the supporting context after the structured candidates exist.
---
## Evaluation
### Question 7. The team says answer quality improved, but there is no measurement. How do you evaluate the RAG pipeline and stop regressions?
**Answer.** "Answers feel better" is not evidence. I build a golden set: questions, the chunks that should be retrieved, and a reference answer. I score retrieval and generation separately.
Retrieval uses Recall@K, Precision@K, MRR, and nDCG. Recall@5 asks whether the known chunk is among the five candidates. Answer quality uses correctness, relevance, faithfulness, and citation accuracy. Faithfulness asks whether each claim is supported by the context that was actually supplied. A system can be faithful and still useless: it honestly says it cannot find the answer because retrieval never fetched the evidence. Semantic similarity between the generated answer and a reference answer is a signal, not proof of factual correctness. I attach the source to each chunk so grounding can be checked.
I also track latency, tokens, cost per request, and failure rate. In production I watch thumbs up and down, task completion, and how often people regenerate. Regeneration, and asking the same question the next day, are weak signals. They are not proof the previous answer was wrong. An LLM-as-judge can score relevance and groundedness, and it needs the source on each chunk so it can see the evidence.
Regression testing means running the same fixed set against the old pipeline and the new one, inspecting the failures, and refusing the release if retrieval or answer metrics fall through a threshold. In production I break those metrics down by document type, query category, tenant, and model version.
---
## Embedding model changes
### Question 8. After you deploy a new embedding model, answer quality drops sharply. The API returns 200, answers sound fluent, and latency is almost unchanged. How do you debug it?
**Answer.** The vector store can be healthy and still be searching the wrong space. An embedding is a point in a space that belongs to one model. Model A might emit 768 dimensions. Model B might emit 1,536. Even when the dimensions match, the spaces are not interchangeable.
Four failures follow. The store rejects the query because the dimension does not match the index. Or the dimension matches, the space does not, and similarity scores become noise. That second case is the quiet failure, and it matches a 200 response with fluent text. Or ranking changes, so relevant chunks fall below the cutoff. Or preprocessing changes: normalization, prefixes, or different encoding for documents and queries.
I debug in this order. I confirm the regression on a fixed set and see whether it hits every query or only some document types. I compare the model id, dimension, and preprocessing used at index time with those used at query time. I inspect top-k chunks and scores to see if the right chunk is missing or only ranked low. If the index holds the old model's vectors, I re-embed into a new index rather than mixing the two. I evaluate Recall@K and answer quality, roll out in stages, and roll back if the threshold is missed. Chunking, filters, thresholds, and the reranker are the next suspects only after the embedding pair is compatible.
### Question 9. Model A is 768 dimensions, Model B is 1,536, and queries now fail with a dimension mismatch. How do you migrate without taking production offline?
**Answer.** I do not rewrite the live index in place, and I do not pad or truncate vectors so they fit the old collection. I use a blue-green migration. Collection A and model A keep serving traffic while a background job builds collection B with model B.
I create `documents_v2` and leave `documents_v1` alone. I read chunks from the source of truth, embed them with the new model in batches, and upsert into the new collection. Batches bound memory, cost, and rate limits, and a resumable worker can restart without starting over.
Writes that arrive during the backfill have to land in both indexes. A change log or a transactional outbox records inserts, updates, and deletes. The new collection is not ready until it has caught up to a checkpoint. A PDF uploaded five minutes after the job starts must not exist only in the old index.
Before the switch I compare both indexes on recall, ranking, answer correctness, faithfulness, latency, embedding cost, metadata filters, and whether recent updates and deletes are present. More dimensions are not automatically a better index.
A versioned router then picks the embedder and the collection together. In-flight requests stay on the version they started with. A small share of traffic can go to the new pair first. If quality drops, the router points back at the old pair, which stays available until the new one has been stable.
The short version: keep the old model and index live, build a new collection, re-embed in batches, dual-write changes, validate against a golden set, switch with a versioned config, and keep the old index for rollback.
---
## Cost and duplicate uploads
### Question 10. The new embedding model costs about 40 percent more per month. How do you reduce embedding and retrieval cost without hurting answer quality?
**Answer.** I do not start by caching popular retrieved chunks. That does not help if the cost is in creating vectors. I split the bill into document embeddings, query embeddings, repeated work, and the compute if I host the model myself. Then I try six moves, in this order.
First I cache an embedding under the content hash, the model version, and the preprocessing version. I cache a query embedding under the normalized question and the model version. I cache a retrieval result under the query, the filters, the tenant, and the index version. If I leave out the tenant or the index version, I can serve another user's documents or a stale index. I never reuse a vector from the old model as if it belonged to the new one.
Second, I embed incrementally. If 100,000 chunks are reprocessed nightly and only 2 percent changed, that is 2,000 embedding calls instead of 100,000, about a 98 percent reduction in those operations, before provider quirks and duplicates.
Third, I batch requests to cut overhead. Batching does not always lower the price per token, so I put it next to rate limits, a bounded queue, and retries.
Fourth, I try a cheaper model on the same golden set and compare recall, rank, and answer quality before switching.
Fifth, I stop embedding headers, boilerplate, duplicate files, and tiny chunks. Structure-aware chunks keep precision without a huge index.
Sixth, at high, steady volume, a self-hosted open-source embedding model can be cheaper per request. I compare GPU, operations, and scaling with the API price, not the API price alone.
I report cost per document, cost per query, and retrieval quality before and after, so a saving that harms answers does not count.
### Question 11. A user uploads the same 100-page PDF five times. How do you stop repeated ingestion and still handle real updates?
**Answer.** Parsing, chunking, and embedding that file on every upload is wasted work. Skipping a file because the filename matches is the wrong shortcut. The same name can hide new bytes. A new upload time can hide the same bytes.
I hash the file and I hash each chunk. If the file hash and the model version are already indexed for that tenant, I return the existing document. If the file is new, I embed only chunks whose content hash and model version are missing. If the file changed, I embed the chunks that changed and delete the chunks that disappeared. A document of 1,000 chunks with 20 edits should cause 20 embedding calls.
Metadata and vectors have to move together. An outbox or a single write path records the chunk id, the hash, the model version, and the source location. A delete removes both the row and the vector. Two models never share one collection. During a migration, changes go to both collections until the router switches.
---
## Agents
### Question 12. When do you use LangChain, and when do you use LangGraph?
**Answer.** I use LangChain for reusable model, prompt, retrieval, and tool pieces. I use LangGraph when the workflow needs explicit state, branches, loops, checkpoints, and a controlled way to resume.
### Question 13. What happens internally when a model calls a tool?
**Answer.** The model emits a structured request. My code validates the tool name and the arguments, checks authorization, runs the function, and returns the result to the model. The model does not get to execute the tool by itself.
### Question 14. How do you stop an agent from looping forever?
**Answer.** I set a max iteration count, a time budget, and a tool-call budget, and I detect repeated actions. When any of those trips, the workflow stops or asks for a decision instead of calling tools again.
### Question 15. How do you implement memory for a long conversation?
**Answer.** I persist conversation events, retrieve the relevant slice, and keep a short summary. I separate durable facts from the recent turn, and I apply access control to both. The whole history is not pasted into every prompt.
### Question 16. When should the workflow be ordinary code instead of an agent?
**Answer.** Calculations, authorization, and stable business rules stay in ordinary code. The model chooses tools only where interpretation or task decomposition is actually useful.
---
## Python backend
### Question 17. Explain async/await in FastAPI, and where it goes wrong.
**Answer.** Async helps when the handler is waiting on network I/O, because the event loop can serve other requests during that wait. A blocking call inside that handler stalls the event loop. Database drivers, file reads, and model calls on the request path have to be non-blocking, or they have to run outside the event loop.
### Question 18. How do you implement long-running document ingestion?
**Answer.** I do not parse and embed inside the HTTP request. I validate the upload, store it, enqueue a job, and return a job id. A worker parses and embeds, records status, and retries transient failures safely.
### Question 19. How do you cache LLM responses in Redis?
**Answer.** The cache key includes the normalized input, the model, the prompt version, the retrieval version, and the generation settings, plus a TTL and a tenant boundary. Without the tenant boundary, one user's answer can be served to another. Without the retrieval version, a stale context can be reused after the index changes.
### Question 20. How do you handle duplicate requests and retries?
**Answer.** Duplicate API calls carry an idempotency key, and the database enforces a uniqueness constraint so the second call does not create a second job or a second charge. Retries are bounded. A retry of a completed idempotent request returns the original result.
### Question 21. How do you stop a slow model provider from taking down the API?
**Answer.** I contain it with timeouts, concurrency limits, rate limits, a circuit breaker, a bounded queue, and a fallback. The request path does not wait forever, and a saturated provider does not fill the process with blocked work.
---
## Coding agents and security
### Question 22. Design a CLI agent that understands and modifies an existing codebase.
**Answer.** The agent discovers files and honors ignore rules. It indexes functions and classes with their paths, retrieves a symbol and its neighbors, plans a change, proposes a patch, and runs tests that were approved. It does not apply a patch and declare success without that verification.
### Question 23. How should that agent run shell commands?
**Answer.** Commands run in a sandbox as a least-privilege identity, with an allowlist, resource limits, a timeout, and no open network unless the task needs it. Model-generated commands are not trusted.
### Question 24. How do you defend against prompt injection in retrieved documents?
**Answer.** Retrieved documents are untrusted data. Instructions inside a PDF do not become permission to call a tool. Authorization stays on the server, outside the model, and I test injection cases on purpose.
### Question 25. How do you build retrieval over a codebase?
