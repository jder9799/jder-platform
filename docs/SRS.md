# **JDER**

Software Requirements Specification

Version 1.0 | Confidential

# **1\. Introduction**

## **1.1 Purpose**

This Software Requirements Specification defines the functional and non-functional requirements for Jder, an India-specific job search aggregator platform. It is the technical reference for all system design, development, and testing decisions.

## **1.2 System Overview**

Jder is a microservice platform consisting of two primary codebases (Python services, Java services) and a Next.js frontend. It scrapes Indian job boards, normalises and indexes listings into Elasticsearch, and serves search and personalised recommendations to end users via a GraphQL API.

## **1.3 Definitions**

| **Term**    | **Definition**                                                                   |
| ----------- | -------------------------------------------------------------------------------- |
| Listing     | A normalised job posting stored in Elasticsearch                                 |
| Raw event   | Unprocessed scraped data published to Kafka topic raw.jobs                       |
| Clean event | Normalised job data published to Kafka topic clean.jobs                          |
| CTC         | Cost to Company - Indian standard for annual salary                              |
| ETL         | Extract, Transform, Load - the ingestion/normalisation pipeline                  |
| gRPC        | Remote Procedure Call protocol using Protobuf for internal service communication |
| DLQ         | Dead Letter Queue - Kafka topic for failed events after max retries              |

# **2\. System Architecture**

## **2.1 Services**

| **Service**            | **Language** | **Runtime**    | **Responsibility**                                                       |
| ---------------------- | ------------ | -------------- | ------------------------------------------------------------------------ |
| API Gateway            | Java         | GraalVM Native | Single entry point. GraphQL federation. Auth enforcement. Rate limiting. |
| User Service           | Java         | GraalVM Native | Auth, profiles, saved jobs, job alerts.                                  |
| Search Service         | Java         | JVM            | Elasticsearch queries, geo filters, facets, caching.                     |
| Notification Service   | Java         | JVM            | Email and push notifications via Kafka events.                           |
| Crawler Service        | Python       | CPython        | Scrapes Naukri, Foundit, Internshala. Publishes to raw.jobs.             |
| Ingestion/ETL          | Python       | CPython        | Normalises, deduplicates, publishes to clean.jobs.                       |
| Recommendation Service | Python       | CPython        | Two-stage retrieve and re-rank personalised recommendations.             |

## **2.2 Communication Protocols**

- External (client to gateway): GraphQL over HTTPS
- Internal synchronous (service to service): gRPC with Protobuf
- Internal asynchronous (pipeline and events): Apache Kafka

## **2.3 Data Stores**

| **Store**     | **Usage**                                              | **Managed By**                           |
| ------------- | ------------------------------------------------------ | ---------------------------------------- |
| Elasticsearch | Job listings, full-text search, geo queries            | Elastic Cloud (India region)             |
| PostgreSQL    | Users, alerts, tracking - schema-per-service isolation | Railway (Phase 1), RDS (Phase 2)         |
| Redis         | Search cache, sessions, Celery broker, rate limiting   | Railway (Phase 1), ElastiCache (Phase 2) |
| Apache Kafka  | raw.jobs, clean.jobs, user.events pipelines            | Upstash (Phase 1), MSK (Phase 2)         |

# **3\. Functional Requirements**

## **3.1 Crawler Service**

| **ID**    | **Category** | **Requirement**                                                                   | **Priority** |
| --------- | ------------ | --------------------------------------------------------------------------------- | ------------ |
| **CR-01** | Crawling     | System shall crawl Naukri.com every 4 hours                                       | P0           |
| **CR-02** | Crawling     | System shall crawl Foundit every 6 hours                                          | P0           |
| **CR-03** | Crawling     | System shall crawl Internshala every 12 hours                                     | P0           |
| **CR-04** | Crawling     | Crawler shall use Playwright with stealth plugin to avoid bot detection           | P0           |
| **CR-05** | Crawling     | Crawler shall rotate residential proxy IPs per session                            | P0           |
| **CR-06** | Crawling     | Crawler shall publish a raw event to Kafka topic raw.jobs for every listing found | P0           |
| **CR-07** | Reliability  | Crawler shall enter exponential backoff after 3 consecutive failures per source   | P0           |
| **CR-08** | Reliability  | Crawler shall alert via Sentry when any source fails for more than 1 hour         | P1           |

## **3.2 Ingestion / ETL Service**

| **ID**    | **Category**  | **Requirement**                                                                           | **Priority** |
| --------- | ------------- | ----------------------------------------------------------------------------------------- | ------------ |
| **ET-01** | Normalisation | System shall normalise all salary variants to annual INR integer range {min, max}         | P0           |
| **ET-02** | Normalisation | System shall map skill variants to canonical form (e.g. React.js, ReactJS -> React)       | P0           |
| **ET-03** | Normalisation | System shall geocode all location strings to {city, state, lat, lng}                      | P0           |
| **ET-04** | Normalisation | System shall resolve relative dates (e.g. '30+ Days Ago') to absolute timestamps          | P0           |
| **ET-05** | Deduplication | System shall deduplicate by source_id within the same source                              | P0           |
| **ET-06** | Deduplication | System shall deduplicate cross-source by company + title + location within a 7-day window | P0           |
| **ET-07** | Deduplication | System shall use Redis bloom filter for fast first-pass deduplication check               | P1           |
| **ET-08** | Pipeline      | System shall publish clean events to Kafka topic clean.jobs after normalisation           | P0           |
| **ET-09** | Pipeline      | Failed events shall be published to raw.jobs.dlq after 3 retry attempts                   | P0           |

## **3.3 Search Service**

| **ID**    | **Category** | **Requirement**                                                                           | **Priority** |
| --------- | ------------ | ----------------------------------------------------------------------------------------- | ------------ |
| **SR-01** | Search       | System shall support full-text search across job title, skills, and description           | P0           |
| **SR-02** | Search       | System shall support geo-distance filtering by city name or lat/lng + radius              | P0           |
| **SR-03** | Search       | System shall support filtering by salary range (normalised INR)                           | P0           |
| **SR-04** | Search       | System shall support filtering by experience range (years)                                | P0           |
| **SR-05** | Search       | System shall support filtering by job type (full_time, contract, internship, remote)      | P0           |
| **SR-06** | Search       | System shall support filtering by date posted (within N days)                             | P0           |
| **SR-07** | Search       | System shall return faceted aggregations (city, job_type, salary range) with every result | P1           |
| **SR-08** | Search       | Search results shall be ranked by relevance score with recency boost                      | P0           |
| **SR-09** | Caching      | System shall cache frequent search queries in Redis with a 15-minute TTL                  | P1           |
| **SR-10** | Pagination   | System shall support pagination via from/size for pages 1-10, search_after beyond         | P1           |

## **3.4 User Service**

| **ID**    | **Category** | **Requirement**                                                                             | **Priority** |
| --------- | ------------ | ------------------------------------------------------------------------------------------- | ------------ |
| **US-01** | Auth         | System shall support email and password registration                                        | P0           |
| **US-02** | Auth         | System shall issue JWT tokens on successful login (24-hour expiry)                          | P0           |
| **US-03** | Auth         | System shall issue refresh tokens (30-day expiry)                                           | P0           |
| **US-04** | Auth         | API Gateway shall validate JWT on every request via gRPC call to User Service               | P0           |
| **US-05** | Profile      | User shall be able to set skills, experience years, preferred locations, salary expectation | P1           |
| **US-06** | Saved Jobs   | User shall be able to save and unsave job listings                                          | P1           |
| **US-07** | Alerts       | User shall be able to create job alerts with search criteria                                | P1           |
| **US-08** | Alerts       | System shall evaluate alerts against new clean.jobs events and trigger notifications        | P1           |

## **3.5 Recommendation Service**

| **ID**    | **Category**       | **Requirement**                                                                                    | **Priority** |
| --------- | ------------------ | -------------------------------------------------------------------------------------------------- | ------------ |
| **RC-01** | Retrieval          | System shall retrieve up to 200 candidate jobs from Search Service via gRPC                        | P1           |
| **RC-02** | Ranking            | System shall re-rank candidates using skill similarity, recency, behaviour, and popularity signals | P1           |
| **RC-03** | Embeddings         | System shall generate skill embeddings using sentence-transformers (all-MiniLM-L6-v2)              | P1           |
| **RC-04** | Cold Start         | System shall fall back to profile-based matching for users with no behaviour history               | P1           |
| **RC-05** | Caching            | Recommendation results shall be cached in Redis per user per page with 30-minute TTL               | P1           |
| **RC-06** | Cache Invalidation | Cache shall be invalidated when user saves or applies to a job                                     | P1           |

# **4\. Non-Functional Requirements**

## **4.1 Performance**

| **Requirement**                          | **Target**      | **Measurement**                           |
| ---------------------------------------- | --------------- | ----------------------------------------- |
| Search response time (p95)               | < 300ms         | End-to-end from gateway to client         |
| Recommendation response time (p95, warm) | < 300ms         | Cache hit path                            |
| Recommendation response time (p95, cold) | < 600ms         | Cache miss path                           |
| Listing freshness                        | < 6 hours       | From crawl to searchable in Elasticsearch |
| Kafka consumer lag (ingestion)           | < 1000 messages | Measured per partition                    |

## **4.2 Reliability**

- API Gateway uptime: > 99.5%
- Crawler success rate per source: > 90% over any 24-hour window
- No data loss on Ingestion service restart - Kafka offset management ensures replay
- DLQ monitored - alert if more than 100 messages accumulate

## **4.3 Security**

- All client-gateway communication over HTTPS
- JWT validation enforced at API Gateway for all authenticated endpoints
- Secrets managed via Doppler - no secrets in version control
- Internal services not publicly exposed - Railway internal DNS only
- Rate limiting enforced at API Gateway via Redis counters

## **4.4 Scalability**

- All services containerised and horizontally scalable
- Elasticsearch index sharded for query parallelism
- Kafka partitions sized to allow up to 6 parallel ingestion consumers
- Phase 2 migration to AWS EKS enables pod-level autoscaling via HPA

# **5\. API Contracts**

## **5.1 External GraphQL API**

All client requests go through the API Gateway at a single /graphql endpoint. Key queries:

searchJobs(input: SearchInput!): SearchResult!

job(id: ID!): Job

recommendations(limit: Int, offset: Int): RecommendationResult!

similarJobs(jobId: ID!, limit: Int): \[Job!\]!

me: User

savedJobs: \[ID!\]!

jobAlerts: \[JobAlert!\]!

Key mutations:

saveJob(jobId: ID!): Boolean!

createAlert(input: CreateAlertInput!): JobAlert!

updateProfile(input: UpdateProfileInput!): User!

## **5.2 Internal gRPC Services**

| **Proto Service**     | **Key RPCs**                                  | **Callers**                         |
| --------------------- | --------------------------------------------- | ----------------------------------- |
| SearchService         | SearchJobs, GetJob                            | API Gateway, Recommendation Service |
| UserService           | GetUser, ValidateToken, SaveJob, GetSavedJobs | API Gateway                         |
| RecommendationService | GetRecommendations, GetSimilarJobs            | API Gateway                         |

## **5.3 Kafka Topics**

| **Topic**    | **Producer**  | **Consumer**         | **Retention** |
| ------------ | ------------- | -------------------- | ------------- |
| raw.jobs     | Crawler       | Ingestion/ETL        | 7 days        |
| clean.jobs   | Ingestion/ETL | Search Service       | 14 days       |
| user.events  | User Service  | Notification Service | 3 days        |
| raw.jobs.dlq | Ingestion/ETL | Manual review        | 30 days       |

# **6\. Key Data Models**

## **6.1 Elasticsearch Job Document**

id: keyword

title: text (job_title_analyzer)

company: text + keyword

skills: text (skill_analyzer) + keyword

location.city: keyword

location.state: keyword

location.geo: geo_point

location.remote: boolean

salary.min: long (annual INR)

salary.max: long (annual INR)

experience.min: integer (years)

experience.max: integer (years)

job_type: keyword

posted_at: date

sources: keyword\[\]

## **6.2 PostgreSQL Schemas**

| **Schema**    | **Tables**                           |
| ------------- | ------------------------------------ |
| users         | users, refresh_tokens, user_profiles |
| alerts        | job_alerts, alert_criteria           |
| tracking      | saved_jobs, application_events       |
| notifications | notification_log, email_queue        |

# **7\. Infrastructure Requirements**

## **7.1 Phase 1 - Railway**

- All services deployed as Docker containers on Railway
- PostgreSQL and Redis via Railway managed services
- Kafka via Upstash managed Kafka
- Elasticsearch via Elastic Cloud India region
- Frontend deployed on Vercel
- Secrets managed via Doppler synced to Railway environment variables

## **7.2 CI/CD**

- GitHub Actions per service - push triggers lint, test, build, deploy
- GraalVM Native Image compilation in CI - expect 3 to 5 minutes per service
- Docker images pushed to GitHub Container Registry
- Separate workflows per service - no full rebuild on unrelated changes

## **7.3 Observability**

| **Concern**         | **Tool**                       |
| ------------------- | ------------------------------ |
| Distributed tracing | OpenTelemetry -> Grafana Tempo |
| Metrics             | Prometheus + Grafana           |
| Log aggregation     | Grafana Loki                   |
| Error tracking      | Sentry (Java + Python SDKs)    |
| Uptime monitoring   | Betterstack                    |
