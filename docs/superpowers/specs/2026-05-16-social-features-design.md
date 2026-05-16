# Social Features — Domain Design (DDD)

**Date:** 2026-05-16
**Status:** Approved
**Scope:** Add social networking capabilities: user profiles, follow/unfollow, activity feed, and similar-user discovery — aligned with existing Clean Architecture and Bounded Context isolation patterns.

---

## 1. Executive Summary

Lorebase currently supports personal media tracking (series, movies). This design adds a social layer enabling users to discover people with similar tastes, follow them, and see their recent activity in a feed — which in turn drives content discovery.

**Key Decisions:**

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Where social logic lives | New `SocialContext` bounded context | Follows existing isolation pattern; zero regression risk |
| Dependency direction | UserTracking is upstream, SocialContext is downstream | No circular dependencies; UserTracking emits domain events without knowing consumers |
| Visibility model | Binary: profile public or private | Simplicity for MVP; public means everything visible and followable |
| Profile scope | Minimal: display name + avatar + media lists | Aligned with MVP scope |
| Discovery algorithm | Hybrid: media overlap (60%) + genre similarity (40%) | Balances concrete overlap with taste preferences |

---

## 2. Domain Model

### 2.1 New Bounded Context: `SocialContext`

| Property | Value |
|---|---|
| **Type** | Core Domain |
| **Technology** | Go + AWS Lambda |
| **Responsibilities** | User profiles, social graph, activity feed, user discovery |

### 2.2 Entities & Value Objects

**UserProfile** (aggregate root):

| Field | Type | Notes |
|-------|------|-------|
| `UserId` | VO (independent) | From Cognito; independent across contexts |
| `DisplayName` | string | Public display name |
| `AvatarUrl` | string | Profile picture URL |
| `IsPublic` | bool | `true` = visible in discovery, followable, activities appear in followers' feeds |
| `CreatedAt` | timestamp | |
| `UpdatedAt` | timestamp | |

**FollowRelationship:**

| Field | Type |
|-------|------|
| `FollowerId` | UserId |
| `FollowedId` | UserId |
| `FollowedAt` | timestamp |

**ActivityEvent:**

| Field | Type | Description |
|-------|------|-------------|
| `EventId` | VO | Unique identifier |
| `UserId` | UserId | Who performed the action |
| `ActivityType` | enum | `STARTED_WATCHING`, `RATED`, `REVIEWED`, `COMPLETED`, `DROPPED`, `ADDED_TO_LIST` |
| `MediaId` | string | TMDB ID |
| `MediaType` | enum | `SERIES` or `MOVIE` |
| `MediaTitle` | string | Denormalized for feed display |
| `Metadata` | map | e.g. `{"rating": 9, "status": "WATCHING"}` |
| `CreatedAt` | timestamp | |

---

## 3. Context Map & Dependencies

### 3.1 Relationship Diagram

```
┌─────────────────────────┐
│  UserTrackingContext    │  UPSTREAM
│  (Core Domain)          │
│                         │
│  Tracking, ratings,     │
│  lists, progress        │
│                         │
│  Emits domain event:    │
│  TrackingActionCompleted│
└────────┬────────────────┘
         │
         │ [U,OHS,PL] Partnership
         │ UserTracking publishes events
         │ SocialContext consumes them
         ▼
┌─────────────────────────┐
│  SocialContext           │  DOWNSTREAM
│  (Core Domain - NEW)    │
│                         │
│  Profile, social graph, │
│  feed, discovery        │
│                         │
│  Knows UserTracking     │
│  (queries for discovery)│
└────────┬────────────────┘
         │
         │ [U,S] Customer-Supplier
         │ BFF aggregates SocialContext
         ▼
┌─────────────────────────┐
│  ExperienceBFFContext    │
│  (BFF)                  │
│                         │
│  New social routes      │
└─────────────────────────┘
```

### 3.2 Dependency Direction (no cycles)

```
UserTracking ──(does NOT know)──► SocialContext   ✅
SocialContext ──(knows)─────────► UserTracking    ✅ (discovery only)
BFF ──(knows)──────────────────► SocialContext    ✅
BFF ──(knows)──────────────────► UserTracking     ✅ (existing)
```

### 3.3 Domain Event: `TrackingActionCompleted`

Published by UserTrackingContext whenever a tracking action completes successfully.

| Field | Type | Description |
|-------|------|-------------|
| `userId` | UserId | Who performed the action |
| `activityType` | enum | `STARTED_WATCHING`, `RATED`, `REVIEWED`, `COMPLETED`, `DROPPED`, `ADDED_TO_LIST` |
| `mediaId` | MediaId | TMDB ID of the media |
| `mediaType` | MediaType | `SERIES` or `MOVIE` |
| `mediaTitle` | string | Denormalized for feed display |
| `metadata` | map | Contextual data (rating, status, review preview) |
| `timestamp` | datetime | When the action occurred |

UserTracking publishes the event but **does not know and does not depend on** who consumes it.

SocialContext consumes this event to build activity feeds.

The transport mechanism (EventBridge, direct HTTP, etc.) is an implementation detail decided at build time.

---

## 4. Public API Contracts (ExperienceBFF)

### 4.1 Profile

```
GET /users/{userId}/profile

Response 200:
{
  "userId": "...",
  "displayName": "Fulano",
  "avatarUrl": "https://...",
  "isPublic": true,
  "followingCount": 42,
  "followerCount": 38,
  "isFollowedByMe": false,
  "createdAt": "2026-05-01T10:00:00Z"
}

Errors:
  404 — Profile not found or private
```

```
PUT /users/me/profile
Body: { "displayName": "...", "avatarUrl": "...", "isPublic": true }
Response: 200 — Updated profile
```

### 4.2 Social Graph

```
POST   /users/{userId}/follow              → 201 | 409 (already following)
DELETE /users/{userId}/follow              → 204
GET    /users/me/following?page=1&limit=20 → { items[], total, hasMore }
GET    /users/me/followers?page=1&limit=20 → { items[], total, hasMore }
GET    /users/{userId}/followers?page=1&limit=20 → { items[], total, hasMore }
GET    /users/{userId}/following?page=1&limit=20 → { items[], total, hasMore }
```

Each item in the response:

```json
{ "userId": "...", "displayName": "...", "avatarUrl": "...", "followedAt": "..." }
```

### 4.3 Feed

```
GET /feed?page=1&limit=20

Response:
{
  "items": [
    {
      "eventId": "...",
      "user": { "userId": "...", "displayName": "...", "avatarUrl": "..." },
      "activityType": "RATED",
      "mediaId": "...",
      "mediaType": "SERIES",
      "mediaTitle": "Breaking Bad",
      "metadata": { "rating": 9 },
      "createdAt": "2026-05-16T14:30:00Z"
    }
  ],
  "hasMore": true
}
```

### 4.4 Discovery

```
GET /discover?limit=10

Response:
{
  "items": [
    {
      "userId": "...",
      "displayName": "...",
      "avatarUrl": "...",
      "affinityScore": 0.85,
      "commonGenres": ["Sci-Fi", "Drama"],
      "commonMedia": ["Breaking Bad", "Interstellar"],
      "isFollowedByMe": false
    }
  ]
}
```

### 4.5 Dashboard

**Unchanged.** Feed and dashboard remain separate. No social section is added to `/dashboard` in this scope.

---

## 5. Inter-Context Communication

### 5.1 SocialContext → UserTrackingContext (Discovery)

SocialContext queries UserTracking for user tracking summaries to compute affinity scores:

```
GET /internal/summary/{userId}

Response:
{
  "genres": {
    "Sci-Fi": 12,
    "Drama": 8,
    "Comedy": 3
  },
  "recentMedia": [
    { "mediaId": "tt0903747", "mediaType": "SERIES", "title": "Breaking Bad", "rating": 9 },
    { "mediaId": "tt0816692", "mediaType": "MOVIE", "title": "Interstellar", "rating": 8 }
  ],
  "totalTracked": 25
}
```

Communication via Lambda Function URL + IAM auth (same pattern as existing contexts).

### 5.2 UserTrackingContext → SocialContext (Activity Events)

UserTracking publishes `TrackingActionCompleted` domain events. SocialContext consumes them.

Transport mechanism (EventBridge, direct HTTP, etc.) is an implementation detail decided during build.

### 5.3 BFF → SocialContext (All social routes)

The BFF adds an HTTP client for SocialContext and exposes social routes as described in Section 4. Same Lambda Function URL + IAM auth pattern.

---

## 6. Discovery Algorithm

The SocialContext computes an affinity score for each public profile candidate against the requesting user:

```
score = (overlapScore × 0.6) + (genreScore × 0.4)

overlapScore = |common media| / |requesting user's total media|

genreScore = cosine similarity between the two users' genre vectors
             e.g., {"Sci-Fi": 12, "Drama": 8} vs {"Sci-Fi": 5, "Drama": 10}
```

**Flow:**
1. Query `UserTracking GET /internal/summary/{userId}` for the requesting user
2. List all public profiles from SocialContext's own store
3. For each candidate, query `UserTracking GET /internal/summary/{candidateId}`
4. Compute affinity score
5. Remove users already followed
6. Sort by score descending, return top N

**MVP scope:** On-demand computation. No caching, no pre-computation.

---

## 7. Visibility & Authorization

### 7.1 Visibility Model

| Profile | Appears in discovery? | Can be followed? | Activities appear in feed? |
|---------|----------------------|-------------------|---------------------------|
| Public | Yes | Yes | Yes (to followers) |
| Private | No | No | — |

When a user switches from public to private:
- Removed from discovery results
- Can no longer be followed by new users
- Existing followers retain the relationship but stop receiving new activities in their feed

### 7.2 Authorization Rules (enforced by BFF)

| Operation | Rule |
|-----------|------|
| `PUT /users/me/profile` | Self only |
| `GET /users/{userId}/profile` | Any authenticated user (if profile is public) |
| `POST /users/{userId}/follow` | Any authenticated user (if target profile is public) |
| `DELETE /users/{userId}/follow` | Self only (unfollowing) |
| `GET /feed` | Self only (feed is personal) |
| `GET /discover` | Any authenticated user |

### 7.3 Authentication

All social endpoints require Cognito JWT in `Authorization` header (same as existing endpoints).

---

## 8. Alignment with Existing Design

This design follows all conventions established in the existing system:

- **Bounded Context isolation:** New `SocialContext` with its own module, Lambda, and data store
- **No Shared Kernel:** Independent Value Objects (`UserId`, `EventId`)
- **Clean Architecture:** SocialContext follows domain/application/infrastructure layers
- **Layered Architecture:** BFF adds handlers + client, no domain logic
- **Inter-context communication:** Lambda Function URLs + IAM auth
- **Domain Events:** Extends the existing event-driven thinking (present in the original spec as "Future")

---

## 9. Unchanged Contexts

- **SeriesCatalogContext** — zero changes
- **MovieCatalogContext** — zero changes
- **RecommendationContext** — zero changes
- **UserTrackingContext** — additions only: new domain event + `/internal/summary` endpoint; no modifications to existing logic

---

## 10. What Is NOT in Scope (Post-MVP)

| Feature | Rationale |
|---------|-----------|
| Comments on reviews | Separate aggregate + domain events needed |
| Groups / watch parties | New aggregate or context |
| Algorithmic feed curation | Current chronological feed is sufficient for MVP |
| Badges / achievements | Separate context |
| Privacy granularity (per-item visibility) | Too complex for MVP; binary public/private is sufficient |
| Push notifications for feed updates | Mobile push infrastructure not yet present |

The design naturally supports these additions without breaking changes to the current domain model.

---

## 11. Open Questions

| Question | Resolution |
|----------|------------|
| Transport for domain events | Deferred to implementation phase (options: EventBridge, direct HTTP, SQS) |
| DynamoDB schema for SocialContext | Deferred to implementation phase |
| Exact file/package structure | Deferred to implementation phase |

---

**End of Specification**
