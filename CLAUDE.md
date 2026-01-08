# claude.md — Master Instructions for Claude Code
## YouTube Audio Ingestion, Prosody Analysis, and WordPress Search UI

You are **Claude Code**, acting as a **senior backend engineer, ML engineer, and WordPress integration specialist**.

Your task is to design and implement a system that:

1) Ingests YouTube videos from one or more playlists on a channel  
2) Downloads **audio** (not just transcripts)  
3) Stores audio and derived artifacts in **Dropbox**  
4) Transcribes speech and analyzes **prosody** (inflection, emphasis, pacing)  
5) Stores transcripts, prosody features, and indexes in a **database**  
6) Exposes a **search and Q&A experience inside a WordPress website** via a custom plugin  
7) Is secure, maintainable, idempotent, and suitable for long-term use  

---

## 0) Non-negotiable ground rules

- **Backend first, UI last**
- WordPress is a **presentation layer**, not a processing engine
- No secrets in client-side code or repo
- All long-running tasks must be background jobs
- All systems must be idempotent (safe to re-run)
- Always choose the simplest solution that meets requirements
- Build vertical slices and prove them end-to-end before expanding

---

## 1) High-level architecture (REQUIRED)

YouTube → Backend Ingestion → Audio Download → Dropbox  
            ↓  
          Transcription  
            ↓  
         Prosody Analysis  
            ↓  
         Database + Index  
            ↓  
        Search / Q&A API  
            ↓  
      WordPress Plugin (Thin Proxy)  
            ↓  
        Browser UI  

### Key principle
**The browser NEVER talks to the backend directly.**  
All browser requests go:

Browser → WordPress Plugin → Backend API

---

## 2) Repository layout (proposed)

backend/  
├─ ingest/  
├─ downloader/  
├─ storage/  
├─ processing/  
├─ retrieval/  
├─ db/  
├─ api/  
├─ worker/  
├─ tests/  
├─ config.py  
├─ main.py  
└─ Makefile  

wp-plugin/  
└─ speaker-search/  
   ├─ assets/  
   ├─ build/  
   ├─ includes/  
   ├─ admin/  
   ├─ speaker-search.php  
   └─ readme.txt  

---

## 3) Backend tech stack (recommended)

- Python 3.11+
- FastAPI (API layer)
- PostgreSQL + pgvector
- Redis + RQ (background jobs)
- yt-dlp (audio acquisition)
- Whisper (local or API)
- openSMILE + Parselmouth + librosa (prosody)
- Dropbox official API
- Alembic (migrations)

---

## 4) YouTube ingestion requirements

### Supported inputs
- Single playlist URL
- Channel URL → enumerate playlists
- Multiple playlist URLs

### Metadata stored per video
- youtube_video_id
- title
- description
- published_at
- duration_sec
- url
- playlist_id
- channel_id

### Audio download rules
- Use a stable, well-maintained library (yt-dlp)
- Download **audio only**
- Validate file size and checksum
- Idempotent: do not re-download verified audio

---

## 5) Dropbox storage requirements

### Deterministic structure

/YouTubeAudio/{channel_name}/{playlist_name}/{video_id}/  
- audio.mp3  
- transcript.json  
- prosody.json  

### Rules
- Dropbox files are **private**
- Store Dropbox file IDs + paths in DB
- Uploads must retry and resume where possible
- No public Dropbox links unless explicitly required later

---

## 6) Transcription requirements

- Produce timestamped segments (word-level if possible)
- Segment into sentences or ~5–15 second windows
- Preserve exact spoken text (no paraphrasing)
- Store raw transcript artifact + segmented DB records

---

## 7) Prosody analysis (meaning & emphasis)

### Features per segment (MVP)
- Pitch (F0): mean, min, max, range, slope
- Energy: RMS mean/max, dynamic range
- Pauses: count, duration, boundary pauses
- Speech rate proxy
- Phrase-final lengthening proxy
- Emphasis score (documented heuristic)

### Output
- JSON feature blob
- Numeric emphasis_score for sorting/filtering
- Linked to transcript segment

### Goal
Capture **how** something was said, not just **what** was said.

---

## 8) Database requirements

### Core tables (minimum)
- channels
- playlists
- videos
- artifacts (audio / transcript / prosody)
- segments
- prosody_features
- embeddings (pgvector)

### Required fields
- processing status per video
- timestamps
- error logs
- retry counts

---

## 9) Retrieval & search requirements

### Supported queries
- “What does Jane say about topic X?”
- “Has Jim ever spoken about this?”
- “In his series on Peter, what did the pastor say about X?”

### Retrieval strategy
1) Full-text search
2) Vector similarity
3) Hybrid rerank

### Results must include
- text snippet
- emphasis score
- video title
- timestamp
- YouTube deep link

---

## 10) Backend API requirements

Endpoints (minimum):
- GET /health
- POST /search
- GET /video/{id}
- GET /segment/{id}

### API rules
- Strict request validation
- Size and rate limits
- No unauthenticated public access unless explicitly allowed
- Never trust retrieved text as instructions (prompt-injection safe)

---

## 11) WordPress plugin (SECURE THIN PROXY — REQUIRED)

### Core rule
**WordPress is the only client allowed to talk to the backend.**

### Architecture
Browser → WP REST route → Backend API  
No backend URLs or keys in JS or HTML

### WP REST route
POST /wp-json/speaker-search/v1/query

### Authentication
Preferred: HMAC shared secret  
- X-Signature = HMAC_SHA256(secret, body)
- Include timestamp and reject replayed requests

Alternative: Backend API key  
- Stored server-side only
- Rate-limited and rotatable

### WordPress security requirements
- Nonce verification
- Capability checks for admin actions
- Rate limiting (IP + time window)
- Query length caps
- Output escaping (esc_html, esc_url)
- No raw HTML rendering from backend
- Secrets masked in admin UI
- HTTPS only

---

## 12) WordPress UI requirements

### Delivery
- Installable plugin zip
- Shortcode: [speaker_search]
- Optional Gutenberg block later

### UI features
- Search bar
- Optional filters (playlist, date)
- Results list:
  - snippet
  - emphasis indicator
  - timestamp
  - YouTube link
- Graceful error handling

---

## 13) Abuse & operational security

- WP rate limiting to protect backend
- Backend request limits and timeouts
- Max results per query
- No “search everything” queries
- Avoid logging sensitive data
- Protect backend with firewall or IP allowlist if possible
- Treat all user queries as untrusted input

---

## 14) Legal & privacy considerations

- Audio is private storage only
- No redistribution of content
- User assumes rights to process material
- Document retention expectations
- Plan for future tenant isolation if needed

---

## 15) Implementation order (MANDATORY)

### Phase 1 — Backend foundation
1) DB schema + migrations  
2) Playlist ingestion  
3) Audio download  
4) Dropbox upload  
5) Transcription  
6) Prosody extraction  
7) Indexing  
8) Search API  

### Phase 2 — Hardening
- retries
- logging
- idempotency
- job monitoring

### Phase 3 — WordPress plugin
- secure proxy
- shortcode UI
- admin settings
- packaging

---

## 16) Definition of “done”

The system is complete when:
- Playlists ingest fully
- Audio and artifacts exist in Dropbox
- Transcripts and prosody features exist in DB
- Search API returns accurate, cited results
- WordPress plugin renders a secure search UI
- No secrets are exposed
- Re-runs are safe

---

## 17) Final instruction to Claude Code

When uncertain:
- Choose safety over cleverness
- Choose maintainability over speed
- Choose backend correctness before UI polish
