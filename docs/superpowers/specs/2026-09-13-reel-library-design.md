# Social Memory — Reel Library Design

## Goal

Build a self-hosted personal library for Instagram Reels where a user submits a Reel URL, the system temporarily downloads the Reel, performs multimodal analysis, persists only the URL and extracted knowledge, deletes temporary media, and later retrieves Reels through hybrid natural-language search.

## Product flow

1. User submits an Instagram Reel URL.
2. API validates and canonicalizes the URL and creates a Reel record plus processing job.
3. A background worker downloads the Reel into an isolated temporary job directory.
4. The worker extracts audio and representative video frames.
5. Whisper transcribes speech; OCR extracts visible text; a vision model describes visual content.
6. A text model synthesizes structured knowledge: title, summaries, topics, keywords, entities, key points, content type, language, transcript, OCR text, visual description, and claims.
7. An embedding is generated from a search document combining the useful textual signals.
8. The knowledge and embedding are persisted transactionally.
9. Temporary video, audio, and frames are deleted after persistence succeeds.
10. Failed jobs are retryable by stage; abandoned temporary files are cleaned automatically.

## Architecture

- ASP.NET Core/.NET 10 API for HTTP endpoints and orchestration.
- PostgreSQL with pgvector for relational data, full-text search, and vector similarity.
- Redis-backed background job queue.
- Dedicated analysis worker isolated from the public API.
- FFmpeg for media inspection/extraction.
- Provider interfaces for Reel downloading, transcription, OCR, vision, LLM synthesis, and embeddings so local or cloud implementations can be swapped without changing domain logic.
- Docker Compose for local/self-hosted deployment.
- Web UI for the MVP; iPhone Share Sheet/PWA integration follows after the core workflow is stable.

## Data retention

Permanent storage must not contain Reel video, extracted audio, or extracted frames. Temporary artifacts live only under an isolated per-job directory. Cleanup occurs after successful persistence and via a TTL cleanup process for abandoned jobs. The database retains the original URL, normalized Reel identifier when available, metadata, transcript, OCR text, visual description, structured analysis, and embeddings.

## Processing states

`Queued`, `Downloading`, `Downloaded`, `Extracting`, `Transcribing`, `Ocr`, `VisionAnalysis`, `AiSynthesis`, `Embedding`, `Persisting`, `Cleanup`, `Completed`, `Failed`.

State transitions must be persisted so the UI can report progress and retries can resume at the appropriate stage.

## Search

Use hybrid search:

- PostgreSQL full-text search for exact terms.
- pgvector cosine similarity for semantic retrieval.
- Metadata filters for creator, language, content type, tags, and saved date.
- A deterministic ranking stage combines lexical and semantic scores.

The search document should include title, summary, detailed summary, transcript, OCR text, visual description, topics, keywords, entities, key points, and claims.

## Core API

- `POST /api/reels` — submit a Reel URL; returns the Reel/job status.
- `GET /api/reels/{id}` — retrieve analysis and processing status.
- `DELETE /api/reels/{id}` — delete the saved knowledge record.
- `GET /api/search?q=...` — hybrid natural-language search.
- `GET /api/tags` — list tags.
- `GET /api/stats` — library/processing statistics.
- `GET /api/jobs/{id}` — processing status.

## Core data model

### reels

- id
- canonical_url
- source_platform
- external_id
- creator_username
- caption
- saved_at
- analyzed_at
- status
- language
- content_type

Unique constraint should prevent duplicate canonical Reels.

### reel_analysis

- reel_id
- title
- summary
- detailed_summary
- transcript
- ocr_text
- visual_description
- key_points
- topics
- keywords
- entities
- claims

### reel_embeddings

- reel_id
- search_document
- embedding
- embedding_model

### tags / reel_tags

Normalized tags for filtering and browsing.

### processing_jobs

- id
- reel_id
- current_stage
- attempts
- last_error
- created_at
- updated_at
- completed_at

## Security

- Authenticate the application even for single-user deployment.
- Validate supported URLs before queueing.
- Protect downloader operations against SSRF and arbitrary filesystem access.
- Run media processing in an isolated worker/container.
- Never accept filesystem paths from API clients.
- Store provider credentials only through environment/secret configuration.
- Apply request and download rate limits.

## Non-goals for MVP

- Native iOS application.
- Permanent media storage.
- Social sharing/collaboration.
- Kubernetes/microservices.
- Separate vector database.
- AI chat over the library.
- Support for multiple social platforms before Instagram flow is stable.

## Success criteria

A user can submit a valid Reel URL, see processing progress, receive a complete multimodal analysis, verify that temporary media is deleted, and find the Reel later using both exact keywords and natural-language descriptions that do not appear verbatim in the Reel caption.
