# Shadow Agent

A system design for a long-running user memory layer that AI agents can query before making decisions on a person's behalf.

Shadow is not trying to make a chatbot remember more chat history. The core idea is that future agents need a separate, durable user model: preferences, commitments, decisions, relationships, recurring context, and the evidence trail behind each memory.

## Thesis

If agents are going to handle meaningful parts of someone's life and work, the question is not only whether they can complete tasks. The harder question is whether they can preserve judgment over time.

Chat logs are too narrow and too task-shaped for that. A user mirror should be:

- searchable
- source-grounded
- continuously updated
- permission-aware
- efficient to inject at decision time

## v0 Scope

The first concrete slice is an audio-to-memory pipeline:

```text
iPhone recording
  -> chunked upload
  -> speech-to-text
  -> speaker review
  -> transcript history
  -> structured memory items
  -> retrieval API
```

v0 is intentionally single-user. The goal is to prove the main chain before adding multi-user permissions, heavy knowledge graphs, or broad product packaging.

## Documents

- [PRD](PRD.md): product goals, scope, and success criteria
- [TECH_SPEC](TECH_SPEC.md): architecture and implementation plan
- [API_SPEC](API_SPEC.md): upload, transcript, speaker, and memory endpoints
- [DATA_SCHEMA](DATA_SCHEMA.md): data model for sessions, chunks, transcripts, speakers, and memories

## Design Principles

- Raw evidence and derived memory should stay separate.
- Memory items should carry confidence and source references.
- Speaker identity should require review when confidence is low.
- Local and cloud boundaries should be explicit.
- The system should prefer a small reliable pipeline over a broad but vague agent memory story.

## Status

This repo is a public system-design snapshot, not a finished product. It is useful as a planning artifact for agent memory, personal data infrastructure, and user-model retrieval.
