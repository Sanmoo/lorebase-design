# Game Tracking Support — Domain Design

**Date:** 2026-06-13
**Status:** Approved for specification review
**Scope:** Add electronic game cataloging and user preferences/tracking to Lorebase, analogous to existing series and movie support.

---

## 1. Executive Summary

Lorebase currently models media tracking around dedicated catalog contexts for series and movies plus a shared `UserTrackingContext` for user lists, ratings, reviews, and status. This design adds electronic games as a first-class media type while preserving the existing bounded-context pattern.

**Key Decisions:**

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Game metadata ownership | New `GameCatalogContext` | Mirrors `SeriesCatalogContext` and `MovieCatalogContext`; avoids regressions in existing catalogs |
| External game provider | IGDB/Twitch API | Rich structured game metadata: platforms, companies, franchises, modes, covers |
| Provider isolation | `GameMetadataProvider` port with IGDB adapter | Keeps domain independent from vendor details and enables future provider replacement |
| User preferences | Extend `UserTrackingContext` with `MediaType = GAME` | Keeps user lists and preferences unified across media types |
| Game statuses | Game-specific statuses: `WANT_TO_PLAY`, `PLAYING`, `COMPLETED`, `DROPPED` | Preserves game semantics instead of forcing watch-oriented terminology |
| Game-specific tracking fields | `selectedPlatform`, `hoursPlayed` | Captures the minimum game-specific preferences/progress requested for MVP |

---

## 2. Domain Model Evolution

### 2.1 New Subdomain: `GameCatalog`

Type: Supporting Domain.

Vision: manage structured metadata for electronic games so users can find games and attach personal preferences/tracking to them.

Entity: `Game`

| Field | Description |
|-------|-------------|
| `gameId` | Internal game identifier, initially mapped to IGDB ID |
| `title` | Game title |
| `summary` | Description/summary |
| `genre` | Primary genre or normalized genre label |
| `coverUrl` | Cover image URL |
| `releaseDate` | First release date |
| `platforms[]` | Supported platforms, e.g. PC, PlayStation, Xbox, Switch |
| `developer` | Main developer company |
| `publisher` | Main publisher company |
| `franchise` | Franchise/collection name when available |
| `gameModes[]` | Modes such as single-player, multiplayer, co-op |
| `igdbId` | External IGDB identifier |
| `lastUpdated` | Cache/update timestamp |

### 2.2 New Bounded Context: `GameCatalogContext`

| Property | Value |
|---|---|
| **Type** | Feature / Supporting Domain implementation |
| **Technology** | Go / AWS Lambda |
| **Responsibilities** | Game search, game detail, metadata normalization, IGDB integration, cache persistence |
| **Knowledge Level** | Concrete |

The context exposes game search/detail operations to the BFF and validation operations to `UserTrackingContext`.

### 2.3 Evolved `UserTrackingContext`

`UserTrackingContext` remains the owner of user-specific lists, preferences, ratings, reviews, and progress. It is extended to support `MediaType = GAME`.

Game records use the existing media-aware tracking model plus two game-specific fields:

| Field | Applies To | Notes |
|-------|------------|-------|
| `mediaType` | All media | Add `GAME` to existing supported values |
| `status` | All media | Validate allowed statuses by media type |
| `rating` | All media | Existing 1-10 user rating |
| `review` | All media | Existing user review |
| `selectedPlatform` | Games only | Platform the user plays or intends to play on |
| `hoursPlayed` | Games only | Non-negative number of hours played |

#### Game Tracking Statuses

Games use their own status vocabulary:

```text
WANT_TO_PLAY
PLAYING
COMPLETED
DROPPED
```

Series and movies keep their existing watch-oriented statuses. The tracking domain validates allowed statuses by `mediaType` instead of forcing one global semantic enum.

---

## 3. Data Model

### 3.1 New DynamoDB Table: `game-catalog`

| PK | SK | Attributes |
|----|----|------------|
| `GAME#<id>` | `META#<id>` | `title`, `summary`, `genre`, `coverUrl`, `releaseDate`, `platforms`, `developer`, `publisher`, `franchise`, `gameModes`, `igdbId`, `lastUpdated` |

Recommended indexes:

| Index | PK | SK | Purpose |
|-------|----|----|---------|
| GSI1 Title Search | `TITLE_PREFIX#<prefix>` | `TITLE#<title>` | Local/cache title search |
| GSI2 Genre | `GENRE#<genre>` | `TITLE#<title>` | Browse/filter by genre |
| GSI3 Platform | `PLATFORM#<platform>` | `TITLE#<title>` | Browse/filter by platform |

The catalog should follow the same hybrid strategy used by existing catalog contexts: query local cache first where useful, fall back to the external provider, and persist selected/detail records.

### 3.2 Evolved `user-tracking` Table

The existing table remains the owner of user tracking records. Game tracking records add optional game-specific attributes.

| PK | SK | Attributes |
|----|----|------------|
| `USER#<userId>` | `TRACKING#<mediaType>#<mediaId>` | `mediaType`, `status`, `rating`, `review`, `selectedPlatform`, `hoursPlayed`, `addedAt`, `lastUpdated` |

If the current implementation still uses `TRACKING#<mediaId>`, the implementation plan should evaluate whether a key migration is required to avoid ID collisions across providers/media types. The target design is media-type-aware keys.

---

## 4. IGDB/Twitch Integration

`GameCatalogContext` uses IGDB as the initial external metadata source.

### 4.1 Authentication

IGDB uses Twitch OAuth app authentication. This is backend-to-backend and does not involve the Lorebase user.

Flow:

1. Store `TWITCH_CLIENT_ID` and `TWITCH_CLIENT_SECRET` as backend secrets.
2. Request an app access token from `https://id.twitch.tv/oauth2/token` with `grant_type=client_credentials`.
3. Call IGDB endpoints with:
   - `Client-ID: <client_id>`
   - `Authorization: Bearer <app_access_token>`
4. Cache and renew the token when it expires.

### 4.2 Provider Port

The domain/application layer depends on a provider port, not directly on IGDB:

```go
type GameMetadataProvider interface {
    SearchGames(ctx context.Context, query string, page, limit int) ([]GameSearchResult, error)
    GetGameDetails(ctx context.Context, providerGameId string) (GameDetails, error)
}
```

The infrastructure layer provides an IGDB adapter responsible for query syntax, authentication, token caching, rate limiting, retries, and field mapping.

### 4.3 Mapping

IGDB data should be normalized into the internal `Game` model:

| Internal Field | IGDB Source Concept |
|----------------|---------------------|
| `gameId`, `igdbId` | Game ID |
| `title` | Game name |
| `summary` | Summary/storyline where available |
| `genre` | Genres |
| `coverUrl` | Cover image |
| `releaseDate` | First release date / release dates |
| `platforms[]` | Platforms |
| `developer` | Involved companies with developer role |
| `publisher` | Involved companies with publisher role |
| `franchise` | Franchise/collection |
| `gameModes[]` | Game modes |

---

## 5. API Contract — ExperienceBFF

### 5.1 New Game Catalog Routes

```http
GET /games?q={query}&page={n}&limit={20}
GET /games/{igdbId}
```

Game search result shape:

```json
{
  "gameId": "1020",
  "title": "Example Game",
  "summary": "...",
  "genre": "RPG",
  "coverUrl": "https://...",
  "releaseDate": "2026-01-01",
  "platforms": ["PC", "PlayStation 5"],
  "developer": "Example Studio",
  "publisher": "Example Publisher",
  "franchise": "Example Franchise",
  "gameModes": ["Single player", "Co-op"]
}
```

### 5.2 Evolved Tracking Routes

Tracking endpoints accept `mediaType = game` and game-specific fields.

```http
POST /tracking
Content-Type: application/json

{
  "mediaId": "1020",
  "mediaType": "game",
  "status": "WANT_TO_PLAY",
  "selectedPlatform": "PC"
}
```

```http
PUT /tracking/{mediaId}
Content-Type: application/json

{
  "mediaType": "game",
  "status": "PLAYING",
  "rating": 9,
  "review": "Great combat and world design.",
  "selectedPlatform": "PC",
  "hoursPlayed": 42
}
```

```http
GET /tracking?status=PLAYING&mediaType=game&page=1&limit=20
GET /dashboard?mediaType=game
DELETE /tracking/{mediaId}?mediaType=game
```

Validation rules:

- `mediaType=game` requires status from the game status set.
- `selectedPlatform`, when provided, should match one of the game's known platforms where feasible.
- `hoursPlayed` must be non-negative.
- `rating`, when provided, follows the existing 1-10 rule.

---

## 6. Inter-Context Communication

### 6.1 BFF → GameCatalogContext

`ExperienceBFFContext` adds a game catalog client and exposes `/games` routes. Communication follows the existing Lambda Function URL + IAM pattern.

### 6.2 UserTrackingContext → GameCatalogContext

`UserTrackingContext` validates game existence before creating/updating game tracking records:

```text
UserTracking Lambda
  ├── mediaType == series → SeriesCatalogContext
  ├── mediaType == movie  → MovieCatalogContext
  └── mediaType == game   → GameCatalogContext
```

### 6.3 SocialContext

`SocialContext` should accept tracking events with `mediaType = GAME`. It can render activity feed entries such as:

- started playing;
- rated;
- reviewed;
- completed;
- dropped;
- added to list.

No direct dependency from `SocialContext` to `GameCatalogContext` is required for MVP if tracking events carry denormalized `mediaTitle` and `mediaType` as they do for other media.

### 6.4 RecommendationContext

`RecommendationContext` can include games in media-type-filtered recommendations when requested for `mediaType=game`. The MVP can reuse existing list-based recommendation behavior, selecting from `WANT_TO_PLAY` records instead of `WANT_TO_WATCH`.

---

## 7. Domain Modeling Artifacts

The implementation must update the existing Context Mapper model and regenerate the derived diagram artifacts.

### 7.1 Rewrite `media-tracker.cml`

Rewrite `docs/domain-modeling/media-tracker.cml` so the model includes:

- `GameCatalog` subdomain;
- `GameCatalogContext` bounded context;
- `GAME` in the media type model;
- game tracking status vocabulary;
- game-specific tracking fields;
- context-map relationships:
  - `UserTrackingContext` consumes `GameCatalogContext` for validation;
  - `ExperienceBFFContext` consumes `GameCatalogContext`;
  - `ExperienceBFFContext` continues to consume `UserTrackingContext`;
  - `ExperienceContext` continues to consume only the BFF.

The rewrite should preserve the existing series, movie, tracking, recommendation, social, BFF, and frontend contexts while adding game support consistently.

### 7.2 Regenerate context map diagrams

After updating the CML, regenerate the Context Mapper diagram outputs under `docs/domain-modeling/`, including at minimum:

- `media-tracker_ContextMap.gv`;
- `media-tracker_ContextMap.png`.

If the existing workflow also produces `media-tracker_ContextMap.svg`, regenerate it as well so all checked-in derived artifacts remain synchronized with `media-tracker.cml`.

---

## 8. Error Handling, Security, Observability

No new system-wide patterns are required.

- **Auth:** BFF endpoints continue to require Cognito JWT.
- **Internal auth:** Lambda Function URL + IAM for context-to-context calls.
- **Secrets:** Twitch client secret stored in AWS Secrets Manager or equivalent encrypted parameter store.
- **External failures:** IGDB unavailable should return a clear catalog-provider error for live search; cached detail records may be served when available.
- **Rate limits:** IGDB adapter handles backoff, retries, and request throttling.
- **Logging:** Structured logs include provider operation, IGDB ID, cache hit/miss, and sanitized error codes.

---

## 9. Out of Scope

- User achievements/trophies.
- Per-platform ownership library import.
- Storefront price tracking.
- Multiplayer matchmaking or social gaming features.
- Advanced game recommendation algorithms beyond existing list-based recommendations.
- Replacing existing series/movie catalog architecture with a unified catalog context.

---

## 10. Acceptance Criteria

The design is complete when:

1. Lorebase has a `GameCatalogContext` design parallel to series and movies.
2. User tracking supports `MediaType = GAME`.
3. Games have status values `WANT_TO_PLAY`, `PLAYING`, `COMPLETED`, `DROPPED`.
4. Game tracking supports `selectedPlatform` and `hoursPlayed`.
5. IGDB/Twitch API is the selected provider, isolated behind `GameMetadataProvider`.
6. BFF routes for game search/detail and game tracking are specified.
7. `docs/domain-modeling/media-tracker.cml` rewrite requirements are explicit.
8. Regeneration of `media-tracker_ContextMap.gv`, `media-tracker_ContextMap.png`, and existing SVG outputs is explicit.

---

**End of Specification**
