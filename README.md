# Academic Regulation and Student Petition Triage Service

Local, evidence-grounded decision support for **synthetic** academic petitions. The prototype retrieves University of Delhi Ordinance VII attendance clauses, checks citations against the corpus, and routes discretionary or unsupported cases to a human officer. It does not issue official decisions.

## Start locally

**Prerequisite:** Docker Desktop with Docker Compose v2.

From the project folder, run:

```powershell
docker compose up --build -d
docker compose ps
```

Open the web form at <http://localhost:8000/>. The API documentation is at <http://localhost:8000/docs>. You can also view service health at <http://localhost:8000/health>, Prometheus metrics at <http://localhost:8000/metrics>, Prometheus at <http://localhost:9090>, Grafana at <http://localhost:3000>, and Jaeger traces at <http://localhost:16686>. Grafana allows anonymous read-only access in this local demo.

### Example synthetic request (PowerShell)

```powershell
$p = @{
  petition_id = 'SYN-01'
  petition_type = 'attendance'
  text = 'Serious illness caused attendance shortage; a medical certificate is available. Please review the attendance rule.'
  attendance_percent = 62
  medical_documented = $true
  semester = 'I'
} | ConvertTo-Json

$run = Invoke-RestMethod -Method Post -Uri 'http://localhost:8000/v1/runs' -ContentType 'application/json' -Body $p
do {
  Start-Sleep -Milliseconds 500
  $status = Invoke-RestMethod "http://localhost:8000$($run.status_url)"
} while ($status.status -in @('queued', 'running'))
$status.result
```

The example should be referred for human review because the medical-hardship provision is discretionary. Use only synthetic information.

## Services and commands

Compose starts six services: FastAPI API, Redis queue, separate agent worker, Prometheus, Grafana, and Jaeger. The browser UI is a client, not a Compose service. SQLite run and feedback records and Redis queue data use named volumes.

To scale the worker for a local comparison:

```powershell
docker compose up -d --scale agent-worker=3
```

The Locust profile is in `locustfile.py`. With Locust installed, run `locust -f locustfile.py --host http://localhost:8000` and use the Locust page to start a load. Locust measures individual HTTP request latency; this is different from complete petition-processing time.

The checked-in corpus works with deterministic lexical retrieval by default. To enable local vector embeddings, set a free local model identifier in `EMBEDDING_MODEL` in your local `.env`, then rebuild the index:

```powershell
docker compose run --rm agent-worker python -m app.ingest
```

Run the 20-scenario deterministic evaluation from an environment where the project dependencies are installed:

```powershell
python -m pip install -r requirements.txt
python evals/run_eval.py
```

The result is written to `evals/latest-results.json`. Stop the services with `docker compose down`. `docker compose down -v` also deletes the local demo data.

## Evaluation and load-test results

The synthetic evaluation initially passed **19 of 20** scenarios. Scenario E09 exposed a missed medical-context keyword (“illness”). After adding that keyword, the same evaluation passed **20 of 20** scenarios. This is a result on the included synthetic scenarios, not a production accuracy claim.

Locust initially reported **92% failures at about 10.9 requests/second**, mainly because both submissions and status polling shared the rate limit. After rate limiting was restricted to submissions, a run at the normal cap still rejected burst submissions (28% overall failures). For the worker comparison, the submission limit was temporarily raised to 1,000 per minute:

| Worker count | HTTP requests | Failures | Throughput | HTTP-request p95 |
|---:|---:|---:|---:|---:|
| 1 | 991 | 0 | 16.02 requests/s | 48 ms |
| 3 | 990 | 0 | 16.16 requests/s | 52 ms |

These approximately one-minute tests used five users. The three-worker run showed no meaningful throughput benefit at this load. The measured p95 values are for individual HTTP requests, not complete petitions. The proposed target for individual API request p95 is **250 ms** under this test load; both runs met it. The proposed complete petition p95 target is **10 seconds**, but it has not been measured. See `docs/evidence/` for the Locust exports and screenshots.

The application makes no model calls, so external model inference cost is **$0 per 100 runs**. Local compute and operating costs were not metered.

## Data and safety

Use synthetic petitions only. Never enter real student identifiers, academic records, or medical information. Review [docs/corpus-sources.md](docs/corpus-sources.md) for sources and scope, [docs/limitations.md](docs/limitations.md) for known gaps, [docs/runbook.md](docs/runbook.md) for recovery steps, and [docs/architecture.md](docs/architecture.md) for the system topology.

## Project map

```text
app/                 API, LangGraph roles, retrieval, worker, SQLite and metrics
data/corpus.json     Curated University of Delhi Ordinance VII clause excerpts
evals/               20 scenarios, prompt versions, evaluation runner and results
docs/                Architecture, corpus sources, runbook, limitations and evidence
monitoring/           Prometheus scrape configuration
grafana/              Grafana datasource provisioning
report/                Editable project report source
outputs/               Final report PDF
Dockerfile             Shared API/worker image
docker-compose.yml     Six-service local stack
locustfile.py          Synthetic stub load-test profile
```
