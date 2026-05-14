# Movie Tracking — CML Domain Model Update Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update the CML domain model to reflect the movie tracking design — adding MovieCatalog subdomain/context, evolving UserTracking to be media-aware, and regenerating the ContextMap PNG.

**Architecture:** CML syntax changes only. No code changes in this repository. The ContextMap PNG is regenerated from the updated `.cml` file using the Context Mapper CLI.

**Tech Stack:** Context Mapper DSL (CML), Context Mapper CLI 6.12.0

---

## File Structure Overview

```
/docs/domain-modeling/
├── media-tracker.cml              ← MODIFIED (all CML changes)
└── media-tracker_ContextMap.png   ← REGENERATED (from updated CML)
```

All changes are in a single CML file. The plan breaks them into logical tasks for clean commits.

---

### Task 1: Add MovieCatalog Subdomain

**Files:**
- Modify: `docs/domain-modeling/media-tracker.cml`

- [ ] **Step 1: Update header comment to reflect movies scope**

Change line 2-3:
```
   Media Tracker - Domain Model (MVP Séries)
   ============================================

   Este arquivo contém a modelagem DDD completa do sistema
   Media Tracker, focado no MVP de rastreamento de séries.
```
To:
```
   Media Tracker - Domain Model (Séries + Filmes)
   ==============================================

   Este arquivo contém a modelagem DDD completa do sistema
   Media Tracker, cobrindo rastreamento de séries e filmes.
```

- [ ] **Step 2: Update subdomain list in header comment**

Change line 8-12:
```
   Subdomínios:
   1. SeriesCatalog (SUPPORTING_DOMAIN) - Dados de séries
   2. UserTracking (CORE_DOMAIN) - Rastreamento pessoal
   3. Recommendation (CORE_DOMAIN) - Sistema de recomendação
   4. UserExperience (CORE_DOMAIN) - Experiência do usuário
```
To:
```
   Subdomínios:
   1. SeriesCatalog (SUPPORTING_DOMAIN) - Dados de séries
   2. MovieCatalog (SUPPORTING_DOMAIN) - Dados de filmes
   3. UserTracking (CORE_DOMAIN) - Rastreamento pessoal
   4. Recommendation (CORE_DOMAIN) - Sistema de recomendação
   5. UserExperience (CORE_DOMAIN) - Experiência do usuário
```

- [ ] **Step 3: Update bounded contexts list in header comment**

Change line 14-19:
```
   Bounded Contexts:
   - SeriesCatalogContext (Backend)
   - UserTrackingContext (Backend)
   - RecommendationContext (Backend)
   - ExperienceContext (Frontend)
   - ExperienceBFFContext (Backend for Frontend)
```
To:
```
   Bounded Contexts:
   - SeriesCatalogContext (Backend)
   - MovieCatalogContext (Backend)
   - UserTrackingContext (Backend)
   - RecommendationContext (Backend)
   - ExperienceContext (Frontend)
   - ExperienceBFFContext (Backend for Frontend)
```

- [ ] **Step 4: Add MovieCatalog Subdomain**

After the `SeriesCatalog` Subdomain block (after line 39 `}`), insert:

```cml
  Subdomain MovieCatalog {
    type = SUPPORTING_DOMAIN
    domainVisionStatement = "Gerencia informações sobre filmes. Dados estruturais do conteúdo cinematográfico."

    Entity Movie {
      String title
      String overview
      String genre
      int runtime
      Date releaseDate
    }
  }
```

- [ ] **Step 5: Commit**

```bash
git add docs/domain-modeling/media-tracker.cml
git commit -m "cml: add MovieCatalog subdomain"
```

---

### Task 2: Evolve UserTracking Subdomain to Media-Aware

**Files:**
- Modify: `docs/domain-modeling/media-tracker.cml`

- [ ] **Step 1: Rename Entity and fields in UserTracking Subdomain**

Change the `UserTracking` Subdomain (currently lines 41-49):
```cml
  Subdomain UserTracking {
    type = CORE_DOMAIN
    domainVisionStatement = "Permite ao usuário rastrear seu progresso, avaliar conteúdo e manter listas pessoais. Diferencial competitivo na organização pessoal."

    Entity TrackedSeries {
      String seriesId
      String status
      int userRating
    }
  }
```
To:
```cml
  Subdomain UserTracking {
    type = CORE_DOMAIN
    domainVisionStatement = "Permite ao usuário rastrear seu progresso, avaliar conteúdo e manter listas pessoais. Diferencial competitivo na organização pessoal."

    Entity TrackedMedia {
      String mediaId
      String mediaType
      String status
      int rating
    }
  }
```

- [ ] **Step 2: Commit**

```bash
git add docs/domain-modeling/media-tracker.cml
git commit -m "cml: evolve UserTracking subdomain — TrackedSeries → TrackedMedia"
```

---

### Task 3: Add MovieCatalogContext Bounded Context

**Files:**
- Modify: `docs/domain-modeling/media-tracker.cml`

- [ ] **Step 1: Insert MovieCatalogContext after SeriesCatalogContext**

After the `SeriesCatalogContext` block (after line 71 `}`), insert:

```cml
BoundedContext MovieCatalogContext implements MovieCatalog {
  type = FEATURE
  domainVisionStatement = "Contexto responsável por gerenciar dados de filmes. Fonte da verdade sobre conteúdo cinematográfico disponível."
  implementationTechnology = "Go / AWS Lambda"
  responsibilities = "Movies", "Metadata"
  knowledgeLevel = CONCRETE
}
```

- [ ] **Step 2: Commit**

```bash
git add docs/domain-modeling/media-tracker.cml
git commit -m "cml: add MovieCatalogContext bounded context"
```

---

### Task 4: Evolve UserTrackingContext Aggregate

**Files:**
- Modify: `docs/domain-modeling/media-tracker.cml`

- [ ] **Step 1: Rename Aggregate, Entity, and fields — add MediaType and MediaId VOs**

Replace the entire `UserTrackingContext` bounded context block (currently starting at line 73). The old block is:

```cml
BoundedContext UserTrackingContext implements UserTracking {
  type = FEATURE
  domainVisionStatement = "Contexto responsável por rastrear o progresso pessoal do usuário, status, avaliações e listas."
  implementationTechnology = "Go / AWS Lambda"
  responsibilities = "Tracking", "Ratings", "UserLists", "Progress"
  knowledgeLevel = CONCRETE

  Aggregate UserSeriesTracking {
    responsibilities = "Rastreamento de uma série específica para um usuário"
    likelihoodForChange = OFTEN
    contentVolatility = OFTEN

    Entity TrackedSeries {
      aggregateRoot

      - TrackingId trackingId
      - UserId userId
      - SeriesId seriesId
      TrackingStatus status
      - List<SeasonRating> seasonRatings
      Integer seriesRating // 1-10, nullable
      String seriesReview
      Date addedAt
      Date lastUpdated
    }

    ValueObject TrackingId {
      String id key
    }

    ValueObject UserId {
      String id key
    }

    ValueObject SeriesId {
      String id key
    }

    ValueObject SeasonRating {
      int seasonNumber key
      int rating // 1-10
      Date ratedAt
    }

    enum TrackingStatus {
      aggregateLifecycle
      WANT_TO_WATCH,    // Quero Assistir
      WATCHING,         // Assistindo
      COMPLETED,        // Concluída
      DROPPED           // Dropada
    }
  }
}
```

Replace with:

```cml
BoundedContext UserTrackingContext implements UserTracking {
  type = FEATURE
  domainVisionStatement = "Contexto responsável por rastrear o progresso pessoal do usuário, status, avaliações e listas de qualquer tipo de mídia."
  implementationTechnology = "Go / AWS Lambda"
  responsibilities = "Tracking", "Ratings", "UserLists", "Progress"
  knowledgeLevel = CONCRETE

  Aggregate UserMediaTracking {
    responsibilities = "Rastreamento de uma mídia específica para um usuário"
    likelihoodForChange = OFTEN
    contentVolatility = OFTEN

    Entity TrackedMedia {
      aggregateRoot

      - TrackingId trackingId
      - UserId userId
      - MediaId mediaId
      - MediaType mediaType
      TrackingStatus status
      - List<SeasonRating> seasonRatings
      Integer rating // 1-10, nullable
      String review
      Date addedAt
      Date lastUpdated
    }

    ValueObject TrackingId {
      String id key
    }

    ValueObject UserId {
      String id key
    }

    ValueObject MediaId {
      String id key
    }

    ValueObject MediaType {
      String type key
    }

    ValueObject SeasonRating {
      int seasonNumber key
      int rating // 1-10
      Date ratedAt
    }

    enum MediaTypeEnum {
      SERIES, MOVIE
    }

    enum TrackingStatus {
      aggregateLifecycle
      WANT_TO_WATCH,    // Quero Assistir
      WATCHING,         // Assistindo
      COMPLETED,        // Concluída
      DROPPED           // Dropada
    }
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add docs/domain-modeling/media-tracker.cml
git commit -m "cml: evolve UserTrackingContext — UserSeriesTracking → UserMediaTracking, add MediaType"
```

---

### Task 5: Update RecommendationContext Vision Statement

**Files:**
- Modify: `docs/domain-modeling/media-tracker.cml`

- [ ] **Step 1: Generalize vision statement**

Change the `RecommendationContext` `domainVisionStatement` from:
```
domainVisionStatement = "Contexto responsável por sugerir séries quando o usuário está ocioso, baseado em sua lista de interesse."
```
To:
```
domainVisionStatement = "Contexto responsável por sugerir conteúdo quando o usuário está ocioso, baseado em sua lista de interesse."
```

- [ ] **Step 2: Commit**

```bash
git add docs/domain-modeling/media-tracker.cml
git commit -m "cml: generalize RecommendationContext vision statement"
```

---

### Task 6: Update Context Map

**Files:**
- Modify: `docs/domain-modeling/media-tracker.cml`

- [ ] **Step 1: Add MovieCatalogContext to context map contains and add relationships**

Replace the entire `ContextMap` block (starting at line 155). Old:

```cml
ContextMap MediaTrackerMap {
  type = SYSTEM_LANDSCAPE
  state = TO_BE

  contains SeriesCatalogContext, UserTrackingContext, RecommendationContext, ExperienceContext, ExperienceBFFContext

  // UserTracking consome dados do catálogo para rastrear séries
  UserTrackingContext [D,CF] <- [U,OHS,PL] SeriesCatalogContext

  // Recommendation consome lista "Quero Assistir" do UserTracking
  RecommendationContext [D,CF] <- [U,OHS,PL] UserTrackingContext

  // ExperienceBFF agrega dados de todos os contextos de domínio
  ExperienceBFFContext [D,CF] <- [U,OHS,PL] SeriesCatalogContext

  ExperienceBFFContext [D,CF] <- [U,OHS,PL] UserTrackingContext

  ExperienceBFFContext [D,CF] <- [U,OHS,PL] RecommendationContext

  // Experience (Frontend) consome apenas o BFF
  ExperienceContext [D,C] <- [U,S] ExperienceBFFContext
}
```

Replace with:

```cml
ContextMap MediaTrackerMap {
  type = SYSTEM_LANDSCAPE
  state = TO_BE

  contains SeriesCatalogContext, MovieCatalogContext, UserTrackingContext, RecommendationContext, ExperienceContext, ExperienceBFFContext

  // UserTracking consome dados dos catálogos para validar mídias
  UserTrackingContext [D,CF] <- [U,OHS,PL] SeriesCatalogContext

  UserTrackingContext [D,CF] <- [U,OHS,PL] MovieCatalogContext

  // Recommendation consome lista "Quero Assistir" do UserTracking
  RecommendationContext [D,CF] <- [U,OHS,PL] UserTrackingContext

  // ExperienceBFF agrega dados de todos os contextos de domínio
  ExperienceBFFContext [D,CF] <- [U,OHS,PL] SeriesCatalogContext

  ExperienceBFFContext [D,CF] <- [U,OHS,PL] MovieCatalogContext

  ExperienceBFFContext [D,CF] <- [U,OHS,PL] UserTrackingContext

  ExperienceBFFContext [D,CF] <- [U,OHS,PL] RecommendationContext

  // Experience (Frontend) consome apenas o BFF
  ExperienceContext [D,C] <- [U,S] ExperienceBFFContext
}
```

- [ ] **Step 2: Commit**

```bash
git add docs/domain-modeling/media-tracker.cml
git commit -m "cml: update context map with MovieCatalogContext and new relationships"
```

---

### Task 7: Validate CML Syntax

**Files:**
- None (validation only)

- [ ] **Step 1: Install Context Mapper CLI (if not present)**

```bash
if [ ! -f /tmp/cm-cli/bin/cm ]; then
  curl -L -o /tmp/cm-cli.tar "https://repo1.maven.org/maven2/org/contextmapper/context-mapper-cli/6.12.0/context-mapper-cli-6.12.0.tar"
  mkdir -p /tmp/cm-cli
  tar -xf /tmp/cm-cli.tar -C /tmp/cm-cli
fi
```

- [ ] **Step 2: Run validation**

```bash
/tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm validate -i docs/domain-modeling/media-tracker.cml
```

Expected: `Validation completed successfully. No errors found.`

- [ ] **Step 3: If validation fails, fix errors and re-validate**

Common pitfalls from the skill reference:
- `description` is a reserved word — use `seriesDescription`, `overview` etc. instead
- Relationship properties need `{ }` blocks
- Only valid DDD patterns: `U`, `D`, `OHS`, `PL`, `ACL`, `CF`, `S`, `C`, `SK`, `P`

---

### Task 8: Regenerate ContextMap PNG

**Files:**
- Create/Overwrite: `docs/domain-modeling/media-tracker_ContextMap.png`

- [ ] **Step 1: Generate context map diagram**

```bash
/tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm generate \
  -i docs/domain-modeling/media-tracker.cml \
  -o /tmp/cm-output/ \
  -g context-map
```

- [ ] **Step 2: Copy generated PNG to docs**

```bash
cp /tmp/cm-output/MediaTrackerMap_ContextMap.png docs/domain-modeling/media-tracker_ContextMap.png
```

- [ ] **Step 3: Verify the new PNG exists and has content**

```bash
ls -lh docs/domain-modeling/media-tracker_ContextMap.png
```

Expected: Shows file size > 0 bytes.

- [ ] **Step 4: Commit**

```bash
git add docs/domain-modeling/media-tracker_ContextMap.png
git commit -m "cml: regenerate ContextMap PNG with movie tracking support"
```

---

## Execution Options

**Plan complete and saved to `docs/superpowers/plans/2026-05-13-movie-tracking-cml-plan.md`**

**Two execution options:**

1. **Subagent-Driven (recommended)** — Dispatch fresh subagent per task, review between tasks, fast iteration

2. **Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?**
