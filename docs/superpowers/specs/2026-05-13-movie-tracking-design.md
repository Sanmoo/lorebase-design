# Movie Tracking Support — Backend Design

**Date:** 2026-05-13
**Status:** Approved
**Scope:** Add movie tracking while keeping series tracking intact, aligned with the frontend spec at `lorebase-frontend/docs/superpowers/specs/2026-05-13-movie-tracking-design.md`.

---

## 1. Executive Summary

The system currently tracks only series. This design adds full movie tracking support while keeping series behavior unchanged and laying the groundwork for future media types (games, books).

**Key Decisions:**

| Decision | Choice | Rationale |
|----------|--------|-----------|
| How to add movies to the catalog | New `MovieCatalogContext` bounded context, parallel to `SeriesCatalogContext` | Zero regression risk; follows isolated bounded context pattern |
| How to track movies | Evolve `UserTrackingContext` to be media-aware (`mediaType` + `mediaId`) | Aligns with the frontend's unified tracking contract |
| DynamoDB for movies | Separate `movie-catalog` table | Isolated infrastructure per bounded context |
| DynamoDB for tracking | Same `user-tracking` table, evolved schema | Unified queries across media types; future-proof |
| BFF endpoints | New `/movies` endpoints; tracking/dashboard refactored with `mediaType` | Matches frontend spec exactly |

---

## 2. Domain Model Evolution

### 2.1 Updated Context Map

```
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│ SeriesCatalogContext │  │ MovieCatalogContext   │  │ UserTrackingContext   │
│  (INALTERADO)        │  │  (NOVO)               │  │  (EVOLUÍDO)           │
│                      │  │  Go + Lambda           │  │  mediaType + mediaId  │
│  DynamoDB:           │  │  DynamoDB:             │  │  rating, review       │
│  series-catalog      │  │  movie-catalog         │  │  DynamoDB:            │
│  TMDB /search/tv     │  │  TMDB /search/movie    │  │  user-tracking        │
│  TMDB /tv/{id}       │  │  TMDB /movie/{id}      │  │  (mesma tabela)       │
└──────────┬───────────┘  └───────────┬────────────┘  └──────────┬───────────┘
           │                          │                           │
           ▼                          ▼                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         ExperienceBFFContext                                   │
│                                                                                │
│  /dashboard?mediaType=series|movie     /tracking?mediaType=series|movie        │
│  /series?q=...  /series/:id            /movies?q=...  /movies/:id    (NOVOS)  │
│  /recommendation/next                                                         │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 New Bounded Context: MovieCatalogContext

Type: Supporting Domain. Implementation: Go + AWS Lambda. Responsibility: movie metadata, search, detail.

**Entities:**
- `Movie`: `MovieId`, `Title`, `Overview`, `Genre`, `Runtime` (minutes), `ReleaseDate`, `PosterUrl`, `TmdbId`, `LastUpdated`

**Value Objects:**
- `MovieId` (independent from `SeriesId` — no shared kernel)

### 2.3 Evolved Bounded Context: UserTrackingContext

The aggregate `UserSeriesTracking` is renamed to `UserMediaTracking`. The entity `TrackedSeries` becomes `TrackedMedia`.

**New Value Object:**
- `MediaType`: enum `{ SERIES, MOVIE }`

**Renamed Fields (entity):**
| Old Name | New Name | Notes |
|----------|----------|-------|
| `SeriesId` | `MediaId` | String, holds TMDB ID |
| `seriesRating` | `rating` | Nullable int 1-10 |
| `seriesReview` | `review` | String |
| — | `mediaType` | New required field |

**Preserved:**
- `TrackingStatus` enum (`WANT_TO_WATCH`, `WATCHING`, `COMPLETED`, `DROPPED`) — unchanged
- `SeasonRating` — kept for series, ignored for movies in UI
- All other VOs (`TrackingId`, `UserId`) — unchanged

### 2.4 Unchanged Contexts

- **SeriesCatalogContext** — zero changes
- **RecommendationContext** — out of scope (frontend spec excludes recommend flow)
- **ExperienceContext** (frontend) — handled in `lorebase-frontend` repo

---

## 3. Data Model (DynamoDB)

### 3.1 New Table: `movie-catalog`

Mirror of `series-catalog` with movie-specific attributes.

| PK | SK | Attributes |
|----|----|-----------|
| `MOVIE#<id>` | `META#<id>` | `title`, `overview`, `genre`, `runtime`, `releaseDate`, `posterUrl`, `lastUpdated`, `tmdbId` |

**GSI1 (Genre Index):** PK=`GENRE#<genre>`, SK=`TITLE#<title>`
**GSI2 (Title Search):** PK=`TITLE_PREFIX#<first_3_chars>`, SK=`TITLE#<title>`

Differences from `series-catalog`: `runtime` + `releaseDate` replace `totalSeasons` + `description`.

### 3.2 Evolved Table: `user-tracking` (same table)

| PK | SK | Attributes |
|----|----|-----------|
| `USER#<userId>` | `TRACKING#<mediaId>` | `mediaType`, `status`, `rating`, `review`, `seasonRatings`, `addedAt`, `lastUpdated` |
| `USER#<userId>` | `SEASON_RATING#<mediaId>#<seasonNum>` | `rating`, `ratedAt` (series-only) |

**GSI1 (Media Index):** PK=`MEDIA#<mediaId>` (was `SERIES#<seriesId>`), SK=`USER#<userId>`
**GSI2 (Status Index):** PK=`STATUS#<status>`, SK=`MEDIA_TYPE#<mediaType>#USER#<userId>#MEDIA#<mediaId>`

GSI2 now includes `mediaType` in the sort key to enable efficient `status + mediaType` filtered queries.

### 3.3 Migration

Existing tracking records have no `mediaType` attribute. A backfill script sets `mediaType = "series"` on all records missing the field. DynamoDB GSIs are rebuilt to reflect the new key schemas (GSI1 `MEDIA#` prefix, GSI2 sort key format).

---

## 4. API Contract (ExperienceBFF)

Aligned with frontend spec `lorebase-frontend/docs/superpowers/specs/2026-05-13-movie-tracking-design.md`.

### 4.1 Unchanged Endpoints

```
GET  /series?q={query}&page={n}&limit={20}
GET  /series/{tmdbId}
GET  /recommendation/next
```

### 4.2 New Endpoints

```
GET  /movies?q={query}&page={n}&limit={20}
GET  /movies/{tmdbId}
```

Response shapes mirror `/series` but with `movieId`, `title`, `overview`, `runtime`, `releaseDate`, `posterUrl`.

### 4.3 Evolved Endpoints

All tracking and dashboard endpoints now accept `mediaType`.

```
GET  /dashboard?mediaType={series|movie}
     Response: stats and status sections filtered by requested mediaType only.

GET  /tracking?status={status}&mediaType={series|movie}&page={n}&limit={20}
     Response items now include:
       - mediaType (string, "series" or "movie")
       - mediaId (string, was seriesId)
       - rating (int, was seriesRating)
       - review (string, was seriesReview)
       - seasonRatings (preserved, series-only in UI)
       - media.mediaType (string, embedded summary)

POST /tracking
     Body: { mediaId, mediaType, status }
     Validates mediaId against SeriesCatalog (if series) or MovieCatalog (if movie).

PUT  /tracking/{mediaId}
     Body: { mediaType, status?, rating?, review?, seasonRatings? }

DELETE /tracking/{mediaId}?mediaType={series|movie}
```

---

## 5. Repository Structure

### 5.1 New Module: `/movie-catalog/`

```
/movie-catalog/
├── go.mod
├── /cmd/api/main.go
├── /internal/
│   ├── /domain/
│   │   ├── movie.go              # Movie entity
│   │   └── value_objects.go      # MovieId
│   ├── /application/
│   │   ├── create_movie.go       # TMDB → DynamoDB
│   │   └── search_movie.go       # DynamoDB + TMDB fallback
│   ├── /infrastructure/
│   │   ├── movie_repository.go   # DynamoDB implementation
│   │   └── tmdb_movie_client.go  # TMDB /search/movie + /movie/{id}
│   └── /ports/
│       ├── repository.go
│       └── tmdb.go
└── /terraform/
```

### 5.2 Modified Module: `/user-tracking/`

Files renamed/added:

| Path | Change |
|------|--------|
| `domain/aggregate/user_series_tracking.go` | → `user_media_tracking.go` |
| `domain/entity/tracked_series.go` | → `tracked_media.go` |
| `domain/valueobject/series_id.go` | → `media_id.go` |
| `domain/valueobject/media_type.go` | New: `MediaType` enum |
| `infrastructure/tracking_repository.go` | Updated PK/SK patterns |
| `infrastructure/movie_catalog_client.go` | New: validates movies via HTTP |
| `terraform/main.tf` | IAM policy for MovieCatalog Function URL |

### 5.3 Modified Module: `/experience-bff/`

| Path | Change |
|------|--------|
| `handlers/movies.go` | New handler for `/movies` routes |
| `handlers/tracking.go` | Accept `mediaType`, map renamed fields |
| `handlers/dashboard.go` | Filter by `mediaType` query param |
| `clients/movie_catalog_client.go` | New HTTP client |
| `dto/responses.go` | Add `mediaType`, rename `seriesId` → `mediaId`, `seriesRating` → `rating` |

### 5.4 Unchanged Modules

- `/shared/` — no changes needed
- `/series-catalog/` — no changes
- `/recommendation/` — no changes

---

## 6. TMDB Integration

The existing `HttpTmdbClient` in SeriesCatalog uses `/search/tv` and `/tv/{id}`.

The new `TmdbMovieClient` in MovieCatalog uses:
- `/search/movie?query=...&language=pt-BR` — search
- `/movie/{id}?language=pt-BR` — details

Same patterns: rate limiting, retry with exponential backoff, hybrid cache strategy (DynamoDB first, TMDB fallback, save on select).

Movie entity mapping from TMDB:
```
TMDB /movie/{id} response:
  id              → MovieId
  title           → Title
  overview        → Overview
  genres[0].name  → Genre
  runtime         → Runtime (minutes)
  release_date    → ReleaseDate
  poster_path     → PosterUrl
```

---

## 7. Inter-Context Communication

UserTrackingContext validates that a `mediaId` exists before creating a tracking entry:

```
UserTracking Lambda
    │
    ├── mediaType == "series" ──HTTP──► SeriesCatalog Lambda Function URL
    │                                   GET /validate?mediaId=...
    │
    └── mediaType == "movie"  ──HTTP──► MovieCatalog Lambda Function URL
                                        GET /validate?mediaId=...
```

Same Lambda Function URL + IAM auth pattern used throughout.

---

## 8. Security, Error Handling, Observability

No new patterns introduced. Existing strategies apply identically:

- **Auth:** Cognito JWT validation in BFF (unchanged)
- **Authorization:** User-scoped data via JWT userId (unchanged)
- **Error handling:** Standard `{ code, message, details }` response (unchanged)
- **Observability:** Structured JSON logs, CloudWatch metrics, SNS alarms (unchanged)
- **Fallbacks:** TMDB unavailable → stale cache; DynamoDB throttling → retry (unchanged)

---

## 9. Alignment with Frontend Spec

This design implements the backend side of the contract defined in `lorebase-frontend/docs/superpowers/specs/2026-05-13-movie-tracking-design.md`:

| Frontend Expectation | Backend Implementation |
|---------------------|----------------------|
| `MediaType` enum (`series`, `movie`) | `MediaType` VO in UserTrackingContext |
| `TrackedMediaModel` with `mediaType`, `mediaId`, `rating`, `review` | `TrackedMedia` entity with renamed fields |
| `GET /tracking` accepts `mediaType` | BFF handler passes param to UserTracking |
| `GET /dashboard` accepts `mediaType` | BFF handler filters stats/sections |
| `POST /tracking { mediaId, mediaType, status }` | Validates against correct Catalog |
| `GET /movies?q=...` and `GET /movies/:id` | MovieCatalogContext + BFF handler |
| `seasonRatings` preserved for series only | Kept in entity, ignored for movies in UI |

---

## 10. Future Media Types

This design intentionally prepares for games, books, etc.:

- `MediaType` enum is extensible (add `GAME`, `BOOK`)
- `UserTrackingContext` is already media-agnostic
- New catalog contexts can be added in parallel (e.g., `GameCatalogContext`)
- Only the BFF needs new handlers and the tracking validation chain needs a new branch

---

## 11. Appendix: CML Updates

The Domain model file `docs/domain-modeling/media-tracker.cml` should be updated to reflect:

- New `MovieCatalog` Subdomain (SUPPORTING_DOMAIN)
- New `MovieCatalogContext` bounded context
- Renamed aggregate: `UserSeriesTracking` → `UserMediaTracking`
- Renamed entity: `TrackedSeries` → `TrackedMedia`
- New VO: `MediaType`
- Updated context map relationships

---

**End of Specification**
