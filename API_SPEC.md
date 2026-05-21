# Shadow Audio v0 API_SPEC

## 1. Purpose

This document defines the minimum API contract for v0:

`iPhone recording -> chunk upload -> STT -> speaker identity review -> transcript/history`

v0 is single-user. Authentication can start as one server-side API token, but every endpoint should still be explicit and idempotent.

## 2. API Principles

### Human version

The iPhone should be able to upload slowly, retry safely, and recover after network failures without creating duplicate sessions or duplicate chunks.

### Technical rules

- Base path: `/v1`
- Request body: JSON unless uploading audio bytes
- Timestamps: ISO 8601 UTC
- IDs: client-generated ULID-style IDs with prefixes
- Audio upload must be idempotent by `chunk_id`
- Server should return the existing object if the same `chunk_id` is uploaded again
- Raw audio goes to R2
- Embeddings live in Postgres + pgvector

## 3. Error Shape

All non-2xx JSON responses should use:

```json
{
  "error": {
    "code": "invalid_request",
    "message": "Human-readable message",
    "details": {}
  }
}
```

## 4. Sessions

## 4.1 Create Or Upsert Session

`PUT /v1/sessions/{session_id}`

Creates a session if it does not exist. Safe to retry.

Request:

```json
{
  "started_at": "2026-04-29T23:00:00Z",
  "status": "ingesting",
  "client": {
    "platform": "ios",
    "app_version": "0.1.0"
  }
}
```

Response:

```json
{
  "session_id": "ses_01...",
  "status": "ingesting"
}
```

## 4.2 Complete Session

`POST /v1/sessions/{session_id}/complete`

Marks recording complete from the client side. Server may still be transcribing or analyzing.

Request:

```json
{
  "ended_at": "2026-04-29T23:42:00Z",
  "chunk_count": 18,
  "speech_chunk_count": 12
}
```

Response:

```json
{
  "session_id": "ses_01...",
  "status": "transcribing"
}
```

## 4.3 List Sessions

`GET /v1/sessions?limit=30&before=2026-04-29T23:42:00Z`

Response:

```json
{
  "sessions": [
    {
      "session_id": "ses_01...",
      "started_at": "2026-04-29T23:00:00Z",
      "ended_at": "2026-04-29T23:42:00Z",
      "status": "ready",
      "summary_short": "Dinner conversation about product direction.",
      "speaker_count_estimate": 2,
      "primary_location": {
        "place_label_short": "Home",
        "locality": "Vancouver"
      }
    }
  ],
  "next_before": "2026-04-29T22:00:00Z"
}
```

## 4.4 Get Session Detail

`GET /v1/sessions/{session_id}`

Response:

```json
{
  "session_id": "ses_01...",
  "status": "ready",
  "started_at": "2026-04-29T23:00:00Z",
  "ended_at": "2026-04-29T23:42:00Z",
  "summary_short": "Short optional summary.",
  "speakers": [
    {
      "session_speaker_id": "ssp_01...",
      "speaker_label": "SPK_0",
      "person_id": "per_self",
      "identity_confidence": 0.98
    }
  ]
}
```

## 5. Location Snapshots

`POST /v1/sessions/{session_id}/location-snapshots`

Uploads location captured during an active session.

Request:

```json
{
  "snapshot_id": "loc_01...",
  "captured_at": "2026-04-29T23:01:00Z",
  "capture_reason": "session_start",
  "latitude": 49.2827,
  "longitude": -123.1207,
  "horizontal_accuracy_m": 18.5,
  "place_label_short": "Home",
  "locality": "Vancouver",
  "admin_area": "BC",
  "country_code": "CA",
  "timezone_id": "America/Vancouver"
}
```

Response:

```json
{
  "snapshot_id": "loc_01...",
  "accepted": true
}
```

## 6. Audio Chunks

## 6.1 Upload Chunk

`PUT /v1/chunks/{chunk_id}`

Use `multipart/form-data`:

- `metadata`: JSON string
- `audio`: `.m4a` file bytes

Metadata:

```json
{
  "session_id": "ses_01...",
  "seq_no": 7,
  "started_at": "2026-04-29T23:07:00Z",
  "ended_at": "2026-04-29T23:08:00Z",
  "duration_ms": 60000,
  "has_speech": true,
  "size_bytes": 481002,
  "sha256": "abc123...",
  "mime_type": "audio/mp4"
}
```

Response:

```json
{
  "chunk_id": "chk_01...",
  "session_id": "ses_01...",
  "ingest_status": "stored",
  "stt_status": "queued",
  "object_key": "sessions/ses_01/chunks/chk_01.m4a",
  "duplicate": false
}
```

If the same `chunk_id` is uploaded again with the same `sha256`, return `200` and `duplicate: true`.

If the same `chunk_id` has a different `sha256`, return `409 conflict`.

## 6.2 List Chunk Status

`GET /v1/sessions/{session_id}/chunks`

Response:

```json
{
  "chunks": [
    {
      "chunk_id": "chk_01...",
      "seq_no": 1,
      "ingest_status": "verified",
      "stt_status": "done"
    }
  ]
}
```

## 7. Transcript

## 7.1 Get Session Transcript

`GET /v1/sessions/{session_id}/transcript`

Response:

```json
{
  "session_id": "ses_01...",
  "segments": [
    {
      "segment_id": "seg_01...",
      "chunk_id": "chk_01...",
      "session_speaker_id": "ssp_01...",
      "speaker_label": "SPK_0",
      "person_id": "per_self",
      "start_ms": 1200,
      "end_ms": 4800,
      "text": "We should keep the review queue identity-only.",
      "confidence": 0.91
    }
  ]
}
```

## 8. Review Queue

## 8.1 Fetch Review Items

`GET /v1/review-items?status=pending&limit=20`

Response:

```json
{
  "items": [
    {
      "review_item_id": "rev_01...",
      "review_type": "identify_person",
      "priority": 100,
      "target_session_speaker_id": "ssp_01...",
      "candidate_person_id": "per_01...",
      "candidate_confidence": 0.74,
      "question": "Who is this speaker?",
      "audio_examples": [
        {
          "example_id": "rea_01...",
          "seq_no": 1,
          "audio_url": "https://signed-url.example/audio.m4a",
          "start_ms": 1200,
          "end_ms": 8200,
          "transcript_hint": "Short optional hint"
        }
      ]
    }
  ]
}
```

Rules:

- `audio_examples` must contain at least 1 item when available
- `audio_examples` must contain at most 3 items
- URLs should be short-lived signed URLs

## 8.2 Submit Review Answer

`POST /v1/review-items/{review_item_id}/answer`

For `identify_person`:

```json
{
  "action": "confirm_person",
  "person_id": "per_01..."
}
```

For manual identity entry:

```json
{
  "action": "create_person",
  "canonical_name": "Mom",
  "person_kind": "known"
}
```

For same-person merge:

```json
{
  "action": "merge_person",
  "source_person_id": "per_tmp...",
  "target_person_id": "per_01..."
}
```

For deferral or dismissal:

```json
{
  "action": "snooze",
  "snooze_until": "2026-05-06T12:00:00Z"
}
```

Supported actions:

- `confirm_person`
- `create_person`
- `merge_person`
- `reject_candidate`
- `snooze`
- `dismiss`

Response:

```json
{
  "review_item_id": "rev_01...",
  "status": "resolved",
  "updated_entities": {
    "person_id": "per_01...",
    "session_speaker_id": "ssp_01...",
    "voiceprint_profile_id": "vpr_01..."
  }
}
```

Server behavior:

- Update `session_speakers.person_id` when identity is confirmed
- Create or update `person_entities` when needed
- Create or update `voiceprint_profiles` only from eligible samples
- Do not update the main voiceprint template from non-trusted samples

## 9. Search And Memory

## 9.1 Search Transcript

`GET /v1/search/transcript?q=keyword&limit=20`

Response:

```json
{
  "results": [
    {
      "session_id": "ses_01...",
      "segment_id": "seg_01...",
      "started_at": "2026-04-29T23:00:00Z",
      "speaker_label": "SPK_0",
      "text": "Matched transcript text"
    }
  ]
}
```

## 9.2 List Recent Memories

`GET /v1/memories?type=decision&limit=20`

Response:

```json
{
  "memories": [
    {
      "memory_id": "mem_01...",
      "memory_type": "decision",
      "title": "Keep review queue identity-only",
      "text": "The system should not ask subjective relationship importance questions in v0.",
      "importance_score": 0.72,
      "confidence": 0.89,
      "created_at": "2026-04-29T23:50:00Z"
    }
  ]
}
```

Memory endpoints are read-only in v0. Memory extraction is handled by workers.

## 10. Worker-Facing Endpoints

These can start as internal endpoints protected by the same server token.

v0 should implement these first with a thin mock worker. The mock worker should prove the database state flow before real STT, diarization, and voiceprint models are connected.

## 10.1 Lease STT Job

`POST /v1/internal/stt-jobs/lease`

Request:

```json
{
  "worker_id": "worker_01",
  "limit": 1
}
```

## 10.2 Complete STT Job

`POST /v1/internal/stt-jobs/{job_id}/complete`

Request:

```json
{
  "segments": [
    {
      "segment_id": "seg_01...",
      "chunk_id": "chk_01...",
      "seq_no": 1,
      "start_ms": 1200,
      "end_ms": 4800,
      "text": "Transcript text",
      "confidence": 0.91
    }
  ]
}
```

## 10.3 Create Analysis Run

`POST /v1/internal/analysis-runs`

Request:

```json
{
  "run_id": "run_01...",
  "session_id": "ses_01...",
  "run_type": "speaker_resolution",
  "model_name": "pyannote+nemo",
  "prompt_version": "v0"
}
```

## 11. v0 Non-Goals

- Multi-user auth
- Public sharing
- Real-time streaming upload
- Full memory editing APIs
- Notification APIs
- Complex relationship-importance review
- Preference-upgrade review

## 12. Implementation Order

1. Implement session upsert and completion.
2. Implement chunk upload to R2 with idempotency.
3. Implement STT job creation and transcript read.
4. Implement location snapshot upload.
5. Implement session list/detail.
6. Implement review item fetch and answer submit.
7. Implement transcript search and recent memories read.

Backend stack:

- `FastAPI`
- `Postgres + pgvector`
- `Cloudflare R2`
- thin mock worker first
- real STT / diarization / voiceprint workers second
