# Academic Regulation and Student Petition Triage Service

**MLOps Mini Project 15 - ML System Design and AgentOps**  
Department of CSE (AI & ML), PES University  
Project report | 15 November 2026

## Abstract
University attendance petitions can require applying a baseline rule together with a separate exception. This project builds a local, evidence-grounded decision-support prototype: synthetic petitions enter through an asynchronous API, a bounded LangGraph workflow retrieves University of Delhi Ordinance VII clauses, and a separate verification role checks citations before the service recommends a route. Discretionary and unsupported cases go to an examination officer; the system does not make final decisions. The deterministic evaluation improved from 19/20 to 20/20 scenarios after a retrieval fix. Five-user Locust runs handled about 16 HTTP requests per second with zero errors under a temporary test rate limit. The measured HTTP-request p95 was 48 ms with one worker and 52 ms with three workers; complete petition p95 was not measured. No generative model is called, so external model cost was $0 per 100 runs; local infrastructure cost was not metered.

**Index Terms -** AgentOps, academic regulation, evidence-grounded retrieval, LangGraph, citation verification, human review, MLOps.

## I. Problem and Scope
Examination staff receive attendance-shortage petitions and related claims. Reviewing a case may require finding a general attendance rule and a separate medical-hardship provision. Returning only the general rule can omit important conditions; treating a discretionary exception as an entitlement can overstate what the regulation permits. The project therefore builds a review aid that retrieves supporting clauses, verifies their text against a local corpus, and refers uncertain or discretionary cases to a human officer.

The prototype uses synthetic petitions and a narrow public University of Delhi Ordinance VII corpus. It does not adjudicate claims, establish a student's official attendance, or determine which regulations apply to a real student's batch or programme. An authorized examination officer must confirm applicability and make every final decision.

## II. Regulation Corpus and Evidence
The corpus is derived from the University of Delhi University Calendar containing Ordinance VII [1]. Clause VII(11)(a) sets a two-thirds attendance baseline for specified semesters and describes a conditional discretionary route for some students with at least 40 percent attendance. Clause VII(11)(c) permits the Principal to consider serious illness or accident using medical certificates, potentially exclude lectures from attendance calculations, and decide each case on its merits. Clauses VII(11)(e) and VII(11)(f) describe a claim window and an exclusion cap.

Each clause record in `data/corpus.json` includes a stable document ID, clause ID, source text, source URL, authority, and applicability notes. Clause-sized records preserve citation identity. The corpus is checked in; the optional FAISS index is derived and reproducible through `python -m app.ingest`. A locally configured sentence-transformers model with FAISS can be used; deterministic lexical retrieval is the offline fallback. Synthetic data only is used in the application and evaluation.

## III. Agents, Tools, and Architecture
The API validates petition fields before storing a run in SQLite and placing a job on Redis. A worker executes a LangGraph workflow with distinct responsibilities:

- **Intake and classification** identifies supported petition categories and uses a field-lookup tool to read structured petition fields.
- **Regulation analyst** searches for the baseline clause, then makes an independent derived search for modifiers, conditions, and authority. Attendance cases search both the baseline and medical-hardship categories.
- **Verification officer** performs exact clause-ID lookup, compares retrieved text with the corpus record, and applies conservative routing. Discretionary medical cases are referred to a human.

The workflow has three tools: structured field lookup, corpus search, and exact clause lookup. If citation verification fails, it makes at most one bounded recovery search and verifies again. It then terminates or escalates; the worker also applies a 60-second timeout. This recovery cap prevents an unbounded agent loop.

### Design decisions and alternatives
The project uses a deterministic local workflow instead of a hosted generative model. This keeps the demo reproducible and avoids external model credentials, inference charges, and unsupported generated legal interpretations. A model-backed prompt comparison was not implemented. Redis plus SQLite was selected for a clear local asynchronous boundary; a synchronous request that waits for all agent work was rejected because it couples client timeouts to processing. The analyst and verifier are separate roles so the component that retrieves evidence does not approve its own citations. Automatic approval of medical hardship was rejected because the source reserves discretion to the Principal. Real student records and personally identifying information were excluded; all test petitions are synthetic.

**Fig. 1. Deployment boundaries and evidence flow.** The browser is outside the Compose deployment. API, queue, worker, corpus/index, run store, and observability components are inside the local deployment boundary.

| Boundary | Components and flow |
|---|---|
| Client | Browser UI submits a petition and polls its run status over HTTP. |
| Local Docker Compose | FastAPI validates and enqueues to Redis; the worker runs intake, analysis, and verification; corpus/index provide evidence; SQLite stores run and feedback state. |
| Observability services | Prometheus scrapes API metrics; Grafana displays Prometheus data; Jaeger receives OpenTelemetry traces. |

The deployment contains six Compose services: API, agent worker, Redis, Prometheus, Grafana, and Jaeger. The browser is a client, not a container. The REST interface includes `POST /v1/runs` (202 Accepted with a run ID), `GET /v1/runs/{id}`, feedback submission, `/health`, and `/metrics`. A browser UI is served at `/`.

## IV. Reliability, Guardrails, and Operations
Pydantic validation rejects malformed requests before enqueue and returns field-level 422 errors. The API applies a fixed-window rate limit to petition submissions. The worker records failure without returning a confident recommendation. Retrieval can fall back to lexical search when vector dependencies or the local model are unavailable; search results are cached in-process. Structured logs carry run and correlation IDs. Prometheus metrics cover requests, queue depth, agent steps, latency, tokens, and estimated cost; OpenTelemetry spans are exported to Jaeger.

Petition text is untrusted input. The deterministic graph does not send free text to a generative model or let it select tools, so prompt-injection text cannot trigger a model tool call. Citations must match exact corpus records. The repository includes a runbook, health checks, and a development incident entry. Grafana's sample credentials are for local demonstration only.

## V. Evaluation and Measured Results
### Scenario evaluation before and after the retrieval fix
The 20-case synthetic suite covers attendance boundaries, medical hardship, unsupported request types, missing or invalid fields, injection wording, and clause cross-references. The initial run passed 19 of 20 cases. Scenario E09 failed because the retrieval keyword handling did not recognize “illness” as medical context. After adding that term, the same suite passed 20 of 20 cases. This is a deterministic scenario pass rate, not a claim of production accuracy or a prompt A/B comparison. The final observed run records a first-case latency of 11.18 seconds and subsequent measured cases mostly around 5-10 milliseconds; the first case appears to include startup/warm-up and is not used as a service SLO.

### Load test before and after rate-limit adjustment
Locust exercised asynchronous submission and status polling for approximately one minute with five users in the worker comparison. The initial middleware rate-limited both submissions and polling from a shared client IP, producing 92% failures at about 10.9 requests per second, mainly HTTP 429 responses. After limiting the middleware to `POST /v1/runs`, polling stopped failing, but a separate run under the normal 30-per-minute cap still showed 35 rejected submissions and 28% overall failures. For the worker comparison only, the submission limit was temporarily raised to 1,000 per minute.

| Locust run | Workers | Total HTTP requests | Failures | Throughput | HTTP-request p95 |
|---|---:|---:|---:|---:|---:|
| Initial limiter behavior | 1 | - | 92% | 10.9 req/s | Not reported |
| Temporary 1,000/min limit | 1 | 991 | 0 | 16.02 req/s | 48 ms |
| Temporary 1,000/min limit | 3 | 990 | 0 | 16.16 req/s | 52 ms |

The one-worker and three-worker runs show no meaningful throughput gain from adding workers at this load. They do not isolate the bottleneck: the initial failures were caused by the rate limiter, while the later comparison may have been constrained by the small fixed client workload, API/queue overhead, or polling pattern. More users and controlled worker-load measurements are needed before drawing a capacity conclusion.

### Latency target and cost
For the next iteration, the target for individual API requests under the five-user synthetic load is **p95 at or below 250 ms**. The measured request p95 of 48 ms (one worker) and 52 ms (three workers) met this target in these runs. The separate **proposed end-to-end petition completion target is p95 at or below 10 seconds**, but full petition completion p95 was not measured; Locust's reported p95 is for individual HTTP requests and must not be presented as petition completion latency.

The application makes no generative-model calls. External model inference cost is therefore **$0 per 100 synthetic runs** in this implementation. CPU, memory, electricity, and Docker-host operating costs were not measured, so total infrastructure cost per 100 runs is unknown.

## VI. What Did Not Work and Remaining Limitations
The first evaluation missed illness-related medical context until the keyword was added. The first load test exposed overbroad rate limiting because status polling shared the same limit as submissions. Restricting the limit to submissions fixed polling errors; high submission bursts can still receive 429 responses under the normal limit. Three workers did not improve measured throughput over one worker in the small comparison.

The prototype has no dead-letter queue for malformed raw Redis jobs, no external-model retry/circuit breaker, no model token or per-agent cost attribution, no prompt A/B experiment, and no automatically sampled live-quality chart. Its measured p95 applies only to individual HTTP requests. End-to-end petition p95 and a workload that saturates the workers remain unmeasured. Token and model-cost counters remain zero because no model call occurs. The results use a 20-case synthetic suite and should not be generalized to real student petitions. Final review remains with examination staff.

## References
[1] University of Delhi, “University Calendar: Ordinance VII,” public PDF. https://www.du.ac.in/uploads/new-web/10062022_University-Calendar-123-321.pdf

[2] LangChain, “LangGraph,” official documentation. https://langchain-ai.github.io/langgraph/

[3] Prometheus, “Prometheus documentation.” https://prometheus.io/docs/

