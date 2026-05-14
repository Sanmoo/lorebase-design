# lorebase Backend - Technical Design Specification

**Date:** 2026-04-18  
**Status:** Approved for Implementation  
**Author:** OpenCode (with user collaboration)  

---

## 1. Executive Summary

This document specifies the technical architecture and implementation approach for the lorebase backend system, a serverless application for tracking TV series consumption with personalized recommendations.

**Key Decisions:**
- **Architecture:** Serverless-first using AWS Lambda, DynamoDB, API Gateway, and Cognito
- **Pattern:** Clean Architecture for domain contexts, Layered Architecture for BFF
- **Repository Structure:** Monorepo with multiple Go modules
- **Data Source:** TMDB API with hybrid caching strategy
- **Communication:** HTTP synchronous (MVP), Event-driven (future)

---

## 2. Domain Context Overview

Based on DDD modeling (see: `docs/domain-modeling/lorebase.cml`):

| Bounded Context | Type | Technology | Responsibility |
|----------------|------|------------|----------------|
| **SeriesCatalogContext** | Supporting Domain | Go + AWS Lambda | Series metadata, seasons, episodes info |
| **UserTrackingContext** | Core Domain | Go + AWS Lambda | User progress, ratings, watch lists |
| **RecommendationContext** | Core Domain | Go + AWS Lambda | Idle-time suggestions from "Want to Watch" list |
| **ExperienceBFFContext** | Backend for Frontend | Go + AWS Lambda | API aggregation and optimization for Flutter |

**Key Principle:** No Shared Kernel. Each context defines its own Value Objects (UserId, SeriesId, etc.) to maintain strict isolation.

---

## 3. System Architecture

### 3.1 High-Level Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      AWS Cloud                               │
│                                                              │
│  ┌──────────────┐      ┌──────────────────┐                  │
│  │  API Gateway │      │   EventBridge    │                  │
│  │   (Public)   │      │  (Future Events) │                  │
│  └──────┬───────┘      └──────────────────┘                  │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────┐        │
│  │          Experience BFF (Lambda)                 │        │
│  │  ┌──────────────┐  ┌────────────────────────┐    │        │
│  │  │ Auth Middleware│  │ Aggregation Layer      │    │        │
│  │  │ (Cognito JWT)  │  │ (Parallel HTTP calls)  │    │        │
│  │  └──────────────┘  └────────────────────────┘    │        │
│  └──────┬──────────────┬──────────────┬──────────────┘        │
│         │              │              │                      │
│         ▼              ▼              ▼                      │
│  ┌────────────┐ ┌──────────────┐ ┌────────────────┐          │
│  │  Series    │ │  User        │ │ Recommendation │          │
│  │  Catalog   │ │  Tracking    │ │   (Stateless)  │          │
│  │  (Lambda)  │ │  (Lambda)    │ │   (Lambda)     │          │
│  └─────┬──────┘ └──────┬───────┘ └────────────────┘          │
│        │               │                                     │
│        │    ┌──────────▼─────────┐                           │
│        │    │    HTTP (sync)      │                           │
│        └────►  (Validation call)  │                           │
│             └────────────────────┘                           │
│                                                              │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────┐        │
│  │DynamoDB    │  │DynamoDB      │  │DynamoDB        │        │
│  │series-     │  │user-         │  │(Not used in    │        │
│  │catalog     │  │tracking      │  │MVP)            │        │
│  └────────────┘  └──────────────┘  └────────────────┘        │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ HTTP/WebSocket
                              ▼
                    ┌─────────────────┐
                    │   Flutter App   │
                    │  (Separate Repo)│
                    └─────────────────┘
```

### 3.2 Communication Patterns

**Current (MVP):**
- **BFF → Domain Contexts:** HTTP synchronous for all operations
- **Domain → Domain:** HTTP synchronous (UserTracking validates Series existence with SeriesCatalog)
- **External → SeriesCatalog:** HTTP to TMDB API (with caching)

**Future (Post-MVP):**
- EventBridge for domain events (SeriesUpdated, TrackingCreated, etc.)
- Asynchronous processing for long-running operations

### 3.3 Lambda Function URLs (Inter-Context Communication)

For communication between bounded contexts (e.g., UserTracking → SeriesCatalog), we use **Lambda Function URLs** with IAM authentication.

**Why Function URLs:**
- No API Gateway cost for internal calls
- HTTP interface (RESTful, testable)
- Native IAM authentication (secure)
- Low latency (10-20ms same-region)

**Architecture:**
```
UserTracking Lambda ──HTTP──► SeriesCatalog Lambda
                                    │
                          Lambda Function URL
                          (with AWS_IAM auth)
```

**Terraform Configuration:**
```hcl
# SeriesCatalog Lambda with Function URL
resource "aws_lambda_function_url" "series_catalog" {
  function_name      = aws_lambda_function.series_catalog_api.function_name
  authorization_type = "AWS_IAM"
}

# IAM Policy allowing UserTracking to call SeriesCatalog
resource "aws_iam_role_policy" "user_tracking_invoke_catalog" {
  name = "invoke-series-catalog"
  role = aws_iam_role.user_tracking_lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = "lambda:InvokeFunctionUrl"
        Resource = aws_lambda_function.series_catalog_api.arn
        Condition = {
          StringEquals = {
            "lambda:FunctionUrlAuthType" = "AWS_IAM"
          }
        }
      }
    ]
  })
}
```

**Go Client Implementation:**
```go
// /shared/pkg/aws/lambda_client.go
package aws

import (
    "context"
    "net/http"
    "github.com/aws/aws-sdk-go-v2/aws"
    v4 "github.com/aws/aws-sdk-go-v2/aws/signer/v4"
)

type LambdaClient struct {
    signer  *v4.Signer
    creds   aws.Credentials
    region  string
}

func (c *LambdaClient) Invoke(ctx context.Context, url string, body []byte) (*http.Response, error) {
    req, _ := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
    
    // Sign request with AWS credentials
    err := c.signer.SignHTTP(ctx, c.creds, req, "lambda", c.region, time.Now())
    if err != nil {
        return nil, err
    }
    
    return http.DefaultClient.Do(req)
}
```

**Usage:**
```go
// UserTracking calls SeriesCatalog
client := aws.NewLambdaClient(cfg)
resp, err := client.Invoke(ctx, 
    "https://<id>.lambda-url.us-east-1.on.aws/validate",
    jsonBody,
)
```

**Environment Variables:**
```bash
SERIES_CATALOG_URL=https://<id>.lambda-url.us-east-1.on.aws
USER_TRACKING_URL=https://<id>.lambda-url.us-east-1.on.aws
RECOMMENDATION_URL=https://<id>.lambda-url.us-east-1.on.aws
```

---

## 4. Repository Structure

### 4.1 Monorepo Layout

```
/lorebase-backend/
├── go.work                          # Go workspace (Go 1.26+)
├── README.md
├── Makefile                         # Build, test, deploy commands
│
├── /shared/                         # Shared utilities (NO business logic)
│   ├── go.mod
│   └── /pkg/
│       ├── /aws/                   # AWS clients (DynamoDB, EventBridge, etc.)
│       ├── /http/                    # Common middleware, response helpers
│       ├── /errors/                  # Standardized error handling
│       └── /logger/                  # Structured JSON logging
│
├── /series-catalog/                # Bounded Context: SeriesCatalog
│   ├── go.mod
│   ├── /cmd/
│   │   └── /api/
│   │       └── main.go               # Lambda entrypoint
│   ├── /internal/
│   │   ├── /domain/                  # Entities, Value Objects
│   │   │   ├── series.go
│   │   │   ├── season.go
│   │   │   └── value_objects.go      # SeriesId, etc.
│   │   ├── /application/             # Use cases, service layer
│   │   │   ├── create_series.go
│   │   │   ├── search_series.go
│   │   │   └── sync_from_tmdb.go
│   │   ├── /infrastructure/          # Repository implementations
│   │   │   ├── series_repository.go  # DynamoDB implementation
│   │   │   └── tmdb_client.go        # TMDB API client
│   │   └── /ports/                   # Interfaces (driven + driving)
│   ├── /terraform/                   # Context-specific infrastructure
│   └── /scripts/
│       └── deploy.sh
│
├── /user-tracking/                 # Bounded Context: UserTracking
│   ├── go.mod
│   ├── /cmd/
│   │   └── /api/
│   │       └── main.go
│   ├── /internal/
│   │   ├── /domain/
│   │   │   ├── aggregate/
│   │   │   │   └── user_series_tracking.go
│   │   │   ├── entity/
│   │   │   │   └── tracked_series.go
│   │   │   └── valueobject/
│   │   │       ├── tracking_id.go
│   │   │       ├── user_id.go       # Independent from other contexts
│   │   │       ├── series_id.go     # Independent from other contexts
│   │   │       └── season_rating.go
│   │   ├── /application/
│   │   │   ├── start_tracking.go
│   │   │   ├── update_progress.go
│   │   │   ├── rate_series.go
│   │   │   └── list_tracking.go
│   │   ├── /infrastructure/
│   │   │   ├── tracking_repository.go
│   │   │   └── series_catalog_client.go  # HTTP client to validate series
│   │   └── /ports/
│   └── /terraform/
│
├── /recommendation/                # Bounded Context: Recommendation (STATELESS)
│   ├── go.mod
│   ├── /cmd/
│   │   └── /api/
│   │       └── main.go
│   ├── /internal/
│   │   ├── /handlers/
│   │   │   └── recommendation.go     # Simple handler, no domain layer
│   │   └── /clients/
│   │       └── user_tracking_client.go
│   └── /terraform/
│
├── /experience-bff/                # Bounded Context: ExperienceBFF
│   ├── go.mod
│   ├── /cmd/
│   │   └── /api/
│   │       └── main.go
│   ├── /internal/
│   │   ├── /handlers/               # HTTP handlers (Layered Architecture)
│   │   │   ├── dashboard.go
│   │   │   ├── series.go
│   │   │   ├── tracking.go
│   │   │   └── recommendation.go
│   │   ├── /aggregator/             # Aggregation logic
│   │   │   └── dashboard_builder.go
│   │   ├── /clients/                # HTTP clients to domain contexts
│   │   │   ├── series_catalog_client.go
│   │   │   ├── user_tracking_client.go
│   │   │   └── recommendation_client.go
│   │   ├── /middleware/
│   │   │   ├── auth.go              # Cognito JWT validation
│   │   │   ├── logging.go
│   │   │   └── cors.go
│   │   └── /dto/                    # Request/Response structs
│   └── /terraform/
│
└── /terraform/                      # Shared infrastructure
    ├── /modules/
    │   ├── /cognito/
    │   ├── /api_gateway/
    │   ├── /dynamodb/
    │   └── /lambda/
    └── /environments/
        └── /dev/
            └── main.tf
```

### 4.2 Design Principles

1. **Strict Isolation:** Each bounded context has its own `go.mod`
2. **No Shared Business Logic:** `/shared/` contains only infrastructure concerns
3. **Clean Architecture:** Domain contexts follow domain/application/infrastructure layers
4. **Layered Architecture:** BFF follows handlers/clients/middleware pattern (no domain layer needed)
5. **Value Object Independence:** Each context defines its own `UserId`, `SeriesId`, etc.

---

## 5. Data Model (DynamoDB)

### 5.1 SeriesCatalog Table

**Table Name:** `series-catalog`

| PK | SK | Attributes |
|----|----|-----------|
| `SERIES#<id>` | `META#<id>` | `title`, `description`, `genre`, `totalSeasons`, `posterUrl`, `lastUpdated`, `tmdbId` |

**GSI1 (Genre Index):**
- PK: `GENRE#<genre>`
- SK: `TITLE#<title>`

**GSI2 (Title Prefix Index for Search):**
- PK: `TITLE_PREFIX#<first_3_chars>`
- SK: `TITLE#<title>`

### 5.2 UserTracking Table

**Table Name:** `user-tracking`

| PK | SK | Attributes |
|----|----|-----------|
| `USER#<userId>` | `TRACKING#<seriesId>` | `status`, `seriesRating`, `seriesReview`, `addedAt`, `lastUpdated` |
| `USER#<userId>` | `SEASON_RATING#<seriesId>#<seasonNum>` | `rating`, `ratedAt` |

**GSI1 (Series Index):**
- PK: `SERIES#<seriesId>`
- SK: `USER#<userId>`

**GSI2 (Status Index):**
- PK: `STATUS#<status>` (WANT_TO_WATCH, WATCHING, COMPLETED, DROPPED)
- SK: `USER#<userId>#SERIES#<seriesId>`

---

## 6. API Specification (Experience BFF)

### 6.1 Authentication

All endpoints require Bearer token from Cognito in `Authorization` header.

### 6.2 Endpoints

#### Dashboard
```
GET /dashboard

Response: {
  "watching": {
    "items": [
      {
        "seriesId": "string",
        "title": "string",
        "posterUrl": "string",
        "status": "WATCHING",
        "lastUpdated": "2026-04-18T10:00:00Z"
      }
    ],
    "total": 5,
    "hasMore": false
  },
  "wantToWatch": {
    "items": [...],
    "total": 12,
    "hasMore": true
  },
  "completed": {
    "items": [...],
    "total": 3,
    "hasMore": false
  },
  "dropped": {
    "items": [...],
    "total": 1,
    "hasMore": false
  },
  "stats": {
    "totalSeries": 21,
    "watchingCount": 5,
    "completedCount": 3
  }
}
```

#### Series Catalog
```
GET /series?q={query}&page={n}&limit={20}

Response: {
  "items": [
    {
      "seriesId": "string",
      "title": "string",
      "description": "string",
      "genre": "string",
      "totalSeasons": 5,
      "posterUrl": "string"
    }
  ],
  "total": 150,
  "page": 1,
  "hasMore": true
}

GET /series/{seriesId}

Response: {
  "seriesId": "string",
  "title": "string",
  "description": "string",
  "genre": "string",
  "totalSeasons": 5,
  "posterUrl": "string",
  "isInUserList": false
}
```

#### Tracking
```
GET /tracking?status={status}&page={n}&limit={20}

Response: {
  "items": [
    {
      "trackingId": "string",
      "seriesId": "string",
      "title": "string",
      "posterUrl": "string",
      "status": "WATCHING",
      "seriesRating": 8,
      "addedAt": "2026-01-15T10:00:00Z",
      "lastUpdated": "2026-04-18T10:00:00Z"
    }
  ],
  "total": 5,
  "hasMore": false
}

POST /tracking
Body: {
  "seriesId": "string"
}

PUT /tracking/{seriesId}
Body: {
  "status": "WATCHING|WANT_TO_WATCH|COMPLETED|DROPPED",
  "seriesRating": 8,
  "seriesReview": "string",
  "seasonRatings": [
    {"seasonNumber": 1, "rating": 9}
  ]
}

DELETE /tracking/{seriesId}
```

#### Recommendation
```
GET /recommendation/next

Response: {
  "series": {
    "seriesId": "string",
    "title": "string",
    "description": "string",
    "genre": "string",
    "totalSeasons": 3,
    "posterUrl": "string"
  },
  "reason": "From your Want to Watch list"
}
```

---

## 7. TMDB Integration Strategy

### 7.1 Overview

SeriesCatalog integrates with TMDB (The Movie Database) API for series metadata.

### 7.2 Architecture

```
┌─────────────────────────────────────────────────┐
│              Series Catalog Context              │
│                                                  │
│  ┌──────────────┐      ┌──────────────┐        │
│  │  Search API  │      │  Sync Job    │        │
│  │   Handler    │      │  (Scheduled) │        │
│  └──────┬───────┘      └──────┬───────┘        │
│         │                      │                │
│         ▼                      ▼                │
│  ┌────────────────────────────────────────┐   │
│  │         TMDB Client                    │   │
│  │  - Rate limiting (40 req/10s)          │   │
│  │  - Retry with exponential backoff       │   │
│  │  - Circuit breaker for failures         │   │
│  └──────────────────┬───────────────────────┘   │
│                     │                           │
└─────────────────────┼───────────────────────────┘
                      │ HTTP
                      ▼
              ┌──────────────┐
              │   TMDB API   │
              └──────────────┘
```

### 7.3 Hybrid Search Flow

1. **User types search query**
2. **BFF → SeriesCatalog** searches DynamoDB (GSI2: Title Prefix)
3. **If not found locally:**
   - SeriesCatalog calls TMDB API
   - Returns results to user (without saving)
4. **User selects a series:**
   - SeriesCatalog fetches full details from TMDB
   - **Saves to DynamoDB** (now part of our cache)
   - Returns complete data

### 7.4 Periodic Sync

- **Trigger:** EventBridge Scheduler (daily at 3 AM UTC)
- **Target:** Lambda function `series-catalog-sync`
- **Logic:**
  - Query DynamoDB for series with `lastUpdated < 7 days ago`
  - Fetch updates from TMDB
  - Update local records
  - Handle TMDB unavailability (skip, log, retry next run)

### 7.5 Fallback Strategy

If TMDB API is unavailable:
- Return cached data from DynamoDB
- Add header: `X-Cache-Status: stale`
- BFF includes warning flag in response
- Flutter displays: "Series information may be outdated"

### 7.6 Data Stored (Minimum Viable)

```go
type Series struct {
    SeriesId       string    // TMDB ID
    Title          string
    Description    string
    Genre          string
    TotalSeasons   int
    PosterUrl      string
    LastUpdated    time.Time
    Source         string    // "tmdb"
}
```

---

## 8. Implementation Phases (MVP Incremental)

### Phase 1: SeriesCatalog (Weeks 1-2)
**Goal:** Reference data available

**Tasks:**
- [ ] Set up monorepo structure with Go workspace
- [ ] Create `/shared/` module with AWS clients, logger, errors
- [ ] Implement SeriesCatalog with Clean Architecture
- [ ] DynamoDB table `series-catalog` with TMDB data
- [ ] TMDB client with rate limiting and caching
- [ ] Terraform for Lambda, IAM, DynamoDB
- [ ] Local testing with SAM CLI

**Success Criteria:**
- Can search series via TMDB integration
- Selected series saved to DynamoDB
- Sync job functional

### Phase 2: UserTracking (Weeks 3-4)
**Goal:** Core tracking functionality

**Tasks:**
- [ ] Implement UserTracking with Clean Architecture
- [ ] Aggregate `UserSeriesTracking` with all rules
- [ ] DynamoDB table `user-tracking` with GSIs
- [ ] HTTP client to SeriesCatalog for validation
- [ ] All tracking operations (CRUD + rating)
- [ ] Terraform infrastructure

**Success Criteria:**
- Can track series progress
- Validation with SeriesCatalog works
- All tracking statuses functional

### Phase 3: ExperienceBFF (Weeks 5-6)
**Goal:** Unified API for Flutter

**Tasks:**
- [ ] BFF Lambda with Layered Architecture
- [ ] Cognito integration (JWT validation)
- [ ] HTTP clients to all domain contexts
- [ ] Aggregation layer (parallel calls)
- [ ] All endpoints: `/dashboard`, `/series`, `/tracking`, `/recommendation`
- [ ] API Gateway configuration
- [ ] CORS, auth middleware

**Success Criteria:**
- Flutter can call all operations through single API
- Dashboard aggregation working
- Authentication flow complete

### Phase 4: Flutter Integration (Weeks 7-8)
**Goal:** Working end-to-end MVP

**Tasks:**
- [ ] Set up Flutter repository
- [ ] Implement authentication with Cognito
- [ ] Dashboard screen
- [ ] Series search and detail
- [ ] Tracking list and management
- [ ] Recommendation feature

**Success Criteria:**
- User can register, login
- Search and add series to tracking
- View dashboard with all lists
- Get recommendations

### Phase 5: Recommendation Polish (Week 9)
**Goal:** Enhanced recommendation

**Tasks:**
- [ ] Recommendation context (stateless)
- [ ] Random selection from "Want to Watch"
- [ ] Exclude recently watched (optional)

**Success Criteria:**
- Recommendation endpoint functional
- Returns valid random series

---

## 9. Infrastructure (Terraform)

### 9.1 Shared Resources

```hcl
# terraform/environments/dev/main.tf

module "cognito" {
  source = "./modules/cognito"
  
  user_pool_name = "lorebase-users"
  # ... Cognito configuration
}

module "api_gateway" {
  source = "./modules/api_gateway"
  
  name = "lorebase-api"
  # Single API Gateway for BFF
}

module "dynamodb" {
  source = "./modules/dynamodb"
  
  tables = [
    {
      name = "series-catalog"
      billing_mode = "PAY_PER_REQUEST"
    },
    {
      name = "user-tracking"
      billing_mode = "PAY_PER_REQUEST"
    }
  ]
}
```

### 9.2 Lambda Functions (per context)

```hcl
# series-catalog/terraform/main.tf

resource "aws_lambda_function" "series_catalog_api" {
  function_name = "series-catalog-api"
  runtime       = "provided.al2"  # Go custom runtime
  handler       = "bootstrap"
  
  filename         = "${path.module}/../bin/api.zip"
  source_code_hash = filebase64sha256("${path.module}/../bin/api.zip")
  
  environment {
    variables = {
      TABLE_NAME     = "series-catalog"
      TMDB_API_KEY   = var.tmdb_api_key
      LOG_LEVEL      = "info"
    }
  }
}
```

### 9.3 Deployment Strategy

```bash
# Makefile targets

# Build all services
make build

# Deploy specific service
make deploy SERVICE=series-catalog

# Deploy all (manual, no CI/CD)
make deploy-all

# Local testing
make local SERVICE=series-catalog
```

---

## 10. Security Considerations

### 10.1 Authentication
- **Cognito User Pools:** User registration, login, password recovery
- **Cognito Identity Pools:** AWS credentials for direct access (future)
- **JWT Validation:** BFF validates Cognito tokens on every request

### 10.2 Authorization
- **User-scoped data:** All tracking queries include `userId` from JWT
- **Row-level security:** DynamoDB keys include user identifier
- **No admin endpoints in MVP:** All users have same permissions

### 10.3 Data Protection
- **DynamoDB encryption:** AWS managed keys (SSE)
- **TMDB API key:** Stored in AWS Secrets Manager
- **No PII beyond email:** Username/sub from Cognito only

### 10.4 API Security
- **CORS:** Restricted to Flutter app domain
- **Rate limiting:** API Gateway throttling (1000 req/sec per method)
- **Input validation:** All DTOs validated before processing
- **No SQL injection:** DynamoDB (NoSQL) with parameterized queries

---

## 11. Monitoring & Observability

### 11.1 Logging
- **Structured JSON logs:** All Lambdas use shared logger
- **Correlation ID:** Passed through all service calls
- **Log levels:** DEBUG (local), INFO (production)

### 11.2 Metrics
- **CloudWatch:** Lambda invocations, duration, errors
- **Custom metrics:** Business metrics (series added, tracks created)
- **API Gateway:** Request count, latency, 4xx/5xx errors

### 11.3 Alerting (Terraform Managed)

CloudWatch Alarms and SNS notifications managed via Terraform:

```hcl
# terraform/modules/monitoring/main.tf

resource "aws_sns_topic" "alarms" {
  name = "lorebase-alarms"
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.alarms.arn
  protocol  = "email"
  endpoint  = var.alert_email  # Your email
}

# Lambda Error Rate Alarm
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "lambda-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "Lambda error count exceeded threshold"
  alarm_actions       = [aws_sns_topic.alarms.arn]
  
  dimensions = {
    FunctionName = "*"  # All Lambda functions
  }
}

# Lambda Duration Alarm
resource "aws_cloudwatch_metric_alarm" "lambda_duration" {
  alarm_name          = "lambda-duration"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "Duration"
  namespace           = "AWS/Lambda"
  period              = 60
  statistic           = "p99"
  threshold           = 5000  # 5 seconds
  alarm_description   = "Lambda duration approaching timeout"
  alarm_actions       = [aws_sns_topic.alarms.arn]
}

# DynamoDB Throttling Alarm
resource "aws_cloudwatch_metric_alarm" "dynamodb_throttling" {
  alarm_name          = "dynamodb-throttling"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ReadThrottleEvents"
  namespace           = "AWS/DynamoDB"
  period              = 60
  statistic           = "Sum"
  threshold           = 1
  alarm_description   = "DynamoDB throttling detected"
  alarm_actions       = [aws_sns_topic.alarms.arn]
}
```

**Alert Delivery:**
- SNS topic publishes to email
- Future: Can add Slack, PagerDuty, or SMS via additional subscriptions

---

## 12. Error Handling Strategy

### 12.1 Standard Error Response
```json
{
  "error": {
    "code": "SERIES_NOT_FOUND",
    "message": "Series with ID 'xyz' not found",
    "details": {
      "seriesId": "xyz"
    }
  }
}
```

### 12.2 Error Categories
- **Client errors (4xx):** Validation, not found, unauthorized
- **Server errors (5xx):** Database unavailable, TMDB failure
- **Domain errors:** Business rule violations (already tracking, invalid rating)

### 12.3 Fallbacks
- **TMDB unavailable:** Use cached data with stale warning
- **DynamoDB throttling:** Retry with exponential backoff
- **Cross-context call fails:** Return partial data if possible

---

## 13. Future Enhancements (Post-MVP)

### 13.1 Event-Driven Architecture
- EventBridge for domain events
- Async processing for heavy operations
- Cross-context decoupling

### 13.2 Advanced Recommendations
- Machine learning model
- Collaborative filtering
- Watch history analysis

### 13.3 Extended Content
- Movies, games, books (other media types)
- Episode-level tracking (when needed)
- Watch party features

### 13.4 Operational Maturity
- CI/CD pipeline (GitHub Actions)
- Automated testing (unit, integration, e2e)
- Blue-green deployments
- Disaster recovery

---

## 14. Open Questions & Decisions

| Question | Decision | Date |
|----------|----------|------|
| Repository structure | Monorepo with multiple go.mod | 2026-04-18 |
| Lambda granularity | One per bounded context | 2026-04-18 |
| Architecture per context | Clean for domains, Layered for BFF | 2026-04-18 |
| Shared Kernel | No - independent Value Objects | 2026-04-18 |
| Recommendation state | Stateless (no table) | 2026-04-18 |
| Inter-context communication | HTTP sync (MVP), Events (future) | 2026-04-18 |
| Data source for series | TMDB API with hybrid cache | 2026-04-18 |
| Frontend technology | Flutter (separate repo) | 2026-04-18 |

---

## 15. Appendix

### A. Environment Variables

```bash
# Shared
AWS_REGION=us-east-1
ENVIRONMENT=dev
LOG_LEVEL=info

# SeriesCatalog
SERIES_CATALOG_TABLE=series-catalog
TMDB_API_KEY=xxx
TMDB_RATE_LIMIT=40
SYNC_SCHEDULE="cron(0 3 * * ? *)"

# UserTracking
USER_TRACKING_TABLE=user-tracking
SERIES_CATALOG_ENDPOINT=https://...

# BFF
COGNITO_USER_POOL_ID=xxx
COGNITO_CLIENT_ID=xxx
SERIES_CATALOG_ENDPOINT=https://...
USER_TRACKING_ENDPOINT=https://...
RECOMMENDATION_ENDPOINT=https://...
```

### B. Useful Commands

```bash
# Local testing with SAM
sam local start-api -t series-catalog/terraform/template.yaml

# Deploy single service
cd series-catalog && make deploy

# View logs
aws logs tail /aws/lambda/series-catalog-api --follow

# Test DynamoDB query
aws dynamodb query \
  --table-name series-catalog \
  --key-condition-expression "PK = :pk" \
  --expression-attribute-values '{":pk":{"S":"SERIES#123"}}'
```

---

**End of Specification**

*This document should be reviewed and updated as the system evolves. Last updated: 2026-04-18*
