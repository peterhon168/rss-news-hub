# rss-news-hub — Requirements
Project: AI-powered personal news hub reviving the classic RSS reader.
Track: Kiroween 2026, Resurrection.
## Pillar 1 — Feed aggregation
- User stories: add/remove feed sources; fetch and deduplicate; browse by source and by time.
- Acceptance: a new source is fetched and appears in the list; duplicate articles appear only once; a failed fetch shows a visible warning instead of silently dropping the source.
- v1 not-do: paid sources; bypassing paywalls for full text.
## Pillar 2 — AI summarization
- User stories: single-article summary; daily digest summary.
- Acceptance: summaries cite their source articles; no summary is fabricated when there is no content; on summarization failure, fall back to the original text.
## Pillar 3 — Offline reading
- User stories: already-fetched content readable offline; offline state clearly indicated.
- Acceptance: cached articles open with no network; a cache failure never loses read state.
## Decided defaults
- OPML import: NOT in v1 (deferred to v2).
- Offline scope: summary + article-body cache only (no full-text offline sync).
## Submission constraints
- Public GitHub repo; LICENSE at root; /.kiro at root and NOT in .gitignore; no code from before 2026-10-30.
- Deliverables before 2026-12-04 14:00 PT: runnable app link, feature description, public YouTube demo video under 3 minutes, Kiro-usage write-up.
