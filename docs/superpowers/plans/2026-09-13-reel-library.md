# Reel Library Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first self-hosted Social Memory MVP that accepts Instagram Reel URLs, temporarily downloads and multimodally analyzes them, persists searchable knowledge, deletes temporary media, and supports hybrid natural-language search.

**Architecture:** A .NET 10 ASP.NET Core API owns the domain and HTTP surface; a Redis-backed worker performs isolated staged media analysis; PostgreSQL + pgvector stores relational knowledge, full-text data, and embeddings. Provider interfaces isolate Reel acquisition, transcription, OCR, vision, LLM synthesis, and embeddings so local implementations can be swapped later.

**Tech Stack:** .NET 10, ASP.NET Core, EF Core, PostgreSQL + pgvector, Redis, Docker Compose, FFmpeg, Whisper, OCR provider, multimodal vision provider, LLM provider, embedding provider, frontend TBD after API foundation.

**Spec:** `docs/superpowers/specs/2026-09-13-reel-library-design.md`

## Global Constraints

- Permanent storage must not contain Reel video, extracted audio, or extracted frames.
- Temporary media must live only in an isolated per-job directory and be deleted after successful persistence or by TTL cleanup.
- API, domain, and worker boundaries must not depend on a specific AI vendor.
- Duplicate canonical Reel URLs/external IDs must not create duplicate library records.
- Search must combine PostgreSQL full-text search and pgvector similarity.
- Media processing must run outside the public API process.
- Tests must cover URL canonicalization, state transitions, persistence, cleanup, provider failures, and search ranking behavior.

---

### Task 1: Repository and solution foundation

**Files:**
- Create: `src/SocialMemory.sln`
- Create: `src/SocialMemory.Api/SocialMemory.Api.csproj`
- Create: `src/SocialMemory.Domain/SocialMemory.Domain.csproj`
- Create: `src/SocialMemory.Infrastructure/SocialMemory.Infrastructure.csproj`
- Create: `src/SocialMemory.Worker/SocialMemory.Worker.csproj`
- Create: `tests/SocialMemory.Domain.Tests/SocialMemory.Domain.Tests.csproj`
- Create: `tests/SocialMemory.Infrastructure.Tests/SocialMemory.Infrastructure.Tests.csproj`
- Create: `.editorconfig`
- Create: `.gitignore`
- Create: `docker-compose.yml`

**Interfaces:**
- Produces a buildable solution with four runtime projects and test projects.
- Domain has no infrastructure dependencies.
- API and Worker reference application/domain abstractions; Infrastructure implements persistence/provider adapters.

- [ ] **Step 1: Write the failing architecture smoke test**

Create a test that loads the solution's domain assembly and asserts it has no dependency on ASP.NET Core or Infrastructure namespaces.

- [ ] **Step 2: Run the test**

Run: `dotnet test tests/SocialMemory.Domain.Tests/SocialMemory.Domain.Tests.csproj -v minimal`
Expected: FAIL because the solution/projects do not yet exist.

- [ ] **Step 3: Create the solution and projects**

Target `net10.0`. Add project references so Domain remains dependency-free, Infrastructure references Domain, API references Domain and Infrastructure, and Worker references Domain and Infrastructure.

- [ ] **Step 4: Add Docker Compose services**

Define `api`, `worker`, `postgres`, and `redis`. Keep model/runtime services configurable rather than requiring a GPU for the first boot.

- [ ] **Step 5: Run build and tests**

Run: `dotnet build src/SocialMemory.sln` and `dotnet test src/SocialMemory.sln`
Expected: PASS.

- [ ] **Step 6: Commit**

Commit: `chore: initialize social memory solution`

---

### Task 2: Domain model and Reel URL canonicalization

**Files:**
- Create: `src/SocialMemory.Domain/Reels/Reel.cs`
- Create: `src/SocialMemory.Domain/Reels/ReelStatus.cs`
- Create: `src/SocialMemory.Domain/Reels/ProcessingStage.cs`
- Create: `src/SocialMemory.Domain/Reels/ReelUrl.cs`
- Create: `src/SocialMemory.Domain/Processing/ProcessingJob.cs`
- Create: `tests/SocialMemory.Domain.Tests/Reels/ReelUrlTests.cs`
- Create: `tests/SocialMemory.Domain.Tests/Reels/ReelStatusTests.cs`

**Interfaces:**
- `ReelUrl.Parse(string)` returns a canonical URL or a validation error.
- `Reel.ExternalId` is nullable because acquisition may not always expose it.
- `ProcessingJob.CurrentStage` uses the staged state machine from the spec.

- [ ] **Step 1: Write URL canonicalization tests**

Cover trailing slashes, `http` to `https`, host casing, tracking query parameters, unsupported hosts, malformed URLs, and missing Reel paths.

- [ ] **Step 2: Run URL tests and verify failure**

Run: `dotnet test tests/SocialMemory.Domain.Tests --filter FullyQualifiedName~ReelUrlTests`
Expected: FAIL.

- [ ] **Step 3: Implement `ReelUrl.Parse`**

Accept Instagram Reel URLs only. Normalize host to `www.instagram.com`, normalize the path, remove known tracking parameters, and retain only the stable Reel identifier in the canonical representation.

- [ ] **Step 4: Write state-transition tests**

Assert valid forward transitions and rejection of invalid transitions such as `Completed -> Downloading` and `Cleanup -> VisionAnalysis`.

- [ ] **Step 5: Implement the state machine**

Use an explicit transition method rather than exposing unrestricted status mutation.

- [ ] **Step 6: Run all domain tests**

Expected: PASS.

- [ ] **Step 7: Commit**

Commit: `feat: add reel domain model and url canonicalization`

---

### Task 3: PostgreSQL schema and repository layer

**Files:**
- Create: `src/SocialMemory.Infrastructure/Persistence/SocialMemoryDbContext.cs`
- Create: `src/SocialMemory.Infrastructure/Persistence/Configurations/ReelConfiguration.cs`
- Create: `src/SocialMemory.Infrastructure/Persistence/Configurations/ReelAnalysisConfiguration.cs`
- Create: `src/SocialMemory.Infrastructure/Persistence/Configurations/ProcessingJobConfiguration.cs`
- Create: `src/SocialMemory.Infrastructure/Persistence/Configurations/ReelEmbeddingConfiguration.cs`
- Create: `src/SocialMemory.Infrastructure/Persistence/Configurations/TagConfiguration.cs`
- Create: `src/SocialMemory.Infrastructure/Persistence/Repositories/ReelRepository.cs`
- Create: `src/SocialMemory.Infrastructure/Persistence/Migrations/*`
- Create: `tests/SocialMemory.Infrastructure.Tests/Persistence/ReelRepositoryTests.cs`

**Interfaces:**
- `IReelRepository.AddAsync(Reel, CancellationToken)`
- `IReelRepository.GetAsync(Guid, CancellationToken)`
- `IReelRepository.FindByCanonicalUrlAsync(string, CancellationToken)`
- `IReelRepository.SaveAnalysisAsync(ReelAnalysis, CancellationToken)`
- `IReelRepository.SaveEmbeddingAsync(ReelEmbedding, CancellationToken)`

- [ ] **Step 1: Write repository tests against PostgreSQL**

Use a disposable PostgreSQL test instance/container. Test creation, canonical URL uniqueness, external ID uniqueness when present, and analysis persistence.

- [ ] **Step 2: Run repository tests and verify failure**

Expected: FAIL because the DbContext/repository does not exist.

- [ ] **Step 3: Implement EF Core entities/configuration**

Use JSON/array columns for structured fields where practical and pgvector for embeddings. Add indexes for canonical URL, external ID, status, saved date, creator, language, and content type.

- [ ] **Step 4: Enable pgvector and generate migration**

Create the initial migration and ensure the vector column uses the selected embedding dimension from configuration.

- [ ] **Step 5: Implement repository methods**

Use async EF Core operations and preserve domain invariants.

- [ ] **Step 6: Run repository tests**

Expected: PASS.

- [ ] **Step 7: Commit**

Commit: `feat: add postgres reel persistence`

---

### Task 4: Provider contracts and temporary media workspace

**Files:**
- Create: `src/SocialMemory.Domain/Ports/IReelSource.cs`
- Create: `src/SocialMemory.Domain/Ports/ITranscriber.cs`
- Create: `src/SocialMemory.Domain/Ports/IOcrAnalyzer.cs`
- Create: `src/SocialMemory.Domain/Ports/IVisionAnalyzer.cs`
- Create: `src/SocialMemory.Domain/Ports/IAiSynthesizer.cs`
- Create: `src/SocialMemory.Domain/Ports/IEmbeddingGenerator.cs`
- Create: `src/SocialMemory.Domain/Ports/IMediaWorkspace.cs`
- Create: `src/SocialMemory.Domain/Analysis/ReelAnalysisDocument.cs`
- Create: `src/SocialMemory.Infrastructure/Media/TemporaryMediaWorkspace.cs`
- Create: `tests/SocialMemory.Infrastructure.Tests/Media/TemporaryMediaWorkspaceTests.cs`

**Interfaces:**
- `IReelSource.DownloadAsync(ReelUrl, MediaWorkspace, CancellationToken)` returns a temporary media artifact.
- `ITranscriber.TranscribeAsync(MediaArtifact, CancellationToken)` returns transcript/language metadata.
- `IOcrAnalyzer.AnalyzeAsync(IReadOnlyList<FrameArtifact>, CancellationToken)` returns OCR text.
- `IVisionAnalyzer.AnalyzeAsync(IReadOnlyList<FrameArtifact>, CancellationToken)` returns visual description.
- `IAiSynthesizer.SynthesizeAsync(AnalysisInputs, CancellationToken)` returns `ReelAnalysisDocument`.
- `IEmbeddingGenerator.GenerateAsync(string, CancellationToken)` returns a vector.
- `IMediaWorkspace.CreateAsync(Guid jobId, CancellationToken)` creates an isolated workspace; `CleanupAsync` removes it recursively.

- [ ] **Step 1: Write workspace lifecycle tests**

Assert creation uses a job-specific directory, path traversal cannot escape the configured root, cleanup is idempotent, and cleanup removes nested video/audio/frame files.

- [ ] **Step 2: Implement workspace lifecycle**

Use a configured root and generated job directory name. Do not accept client-controlled paths.

- [ ] **Step 3: Define provider contracts and analysis DTOs**

Keep contracts in Domain so provider implementations can be local or cloud-based.

- [ ] **Step 4: Run tests**

Expected: PASS.

- [ ] **Step 5: Commit**

Commit: `feat: add media workspace and ai provider contracts`

---

### Task 5: Reel acquisition adapter and media extraction

**Files:**
- Create: `src/SocialMemory.Infrastructure/Acquisition/InstagramReelSource.cs`
- Create: `src/SocialMemory.Infrastructure/Media/FfmpegMediaExtractor.cs`
- Create: `src/SocialMemory.Infrastructure/Acquisition/InstagramAcquisitionOptions.cs`
- Create: `tests/SocialMemory.Infrastructure.Tests/Acquisition/InstagramReelSourceTests.cs`
- Create: `tests/SocialMemory.Infrastructure.Tests/Media/FfmpegMediaExtractorTests.cs`

**Interfaces:**
- `InstagramReelSource` implements `IReelSource`.
- `FfmpegMediaExtractor` exposes extraction of audio and representative frames into the temporary workspace.

- [ ] **Step 1: Write acquisition tests with a fake HTTP/acquisition boundary**

Test URL validation, canonical Reel ID handling, bounded download behavior, and failure mapping. Do not make unit tests depend on Instagram availability.

- [ ] **Step 2: Implement acquisition adapter**

Keep the acquisition mechanism isolated behind `IReelSource`. Reject non-Instagram URLs before network activity and enforce configured size/time limits.

- [ ] **Step 3: Write media extraction tests**

Use a small fixture media file and assert audio extraction plus deterministic representative frame generation.

- [ ] **Step 4: Implement FFmpeg adapter**

Invoke FFmpeg with argument arrays/process APIs rather than shell string concatenation. Write all outputs inside the job workspace.

- [ ] **Step 5: Run infrastructure tests**

Expected: PASS. Integration tests that require external Instagram access must be opt-in and excluded from the normal suite.

- [ ] **Step 6: Commit**

Commit: `feat: add instagram acquisition and ffmpeg extraction`

---

### Task 6: Local AI adapters

**Files:**
- Create: `src/SocialMemory.Infrastructure/Ai/WhisperTranscriber.cs`
- Create: `src/SocialMemory.Infrastructure/Ai/OcrAnalyzer.cs`
- Create: `src/SocialMemory.Infrastructure/Ai/OllamaVisionAnalyzer.cs`
- Create: `src/SocialMemory.Infrastructure/Ai/OllamaAiSynthesizer.cs`
- Create: `src/SocialMemory.Infrastructure/Ai/OllamaEmbeddingGenerator.cs`
- Create: `src/SocialMemory.Infrastructure/Ai/AiOptions.cs`
- Create: `tests/SocialMemory.Infrastructure.Tests/Ai/*`

**Interfaces:**
- Implement the five Domain provider contracts without leaking Ollama/Whisper-specific types into Domain.

- [ ] **Step 1: Write adapter contract tests with mocked HTTP/process boundaries**

Assert request construction, structured JSON parsing, malformed provider responses, timeouts, cancellation, and model configuration.

- [ ] **Step 2: Implement Whisper adapter**

Send extracted audio to the configured local Whisper runtime and map output to transcript/language data.

- [ ] **Step 3: Implement OCR adapter**

Process sampled frames and return consolidated OCR text while preserving source frame/timestamp information where available.

- [ ] **Step 4: Implement vision adapter**

Send representative frames with bounded payload sizes and request a factual visual description, avoiding unsupported claims.

- [ ] **Step 5: Implement synthesis adapter**

Use a strict JSON schema/prompt to produce `ReelAnalysisDocument`. Reject invalid schema output rather than persisting malformed analysis.

- [ ] **Step 6: Implement embedding adapter**

Generate vectors from the constructed search document and validate vector dimensions before persistence.

- [ ] **Step 7: Run tests**

Expected: PASS without requiring a running local model service. Add separate opt-in smoke tests for real Ollama/Whisper installations.

- [ ] **Step 8: Commit**

Commit: `feat: add local ai adapters`

---

### Task 7: Staged analysis pipeline and cleanup guarantees

**Files:**
- Create: `src/SocialMemory.Worker/Jobs/ReelAnalysisJob.cs`
- Create: `src/SocialMemory.Worker/Jobs/ReelAnalysisPipeline.cs`
- Create: `src/SocialMemory.Worker/Jobs/JobRetryPolicy.cs`
- Create: `src/SocialMemory.Worker/Services/TemporaryMediaCleanupService.cs`
- Create: `tests/SocialMemory.Worker.Tests/Jobs/ReelAnalysisPipelineTests.cs`
- Create: `tests/SocialMemory.Worker.Tests/Services/TemporaryMediaCleanupServiceTests.cs`

**Interfaces:**
- `ReelAnalysisPipeline.ProcessAsync(Guid reelId, CancellationToken)` performs stage-aware processing.
- Each stage reads/writes only the artifacts needed by subsequent stages.
- Cleanup runs after successful persistence and on abandoned workspaces.

- [ ] **Step 1: Write pipeline tests**

Cover successful end-to-end orchestration with fakes; provider failure at each stage; retry from failed stage; cancellation; persistence failure; and the invariant that media cleanup is attempted after persistence.

- [ ] **Step 2: Implement stage transitions**

Persist the stage before invoking the stage operation and persist completion after it succeeds. Never mark a Reel `Completed` before analysis and embedding are durable.

- [ ] **Step 3: Implement retry policy**

Retry transient failures with bounded exponential backoff. Do not retry permanent URL validation/schema failures indefinitely.

- [ ] **Step 4: Implement cleanup service**

Delete the job workspace after successful persistence. Scan for abandoned workspaces older than the configured TTL and remove them safely.

- [ ] **Step 5: Run worker tests**

Expected: PASS.

- [ ] **Step 6: Commit**

Commit: `feat: add staged reel analysis pipeline`

---

### Task 8: Redis queue and background worker host

**Files:**
- Create: `src/SocialMemory.Infrastructure/Jobs/RedisJobQueue.cs`
- Create: `src/SocialMemory.Domain/Ports/IJobQueue.cs`
- Modify: `src/SocialMemory.Worker/Program.cs`
- Create: `tests/SocialMemory.Infrastructure.Tests/Jobs/RedisJobQueueTests.cs`

**Interfaces:**
- `IJobQueue.EnqueueAsync(Guid reelId, CancellationToken)`.
- `IJobQueue.DequeueAsync(CancellationToken)`.

- [ ] **Step 1: Write queue contract tests**

Test enqueue/dequeue semantics, cancellation, malformed payload handling, and duplicate job protection.

- [ ] **Step 2: Implement Redis queue**

Use a durable Redis list/stream strategy with a visibility/retry mechanism appropriate for the chosen client library. Store only job IDs, not media.

- [ ] **Step 3: Connect Worker hosted service**

Dequeue jobs and invoke `ReelAnalysisPipeline.ProcessAsync` with bounded concurrency.

- [ ] **Step 4: Run tests**

Expected: PASS.

- [ ] **Step 5: Commit**

Commit: `feat: add redis analysis queue`

---

### Task 9: API endpoints and job status

**Files:**
- Create: `src/SocialMemory.Api/Endpoints/ReelEndpoints.cs`
- Create: `src/SocialMemory.Api/Endpoints/SearchEndpoints.cs`
- Create: `src/SocialMemory.Api/Contracts/CreateReelRequest.cs`
- Create: `src/SocialMemory.Api/Contracts/ReelResponse.cs`
- Create: `src/SocialMemory.Api/Contracts/JobResponse.cs`
- Create: `src/SocialMemory.Api/Services/ReelSubmissionService.cs`
- Create: `tests/SocialMemory.Api.Tests/ReelEndpointsTests.cs`

**Interfaces:**
- `POST /api/reels` returns 201 for a new Reel or 200/409 according to the duplicate policy, with status and job ID.
- `GET /api/reels/{id}` returns the current status and persisted analysis when available.
- `GET /api/jobs/{id}` returns current processing stage/error information.
- `DELETE /api/reels/{id}` removes permanent knowledge but never attempts to restore media.

- [ ] **Step 1: Write API integration tests**

Test validation, duplicate URLs, queue enqueueing, retrieval, deletion, and appropriate error responses.

- [ ] **Step 2: Implement submission service**

Canonicalize the URL, check for duplicates, create Reel and ProcessingJob records transactionally, then enqueue the job after commit.

- [ ] **Step 3: Implement endpoints**

Use ProblemDetails for errors and return only DTOs, never EF entities.

- [ ] **Step 4: Add authentication boundary**

Configure a single-user authentication mechanism suitable for self-hosting without embedding secrets in source control.

- [ ] **Step 5: Run API tests**

Expected: PASS.

- [ ] **Step 6: Commit**

Commit: `feat: add reel and job api`

---

### Task 10: Hybrid search

**Files:**
- Create: `src/SocialMemory.Domain/Ports/IReelSearch.cs`
- Create: `src/SocialMemory.Infrastructure/Search/ReelSearchService.cs`
- Create: `src/SocialMemory.Api/Contracts/SearchResponse.cs`
- Modify: `src/SocialMemory.Api/Endpoints/SearchEndpoints.cs`
- Create: `tests/SocialMemory.Infrastructure.Tests/Search/ReelSearchServiceTests.cs`

**Interfaces:**
- `IReelSearch.SearchAsync(string query, SearchFilters filters, int limit, CancellationToken)` returns ranked Reel results.

- [ ] **Step 1: Write search tests**

Use known fixtures to prove exact keyword matches, semantic matches, metadata filters, empty queries, pagination, and deterministic ranking.

- [ ] **Step 2: Implement full-text search**

Build a PostgreSQL `tsvector`/GIN index over the search document fields or equivalent generated search vector.

- [ ] **Step 3: Implement vector similarity**

Generate a query embedding and execute pgvector similarity search with the configured distance metric.

- [ ] **Step 4: Implement hybrid ranking**

Normalize lexical and semantic scores, combine them with configured weights, then apply metadata constraints. Return score components for diagnostics in development mode.

- [ ] **Step 5: Add search endpoint**

Implement `GET /api/search?q=...` with optional creator/language/content-type/tag/date filters and pagination.

- [ ] **Step 6: Run search tests**

Expected: PASS.

- [ ] **Step 7: Commit**

Commit: `feat: add hybrid reel search`

---

### Task 11: Minimal web UI

**Files:**
- Create: `src/SocialMemory.Web/*`
- Create: `tests/SocialMemory.Web.Tests/*` where practical
- Modify: `docker-compose.yml`

**Interfaces:**
- UI calls the API only through documented HTTP contracts.
- UI exposes save URL, processing status, search, result cards, and Open Reel.

- [ ] **Step 1: Write UI behavior tests**

Cover submitting a URL, displaying queued/processing/completed/failed states, entering a semantic query, rendering results, and opening the original URL.

- [ ] **Step 2: Implement minimal search-first UI**

Use a responsive layout suitable for desktop and mobile browsers. Keep the MVP free of unnecessary dashboards.

- [ ] **Step 3: Implement save/progress flow**

After submission, poll job status until completion/failure and display the structured analysis when available.

- [ ] **Step 4: Implement result cards**

Show title, summary, creator, topics, keywords, saved date, processing status, and an Open Reel action.

- [ ] **Step 5: Run UI tests/build**

Expected: PASS/build succeeds.

- [ ] **Step 6: Commit**

Commit: `feat: add reel library web ui`

---

### Task 12: Dockerized integration environment and operational checks

**Files:**
- Modify: `docker-compose.yml`
- Create: `docker-compose.dev.yml`
- Create: `.env.example`
- Create: `docs/deployment.md`
- Create: `tests/SocialMemory.IntegrationTests/*`
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- One documented command starts PostgreSQL, Redis, API, Worker, and Web.
- CI builds all projects and runs unit/integration tests without external Instagram or paid AI services.

- [ ] **Step 1: Write integration tests**

Exercise `POST /api/reels` through PostgreSQL/Redis using a fake ReelSource and fake AI providers, then verify analysis persistence, search, and temporary-media cleanup.

- [ ] **Step 2: Implement development compose configuration**

Mount only the temporary-media directory where necessary; use named volumes for PostgreSQL/Redis. Never mount a permanent Reel-media volume.

- [ ] **Step 3: Add environment template**

Document database, Redis, model provider URLs/models, temporary workspace root, TTL, queue concurrency, and authentication settings.

- [ ] **Step 4: Add CI**

Run formatting/build/test and integration services. Ensure no secrets or real social-media URLs are required.

- [ ] **Step 5: Document deployment and cleanup guarantees**

Document startup, migrations, local AI dependencies, health checks, backup scope, and the fact that Reel media is intentionally not backed up because it is temporary.

- [ ] **Step 6: Run the complete verification suite**

Run: `dotnet test src/SocialMemory.sln` plus the web build and Docker Compose configuration validation.
Expected: PASS.

- [ ] **Step 7: Commit**

Commit: `chore: add local deployment and ci`

---

## Final verification checklist

- [ ] Submit a real Reel URL in an opt-in/manual integration environment.
- [ ] Confirm the Reel reaches `Completed`.
- [ ] Confirm transcript, OCR, visual description, structured analysis, and embedding are persisted.
- [ ] Confirm the temporary workspace is empty after successful processing.
- [ ] Confirm a semantic query retrieves the Reel without relying on exact caption text.
- [ ] Confirm exact keyword search also works.
- [ ] Confirm duplicate submission does not create a second Reel.
- [ ] Force a provider failure and verify retry resumes at the failed stage.
- [ ] Leave a workspace abandoned and confirm TTL cleanup removes it.
- [ ] Verify no video/audio/frame files exist in permanent Docker volumes or PostgreSQL.
- [ ] Run the full automated suite before claiming completion.
