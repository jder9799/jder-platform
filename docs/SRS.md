# SRS — jder-platform

**Repo:** `jder-platform`
**Language:** Python
**Services:** Crawler, Ingestion/ETL, Recommendation
**Parent SRS:** JDER_SRS.docx (Master)
**Version:** 1.0

---

## 1. Scope

This document covers all Python services in `jder-platform`. It defines the functional requirements, data contracts, infrastructure, and testing expectations for:

- `crawler/` — scrapes Indian job boards, produces to Kafka
- `ingestion/` — consumes raw events, normalises, deduplicates, produces clean events
- `recommendation/` — serves personalised job recommendations via FastAPI + gRPC

---

## 2. Repo Structure

```
jder-platform/
├── docker-compose.yml
├── pyproject.toml
├── proto/
│   └── search.proto
│
├── crawler/
│   ├── Dockerfile
│   ├── pyproject.toml
│   ├── crawler/
│   │   ├── main.py
│   │   ├── tasks.py              # celery task definitions
│   │   ├── config.py
│   │   └── adapters/
│   │       ├── base_adapter.py   # abstract base class
│   │       ├── naukri.py
│   │       ├── foundit.py
│   │       └── internshala.py
│   └── tests/
│
├── ingestion/
│   ├── Dockerfile
│   ├── pyproject.toml
│   ├── ingestion/
│   │   ├── main.py
│   │   ├── consumer.py           # kafka consumer loop
│   │   ├── deduplicator.py
│   │   ├── config.py
│   │   └── normalizers/
│   │       ├── salary.py
│   │       ├── skills.py
│   │       ├── location.py
│   │       └── date.py
│   └── tests/
│
├── recommendation/
│   ├── Dockerfile
│   ├── pyproject.toml
│   ├── recommendation/
│   │   ├── main.py               # fastapi app
│   │   ├── embeddings.py
│   │   ├── ranker.py
│   │   ├── collaborative.py
│   │   ├── config.py
│   │   └── proto_generated/      # generated grpc stubs
│   └── tests/
│
└── shared/
    ├── kafka_client.py
    ├── schemas.py                 # pydantic models shared across services
    └── proto_generated/
```

---

## 3. Crawler Service

### 3.1 Responsibilities

- Schedule and execute scrape jobs against Indian job boards
- Produce raw job events to Kafka topic `raw.jobs`
- Track crawl state per source in Redis
- Handle bot detection gracefully with retries and backoff

### 3.2 Dependencies

```
playwright
playwright-stealth
celery
redis
kafka-python
pydantic
sentry-sdk
```

### 3.3 Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| CR-01 | Crawl Naukri.com every 4 hours via Celery Beat schedule | P0 |
| CR-02 | Crawl Foundit every 6 hours | P0 |
| CR-03 | Crawl Internshala every 12 hours | P0 |
| CR-04 | Use Playwright with playwright-stealth plugin on all crawlers | P0 |
| CR-05 | Rotate residential proxy IP per crawl session | P0 |
| CR-06 | Randomise request delay between 2 and 8 seconds per page | P0 |
| CR-07 | Publish one raw event to `raw.jobs` per listing found | P0 |
| CR-08 | Use `source:source_id` as the Kafka message key | P0 |
| CR-09 | Enter exponential backoff (1s, 2s, 4s) after 3 consecutive page failures | P0 |
| CR-10 | Alert via Sentry when a source has 0 successful listings over 1 hour | P1 |
| CR-11 | Store last successful crawl timestamp per source in Redis | P1 |
| CR-12 | Never block the entire crawl on a single failed page — skip and continue | P0 |

### 3.4 Base Adapter Interface

Every source adapter must implement:

```python
class BaseAdapter:
    source: str                        # "naukri" | "foundit" | "internshala"

    async def crawl(self) -> list[RawJob]:
        raise NotImplementedError

    async def parse_listing(self, page) -> RawJob:
        raise NotImplementedError
```

### 3.5 Raw Job Schema (Kafka `raw.jobs`)

```python
class RawJob(BaseModel):
    event_id: str           # uuid4
    event_type: str         # "job.raw.crawled"
    produced_at: datetime
    source: str             # "naukri" | "foundit" | "internshala"
    source_id: str          # platform-native job ID
    crawled_at: datetime
    url: str
    raw: dict               # unmodified scraped fields
```

`raw` dict must contain at minimum: `title`, `company`, `location_raw`. All other fields are optional — ingestion handles missing fields.

### 3.6 Celery Configuration

```
broker:           Redis
result_backend:   Redis
task_serializer:  json
timezone:         Asia/Kolkata
beat_schedule:
  naukri:         every 4 hours
  foundit:        every 6 hours
  internshala:    every 12 hours
```

### 3.7 Environment Variables

| Variable | Description |
|---|---|
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka broker URL |
| `KAFKA_API_KEY` | Upstash Kafka API key |
| `KAFKA_API_SECRET` | Upstash Kafka API secret |
| `REDIS_URL` | Redis connection URL |
| `PROXY_HOST` | Residential proxy host |
| `PROXY_USERNAME` | Proxy auth username |
| `PROXY_PASSWORD` | Proxy auth password |
| `SENTRY_DSN` | Sentry project DSN |

---

## 4. Ingestion / ETL Service

### 4.1 Responsibilities

- Consume raw events from Kafka `raw.jobs`
- Normalise salary, skills, location, and dates
- Deduplicate within source and across sources
- Publish clean events to Kafka `clean.jobs`
- Publish failed events to `raw.jobs.dlq` after max retries

### 4.2 Dependencies

```
kafka-python
pydantic
spacy                  # NER for skill extraction from descriptions
redis
psycopg2-binary        # for cross-source dedup ground truth
sentry-sdk
```

### 4.3 Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| ET-01 | Normalise all salary formats to `{min: int, max: int, currency: "INR", period: "annual"}` | P0 |
| ET-02 | Map skill variants to canonical form via lookup table | P0 |
| ET-03 | Use spaCy NER to extract implicit skills from job description | P1 |
| ET-04 | Geocode all location strings to `{city, state, lat, lng}` | P0 |
| ET-05 | Resolve relative dates to absolute UTC timestamps | P0 |
| ET-06 | Deduplicate by `source + source_id` (exact duplicate — skip) | P0 |
| ET-07 | Deduplicate cross-source by `company + title + location` within 7-day window | P0 |
| ET-08 | Use Redis bloom filter for fast first-pass dedup check | P1 |
| ET-09 | Publish clean event to `clean.jobs` after successful normalisation | P0 |
| ET-10 | Retry failed events up to 3 times with exponential backoff | P0 |
| ET-11 | Publish failed events to `raw.jobs.dlq` after 3 failed attempts with error metadata | P0 |
| ET-12 | Never crash the consumer loop on a single bad event — catch, DLQ, continue | P0 |
| ET-13 | Expose Prometheus metrics endpoint for consumer lag and processing rate | P1 |

### 4.4 Salary Normalisation Rules

| Raw Format | Normalised |
|---|---|
| `12-18 Lacs PA` | `{min: 1200000, max: 1800000}` |
| `₹50,000/month` | `{min: 600000, max: 600000}` |
| `500000 per annum` | `{min: 500000, max: 500000}` |
| `10-15 LPA` | `{min: 1000000, max: 1500000}` |
| `Not Disclosed` / missing | `null` |

### 4.5 Skills Canonical Map (Starter — Expand Over Time)

| Raw Variants | Canonical |
|---|---|
| `React.js`, `ReactJS`, `React JS` | `React` |
| `Node JS`, `NodeJS`, `Node.js` | `Node.js` |
| `Postgres`, `PostgreSQL`, `psql` | `PostgreSQL` |
| `ML`, `Machine Learning` | `Machine Learning` |
| `JS`, `Javascript`, `JavaScript` | `JavaScript` |
| `K8s`, `Kubernetes` | `Kubernetes` |
| `AWS`, `Amazon Web Services` | `AWS` |

### 4.6 Clean Job Schema (Kafka `clean.jobs`)

```python
class CleanJob(BaseModel):
    event_id: str
    event_type: str         # "job.clean.upserted"
    produced_at: datetime
    job: JobDocument

class JobDocument(BaseModel):
    id: str                 # uuid4 — stable across upserts
    sources: list[str]
    title: str
    company: str
    location: Location
    salary: Salary | None
    experience: ExperienceRange
    skills: list[str]
    description: str
    job_type: str           # "full_time" | "contract" | "internship" | "remote"
    posted_at: datetime
    crawled_at: datetime
    apply_url: str
```

### 4.7 DLQ Event Schema

```python
class DLQEvent(BaseModel):
    original_event: RawJob
    error: str              # exception message
    service: str            # "ingestion-etl"
    failed_at: datetime
    attempt: int            # always 3 (max retries)
```

### 4.8 Environment Variables

| Variable | Description |
|---|---|
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka broker URL |
| `KAFKA_API_KEY` | Upstash Kafka API key |
| `KAFKA_API_SECRET` | Upstash Kafka API secret |
| `KAFKA_CONSUMER_GROUP` | `ingestion-etl` |
| `REDIS_URL` | Redis for bloom filter |
| `DATABASE_URL` | PostgreSQL for dedup ground truth |
| `SENTRY_DSN` | Sentry project DSN |

---

## 5. Recommendation Service

### 5.1 Responsibilities

- Expose gRPC endpoints for personalised recommendations and similar jobs
- Retrieve candidate jobs from Search Service via gRPC
- Re-rank candidates using skill embeddings, recency, behaviour, and popularity signals
- Cache results per user in Redis

### 5.2 Dependencies

```
fastapi
grpcio
grpcio-tools
sentence-transformers
scikit-learn
numpy
redis
psycopg2-binary          # for behaviour signal (saved/applied jobs)
sentry-sdk
prometheus-fastapi-instrumentator
```

### 5.3 Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| RC-01 | Expose `GetRecommendations(user_id, limit, offset)` via gRPC | P1 |
| RC-02 | Expose `GetSimilarJobs(job_id, limit)` via gRPC | P1 |
| RC-03 | Retrieve up to 200 candidate jobs from Search Service via gRPC | P1 |
| RC-04 | Re-rank candidates using weighted score: 45% skill similarity + 25% recency + 20% behaviour + 10% popularity | P1 |
| RC-05 | Generate user skill embeddings using `sentence-transformers/all-MiniLM-L6-v2` | P1 |
| RC-06 | Generate job embeddings using title + top 5 skills | P1 |
| RC-07 | Fall back to profile-based retrieval only for users with no behaviour history | P1 |
| RC-08 | Cache recommendation results in Redis: key `rec:{user_id}:{page}`, TTL 30 minutes | P1 |
| RC-09 | Invalidate cache when user saves or applies to a job | P1 |
| RC-10 | Cache user embeddings in Redis: key `emb:{user_id}`, TTL 1 hour | P1 |
| RC-11 | P95 latency on warm path (cache hit) must be < 300ms | P1 |
| RC-12 | P95 latency on cold path (cache miss) must be < 600ms | P1 |

### 5.4 Scoring Weights

```python
WEIGHTS = {
    "skill_similarity": 0.45,
    "recency":          0.25,
    "behaviour":        0.20,
    "popularity":       0.10,
}
```

Recency score uses exponential decay: `score = e^(-days_since_posted / 14)`. A listing posted today scores 1.0; posted 14 days ago scores ~0.37.

### 5.5 gRPC Contract

Defined in `/proto/recommendation.proto`. Key messages:

```proto
service RecommendationService {
  rpc GetRecommendations(RecommendationRequest) returns (RecommendationResponse);
  rpc GetSimilarJobs(SimilarJobsRequest) returns (SimilarJobsResponse);
}

message RecommendationRequest {
  string user_id = 1;
  int32 limit = 2;
  int32 offset = 3;
}
```

Full proto definition lives in `/proto/recommendation.proto`. Generated Python stubs go in `recommendation/proto_generated/`.

### 5.6 Environment Variables

| Variable | Description |
|---|---|
| `SEARCH_SERVICE_GRPC_HOST` | Search Service gRPC host:port |
| `REDIS_URL` | Redis for caching |
| `DATABASE_URL` | PostgreSQL for behaviour signals |
| `SENTRY_DSN` | Sentry project DSN |
| `GRPC_PORT` | Port to expose gRPC server (default: 9003) |

---

## 6. Shared Module

`shared/` contains code used across all three services.

| File | Contents |
|---|---|
| `kafka_client.py` | Producer and consumer wrappers with retry logic |
| `schemas.py` | Pydantic models: `RawJob`, `CleanJob`, `JobDocument`, `DLQEvent` |
| `proto_generated/` | Generated gRPC stubs for `search.proto` |

All services import from `shared/` via relative path. Do not duplicate schemas across services.

---

## 7. Kafka Topics

| Topic | Producer | Consumer | Partitions | Retention |
|---|---|---|---|---|
| `raw.jobs` | Crawler | Ingestion/ETL | 6 | 7 days |
| `clean.jobs` | Ingestion/ETL | Search Service (jder-core) | 6 | 14 days |
| `raw.jobs.dlq` | Ingestion/ETL | Manual | 1 | 30 days |
| `user.events` | User Service (jder-core) | Notification Service (jder-core) | 3 | 3 days |

`jder-platform` produces to `raw.jobs` and `clean.jobs`. It does not produce to or consume from `user.events`.

---

## 8. Testing Requirements

| Layer | Tool | Requirement |
|---|---|---|
| Unit | pytest | All normaliser functions must have unit tests with real-world raw format samples |
| Integration | pytest + Testcontainers | Crawler → Kafka → Ingestion flow tested against a real Kafka container |
| Contract | pydantic validation | Every event published to Kafka must pass schema validation before publish |
| Performance | locust | Recommendation service cold path must sustain 50 req/s under 600ms p95 |

CI runs unit and integration tests on every push. Performance tests run manually before release.

---

## 9. Local Development

```bash
# Start all infra (Kafka, Redis, Postgres, Elasticsearch)
docker compose up -d

# Run crawler locally (single manual trigger, no Celery)
cd crawler && doppler run -- python -m crawler.tasks trigger_naukri

# Run ingestion consumer
cd ingestion && doppler run -- python -m ingestion.main

# Run recommendation service
cd recommendation && doppler run -- python -m recommendation.main
```

All secrets injected via Doppler CLI. No `.env` files.
