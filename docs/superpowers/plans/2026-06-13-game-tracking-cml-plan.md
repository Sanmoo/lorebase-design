# Game Tracking CML Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update Lorebase's domain model artifacts to add electronic games as a first-class media type with a dedicated `GameCatalogContext`, game-specific tracking status, IGDB-backed catalog metadata, and regenerated context map diagrams.

**Architecture:** This repository is a design/modeling repository. The implementation updates `docs/domain-modeling/media-tracker.cml` and regenerates Context Mapper outputs so the CML source, `.gv`, `.png`, and `.svg` artifacts stay synchronized.

**Tech Stack:** Context Mapper DSL (CML), Context Mapper CLI 6.12.0, Graphviz-derived diagram artifacts.

---

## File Structure Overview

```
/docs/domain-modeling/
├── media-tracker.cml              ← MODIFIED (domain model source)
├── media-tracker_ContextMap.gv    ← REGENERATED (Context Mapper output)
├── media-tracker_ContextMap.png   ← REGENERATED (Context Mapper output)
└── media-tracker_ContextMap.svg   ← REGENERATED if CLI emits it
```

The source of truth is `media-tracker.cml`. Diagram artifacts must be regenerated from the updated CML, not edited by hand.

---

### Task 1: Update CML Header and Add `GameCatalog` Subdomain

**Files:**
- Modify: `docs/domain-modeling/media-tracker.cml`

- [ ] **Step 1: Update the header title and description**

Replace:

```cml
   Media Tracker - Domain Model (Séries + Filmes)
   ==============================================

   Este arquivo contém a modelagem DDD completa do sistema
   Media Tracker, cobrindo rastreamento de séries e filmes.
```

With:

```cml
   Media Tracker - Domain Model (Séries + Filmes + Jogos)
   ======================================================

   Este arquivo contém a modelagem DDD completa do sistema
   Media Tracker, cobrindo rastreamento de séries, filmes e jogos eletrônicos.
```

- [ ] **Step 2: Update the subdomain list in the header**

Replace:

```cml
   Subdomínios:
   1. SeriesCatalog (SUPPORTING_DOMAIN) - Dados de séries
   2. MovieCatalog (SUPPORTING_DOMAIN) - Dados de filmes
   3. UserTracking (CORE_DOMAIN) - Rastreamento pessoal
   4. Recommendation (CORE_DOMAIN) - Sistema de recomendação
   5. UserExperience (CORE_DOMAIN) - Experiência do usuário
   6. Social (CORE_DOMAIN) - Rede social e descoberta de usuários
```

With:

```cml
   Subdomínios:
   1. SeriesCatalog (SUPPORTING_DOMAIN) - Dados de séries
   2. MovieCatalog (SUPPORTING_DOMAIN) - Dados de filmes
   3. GameCatalog (SUPPORTING_DOMAIN) - Dados de jogos eletrônicos
   4. UserTracking (CORE_DOMAIN) - Rastreamento pessoal
   5. Recommendation (CORE_DOMAIN) - Sistema de recomendação
   6. UserExperience (CORE_DOMAIN) - Experiência do usuário
   7. Social (CORE_DOMAIN) - Rede social e descoberta de usuários
```

- [ ] **Step 3: Update the bounded contexts list in the header**

Replace:

```cml
   Bounded Contexts:
   - SeriesCatalogContext (Backend)
   - MovieCatalogContext (Backend)
   - UserTrackingContext (Backend)
   - RecommendationContext (Backend)
   - SocialContext (Backend) - NOVO
   - ExperienceContext (Frontend)
   - ExperienceBFFContext (Backend for Frontend)
```

With:

```cml
   Bounded Contexts:
   - SeriesCatalogContext (Backend)
   - MovieCatalogContext (Backend)
   - GameCatalogContext (Backend) - NOVO
   - UserTrackingContext (Backend)
   - RecommendationContext (Backend)
   - SocialContext (Backend)
   - ExperienceContext (Frontend)
   - ExperienceBFFContext (Backend for Frontend)
```

- [ ] **Step 4: Update the timestamp comment**

Replace:

```cml
   Atualizado em: 2026-05-16
```

With:

```cml
   Atualizado em: 2026-06-13
```

- [ ] **Step 5: Add `GameCatalog` subdomain after `MovieCatalog`**

After the existing `Subdomain MovieCatalog { ... }` block, insert:

```cml
  Subdomain GameCatalog {
    type = SUPPORTING_DOMAIN
    domainVisionStatement = "Gerencia informações sobre jogos eletrônicos. Dados estruturais de catálogo, plataformas, empresas, franquias e modos de jogo."

    Entity Game {
      String title
      String summary
      String genre
      String coverUrl
      Date releaseDate
      String platforms
      String developer
      String publisher
      String franchise
      String gameModes
      String igdbId
    }
  }
```

Use `summary` instead of `description` because `description` is a reserved/fragile CML term in this project history.

- [ ] **Step 6: Validate syntax after the subdomain edit**

Run:

```bash
/tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm validate -i docs/domain-modeling/media-tracker.cml
```

Expected if the CLI is already installed:

```text
Validation completed successfully. No errors found.
```

If `/tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm` does not exist yet, skip validation for this task and install it in Task 6 before final validation.

- [ ] **Step 7: Commit**

```bash
git add docs/domain-modeling/media-tracker.cml
git commit -m "cml: add GameCatalog subdomain"
```

---

### Task 2: Add `GameCatalogContext` Bounded Context

**Files:**
- Modify: `docs/domain-modeling/media-tracker.cml`

- [ ] **Step 1: Insert the new bounded context after `MovieCatalogContext`**

After the existing `BoundedContext MovieCatalogContext implements MovieCatalog { ... }` block, insert:

```cml
BoundedContext GameCatalogContext implements GameCatalog {
  type = FEATURE
  domainVisionStatement = "Contexto responsável por gerenciar dados de jogos eletrônicos. Fonte da verdade sobre metadados de jogos disponíveis."
  implementationTechnology = "Go / AWS Lambda"
  responsibilities = "Games", "Platforms", "Companies", "Franchises", "GameModes", "IGDBIntegration", "Metadata"
  knowledgeLevel = CONCRETE

  Aggregate GameCatalog {
    responsibilities = "Catálogo de jogos eletrônicos e metadados normalizados a partir da IGDB"
    likelihoodForChange = NORMAL
    contentVolatility = OFTEN

    Entity Game {
      aggregateRoot

      - GameId gameId
      String title
      String summary
      String genre
      String coverUrl
      Date releaseDate
      String platforms
      String developer
      String publisher
      String franchise
      String gameModes
      String igdbId
      Date lastUpdated
    }

    ValueObject GameId {
      String id key
    }

    ValueObject ProviderGameId {
      String id key
    }
  }
}
```

- [ ] **Step 2: Validate syntax if the CLI is available**

Run:

```bash
/tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm validate -i docs/domain-modeling/media-tracker.cml
```

Expected:

```text
Validation completed successfully. No errors found.
```

If the CLI is missing, defer validation to Task 6.

- [ ] **Step 3: Commit**

```bash
git add docs/domain-modeling/media-tracker.cml
git commit -m "cml: add GameCatalogContext bounded context"
```

---

### Task 3: Evolve `UserTrackingContext` for Game Tracking

**Files:**
- Modify: `docs/domain-modeling/media-tracker.cml`

- [ ] **Step 1: Add game-specific fields to `TrackedMedia`**

Inside `Aggregate UserMediaTracking`, find the `Entity TrackedMedia` fields:

```cml
      - MediaType mediaType
      TrackingStatus status
      - List<SeasonRating> seasonRatings
      Integer rating // 1-10, nullable
      String review
      Date addedAt
      Date lastUpdated
```

Replace them with:

```cml
      - MediaType mediaType
      TrackingStatus status
      GameTrackingStatus gameStatus
      - List<SeasonRating> seasonRatings
      Integer rating // 1-10, nullable
      String review
      String selectedPlatform // Jogos: plataforma escolhida pelo usuário
      int hoursPlayed // Jogos: horas jogadas, não negativo
      Date addedAt
      Date lastUpdated
```

Rationale: CML does not express conditional fields cleanly. The model records that `selectedPlatform`, `hoursPlayed`, and `gameStatus` apply to game records, while implementation validates applicability by `mediaType`.

- [ ] **Step 2: Extend `MediaTypeEnum`**

Replace:

```cml
    enum MediaTypeEnum {
      SERIES, MOVIE
    }
```

With:

```cml
    enum MediaTypeEnum {
      SERIES, MOVIE, GAME
    }
```

- [ ] **Step 3: Add `GameTrackingStatus` enum after `TrackingStatus`**

After the existing `enum TrackingStatus { ... }` block, insert:

```cml

    enum GameTrackingStatus {
      aggregateLifecycle
      WANT_TO_PLAY,     // Quero Jogar
      PLAYING,          // Jogando
      COMPLETED,        // Concluído/Zerado
      DROPPED           // Dropado
    }
```

- [ ] **Step 4: Update the `UserTrackingContext` vision statement**

Replace:

```cml
domainVisionStatement = "Contexto responsável por rastrear o progresso pessoal do usuário, status, avaliações e listas de qualquer tipo de mídia."
```

With:

```cml
domainVisionStatement = "Contexto responsável por rastrear o progresso pessoal do usuário, status, avaliações, reviews e preferências específicas de cada tipo de mídia."
```

- [ ] **Step 5: Validate syntax if the CLI is available**

Run:

```bash
/tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm validate -i docs/domain-modeling/media-tracker.cml
```

Expected:

```text
Validation completed successfully. No errors found.
```

If the CLI is missing, defer validation to Task 6.

- [ ] **Step 6: Commit**

```bash
git add docs/domain-modeling/media-tracker.cml
git commit -m "cml: add game tracking fields and statuses"
```

---

### Task 4: Update Social and Recommendation Semantics for Games

**Files:**
- Modify: `docs/domain-modeling/media-tracker.cml`

- [ ] **Step 1: Update the `RecommendationContext` vision statement**

Replace:

```cml
domainVisionStatement = "Contexto responsável por sugerir conteúdo quando o usuário está ocioso, baseado em sua lista de interesse."
```

With:

```cml
domainVisionStatement = "Contexto responsável por sugerir conteúdo quando o usuário está ocioso, baseado em sua lista de interesse, incluindo séries, filmes e jogos."
```

- [ ] **Step 2: Extend `ActivityType` with game-specific activity**

Inside `BoundedContext SocialContext`, find:

```cml
    enum ActivityType {
      STARTED_WATCHING,  // Começou a assistir
      RATED,             // Avaliou com nota
      REVIEWED,          // Escreveu/revisou review
      COMPLETED,         // Concluiu
      DROPPED,           // Dropou
      ADDED_TO_LIST      // Adicionou à lista
    }
```

Replace it with:

```cml
    enum ActivityType {
      STARTED_WATCHING,  // Começou a assistir
      STARTED_PLAYING,   // Começou a jogar
      RATED,             // Avaliou com nota
      REVIEWED,          // Escreveu/revisou review
      COMPLETED,         // Concluiu
      DROPPED,           // Dropou
      ADDED_TO_LIST      // Adicionou à lista
    }
```

- [ ] **Step 3: Validate syntax if the CLI is available**

Run:

```bash
/tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm validate -i docs/domain-modeling/media-tracker.cml
```

Expected:

```text
Validation completed successfully. No errors found.
```

If the CLI is missing, defer validation to Task 6.

- [ ] **Step 4: Commit**

```bash
git add docs/domain-modeling/media-tracker.cml
git commit -m "cml: update recommendation and social semantics for games"
```

---

### Task 5: Update Context Map Relationships

**Files:**
- Modify: `docs/domain-modeling/media-tracker.cml`

- [ ] **Step 1: Add `GameCatalogContext` to the context map `contains` line**

Replace:

```cml
  contains SeriesCatalogContext, MovieCatalogContext, UserTrackingContext, RecommendationContext, SocialContext, ExperienceContext, ExperienceBFFContext
```

With:

```cml
  contains SeriesCatalogContext, MovieCatalogContext, GameCatalogContext, UserTrackingContext, RecommendationContext, SocialContext, ExperienceContext, ExperienceBFFContext
```

- [ ] **Step 2: Add UserTracking validation relationship for games**

After:

```cml
  UserTrackingContext [D,CF] <- [U,OHS,PL] MovieCatalogContext
```

Insert:

```cml

  UserTrackingContext [D,CF] <- [U,OHS,PL] GameCatalogContext
```

- [ ] **Step 3: Add BFF aggregation relationship for games**

After:

```cml
  ExperienceBFFContext [D,CF] <- [U,OHS,PL] MovieCatalogContext
```

Insert:

```cml

  ExperienceBFFContext [D,CF] <- [U,OHS,PL] GameCatalogContext
```

- [ ] **Step 4: Update the recommendation relationship comment**

Replace:

```cml
  // Recommendation consome lista "Quero Assistir" do UserTracking
```

With:

```cml
  // Recommendation consome listas de interesse do UserTracking, incluindo WANT_TO_WATCH e WANT_TO_PLAY
```

- [ ] **Step 5: Validate syntax if the CLI is available**

Run:

```bash
/tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm validate -i docs/domain-modeling/media-tracker.cml
```

Expected:

```text
Validation completed successfully. No errors found.
```

If the CLI is missing, defer validation to Task 6.

- [ ] **Step 6: Commit**

```bash
git add docs/domain-modeling/media-tracker.cml
git commit -m "cml: connect GameCatalogContext in context map"
```

---

### Task 6: Install Context Mapper CLI and Validate Final CML

**Files:**
- None, unless validation reveals CML syntax fixes are needed.

- [ ] **Step 1: Install Context Mapper CLI if missing**

Run:

```bash
if [ ! -x /tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm ]; then
  rm -rf /tmp/cm-cli /tmp/cm-cli.tar
  curl -L -o /tmp/cm-cli.tar "https://repo1.maven.org/maven2/org/contextmapper/context-mapper-cli/6.12.0/context-mapper-cli-6.12.0.tar"
  mkdir -p /tmp/cm-cli
  tar -xf /tmp/cm-cli.tar -C /tmp/cm-cli
fi
```

Expected:

```text
# Command exits 0 and /tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm exists.
```

- [ ] **Step 2: Validate the CML model**

Run:

```bash
/tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm validate -i docs/domain-modeling/media-tracker.cml
```

Expected:

```text
Validation completed successfully. No errors found.
```

- [ ] **Step 3: Fix specific validation errors if they occur**

Use these exact fixes for common failures:

| Validation message pattern | Fix |
|---|---|
| `mismatched input 'description'` | Rename field to `summary`, `overview`, or `seriesDescription`. |
| `no viable alternative at input '['` | Ensure context map relationship syntax is `Downstream [D,CF] <- [U,OHS,PL] Upstream`. |
| `Extraneous input` near a relationship property | Wrap relationship properties in `{ implementationTechnology = "..." }`. |
| Unknown pattern abbreviation | Use only valid abbreviations already present in this file: `U`, `D`, `OHS`, `PL`, `CF`, `S`, `C`. |

After any fix, rerun:

```bash
/tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm validate -i docs/domain-modeling/media-tracker.cml
```

Expected:

```text
Validation completed successfully. No errors found.
```

- [ ] **Step 4: Commit validation fixes if any were needed**

If `git status --short` shows changes, run:

```bash
git add docs/domain-modeling/media-tracker.cml
git commit -m "cml: fix game tracking model validation"
```

If there are no changes, do not create an empty commit.

---

### Task 7: Regenerate Context Map Artifacts

**Files:**
- Modify: `docs/domain-modeling/media-tracker_ContextMap.gv`
- Modify: `docs/domain-modeling/media-tracker_ContextMap.png`
- Modify: `docs/domain-modeling/media-tracker_ContextMap.svg` if generated

- [ ] **Step 1: Remove old temporary output and generate new context map**

Run:

```bash
rm -rf /tmp/cm-output
mkdir -p /tmp/cm-output
/tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm generate \
  -i docs/domain-modeling/media-tracker.cml \
  -o /tmp/cm-output/ \
  -g context-map
```

Expected:

```text
# Command exits 0 and writes ContextMap artifacts under /tmp/cm-output.
```

- [ ] **Step 2: Inspect generated files**

Run:

```bash
find /tmp/cm-output -maxdepth 2 -type f | sort
```

Expected output includes a generated ContextMap `.gv` and `.png`. It may also include `.svg`.

- [ ] **Step 3: Copy generated artifacts into `docs/domain-modeling`**

Run:

```bash
GENERATED_GV=$(find /tmp/cm-output -type f -name '*ContextMap*.gv' | head -1)
GENERATED_PNG=$(find /tmp/cm-output -type f -name '*ContextMap*.png' | head -1)
GENERATED_SVG=$(find /tmp/cm-output -type f -name '*ContextMap*.svg' | head -1)

cp "$GENERATED_GV" docs/domain-modeling/media-tracker_ContextMap.gv
cp "$GENERATED_PNG" docs/domain-modeling/media-tracker_ContextMap.png

if [ -n "$GENERATED_SVG" ]; then
  cp "$GENERATED_SVG" docs/domain-modeling/media-tracker_ContextMap.svg
fi
```

- [ ] **Step 4: Verify artifacts exist and are non-empty**

Run:

```bash
ls -lh docs/domain-modeling/media-tracker_ContextMap.gv \
       docs/domain-modeling/media-tracker_ContextMap.png \
       docs/domain-modeling/media-tracker_ContextMap.svg
```

Expected: all listed files exist and each size is greater than 0 bytes.

- [ ] **Step 5: Verify the graph source mentions `GameCatalogContext`**

Run:

```bash
grep -n "GameCatalogContext" docs/domain-modeling/media-tracker_ContextMap.gv
```

Expected: at least one line containing `GameCatalogContext`.

- [ ] **Step 6: Commit regenerated artifacts**

```bash
git add docs/domain-modeling/media-tracker_ContextMap.gv \
        docs/domain-modeling/media-tracker_ContextMap.png \
        docs/domain-modeling/media-tracker_ContextMap.svg
git commit -m "cml: regenerate context map with game catalog"
```

---

### Task 8: Final Verification Against Spec

**Files:**
- Verify: `docs/superpowers/specs/2026-06-13-game-tracking-design.md`
- Verify: `docs/domain-modeling/media-tracker.cml`
- Verify: `docs/domain-modeling/media-tracker_ContextMap.gv`
- Verify: `docs/domain-modeling/media-tracker_ContextMap.png`
- Verify: `docs/domain-modeling/media-tracker_ContextMap.svg`

- [ ] **Step 1: Run final CML validation**

```bash
/tmp/cm-cli/context-mapper-cli-6.12.0/bin/cm validate -i docs/domain-modeling/media-tracker.cml
```

Expected:

```text
Validation completed successfully. No errors found.
```

- [ ] **Step 2: Verify required CML concepts are present**

Run:

```bash
grep -nE "GameCatalog|GameCatalogContext|GAME|WANT_TO_PLAY|PLAYING|selectedPlatform|hoursPlayed|STARTED_PLAYING" docs/domain-modeling/media-tracker.cml
```

Expected: output lines for each required concept.

- [ ] **Step 3: Verify generated graph includes the game context**

Run:

```bash
grep -n "GameCatalogContext" docs/domain-modeling/media-tracker_ContextMap.gv
```

Expected: output contains `GameCatalogContext` node/edge lines.

- [ ] **Step 4: Verify git working tree is clean**

Run:

```bash
git status --short
```

Expected:

```text
# no output
```

If there are uncommitted changes, inspect them with `git diff`, commit intended changes, and leave unrelated changes untouched.

---

## Execution Options

Plan complete and saved to `docs/superpowers/plans/2026-06-13-game-tracking-cml-plan.md`.

Two execution options:

1. **Subagent-Driven (recommended)** - Dispatch a fresh subagent per task, review between tasks, fast iteration

2. **Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?
