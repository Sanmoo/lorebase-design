# lorebase Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the complete lorebase backend system with 4 bounded contexts (SeriesCatalog, UserTracking, Recommendation, ExperienceBFF) using Go, AWS Lambda, DynamoDB, and serverless architecture.

**Architecture:** Monorepo with multiple Go modules per bounded context. Clean Architecture for domain contexts, Layered Architecture for BFF. Communication via Lambda Function URLs with IAM auth.

**Tech Stack:** Go 1.26, AWS Lambda (provided.al2), DynamoDB, API Gateway, Cognito, Terraform, TMDB API

---

## File Structure Overview

```
/lorebase-backend/                    # New repository (separate from lorebase-design)
├── go.work
├── Makefile
├── README.md
│
├── /shared/
│   ├── go.mod
│   └── /pkg/
│       ├── /aws/
│       │   ├── config.go
│       │   ├── dynamodb.go
│       │   ├── lambda_client.go      # For Function URL calls
│       │   └── logger.go
│       ├── /errors/
│       │   └── errors.go
│       └── /http/
│           ├── middleware.go
│           └── response.go
│
├── /series-catalog/
│   ├── go.mod
│   ├── /cmd/api/main.go
│   ├── /internal/
│   │   ├── /domain/
│   │   │   ├── series.go
│   │   │   ├── season.go
│   │   │   └── value_objects.go
│   │   ├── /application/
│   │   │   ├── create_series.go
│   │   │   ├── search_series.go
│   │   │   └── sync_from_tmdb.go
│   │   ├── /infrastructure/
│   │   │   ├── series_repository.go
│   │   │   └── tmdb_client.go
│   │   └── /ports/
│   │       ├── repository.go
│   │       └── tmdb.go
│   ├── /terraform/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── /scripts/
│       └── deploy.sh
│
├── /user-tracking/
│   ├── go.mod
│   ├── /cmd/api/main.go
│   ├── /internal/
│   │   ├── /domain/
│   │   │   ├── aggregate/
│   │   │   │   └── user_series_tracking.go
│   │   │   ├── entity/
│   │   │   │   └── tracked_series.go
│   │   │   └── valueobject/
│   │   │       ├── tracking_id.go
│   │   │       ├── user_id.go
│   │   │       ├── series_id.go
│   │   │       └── season_rating.go
│   │   ├── /application/
│   │   │   ├── start_tracking.go
│   │   │   ├── update_progress.go
│   │   │   ├── rate_series.go
│   │   │   └── list_tracking.go
│   │   ├── /infrastructure/
│   │   │   ├── tracking_repository.go
│   │   │   └── series_catalog_client.go
│   │   └── /ports/
│   ├── /terraform/
│   └── /scripts/
│
├── /recommendation/
│   ├── go.mod
│   ├── /cmd/api/main.go
│   ├── /internal/
│   │   ├── /handlers/
│   │   │   └── recommendation.go
│   │   └── /clients/
│   │       └── user_tracking_client.go
│   └── /terraform/
│
├── /experience-bff/
│   ├── go.mod
│   ├── /cmd/api/main.go
│   ├── /internal/
│   │   ├── /handlers/
│   │   │   ├── dashboard.go
│   │   │   ├── series.go
│   │   │   ├── tracking.go
│   │   │   └── recommendation.go
│   │   ├── /aggregator/
│   │   │   └── dashboard_builder.go
│   │   ├── /clients/
│   │   │   ├── series_catalog_client.go
│   │   │   ├── user_tracking_client.go
│   │   │   └── recommendation_client.go
│   │   ├── /middleware/
│   │   │   ├── auth.go
│   │   │   ├── logging.go
│   │   │   └── cors.go
│   │   └── /dto/
│   │       ├── requests.go
│   │       └── responses.go
│   └── /terraform/
│
└── /terraform/
    └── /environments/
        └── /dev/
            ├── main.tf
            ├── variables.tf
            └── outputs.tf
```

---

## Phase 1: Shared Module Foundation

**Goal:** Create shared utilities used by all bounded contexts.

---

### Task 1.1: Initialize Shared Module

**Files:**
- Create: `/shared/go.mod`
- Create: `/shared/pkg/aws/config.go`

- [ ] **Step 1: Create shared go.mod**

```bash
cd /shared
go mod init github.com/lorebase/lorebase-backend/shared
go mod tidy
```

- [ ] **Step 2: Create AWS config loader**

Create: `/shared/pkg/aws/config.go`
```go
package aws

import (
    "context"
    "os"

    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/aws"
)

type Config struct {
    Region    string
    TableName string
}

func LoadConfig(ctx context.Context) (aws.Config, error) {
    region := os.Getenv("AWS_REGION")
    if region == "" {
        region = "us-east-1"
    }

    cfg, err := config.LoadDefaultConfig(ctx,
        config.WithRegion(region),
    )
    if err != nil {
        return aws.Config{}, err
    }

    return cfg, nil
}

func LoadTableName() string {
    tableName := os.Getenv("TABLE_NAME")
    if tableName == "" {
        return "default-table"
    }
    return tableName
}
```

- [ ] **Step 3: Commit**

```bash
git add shared/
git commit -m "feat(shared): initialize shared module with AWS config"
```

---

### Task 1.2: Structured Logger

**Files:**
- Create: `/shared/pkg/logger/logger.go`

- [ ] **Step 1: Create JSON logger**

Create: `/shared/pkg/logger/logger.go`
```go
package logger

import (
    "encoding/json"
    "os"
    "time"
)

type Logger struct {
    service string
}

func New(service string) *Logger {
    return &Logger{service: service}
}

func (l *Logger) Info(msg string, fields map[string]interface{}) {
    l.log("INFO", msg, fields)
}

func (l *Logger) Error(msg string, err error, fields map[string]interface{}) {
    if fields == nil {
        fields = make(map[string]interface{})
    }
    if err != nil {
        fields["error"] = err.Error()
    }
    l.log("ERROR", msg, fields)
}

func (l *Logger) log(level, msg string, fields map[string]interface{}) {
    entry := map[string]interface{}{
        "timestamp": time.Now().UTC().Format(time.RFC3339),
        "level":     level,
        "service":   l.service,
        "message":   msg,
    }
    
    for k, v := range fields {
        entry[k] = v
    }
    
    json.NewEncoder(os.Stdout).Encode(entry)
}
```

- [ ] **Step 2: Commit**

```bash
git add shared/pkg/logger/
git commit -m "feat(shared): add structured JSON logger"
```

---

### Task 1.3: Custom Error Types

**Files:**
- Create: `/shared/pkg/errors/errors.go`

- [ ] **Step 1: Create domain errors**

Create: `/shared/pkg/errors/errors.go`
```go
package errors

import "fmt"

type DomainError struct {
    Code    string
    Message string
}

func (e *DomainError) Error() string {
    return fmt.Sprintf("[%s] %s", e.Code, e.Message)
}

func NewNotFound(resource string) *DomainError {
    return &DomainError{
        Code:    "NOT_FOUND",
        Message: resource + " not found",
    }
}

func NewValidation(field string, reason string) *DomainError {
    return &DomainError{
        Code:    "VALIDATION_ERROR",
        Message: fmt.Sprintf("%s: %s", field, reason),
    }
}

func NewConflict(message string) *DomainError {
    return &DomainError{
        Code:    "CONFLICT",
        Message: message,
    }
}

func IsNotFound(err error) bool {
    if de, ok := err.(*DomainError); ok {
        return de.Code == "NOT_FOUND"
    }
    return false
}
```

- [ ] **Step 2: Commit**

```bash
git add shared/pkg/errors/
git commit -m "feat(shared): add custom domain error types"
```

---

### Task 1.4: Lambda Function URL Client

**Files:**
- Create: `/shared/pkg/aws/lambda_client.go`

- [ ] **Step 1: Create Lambda client for Function URLs**

Create: `/shared/pkg/aws/lambda_client.go`
```go
package aws

import (
    "bytes"
    "context"
    "io"
    "net/http"
    "time"

    "github.com/aws/aws-sdk-go-v2/aws"
    v4 "github.com/aws/aws-sdk-go-v2/aws/signer/v4"
)

type LambdaClient struct {
    signer   *v4.Signer
    creds    aws.Credentials
    region   string
    client   *http.Client
}

func NewLambdaClient(cfg aws.Config) *LambdaClient {
    return &LambdaClient{
        signer: v4.NewSigner(),
        creds:  cfg.Credentials,
        region: cfg.Region,
        client: &http.Client{
            Timeout: 5 * time.Second,
        },
    }
}

func (c *LambdaClient) Invoke(ctx context.Context, url string, method string, body []byte) ([]byte, error) {
    var bodyReader io.Reader
    if body != nil {
        bodyReader = bytes.NewReader(body)
    }

    req, err := http.NewRequestWithContext(ctx, method, url, bodyReader)
    if err != nil {
        return nil, err
    }

    if body != nil {
        req.Header.Set("Content-Type", "application/json")
    }

    // Sign request with AWS credentials
    creds, err := c.creds.Retrieve(ctx)
    if err != nil {
        return nil, err
    }

    err = c.signer.SignHTTP(ctx, creds, req, "lambda", c.region, time.Now())
    if err != nil {
        return nil, err
    }

    resp, err := c.client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    responseBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, err
    }

    if resp.StatusCode >= 400 {
        return nil, &DomainError{
            Code:    "LAMBDA_ERROR",
            Message: string(responseBody),
        }
    }

    return responseBody, nil
}
```

- [ ] **Step 2: Update go.mod dependencies**

```bash
cd /shared
go get github.com/aws/aws-sdk-go-v2
go get github.com/aws/aws-sdk-go-v2/aws/signer/v4
go mod tidy
```

- [ ] **Step 3: Commit**

```bash
git add shared/
git commit -m "feat(shared): add Lambda Function URL client with IAM auth"
```

---

### Task 1.5: HTTP Response Helpers

**Files:**
- Create: `/shared/pkg/http/response.go`

- [ ] **Step 1: Create HTTP response utilities**

Create: `/shared/pkg/http/response.go`
```go
package http

import (
    "encoding/json"
    "net/http"
)

type Response struct {
    StatusCode int
    Body       interface{}
}

func JSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func Error(w http.ResponseWriter, status int, code, message string) {
    JSON(w, status, map[string]interface{}{
        "error": map[string]string{
            "code":    code,
            "message": message,
        },
    })
}

func Success(w http.ResponseWriter, data interface{}) {
    JSON(w, http.StatusOK, data)
}
```

- [ ] **Step 2: Commit**

```bash
git add shared/pkg/http/
git commit -m "feat(shared): add HTTP response helpers"
```

---

## Phase 2: SeriesCatalogContext

**Goal:** Implement the supporting domain for series metadata with TMDB integration.

---

### Task 2.1: Initialize SeriesCatalog Module

**Files:**
- Create: `/series-catalog/go.mod`
- Create: `/series-catalog/cmd/api/main.go`

- [ ] **Step 1: Create go.mod with shared dependency**

Create: `/series-catalog/go.mod`
```go
module github.com/lorebase/lorebase-backend/series-catalog

go 1.26

require (
    github.com/aws/aws-lambda-go v1.47.0
    github.com/aws/aws-sdk-go-v2 v1.30.0
    github.com/aws/aws-sdk-go-v2/config v1.27.0
    github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue v1.13.0
    github.com/aws/aws-sdk-go-v2/service/dynamodb v1.32.0
    github.com/lorebase/lorebase-backend/shared v0.0.0
)

replace github.com/lorebase/lorebase-backend/shared => ../shared
```

- [ ] **Step 2: Create Lambda entrypoint**

Create: `/series-catalog/cmd/api/main.go`
```go
package main

import (
    "context"
    "net/http"

    "github.com/aws/aws-lambda-go/events"
    "github.com/aws/aws-lambda-go/lambda"
    "github.com/lorebase/lorebase-backend/shared/pkg/aws"
    "github.com/lorebase/lorebase-backend/shared/pkg/logger"
)

var log *logger.Logger

func init() {
    log = logger.New("series-catalog")
}

func handler(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    log.Info("Received request", map[string]interface{}{
        "path":   request.Path,
        "method": request.HTTPMethod,
    })

    // Route handling
    switch request.HTTPMethod {
    case "GET":
        if request.Path == "/search" {
            return handleSearch(ctx, request)
        }
        return handleGetSeries(ctx, request)
    case "POST":
        return handleCreateSeries(ctx, request)
    default:
        return events.APIGatewayProxyResponse{
            StatusCode: http.StatusMethodNotAllowed,
            Body:       `{"error": "Method not allowed"}`,
        }, nil
    }
}

func handleSearch(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    query := request.QueryStringParameters["q"]
    if query == "" {
        return events.APIGatewayProxyResponse{
            StatusCode: http.StatusBadRequest,
            Body:       `{"error": "Query parameter 'q' is required"}`,
        }, nil
    }

    // TODO: Implement search logic
    return events.APIGatewayProxyResponse{
        StatusCode: http.StatusOK,
        Body:       `{"items": [], "query": "` + query + `"}`,
    }, nil
}

func handleGetSeries(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    // TODO: Implement get series by ID
    return events.APIGatewayProxyResponse{
        StatusCode: http.StatusOK,
        Body:       `{"seriesId": "test"}`,
    }, nil
}

func handleCreateSeries(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    // TODO: Implement create series from TMDB
    return events.APIGatewayProxyResponse{
        StatusCode: http.StatusCreated,
        Body:       `{"seriesId": "created"}`,
    }, nil
}

func main() {
    lambda.Start(handler)
}
```

- [ ] **Step 3: Download dependencies**

```bash
cd /series-catalog
go mod tidy
```

- [ ] **Step 4: Commit**

```bash
git add series-catalog/
git commit -m "feat(series-catalog): initialize module and lambda entrypoint"
```

---

### Task 2.2: Domain Models (Value Objects & Entities)

**Files:**
- Create: `/series-catalog/internal/domain/value_objects.go`
- Create: `/series-catalog/internal/domain/series.go`

- [ ] **Step 1: Create Value Objects**

Create: `/series-catalog/internal/domain/value_objects.go`
```go
package domain

type SeriesId struct {
    value string
}

func NewSeriesId(id string) SeriesId {
    return SeriesId{value: id}
}

func (s SeriesId) String() string {
    return s.value
}

type Genre string

const (
    GenreDrama    Genre = "Drama"
    GenreComedy   Genre = "Comedy"
    GenreAction   Genre = "Action"
    GenreSciFi    Genre = "Sci-Fi"
    GenreHorror   Genre = "Horror"
    GenreThriller Genre = "Thriller"
    GenreRomance  Genre = "Romance"
    GenreOther    Genre = "Other"
)
```

- [ ] **Step 2: Create Series Entity**

Create: `/series-catalog/internal/domain/series.go`
```go
package domain

import (
    "time"
    "github.com/lorebase/lorebase-backend/shared/pkg/errors"
)

type Series struct {
    SeriesId      SeriesId
    Title         string
    Description   string
    Genre         Genre
    TotalSeasons  int
    PosterUrl     string
    TmdbId        string
    LastUpdated   time.Time
}

func NewSeries(id SeriesId, title, description string, genre Genre, totalSeasons int, posterUrl, tmdbId string) (*Series, error) {
    if title == "" {
        return nil, errors.NewValidation("title", "title is required")
    }
    
    if totalSeasons < 1 {
        return nil, errors.NewValidation("totalSeasons", "must be at least 1")
    }

    return &Series{
        SeriesId:     id,
        Title:        title,
        Description:  description,
        Genre:        genre,
        TotalSeasons: totalSeasons,
        PosterUrl:    posterUrl,
        TmdbId:       tmdbId,
        LastUpdated:  time.Now().UTC(),
    }, nil
}

func (s *Series) UpdateMetadata(title, description string, genre Genre, totalSeasons int, posterUrl string) error {
    if title != "" {
        s.Title = title
    }
    if description != "" {
        s.Description = description
    }
    if genre != "" {
        s.Genre = genre
    }
    if totalSeasons > 0 {
        s.TotalSeasons = totalSeasons
    }
    if posterUrl != "" {
        s.PosterUrl = posterUrl
    }
    
    s.LastUpdated = time.Now().UTC()
    return nil
}
```

- [ ] **Step 3: Commit**

```bash
git add series-catalog/internal/domain/
git commit -m "feat(series-catalog): add domain models (Series entity and Value Objects)"
```

---

### Task 2.3: Repository Port & DynamoDB Implementation

**Files:**
- Create: `/series-catalog/internal/ports/repository.go`
- Create: `/series-catalog/internal/infrastructure/series_repository.go`

- [ ] **Step 1: Define Repository Port**

Create: `/series-catalog/internal/ports/repository.go`
```go
package ports

import (
    "context"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/domain"
)

type SeriesRepository interface {
    GetById(ctx context.Context, id domain.SeriesId) (*domain.Series, error)
    SearchByTitle(ctx context.Context, query string, limit int) ([]*domain.Series, error)
    SearchByGenre(ctx context.Context, genre domain.Genre, limit int) ([]*domain.Series, error)
    Save(ctx context.Context, series *domain.Series) error
    Delete(ctx context.Context, id domain.SeriesId) error
}
```

- [ ] **Step 2: Create DynamoDB Repository**

Create: `/series-catalog/internal/infrastructure/series_repository.go`
```go
package infrastructure

import (
    "context"
    "fmt"
    "time"

    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue"
    "github.com/aws/aws-sdk-go-v2/service/dynamodb"
    "github.com/aws/aws-sdk-go-v2/service/dynamodb/types"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/domain"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/ports"
    "github.com/lorebase/lorebase-backend/shared/pkg/errors"
)

type SeriesDynamoRepository struct {
    client    *dynamodb.Client
    tableName string
}

func NewSeriesDynamoRepository(client *dynamodb.Client, tableName string) ports.SeriesRepository {
    return &SeriesDynamoRepository{
        client:    client,
        tableName: tableName,
    }
}

type seriesRecord struct {
    PK           string    `dynamodbav:"PK"`
    SK           string    `dynamodbav:"SK"`
    SeriesId     string    `dynamodbav:"seriesId"`
    Title        string    `dynamodbav:"title"`
    Description  string    `dynamodbav:"description"`
    Genre        string    `dynamodbav:"genre"`
    TotalSeasons int       `dynamodbav:"totalSeasons"`
    PosterUrl    string    `dynamodbav:"posterUrl"`
    TmdbId       string    `dynamodbav:"tmdbId"`
    LastUpdated  time.Time `dynamodbav:"lastUpdated"`
}

func (r *SeriesDynamoRepository) GetById(ctx context.Context, id domain.SeriesId) (*domain.Series, error) {
    result, err := r.client.GetItem(ctx, &dynamodb.GetItemInput{
        TableName: aws.String(r.tableName),
        Key: map[string]types.AttributeValue{
            "PK": &types.AttributeValueMemberS{Value: fmt.Sprintf("SERIES#%s", id.String())},
            "SK": &types.AttributeValueMemberS{Value: fmt.Sprintf("META#%s", id.String())},
        },
    })
    if err != nil {
        return nil, err
    }

    if result.Item == nil {
        return nil, errors.NewNotFound("Series")
    }

    var record seriesRecord
    err = attributevalue.UnmarshalMap(result.Item, &record)
    if err != nil {
        return nil, err
    }

    return r.toDomain(record), nil
}

func (r *SeriesDynamoRepository) SearchByTitle(ctx context.Context, query string, limit int) ([]*domain.Series, error) {
    if limit == 0 {
        limit = 20
    }

    // Use begins_with on title for prefix search
    result, err := r.client.Query(ctx, &dynamodb.QueryInput{
        TableName:              aws.String(r.tableName),
        IndexName:              aws.String("title-index"),
        KeyConditionExpression: aws.String("begins_with(TITLE_PREFIX, :prefix)"),
        ExpressionAttributeValues: map[string]types.AttributeValue{
            ":prefix": &types.AttributeValueMemberS{Value: query[:min(3, len(query))]},
        },
        Limit: aws.Int32(int32(limit)),
    })
    if err != nil {
        return nil, err
    }

    var series []*domain.Series
    for _, item := range result.Items {
        var record seriesRecord
        err := attributevalue.UnmarshalMap(item, &record)
        if err != nil {
            continue
        }
        series = append(series, r.toDomain(record))
    }

    return series, nil
}

func (r *SeriesDynamoRepository) SearchByGenre(ctx context.Context, genre domain.Genre, limit int) ([]*domain.Series, error) {
    if limit == 0 {
        limit = 20
    }

    result, err := r.client.Query(ctx, &dynamodb.QueryInput{
        TableName:              aws.String(r.tableName),
        IndexName:              aws.String("genre-index"),
        KeyConditionExpression: aws.String("PK = :genre"),
        ExpressionAttributeValues: map[string]types.AttributeValue{
            ":genre": &types.AttributeValueMemberS{Value: fmt.Sprintf("GENRE#%s", genre)},
        },
        Limit: aws.Int32(int32(limit)),
    })
    if err != nil {
        return nil, err
    }

    var series []*domain.Series
    for _, item := range result.Items {
        var record seriesRecord
        err := attributevalue.UnmarshalMap(item, &record)
        if err != nil {
            continue
        }
        series = append(series, r.toDomain(record))
    }

    return series, nil
}

func (r *SeriesDynamoRepository) Save(ctx context.Context, series *domain.Series) error {
    record := r.toRecord(series)
    item, err := attributevalue.MarshalMap(record)
    if err != nil {
        return err
    }

    _, err = r.client.PutItem(ctx, &dynamodb.PutItemInput{
        TableName: aws.String(r.tableName),
        Item:      item,
    })
    return err
}

func (r *SeriesDynamoRepository) Delete(ctx context.Context, id domain.SeriesId) error {
    _, err := r.client.DeleteItem(ctx, &dynamodb.DeleteItemInput{
        TableName: aws.String(r.tableName),
        Key: map[string]types.AttributeValue{
            "PK": &types.AttributeValueMemberS{Value: fmt.Sprintf("SERIES#%s", id.String())},
            "SK": &types.AttributeValueMemberS{Value: fmt.Sprintf("META#%s", id.String())},
        },
    })
    return err
}

func (r *SeriesDynamoRepository) toRecord(s *domain.Series) seriesRecord {
    return seriesRecord{
        PK:           fmt.Sprintf("SERIES#%s", s.SeriesId.String()),
        SK:           fmt.Sprintf("META#%s", s.SeriesId.String()),
        SeriesId:     s.SeriesId.String(),
        Title:        s.Title,
        Description:  s.Description,
        Genre:        string(s.Genre),
        TotalSeasons: s.TotalSeasons,
        PosterUrl:    s.PosterUrl,
        TmdbId:       s.TmdbId,
        LastUpdated:  s.LastUpdated,
    }
}

func (r *SeriesDynamoRepository) toDomain(record seriesRecord) *domain.Series {
    return &domain.Series{
        SeriesId:     domain.NewSeriesId(record.SeriesId),
        Title:        record.Title,
        Description:  record.Description,
        Genre:        domain.Genre(record.Genre),
        TotalSeasons: record.TotalSeasons,
        PosterUrl:    record.PosterUrl,
        TmdbId:       record.TmdbId,
        LastUpdated:  record.LastUpdated,
    }
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

- [ ] **Step 3: Commit**

```bash
git add series-catalog/internal/ports/ series-catalog/internal/infrastructure/
git commit -m "feat(series-catalog): add DynamoDB repository implementation"
```

---

### Task 2.4: TMDB Client

**Files:**
- Create: `/series-catalog/internal/ports/tmdb.go`
- Create: `/series-catalog/internal/infrastructure/tmdb_client.go`

- [ ] **Step 1: Define TMDB Port**

Create: `/series-catalog/internal/ports/tmdb.go`
```go
package ports

import (
    "context"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/domain"
)

type TmdbSearchResult struct {
    Id          int
    Title       string
    Description string
    Genre       string
    PosterUrl   string
}

type TmdbClient interface {
    SearchSeries(ctx context.Context, query string) ([]TmdbSearchResult, error)
    GetSeriesDetails(ctx context.Context, tmdbId int) (*domain.Series, error)
}
```

- [ ] **Step 2: Implement TMDB HTTP Client**

Create: `/series-catalog/internal/infrastructure/tmdb_client.go`
```go
package infrastructure

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "net/url"
    "strconv"
    "time"

    "github.com/lorebase/lorebase-backend/series-catalog/internal/domain"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/ports"
    sharedaws "github.com/lorebase/lorebase-backend/shared/pkg/aws"
)

const tmdbBaseUrl = "https://api.themoviedb.org/3"

type HttpTmdbClient struct {
    apiKey string
    client *http.Client
}

func NewHttpTmdbClient(apiKey string) ports.TmdbClient {
    return &HttpTmdbClient{
        apiKey: apiKey,
        client: &http.Client{
            Timeout: 5 * time.Second,
        },
    }
}

type tmdbSearchResponse struct {
    Results []struct {
        Id          int    `json:"id"`
        Name        string `json:"name"`
        Overview    string `json:"overview"`
        PosterPath  string `json:"poster_path"`
        GenreIds    []int  `json:"genre_ids"`
    } `json:"results"`
}

type tmdbSeriesDetails struct {
    Id            int    `json:"id"`
    Name          string `json:"name"`
    Overview      string `json:"overview"`
    PosterPath    string `json:"poster_path"`
    NumberOfSeasons int  `json:"number_of_seasons"`
    Genres        []struct {
        Id   int    `json:"id"`
        Name string `json:"name"`
    } `json:"genres"`
}

func (c *HttpTmdbClient) SearchSeries(ctx context.Context, query string) ([]ports.TmdbSearchResult, error) {
    u, _ := url.Parse(tmdbBaseUrl + "/search/tv")
    q := u.Query()
    q.Set("api_key", c.apiKey)
    q.Set("query", query)
    q.Set("language", "pt-BR")
    u.RawQuery = q.Encode()

    req, err := http.NewRequestWithContext(ctx, "GET", u.String(), nil)
    if err != nil {
        return nil, err
    }

    resp, err := c.client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("TMDB API error: %d", resp.StatusCode)
    }

    var result tmdbSearchResponse
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, err
    }

    var results []ports.TmdbSearchResult
    for _, item := range result.Results {
        var genre string
        if len(item.GenreIds) > 0 {
            genre = mapGenreId(item.GenreIds[0])
        }

        results = append(results, ports.TmdbSearchResult{
            Id:          item.Id,
            Title:       item.Name,
            Description: item.Overview,
            Genre:       genre,
            PosterUrl:   fmt.Sprintf("https://image.tmdb.org/t/p/w500%s", item.PosterPath),
        })
    }

    return results, nil
}

func (c *HttpTmdbClient) GetSeriesDetails(ctx context.Context, tmdbId int) (*domain.Series, error) {
    u := fmt.Sprintf("%s/tv/%d?api_key=%s&language=pt-BR", tmdbBaseUrl, tmdbId, c.apiKey)

    req, err := http.NewRequestWithContext(ctx, "GET", u, nil)
    if err != nil {
        return nil, err
    }

    resp, err := c.client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("TMDB API error: %d", resp.StatusCode)
    }

    var details tmdbSeriesDetails
    if err := json.NewDecoder(resp.Body).Decode(&details); err != nil {
        return nil, err
    }

    var genre domain.Genre = domain.GenreOther
    if len(details.Genres) > 0 {
        genre = mapGenreName(details.Genres[0].Name)
    }

    seriesId := domain.NewSeriesId(strconv.Itoa(details.Id))
    series, err := domain.NewSeries(
        seriesId,
        details.Name,
        details.Overview,
        genre,
        details.NumberOfSeasons,
        fmt.Sprintf("https://image.tmdb.org/t/p/w500%s", details.PosterPath),
        strconv.Itoa(details.Id),
    )
    if err != nil {
        return nil, err
    }

    return series, nil
}

func mapGenreId(id int) string {
    // TMDB genre IDs to names
    genreMap := map[int]string{
        18: "Drama",
        35: "Comedy",
        28: "Action",
        878: "Sci-Fi",
        27: "Horror",
        53: "Thriller",
        10749: "Romance",
    }
    if name, ok := genreMap[id]; ok {
        return name
    }
    return "Other"
}

func mapGenreName(name string) domain.Genre {
    switch name {
    case "Drama":
        return domain.GenreDrama
    case "Comedy":
        return domain.GenreComedy
    case "Action":
        return domain.GenreAction
    case "Sci-Fi", "Science Fiction":
        return domain.GenreSciFi
    case "Horror":
        return domain.GenreHorror
    case "Thriller":
        return domain.GenreThriller
    case "Romance":
        return domain.GenreRomance
    default:
        return domain.GenreOther
    }
}
```

- [ ] **Step 3: Commit**

```bash
git add series-catalog/internal/ports/tmdb.go series-catalog/internal/infrastructure/tmdb_client.go
git commit -m "feat(series-catalog): add TMDB API client with search and details"
```

---

### Task 2.5: Application Use Cases

**Files:**
- Create: `/series-catalog/internal/application/create_series.go`
- Create: `/series-catalog/internal/application/search_series.go`

- [ ] **Step 1: Create Series Use Case**

Create: `/series-catalog/internal/application/create_series.go`
```go
package application

import (
    "context"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/domain"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/ports"
)

type CreateSeriesUseCase struct {
    tmdbClient ports.TmdbClient
    repository ports.SeriesRepository
}

func NewCreateSeriesUseCase(tmdb ports.TmdbClient, repo ports.SeriesRepository) *CreateSeriesUseCase {
    return &CreateSeriesUseCase{
        tmdbClient: tmdb,
        repository: repo,
    }
}

func (uc *CreateSeriesUseCase) Execute(ctx context.Context, tmdbId int) (*domain.Series, error) {
    // Fetch from TMDB
    series, err := uc.tmdbClient.GetSeriesDetails(ctx, tmdbId)
    if err != nil {
        return nil, err
    }

    // Check if already exists
    existing, _ := uc.repository.GetById(ctx, series.SeriesId)
    if existing != nil {
        // Update instead of create
        existing.UpdateMetadata(series.Title, series.Description, series.Genre, series.TotalSeasons, series.PosterUrl)
        err = uc.repository.Save(ctx, existing)
        return existing, err
    }

    // Save to database
    err = uc.repository.Save(ctx, series)
    if err != nil {
        return nil, err
    }

    return series, nil
}
```

- [ ] **Step 2: Search Series Use Case**

Create: `/series-catalog/internal/application/search_series.go`
```go
package application

import (
    "context"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/domain"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/ports"
)

type SearchSeriesUseCase struct {
    tmdbClient ports.TmdbClient
    repository ports.SeriesRepository
}

func NewSearchSeriesUseCase(tmdb ports.TmdbClient, repo ports.SeriesRepository) *SearchSeriesUseCase {
    return &SearchSeriesUseCase{
        tmdbClient: tmdb,
        repository: repo,
    }
}

type SearchResult struct {
    Series    *domain.Series `json:"series,omitempty"`
    TmdbId    int            `json:"tmdbId,omitempty"`
    Title     string         `json:"title"`
    InCache   bool           `json:"inCache"`
}

func (uc *SearchSeriesUseCase) Execute(ctx context.Context, query string, limit int) ([]SearchResult, error) {
    if limit == 0 {
        limit = 20
    }

    // First, search in local cache
    cached, err := uc.repository.SearchByTitle(ctx, query, limit)
    if err != nil {
        // Log but continue to TMDB
        cached = []*domain.Series{}
    }

    // Then search TMDB
    tmdbResults, err := uc.tmdbClient.SearchSeries(ctx, query)
    if err != nil {
        // If TMDB fails, return only cached results
        var results []SearchResult
        for _, s := range cached {
            results = append(results, SearchResult{
                Series:  s,
                InCache: true,
            })
        }
        return results, nil
    }

    // Merge results: prefer cached, include TMDB results not in cache
    var results []SearchResult
    cachedIds := make(map[string]bool)

    for _, s := range cached {
        results = append(results, SearchResult{
            Series:  s,
            InCache: true,
        })
        cachedIds[s.SeriesId.String()] = true
    }

    for _, tmdb := range tmdbResults {
        if !cachedIds[strconv.Itoa(tmdb.Id)] {
            results = append(results, SearchResult{
                TmdbId:  tmdb.Id,
                Title:   tmdb.Title,
                InCache: false,
            })
        }
    }

    if len(results) > limit {
        results = results[:limit]
    }

    return results, nil
}
```

- [ ] **Step 3: Add missing import to search_series.go**

Edit: `/series-catalog/internal/application/search_series.go`
Add import: `strconv`

```go
import (
    "context"
    "strconv"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/domain"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/ports"
)
```

- [ ] **Step 4: Commit**

```bash
git add series-catalog/internal/application/
git commit -m "feat(series-catalog): add create and search use cases"
```

---

### Task 2.6: Update Lambda Handler with Use Cases

**Files:**
- Modify: `/series-catalog/cmd/api/main.go`

- [ ] **Step 1: Initialize dependencies in handler**

Modify: `/series-catalog/cmd/api/main.go`
```go
package main

import (
    "context"
    "encoding/json"
    "net/http"
    "os"
    "strconv"

    "github.com/aws/aws-lambda-go/events"
    "github.com/aws/aws-lambda-go/lambda"
    "github.com/aws/aws-sdk-go-v2/service/dynamodb"
    sharedaws "github.com/lorebase/lorebase-backend/shared/pkg/aws"
    sharedhttp "github.com/lorebase/lorebase-backend/shared/pkg/http"
    "github.com/lorebase/lorebase-backend/shared/pkg/logger"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/application"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/domain"
    "github.com/lorebase/lorebase-backend/series-catalog/internal/infrastructure"
)

var (
    log              *logger.Logger
    createSeriesUC   *application.CreateSeriesUseCase
    searchSeriesUC   *application.SearchSeriesUseCase
)

func init() {
    log = logger.New("series-catalog")
    
    ctx := context.Background()
    cfg, err := sharedaws.LoadConfig(ctx)
    if err != nil {
        log.Error("Failed to load AWS config", err, nil)
        return
    }

    dynamoClient := dynamodb.NewFromConfig(cfg)
    tableName := sharedaws.LoadTableName()
    
    tmdbApiKey := os.Getenv("TMDB_API_KEY")
    if tmdbApiKey == "" {
        log.Error("TMDB_API_KEY not set", nil, nil)
        return
    }

    repo := infrastructure.NewSeriesDynamoRepository(dynamoClient, tableName)
    tmdbClient := infrastructure.NewHttpTmdbClient(tmdbApiKey)
    
    createSeriesUC = application.NewCreateSeriesUseCase(tmdbClient, repo)
    searchSeriesUC = application.NewSearchSeriesUseCase(tmdbClient, repo)
}

func handler(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    log.Info("Received request", map[string]interface{}{
        "path":   request.Path,
        "method": request.HTTPMethod,
    })

    switch request.HTTPMethod {
    case "GET":
        if request.Path == "/search" {
            return handleSearch(ctx, request)
        }
        return handleGetSeries(ctx, request)
    case "POST":
        return handleCreateSeries(ctx, request)
    default:
        return sharedhttp.ErrorResponse(http.StatusMethodNotAllowed, "METHOD_NOT_ALLOWED", "Method not allowed"), nil
    }
}

func handleSearch(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    query := request.QueryStringParameters["q"]
    if query == "" {
        return sharedhttp.ErrorResponse(http.StatusBadRequest, "MISSING_QUERY", "Query parameter 'q' is required"), nil
    }

    limit := 20
    if l := request.QueryStringParameters["limit"]; l != "" {
        if parsed, err := strconv.Atoi(l); err == nil && parsed > 0 {
            limit = parsed
        }
    }

    results, err := searchSeriesUC.Execute(ctx, query, limit)
    if err != nil {
        log.Error("Search failed", err, map[string]interface{}{"query": query})
        return sharedhttp.ErrorResponse(http.StatusInternalServerError, "SEARCH_FAILED", "Failed to search series"), nil
    }

    body, _ := json.Marshal(map[string]interface{}{
        "items": results,
        "total": len(results),
    })

    return events.APIGatewayProxyResponse{
        StatusCode: http.StatusOK,
        Body:       string(body),
    }, nil
}

func handleGetSeries(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    // Extract seriesId from path
    seriesId := request.PathParameters["seriesId"]
    if seriesId == "" {
        return sharedhttp.ErrorResponse(http.StatusBadRequest, "MISSING_ID", "Series ID is required"), nil
    }

    // TODO: Implement get by ID
    return events.APIGatewayProxyResponse{
        StatusCode: http.StatusOK,
        Body:       `{"seriesId": "` + seriesId + `"}`,
    }, nil
}

func handleCreateSeries(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    var req struct {
        TmdbId int `json:"tmdbId"`
    }
    
    if err := json.Unmarshal([]byte(request.Body), &req); err != nil {
        return sharedhttp.ErrorResponse(http.StatusBadRequest, "INVALID_BODY", "Invalid request body"), nil
    }

    if req.TmdbId == 0 {
        return sharedhttp.ErrorResponse(http.StatusBadRequest, "MISSING_TMDB_ID", "tmdbId is required"), nil
    }

    series, err := createSeriesUC.Execute(ctx, req.TmdbId)
    if err != nil {
        log.Error("Create series failed", err, map[string]interface{}{"tmdbId": req.TmdbId})
        return sharedhttp.ErrorResponse(http.StatusInternalServerError, "CREATE_FAILED", "Failed to create series"), nil
    }

    body, _ := json.Marshal(series)
    return events.APIGatewayProxyResponse{
        StatusCode: http.StatusCreated,
        Body:       string(body),
    }, nil
}

func main() {
    lambda.Start(handler)
}
```

- [ ] **Step 2: Add ErrorResponse helper to shared**

Create: `/shared/pkg/http/response.go` (if not exists)
Add function:
```go
func ErrorResponse(status int, code, message string) events.APIGatewayProxyResponse {
    body, _ := json.Marshal(map[string]interface{}{
        "error": map[string]string{
            "code":    code,
            "message": message,
        },
    })
    
    return events.APIGatewayProxyResponse{
        StatusCode: status,
        Body:       string(body),
    }
}
```

- [ ] **Step 3: Update go.mod and build**

```bash
cd /series-catalog
go mod tidy
go build -o bin/api ./cmd/api
```

- [ ] **Step 4: Commit**

```bash
git add series-catalog/
git commit -m "feat(series-catalog): wire use cases to lambda handler"
```

---

### Task 2.7: Terraform Infrastructure

**Files:**
- Create: `/series-catalog/terraform/main.tf`
- Create: `/series-catalog/terraform/variables.tf`

- [ ] **Step 1: Create Terraform configuration**

Create: `/series-catalog/terraform/main.tf`
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# DynamoDB Table
resource "aws_dynamodb_table" "series_catalog" {
  name         = "series-catalog"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "PK"
  range_key    = "SK"

  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  attribute {
    name = "TITLE"
    type = "S"
  }

  attribute {
    name = "TITLE_PREFIX"
    type = "S"
  }

  attribute {
    name = "GENRE"
    type = "S"
  }

  # GSI for genre search
  global_secondary_index {
    name            = "genre-index"
    hash_key        = "GENRE"
    range_key       = "TITLE"
    projection_type = "ALL"
  }

  # GSI for title prefix search
  global_secondary_index {
    name            = "title-index"
    hash_key        = "TITLE_PREFIX"
    range_key       = "TITLE"
    projection_type = "ALL"
  }
}

# Lambda Function
resource "aws_lambda_function" "series_catalog_api" {
  function_name = "series-catalog-api"
  runtime       = "provided.al2"
  handler       = "bootstrap"
  role          = aws_iam_role.lambda_exec.arn

  filename         = "${path.module}/../bin/api.zip"
  source_code_hash = filebase64sha256("${path.module}/../bin/api.zip")

  environment {
    variables = {
      TABLE_NAME     = aws_dynamodb_table.series_catalog.name
      TMDB_API_KEY   = var.tmdb_api_key
      LOG_LEVEL      = "info"
    }
  }

  timeout = 10
  memory_size = 512
}

# Lambda Function URL
resource "aws_lambda_function_url" "series_catalog" {
  function_name      = aws_lambda_function.series_catalog_api.function_name
  authorization_type = "AWS_IAM"
}

# IAM Role for Lambda
resource "aws_iam_role" "lambda_exec" {
  name = "series-catalog-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy" "dynamodb_access" {
  name = "dynamodb-access"
  role = aws_iam_role.lambda_exec.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:Query",
          "dynamodb:DeleteItem"
        ]
        Resource = aws_dynamodb_table.series_catalog.arn
      }
    ]
  })
}

# Outputs
output "lambda_function_url" {
  value = aws_lambda_function_url.series_catalog.function_url
}

output "lambda_function_name" {
  value = aws_lambda_function.series_catalog_api.function_name
}
```

- [ ] **Step 2: Create variables**

Create: `/series-catalog/terraform/variables.tf`
```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "tmdb_api_key" {
  description = "TMDB API Key"
  type        = string
  sensitive   = true
}
```

- [ ] **Step 3: Create deployment script**

Create: `/series-catalog/scripts/deploy.sh`
```bash
#!/bin/bash
set -e

echo "Building series-catalog..."
cd ..
go build -o bin/api bootstrap ./cmd/api
zip bin/api.zip bin/api

echo "Deploying to AWS..."
cd terraform
terraform init
terraform apply -auto-approve \
  -var="tmdb_api_key=$TMDB_API_KEY"

echo "Done!"
```

```bash
chmod +x series-catalog/scripts/deploy.sh
```

- [ ] **Step 4: Commit**

```bash
git add series-catalog/terraform/ series-catalog/scripts/
git commit -m "feat(series-catalog): add terraform infrastructure and deployment script"
```

---

## Phase 3: UserTrackingContext

**Goal:** Implement core domain for user series tracking with ratings and progress.

---

### Task 3.1: Initialize UserTracking Module

**Files:**
- Create: `/user-tracking/go.mod`
- Create: `/user-tracking/cmd/api/main.go`

- [ ] **Step 1: Create go.mod**

Create: `/user-tracking/go.mod`
```go
module github.com/lorebase/lorebase-backend/user-tracking

go 1.26

require (
    github.com/aws/aws-lambda-go v1.47.0
    github.com/aws/aws-sdk-go-v2 v1.30.0
    github.com/aws/aws-sdk-go-v2/config v1.27.0
    github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue v1.13.0
    github.com/aws/aws-sdk-go-v2/service/dynamodb v1.32.0
    github.com/lorebase/lorebase-backend/shared v0.0.0
)

replace github.com/lorebase/lorebase-backend/shared => ../shared
```

- [ ] **Step 2: Create Lambda entrypoint skeleton**

Create: `/user-tracking/cmd/api/main.go`
```go
package main

import (
    "context"
    "net/http"

    "github.com/aws/aws-lambda-go/events"
    "github.com/aws/aws-lambda-go/lambda"
    "github.com/lorebase/lorebase-backend/shared/pkg/logger"
)

var log *logger.Logger

func init() {
    log = logger.New("user-tracking")
}

func handler(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    log.Info("Received request", map[string]interface{}{
        "path":   request.Path,
        "method": request.HTTPMethod,
    })

    return events.APIGatewayProxyResponse{
        StatusCode: http.StatusOK,
        Body:       `{"status": "ok"}`,
    }, nil
}

func main() {
    lambda.Start(handler)
}
```

- [ ] **Step 3: Download dependencies**

```bash
cd /user-tracking
go mod tidy
```

- [ ] **Step 4: Commit**

```bash
git add user-tracking/
git commit -m "feat(user-tracking): initialize module and lambda skeleton"
```

---

### Task 3.2: Domain Models (Aggregate, Entity, Value Objects)

**Files:**
- Create: `/user-tracking/internal/domain/valueobject/tracking_id.go`
- Create: `/user-tracking/internal/domain/valueobject/user_id.go`
- Create: `/user-tracking/internal/domain/valueobject/series_id.go`
- Create: `/user-tracking/internal/domain/valueobject/season_rating.go`

- [ ] **Step 1: Create Value Objects**

Create all VO files:

`/user-tracking/internal/domain/valueobject/tracking_id.go`:
```go
package valueobject

type TrackingId struct {
    value string
}

func NewTrackingId(id string) TrackingId {
    return TrackingId{value: id}
}

func (t TrackingId) String() string {
    return t.value
}
```

`/user-tracking/internal/domain/valueobject/user_id.go`:
```go
package valueobject

type UserId struct {
    value string
}

func NewUserId(id string) UserId {
    return UserId{value: id}
}

func (u UserId) String() string {
    return u.value
}
```

`/user-tracking/internal/domain/valueobject/series_id.go`:
```go
package valueobject

type SeriesId struct {
    value string
}

func NewSeriesId(id string) SeriesId {
    return SeriesId{value: id}
}

func (s SeriesId) String() string {
    return s.value
}
```

`/user-tracking/internal/domain/valueobject/season_rating.go`:
```go
package valueobject

import (
    "time"
    "github.com/lorebase/lorebase-backend/shared/pkg/errors"
)

type SeasonRating struct {
    SeasonNumber int
    Rating       int       // 1-10
    RatedAt      time.Time
}

func NewSeasonRating(seasonNumber, rating int) (*SeasonRating, error) {
    if seasonNumber < 1 {
        return nil, errors.NewValidation("seasonNumber", "must be at least 1")
    }
    if rating < 1 || rating > 10 {
        return nil, errors.NewValidation("rating", "must be between 1 and 10")
    }
    
    return &SeasonRating{
        SeasonNumber: seasonNumber,
        Rating:       rating,
        RatedAt:      time.Now().UTC(),
    }, nil
}
```

- [ ] **Step 2: Create TrackingStatus enum**

Create: `/user-tracking/internal/domain/tracking_status.go`
```go
package domain

type TrackingStatus string

const (
    TrackingStatusWantToWatch TrackingStatus = "WANT_TO_WATCH"
    TrackingStatusWatching    TrackingStatus = "WATCHING"
    TrackingStatusCompleted   TrackingStatus = "COMPLETED"
    TrackingStatusDropped     TrackingStatus = "DROPPED"
)

func (t TrackingStatus) IsValid() bool {
    switch t {
    case TrackingStatusWantToWatch, TrackingStatusWatching, 
         TrackingStatusCompleted, TrackingStatusDropped:
        return true
    }
    return false
}
```

- [ ] **Step 3: Create TrackedSeries Entity**

Create: `/user-tracking/internal/domain/entity/tracked_series.go`
```go
package entity

import (
    "time"
    "github.com/lorebase/lorebase-backend/shared/pkg/errors"
    "github.com/lorebase/lorebase-backend/user-tracking/internal/domain"
    "github.com/lorebase/lorebase-backend/user-tracking/internal/domain/valueobject"
)

type TrackedSeries struct {
    TrackingId    valueobject.TrackingId
    UserId        valueobject.UserId
    SeriesId      valueobject.SeriesId
    Status        domain.TrackingStatus
    SeriesRating  *int                    // Optional, 1-10
    SeriesReview  string
    SeasonRatings []valueobject.SeasonRating
    AddedAt       time.Time
    LastUpdated   time.Time
}

func NewTrackedSeries(trackingId valueobject.TrackingId, userId valueobject.UserId, 
    seriesId valueobject.SeriesId, status domain.TrackingStatus) (*TrackedSeries, error) {
    
    if !status.IsValid() {
        return nil, errors.NewValidation("status", "invalid tracking status")
    }

    now := time.Now().UTC()
    return &TrackedSeries{
        TrackingId:      trackingId,
        UserId:          userId,
        SeriesId:        seriesId,
        Status:          status,
        SeasonRatings:   []valueobject.SeasonRating{},
        AddedAt:         now,
        LastUpdated:     now,
    }, nil
}

func (t *TrackedSeries) UpdateStatus(status domain.TrackingStatus) error {
    if !status.IsValid() {
        return errors.NewValidation("status", "invalid tracking status")
    }
    
    t.Status = status
    t.LastUpdated = time.Now().UTC()
    return nil
}

func (t *TrackedSeries) UpdateRating(rating int) error {
    if rating < 1 || rating > 10 {
        return errors.NewValidation("rating", "must be between 1 and 10")
    }
    
    t.SeriesRating = &rating
    t.LastUpdated = time.Now().UTC()
    return nil
}

func (t *TrackedSeries) UpdateReview(review string) {
    t.SeriesReview = review
    t.LastUpdated = time.Now().UTC()
}

func (t *TrackedSeries) AddSeasonRating(rating valueobject.SeasonRating) {
    // Replace if exists
    for i, sr := range t.SeasonRatings {
        if sr.SeasonNumber == rating.SeasonNumber {
            t.SeasonRatings[i] = rating
            t.LastUpdated = time.Now().UTC()
            return
        }
    }
    
    // Add new
    t.SeasonRatings = append(t.SeasonRatings, rating)
    t.LastUpdated = time.Now().UTC()
}
```

- [ ] **Step 4: Commit**

```bash
git add user-tracking/internal/domain/
git commit -m "feat(user-tracking): add domain models (VOs, Entity, Aggregate)"
```

---

## Remaining Tasks Summary

Due to document length limits, remaining tasks are outlined below. Each should be implemented as separate commits following the same pattern:

### Phase 3 Continued (UserTracking):
- Task 3.3: Repository Port & DynamoDB Implementation
- Task 3.4: Series Catalog Client (HTTP to validate series)
- Task 3.5: Application Use Cases (StartTracking, UpdateProgress, RateSeries, ListTracking)
- Task 3.6: Update Lambda Handler with Use Cases
- Task 3.7: Terraform Infrastructure

### Phase 4: RecommendationContext (Stateless):
- Task 4.1: Initialize Module
- Task 4.2: User Tracking Client
- Task 4.3: Recommendation Handler (random from Want to Watch)
- Task 4.4: Terraform Infrastructure

### Phase 5: ExperienceBFF:
- Task 5.1: Initialize Module
- Task 5.2: Cognito Auth Middleware (JWT validation)
- Task 5.3: HTTP Clients to Domain Contexts (Lambda Function URLs)
- Task 5.4: Aggregator Layer (Dashboard Builder)
- Task 5.5: Handlers (Dashboard, Series, Tracking, Recommendation)
- Task 5.6: Terraform Infrastructure (API Gateway, Cognito)

### Phase 6: Shared Infrastructure:
- Task 6.1: Terraform Backend (S3 + DynamoDB for state)
- Task 6.2: CloudWatch Alarms + SNS (email notifications)
- Task 6.3: Root Terraform Configuration

### Phase 7: Testing & Documentation:
- Task 7.1: Unit Tests (domain layer)
- Task 7.2: Integration Tests (DynamoDB with testcontainers)
- Task 7.3: Local Testing with SAM CLI
- Task 7.4: README with Setup Instructions

---

## Execution Options

**Plan complete and saved to `/home/sanmoo/dev/github.com/lorebase/lorebase-design/docs/superpowers/plans/2026-04-18-lorebase-implementation-plan.md`**

**Two execution options:**

1. **Subagent-Driven (recommended)** - Dispatch fresh subagent per task, review between tasks, fast iteration

2. **Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach would you like to use?**

**Note:** The design document has been successfully moved to:
`/home/sanmoo/dev/github.com/lorebase/lorebase-design/docs/superpowers/specs/2026-04-18-lorebase-system-design.md`

Both documents are now in the correct location (`lorebase-design` repository).
