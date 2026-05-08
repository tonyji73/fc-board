# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Build
./gradlew build

# Run (defaults to dev profile — AWS RDS + ElastiCache)
./gradlew bootRun

# Run with local profile (local MySQL + Redis)
./gradlew bootRun --args='--spring.profiles.active=local'

# Test (uses H2 + TestContainers Redis)
./gradlew test

# Run a single test class
./gradlew test --tests "com.fastcampus.fcboard.service.PostServiceTest"

# Lint (ktlint)
./gradlew ktlint

# Lint auto-fix
./gradlew ktlintFormat
```

**Profiles:**
- `local` — MySQL at `localhost:3306/board` (root/1234), Redis at `localhost:6379`
- `dev` — AWS RDS + AWS ElastiCache (default active profile)
- `test` — H2 in-memory + TestContainers Redis on port 16379

## Architecture

Spring Boot 3.1.1 / Kotlin REST API for a bulletin board system (posts, comments, likes, tags).

**Layer flow:** `Controller → Service → Repository → Domain entity → DB`

```
src/main/kotlin/com/fastcampus/fcboard/
├── controller/      # REST endpoints; controller/dto/ holds HTTP request/response types
├── service/         # Business logic; service/dto/ holds service-layer types
├── repository/      # JpaRepository + QueryDSL custom implementations
├── domain/          # JPA entities (Post, Comment, Like, Tag, BaseEntity)
├── event/           # Async like handling (LikeEvent, LikeEventHandler)
├── config/          # RedisConfig
├── exception/       # PostException, CommentException hierarchies
└── util/            # RedisUtil (cache-aside helpers)
```

**DTO conversion:** Extension functions (`.toDto()`, `.toEntity()`, `.toResponse()`) in each `service/dto/` file bridge the layers — always use these rather than constructing across layers by hand.

**QueryDSL:** Custom repository interfaces (`CustomPostRepository`, `CustomTagRepository`) with QueryDSL implementations handle dynamic filtering (title search, creator filter, tag search) and pagination. Q-classes are generated at build time.

**Like system (async):** Likes are created via `ApplicationEventPublisher` → `LikeEvent` → `@Async @TransactionalEventListener LikeEventHandler`. The handler delays 3 seconds then writes to DB. Like counts are cached in Redis with key pattern `like:{postId}` using cache-aside (check Redis → fall back to DB → populate Redis → atomic increment).

**Authorization:** Only the original creator (`createdBy`) may update or delete their posts and comments — enforced in the service layer by comparing the `updatedBy` parameter.

**Tags:** Updating a post replaces the entire tag list (delete-all + re-insert), not a diff.

## Testing

Tests use Kotest `BehaviorSpec` (Given/When/Then) with `@SpringBootTest`. All service-layer tests run against H2 + a TestContainers Redis instance (started automatically).

```kotlin
// Example test structure
class PostServiceTest : BehaviorSpec({
    given("...") {
        `when`("...") {
            then("...") { ... }
        }
    }
})
```

## Key Conventions

- **Line length:** 120 characters (enforced by ktlint and `.editorconfig`)
- **Indentation:** 4 spaces
- **FK constraints:** All JPA foreign keys use `ConstraintMode.NO_CONSTRAINT` — schema integrity is enforced at the application layer
- **Immutability:** Domain entity properties have private setters; mutation goes through dedicated methods (e.g., `updatedBy(String)` on `BaseEntity`)
- **No global exception handler:** Exceptions bubble to Spring Boot defaults; custom exceptions extend `RuntimeException`
