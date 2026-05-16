# Social Features Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add social networking capabilities (profiles, follow/unfollow, activity feed, user discovery) through a new `SocialContext` bounded context, with supporting additions to `UserTrackingContext` and `ExperienceBFFContext`.

**Architecture:** New `SocialContext` (Go + AWS Lambda) isolated from existing contexts. `UserTrackingContext` emits `TrackingActionCompleted` domain events (upstream, no dependency on SocialContext). `SocialContext` consumes events and queries `UserTrackingContext` for discovery. `ExperienceBFFContext` exposes new social routes aggregating `SocialContext`.

**Tech Stack:** Go + AWS Lambda + DynamoDB (same as existing contexts). Inter-context communication via Lambda Function URLs + IAM auth. Domain events via EventBridge (or equivalent — transport decided during implementation).

---

### Task 1: UserTrackingContext — Domain Event + Summary Endpoint

**Context:** UserTrackingContext already handles tracking actions (add to list, update status, rate, review, complete, drop). This task adds: (a) emitting `TrackingActionCompleted` domain events after each action, and (b) a new internal endpoint for discovery.

**Deliverables:**

- [ ] **Step 1: Define the `TrackingActionCompleted` domain event contract**

Define the event structure exactly as specified:

```go
// domain event fields
type TrackingActionCompleted struct {
    UserId       string            // Cognito user ID
    ActivityType string            // STARTED_WATCHING | RATED | REVIEWED | COMPLETED | DROPPED | ADDED_TO_LIST
    MediaId      string            // TMDB ID
    MediaType    string            // SERIES | MOVIE
    MediaTitle   string            // Denormalized title for feed display
    Metadata     map[string]string // Contextual data (rating, status, review preview)
    Timestamp    time.Time
}
```

- [ ] **Step 2: Map existing tracking actions to ActivityType values**

Ensure every tracking operation in `UserTrackingContext` maps to exactly one `ActivityType`:

| Tracking Operation | ActivityType |
|--------------------|--------------|
| `startTracking` (new tracking record) | `ADDED_TO_LIST` |
| `updateProgress` to WATCHING | `STARTED_WATCHING` |
| `rateSeries` / `rateMovie` | `RATED` |
| `updateReview` | `REVIEWED` |
| `updateStatus` to COMPLETED | `COMPLETED` |
| `updateStatus` to DROPPED | `DROPPED` |

- [ ] **Step 3: Define the event publisher interface (port)**

```go
// internal/ports/event_publisher.go
type EventPublisher interface {
    PublishTrackingActionCompleted(ctx context.Context, event TrackingActionCompleted) error
}
```

- [ ] **Step 4: Implement the event publisher**

Choose transport (EventBridge recommended per existing architecture direction; direct HTTP to SocialContext also valid for MVP simplicity). The publisher must be non-blocking — if publishing fails, the tracking operation must still succeed (log the error, don't roll back).

```go
// internal/infrastructure/eventbridge_publisher.go
// (or internal/infrastructure/http_event_publisher.go)
```

- [ ] **Step 5: Wire event publishing into existing use cases**

After each tracking action persists in DynamoDB, publish the event. Example for rating:

```go
// In rate_series.go (or rate_media.go), after successful persistence:
event := TrackingActionCompleted{
    UserId:       userId,
    ActivityType: "RATED",
    MediaId:      mediaId,
    MediaType:    "SERIES",
    MediaTitle:   mediaTitle,
    Metadata:     map[string]string{"rating": strconv.Itoa(rating)},
    Timestamp:    time.Now(),
}
if err := publisher.PublishTrackingActionCompleted(ctx, event); err != nil {
    logger.Warn("failed to publish tracking event", "error", err)
    // Do NOT return error — tracking is already persisted
}
```

- [ ] **Step 6: Implement GET /internal/summary/{userId} endpoint**

This endpoint returns a summary of the user's tracking data for discovery:

```go
// Response shape
type TrackingSummary struct {
    Genres      map[string]int  // genre name → count (e.g., {"Sci-Fi": 12})
    RecentMedia []MediaSummary  // user's tracked media with ratings
    TotalTracked int
}

type MediaSummary struct {
    MediaId   string
    MediaType string // SERIES | MOVIE
    Title     string
    Rating    *int   // nullable, 1-10
}
```

This endpoint queries the existing `user-tracking` DynamoDB table (GSI by user) and, for genre data, may need to query SeriesCatalog/MovieCatalog or cache genre data locally on the tracking record. If genre data isn't readily available on tracking records, add it as a denormalized field during tracking creation/update.

- [ ] **Step 7: Add IAM policy for the summary endpoint**

Ensure the `GET /internal/summary/{userId}` endpoint is accessible via Lambda Function URL with IAM auth (same pattern as existing inter-context calls).

- [ ] **Step 8: Add tests**

- Unit test: event publisher is called with correct fields after each tracking operation
- Unit test: tracking operation succeeds even when event publishing fails
- Unit test: summary endpoint returns correct genre counts and media list
- Integration test: end-to-end summary endpoint via HTTP

- [ ] **Step 9: Commit**

```bash
git add user-tracking/internal/...
git commit -m "feat(user-tracking): add domain event publishing and summary endpoint"
```

---

### Task 2: SocialContext — New Bounded Context (Core)

**Context:** This is the main deliverable. A new Go module at `/social-context/` following the same Clean Architecture pattern as existing contexts.

**Deliverables:**

- [ ] **Step 1: Scaffold module structure**

```bash
mkdir -p social-context/cmd/api
mkdir -p social-context/internal/{domain,application,infrastructure,ports}
mkdir -p social-context/terraform
```

Initialize `go.mod`, set up the Lambda entrypoint (`cmd/api/main.go`), and configure the Go workspace (`go.work`).

- [ ] **Step 2: Define domain entities and value objects**

```go
// internal/domain/user_profile.go
type UserProfile struct {
    UserId      UserId
    DisplayName string
    AvatarUrl   string
    IsPublic    bool
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

// internal/domain/value_objects.go
type UserId struct { Id string }
type EventId struct { Id string }
type ActivityType string
type MediaType string
type TrackingStatus string

// ActivityType constants
const (
    StartedWatching ActivityType = "STARTED_WATCHING"
    Rated           ActivityType = "RATED"
    Reviewed        ActivityType = "REVIEWED"
    Completed       ActivityType = "COMPLETED"
    Dropped         ActivityType = "DROPPED"
    AddedToList     ActivityType = "ADDED_TO_LIST"
)
```

```go
// internal/domain/follow_relationship.go
type FollowRelationship struct {
    FollowerId UserId
    FollowedId UserId
    FollowedAt time.Time
}
```

```go
// internal/domain/activity_event.go
type ActivityEvent struct {
    EventId      EventId
    UserId       UserId
    ActivityType ActivityType
    MediaId      string
    MediaType    MediaType
    MediaTitle   string
    Metadata     map[string]string
    CreatedAt    time.Time
}
```

- [ ] **Step 3: Define repository interface (port)**

```go
// internal/ports/repository.go
type SocialRepository interface {
    // Profile
    GetProfile(ctx context.Context, userId UserId) (*UserProfile, error)
    CreateProfile(ctx context.Context, profile *UserProfile) error
    UpdateProfile(ctx context.Context, profile *UserProfile) error

    // Follow
    Follow(ctx context.Context, followerId, followedId UserId) error
    Unfollow(ctx context.Context, followerId, followedId UserId) error
    IsFollowing(ctx context.Context, followerId, followedId UserId) (bool, error)
    GetFollowing(ctx context.Context, userId UserId, page, limit int) ([]FollowRelationship, int, error)
    GetFollowers(ctx context.Context, userId UserId, page, limit int) ([]FollowRelationship, int, error)

    // Activities
    SaveActivity(ctx context.Context, event *ActivityEvent) error
    GetUserActivities(ctx context.Context, userId UserId, limit int) ([]ActivityEvent, error)

    // Discovery
    GetPublicProfiles(ctx context.Context) ([]UserProfile, error)
}
```

- [ ] **Step 4: Define UserTracking client interface (port)**

```go
// internal/ports/user_tracking.go
type UserTrackingClient interface {
    GetTrackingSummary(ctx context.Context, userId string) (*TrackingSummary, error)
}
```

- [ ] **Step 5: Implement DynamoDB repository**

Single-table design for `social-graph`. The exact PK/SK schema is decided and implemented here — follow the access patterns listed below.

**Access patterns to support:**
- Get profile by userId
- Create/update profile
- List following by followerId (paginated)
- List followers by followedId (paginated)
- Check if following (single item get)
- Follow (write two items: follower index + following index)
- Unfollow (delete two items)
- Save activity event
- Get recent activities by userId (sorted by timestamp, descending)
- List all public profiles

- [ ] **Step 6: Implement UserTracking HTTP client**

```go
// internal/infrastructure/user_tracking_client.go
// Calls GET https://<user-tracking-lambda-url>/internal/summary/{userId}
// Signs request with IAM credentials (same pattern as existing Lambda clients)
```

- [ ] **Step 7: Implement use cases — Profile**

```go
// internal/application/create_profile.go
func CreateProfile(ctx, repo, userId, displayName, avatarUrl, isPublic) (*UserProfile, error)

// internal/application/update_profile.go
func UpdateProfile(ctx, repo, userId, displayName, avatarUrl, isPublic) (*UserProfile, error)

// internal/application/get_profile.go
func GetProfile(ctx, repo, userId, requesterUserId) (*UserProfile, error)
// Returns nil if profile is private and requesterUserId != userId
```

- [ ] **Step 8: Implement use cases — Social Graph**

```go
// internal/application/follow_user.go
func FollowUser(ctx, repo, followerId, followedId) error
// Validates: cannot follow self, cannot follow private profile, cannot duplicate

// internal/application/unfollow_user.go
func UnfollowUser(ctx, repo, followerId, followedId) error

// internal/application/get_following.go
func GetFollowing(ctx, repo, userId, page, limit) ([]FollowResult, int, error)

// internal/application/get_followers.go
func GetFollowers(ctx, repo, userId, page, limit) ([]FollowResult, int, error)
```

- [ ] **Step 9: Implement use cases — Activity Feed**

```go
// internal/application/get_feed.go
func GetFeed(ctx, repo, userId, page, limit) ([]FeedItem, error)
// 1. Get list of followed userIds
// 2. For each, query recent activities
// 3. Merge + sort by CreatedAt descending
// 4. Apply pagination
// MVP: fan-out on read (query each followed user)

// internal/application/ingest_activity.go
func IngestActivity(ctx, repo, event ActivityEvent) error
// Called by the event consumer — persists activity to repo
```

- [ ] **Step 10: Implement use cases — Discovery**

```go
// internal/application/discover_users.go
func DiscoverUsers(ctx, repo, trackingClient, userId, limit) ([]DiscoveryResult, error)
// 1. Get requesting user's tracking summary (via trackingClient)
// 2. Get all public profiles (via repo)
// 3. For each candidate, get tracking summary (via trackingClient)
// 4. Compute affinity score:
//    overlapScore = |common media| / |requesting user's media|
//    genreScore = cosine similarity of genre vectors
//    score = (overlapScore * 0.6) + (genreScore * 0.4)
// 5. Remove already-followed users
// 6. Sort by score descending, return top N
```

- [ ] **Step 11: Implement Lambda handlers (API)**

The main Lambda (`social-context-api`) routes HTTP requests to use cases. Each handler:

```go
// GET /profiles/{userId}
// PUT /profiles/{userId}
// POST /follow
// DELETE /follow/{followedId}?followerId={followerId}
// GET /following/{userId}?page=1&limit=20
// GET /followers/{userId}?page=1&limit=20
// GET /feed/{userId}?page=1&limit=20
// GET /discover/{userId}?limit=10
```

All handlers extract `userId` from the request context (injected by IAM auth or passed by BFF), call the corresponding use case, and return JSON responses matching the API contracts defined in the spec.

- [ ] **Step 12: Implement event consumer**

The domain event `TrackingActionCompleted` from UserTrackingContext must be consumed. This can be:
- **Option A (EventBridge):** A dedicated Lambda triggered by EventBridge rule → calls `IngestActivity`
- **Option B (Same Lambda):** API Lambda also has an EventBridge trigger → routes to `IngestActivity`
- **Option C (HTTP endpoint):** UserTracking directly POSTs to an internal endpoint

Choose based on existing infrastructure patterns. The consumer's sole job: receive event → call `IngestActivity` → persist. No response needed.

- [ ] **Step 13: Terraform infrastructure**

```hcl
# social-context/terraform/main.tf
# - Lambda function (social-context-api)
# - Lambda Function URL with IAM auth
# - DynamoDB table (social-graph) with GSIs
# - IAM role + policies for DynamoDB access
# - IAM policy for UserTracking to allow invoke (if using EventBridge: EventBridge rule + target)
# - Environment variables: TABLE_NAME, USER_TRACKING_URL, LOG_LEVEL
```

- [ ] **Step 14: Add tests**

- Unit tests for each use case (mocking repository + tracking client)
- Unit test: `UpdateProfile` validation (displayName required, etc.)
- Unit test: `FollowUser` edge cases (self-follow, duplicate, private target)
- Unit test: `GetFeed` merges and sorts correctly from multiple followed users
- Unit test: `DiscoverUsers` algorithm computes correct scores
- Unit test: `IngestActivity` persists correctly
- Integration test: end-to-end Lambda invocation for each endpoint

- [ ] **Step 15: Commit**

```bash
git add social-context/
git commit -m "feat(social): add SocialContext bounded context

- User profiles (public/private)
- Follow/unfollow with paginated follower/following lists
- Activity feed (chronological aggregation)
- User discovery (hybrid affinity: media overlap + genre similarity)
- Event consumer for TrackingActionCompleted domain events"
```

---

### Task 3: ExperienceBFFContext — New Social Routes

**Context:** The BFF already aggregates data from domain contexts and exposes public REST endpoints. This task adds social routes.

**Deliverables:**

- [ ] **Step 1: Define SocialContext HTTP client**

```go
// internal/clients/social_context_client.go
// Uses the same Lambda Function URL + IAM signing pattern as existing clients
// Methods:
//   GetProfile(userId) → ProfileResponse
//   UpdateProfile(userId, body) → ProfileResponse
//   Follow(followerId, followedId) → error
//   Unfollow(followerId, followedId) → error
//   GetFollowing(userId, page, limit) → PaginatedUsersResponse
//   GetFollowers(userId, page, limit) → PaginatedUsersResponse
//   GetFeed(userId, page, limit) → PaginatedFeedResponse
//   GetDiscover(userId, limit) → DiscoverResponse
```

- [ ] **Step 2: Define DTOs for social responses**

```go
// internal/dto/social.go
type ProfileResponse struct { ... }
type FeedItemResponse struct { ... }
type DiscoverItemResponse struct { ... }
type PaginatedUsersResponse struct { ... }
type PaginatedFeedResponse struct { ... }
```

Match the exact JSON shapes defined in the spec (Section 4).

- [ ] **Step 3: Implement profile handlers**

```go
// GET /users/{userId}/profile
func (h *ProfileHandler) GetProfile(w, r)
// Extracts target userId from path, requester userId from JWT
// Calls socialClient.GetProfile(targetUserId)
// Returns 404 if profile is private (or not found)

// PUT /users/me/profile
func (h *ProfileHandler) UpdateProfile(w, r)
// Extracts userId from JWT
// Validates body (displayName required, max length, etc.)
// Calls socialClient.UpdateProfile(userId, body)
```

- [ ] **Step 4: Implement social graph handlers**

```go
// POST /users/{userId}/follow
// DELETE /users/{userId}/follow
// GET /users/me/following
// GET /users/me/followers
// GET /users/{userId}/followers
// GET /users/{userId}/following
```

Each handler extracts userId from JWT, validates authorization (see spec Section 7.2), calls the SocialContext client, and maps the response.

- [ ] **Step 5: Implement feed handler**

```go
// GET /feed?page=1&limit=20
func (h *FeedHandler) GetFeed(w, r)
// Extracts userId from JWT
// Calls socialClient.GetFeed(userId, page, limit)
// Returns paginated feed items
```

- [ ] **Step 6: Implement discovery handler**

```go
// GET /discover?limit=10
func (h *DiscoverHandler) GetDiscover(w, r)
// Extracts userId from JWT
// Calls socialClient.GetDiscover(userId, limit)
// Returns ranked list of similar users
```

- [ ] **Step 7: Register new routes**

Add routes to the existing BFF router:
```go
// Perfil
r.GET("/users/:userId/profile", profileHandler.GetProfile)
r.PUT("/users/me/profile", profileHandler.UpdateProfile)

// Social Graph
r.POST("/users/:userId/follow", socialHandler.Follow)
r.DELETE("/users/:userId/follow", socialHandler.Unfollow)
r.GET("/users/me/following", socialHandler.GetFollowing)
r.GET("/users/me/followers", socialHandler.GetFollowers)
r.GET("/users/:userId/followers", socialHandler.GetUserFollowers)
r.GET("/users/:userId/following", socialHandler.GetUserFollowing)

// Feed + Discovery
r.GET("/feed", feedHandler.GetFeed)
r.GET("/discover", discoverHandler.GetDiscover)
```

All routes go through the existing auth middleware (Cognito JWT validation).

- [ ] **Step 8: Add SocialContext URL to BFF environment**

```bash
SOCIAL_CONTEXT_URL=https://<lambda-id>.lambda-url.<region>.on.aws
```

- [ ] **Step 9: Update Terraform (BFF IAM policy)**

Add IAM policy allowing BFF Lambda to invoke SocialContext Lambda Function URL:
```hcl
resource "aws_iam_role_policy" "bff_invoke_social" {
  name = "invoke-social-context"
  role = aws_iam_role.bff_lambda.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "lambda:InvokeFunctionUrl"
      Resource = aws_lambda_function.social_context_api.arn
      Condition = {
        StringEquals = {
          "lambda:FunctionUrlAuthType" = "AWS_IAM"
        }
      }
    }]
  })
}
```

- [ ] **Step 10: Add tests**

- Unit test: BFF handlers call correct SocialContext client methods
- Unit test: authorization rules enforced (self-only for PUT profile, DELETE follow)
- Unit test: 404 for private profiles
- Integration test: end-to-end BFF → SocialContext for each route

- [ ] **Step 11: Commit**

```bash
git add experience-bff/internal/...
git commit -m "feat(bff): add social routes (profile, follow, feed, discovery)

- New SocialContext HTTP client
- Profile endpoints (GET public, PUT own)
- Social graph endpoints (follow/unfollow, followers/following lists)
- Feed endpoint (chronological activities from followed users)
- Discovery endpoint (affinity-ranked user suggestions)"
```

---

### Task 4: Integration Verification

**Context:** Ensure all three contexts work together end-to-end before declaring the feature done.

**Deliverables:**

- [ ] **Step 1: Deploy all contexts to dev environment**

```bash
make deploy-all
# or per-context:
cd user-tracking && make deploy
cd social-context && make deploy
cd experience-bff && make deploy
```

- [ ] **Step 2: Verify full flow manually**

Execute these scenarios via the BFF public API:

1. **Create profile:** `PUT /users/me/profile { "displayName": "Alice", "isPublic": true }` → 200
2. **Verify discovery:** `GET /discover?limit=10` → returns public profiles
3. **Follow someone:** `POST /users/bob-id/follow` → 201
4. **Verify following list:** `GET /users/me/following` → includes Bob
5. **Trigger activity:** Add/rate a series via existing tracking endpoints
6. **Verify feed:** `GET /feed` → includes the activity event
7. **Unfollow:** `DELETE /users/bob-id/follow` → 204
8. **Private profile:** `PUT /users/me/profile { "isPublic": false }` → profile no longer appears in discovery

- [ ] **Step 3: Verify error cases**

- Get private profile → 404
- Follow private profile → 404/403
- Follow self → 409
- Double follow → 409
- Unauthenticated request → 401

- [ ] **Step 4: Commit**

```bash
git add docs/superpowers/plans/2026-05-16-social-features-plan.md
git commit -m "docs: add social features implementation plan"
```
