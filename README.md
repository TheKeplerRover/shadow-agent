# Shadow Agent

Shadow is a user-owned, local-first mirror system for AI agents.

The design assumes explicit user consent, user-controlled data boundaries, and private-by-default storage. Sensitive signals such as conversations, device activity, or health data should only be collected when the user deliberately opts in, with clear retention, deletion, and audit controls.

Its core thesis comes from this essay: [Why A2A Is Inevitable](https://alexlyu.com/blog/why-a2a-is-inevitable.html)

If agents are going to handle most of a person's life and work, the real question is not whether AI can complete tasks. The real question is whether AI can process information, preserve preferences, form judgments, and stay consistent over time in a way that genuinely resembles the person it represents.

I do not believe chat history alone can solve that. Conversation context is too narrow, too task shaped, and too lossy. Shadow is meant to build a fuller, longer term, searchable, evolving model of the user by continuously collecting signals across many parts of that person's life, then exposing that model as an external personality layer for AI agents.

Those signals may include device behavior and daily activity, online traces, conversations with other people, explicit user input, and even health data such as information from the iOS Health app. The system first collects raw information on a daily cycle, then batches, translates, and uploads or stores it when the user becomes inactive. After that, it runs a first pass that classifies, distills, and structures the raw data, preferably with a local model doing the initial layer of understanding and compression. On top of that, the system produces higher level summaries across weekly, monthly, quarterly, and yearly intervals, assigning different memory weights to build a long term and trackable model of the user's patterns and preferences.

In phase one, Shadow is built for a single user, myself. The point is not to start as a broad consumer product. The point is to prove that this system can actually make an agent behave more like me. That means core parts of the system need to stay configurable, including the models used for processing, the storage target, and the boundary between local and remote infrastructure. Regardless of the underlying implementation, the output has to remain searchable, traceable, continuously updateable, and efficient to inject into an agent at decision time.

The immediate priority is the main chain: collection, processing, summarization, storage, and retrieval interfaces. Large scale prioritization and more mature anti-forgetting mechanisms can be addressed in phase two by adopting the right open source approach. At the same time, Shadow needs to leave room for every future product concern, including permission and privacy settings, the boundary between local and cloud, encryption and key ownership, retention and deletion policies, audit logging and access control, sensitive content classification, isolation when multiple agents share the same Shadow, and eventual compliance requirements for launch.

## Thesis

A user mirror should be:

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
