# rss-news-hub — Design

## 1. Overview

rss-news-hub is a single-user, client-side-first web application.  All three
pillars (feed aggregation, AI summarization, offline reading) run in the
browser, backed by a lightweight local persistence layer.  The only external
network calls are feed fetches and AI API calls; no proprietary backend server
is required for v1.

```
┌─────────────────────────────────────────────────────────┐
│                        Browser                          │
│                                                         │
│  ┌──────────┐   ┌────────────┐   ┌───────────────────┐ │
│  │  UI Layer │   │ App Layer  │   │ Persistence Layer │ │
│  │  (React)  │◄─►│ (Services) │◄─►│  (IndexedDB /    │ │
│  └──────────┘   └─────┬──────┘   │   localStorage)   │ │
│                        │          └───────────────────┘ │
│                ┌───────┴───────┐                        │
│                │  Feed Poller  │  (Web Worker / setInterval) │
│                └───────┬───────┘                        │
└────────────────────────┼────────────────────────────────┘
                         │ HTTP(S)
         ┌───────────────┼───────────────┐
         │               │               │
    RSS / Atom       AI LLM API       (nothing else)
    feed servers   (OpenAI-compatible)
```

---

## 2. Technology Choices

| Concern | Choice | Rationale |
|---------|--------|-----------|
| UI framework | React 18 + TypeScript | Ecosystem breadth, component model |
| Build tool | Vite | Fast HMR, ES module output |
| State management | Zustand | Lightweight, no boilerplate |
| Local persistence | IndexedDB via `idb` wrapper | Large storage quota; survives restarts |
| Feed parsing | `fast-xml-parser` (in-browser) | No server dependency |
| AI calls | `openai` JS SDK (OpenAI-compatible) | Swap-friendly; key in env var |
| Offline detection | Navigator.onLine + `online`/`offline` events | Standard, no library needed |
| Styling | Tailwind CSS | Utility-first; rapid iteration |

---

## 3. Module Structure

```
src/
├── components/          # React UI components
│   ├── FeedList/        # Sidebar: list of feeds + add/remove
│   ├── ArticleList/     # Main panel: unified or per-feed article list
│   ├── ArticleView/     # Article detail: body + Summarize button
│   ├── DigestView/      # Daily Digest panel
│   └── OfflineBanner/   # Persistent offline indicator
│
├── services/
│   ├── feedService.ts   # Add, remove, persist feed configs
│   ├── fetchService.ts  # HTTP fetch of RSS/Atom; error handling
│   ├── parseService.ts  # XML → Article model (RSS 2.0 + Atom 1.0)
│   ├── pollService.ts   # Polling scheduler (setInterval / Web Worker)
│   ├── cacheService.ts  # IndexedDB read/write/eviction
│   ├── dedupService.ts  # GUID / URL-hash deduplication key
│   └── aiService.ts     # OpenAI-compatible summarization calls
│
├── store/
│   └── appStore.ts      # Zustand slices: feeds, articles, ui, offline
│
├── db/
│   └── schema.ts        # IndexedDB schema, version migrations
│
├── types/
│   └── index.ts         # Shared TypeScript types
│
└── main.tsx             # Entry point
```

---

## 4. Data Models

### Feed

```typescript
interface Feed {
  id: string;            // uuid
  url: string;           // canonical feed URL
  title: string;         // parsed from feed metadata
  lastFetchedAt: number | null;  // epoch ms; null = never
  fetchStatus: 'ok' | 'error';
  fetchError: string | null;
  pollIntervalMs: number;        // default 15 * 60 * 1000
}
```

### Article

```typescript
interface Article {
  id: string;            // feedId + ':' + dedupKey
  feedId: string;
  feedTitle: string;     // denormalized for display
  guid: string;          // original GUID / id from feed
  title: string;
  link: string;
  description: string;   // body text as provided by feed
  publishedAt: number;   // epoch ms
  isRead: boolean;
  summary: string | null;  // AI-generated; null = not yet generated
  cachedAt: number;        // epoch ms; used for eviction ordering
}
```

### AppSettings

```typescript
interface AppSettings {
  maxCachedArticles: number;   // default 500
  aiApiKey: string | null;     // stored via platform credential store
  aiBaseUrl: string;           // default OpenAI endpoint
  aiModel: string;             // e.g. 'gpt-4o-mini'
}
```

---

## 5. Pillar Data Flows

### 5.1 Feed Aggregation — Add Feed Flow

```
User enters URL
      │
      ▼
feedService.addFeed(url)
      │
      ├─► fetchService.fetchRaw(url)  ─── network ──► feed server
      │         │ error                   │ success
      │         ▼                         ▼
      │   UI: validation error      parseService.parse(xml)
      │                                   │
      │                             Article[]  +  Feed metadata
      │                                   │
      │                             dedupService.key(article)
      │                                   │
      │                             cacheService.upsertArticles()
      │                                   │
      │                             feedService.persistFeed()
      │                                   │
      ▼                             appStore.addFeed()  ──► UI refresh
```

### 5.2 Feed Aggregation — Poll Cycle

```
pollService.tick(feedId)
      │
      ├─► fetchService.fetchRaw(feed.url)
      │         │ error
      │         └─► feedService.setFetchError(feedId, msg)
      │                   └─► appStore ──► UI shows warning badge
      │         │ success
      │         ▼
      │   parseService.parse(xml) ──► Article[]
      │         │
      │   dedupService.key() per article
      │         │
      │   cacheService.upsertArticles()   (insert new / update existing)
      │         │
      │   feedService.setLastFetched(feedId, now)
      │         │
      │   cacheService.evictIfOverLimit()
      │         │
      └─► appStore.setArticles()  ──► UI refresh
```

### 5.3 AI Summarization — Single Article

```
User clicks "Summarize"
      │
      ├─► guard: article.description empty? ──► disable button (REQ-SUM-04)
      │
      ▼
aiService.summarizeArticle(article)
      │
      ├─► Build prompt: "Summarize the following article. Cite source: {feedTitle} — {title}.\n\n{description}"
      │
      ├─► openai.chat.completions.create(...)  ─── network ──► AI API
      │         │ error / rate-limit
      │         └─► UI: error toast; article.description shown as fallback (REQ-SUM-05)
      │         │ success
      │         ▼
      │   cacheService.updateArticleSummary(articleId, summaryText)
      │         │
      └─► appStore.setArticleSummary()  ──► ArticleView shows summary
```

### 5.4 AI Summarization — Daily Digest

```
User opens DigestView
      │
      ├─► appStore.todayUnreadArticles() ──► empty? show empty state (REQ-SUM-09)
      │
      ▼
aiService.summarizeDigest(articles[])
      │
      ├─► Build prompt: headlines + descriptions + per-article citation block
      │
      ├─► openai.chat.completions.create(...)  ─── network ──► AI API
      │         │ error / rate-limit
      │         └─► UI: error toast with quota note (REQ-SUM-10)
      │         │ success
      │         ▼
      └─► DigestView renders summary + citation list
```

### 5.5 Offline Reading

```
Navigator 'offline' event
      │
      ▼
appStore.setOffline(true)
      ├─► OfflineBanner renders (REQ-OFF-07)
      ├─► pollService.pause()  (REQ-OFF-08)
      └─► aiService.disable()  (REQ-OFF-08)

UI reads from appStore.articles  (sourced from IndexedDB cache)
      └─► Articles and summaries displayed from cache (REQ-OFF-06)

Navigator 'online' event
      │
      ▼
appStore.setOffline(false)
      ├─► OfflineBanner hides (REQ-OFF-09)
      └─► pollService.resume()  (REQ-OFF-09)
```

---

## 6. Persistence Layer

All persistent state lives in a single IndexedDB database: `rss-news-hub-db`.

| Object Store | Key | Indexes | Notes |
|---|---|---|---|
| `feeds` | `id` (uuid) | — | One record per feed |
| `articles` | `id` (feedId:dedupKey) | `feedId`, `publishedAt`, `isRead`, `cachedAt` | Main article store |
| `settings` | `'singleton'` | — | Single AppSettings record |

Migrations are versioned in `db/schema.ts` using the idb `upgrade` callback.

Read state (`isRead`) is stored as a dedicated field on the Article record and is
never cleared during cache eviction; eviction only removes entire Article
records for the oldest-by-`cachedAt`, and only after confirming the article
is marked read (preferring to retain unread articles, per REQ-OFF-05 and
REQ-OFF-10).

---

## 7. Cache Eviction Algorithm

```
function evictIfOverLimit(maxArticles: number):
  total = count(articles store)
  if total <= maxArticles: return

  overflow = total - maxArticles

  // Phase 1: evict read articles oldest-first
  readOldest = query articles WHERE isRead=true ORDER BY cachedAt ASC LIMIT overflow
  delete readOldest
  overflow -= len(readOldest)

  // Phase 2: if still over limit, evict unread articles oldest-first
  if overflow > 0:
    unreadOldest = query articles ORDER BY cachedAt ASC LIMIT overflow
    delete unreadOldest
```

---

## 8. Security Considerations

- AI API keys are never stored in IndexedDB or localStorage; they are supplied
  via a `.env` file (excluded from version control) or the OS credential store.
- Feed fetch requests are made directly from the browser; where CORS prevents
  direct fetching, a simple same-origin proxy endpoint can be added in v2 — v1
  may surface CORS errors as fetch errors per REQ-AGG-08.
- No user content is transmitted to external services other than article text
  sent to the configured AI API (REQ-SUM-11).

---

## 9. Offline-First Guarantee

The app is structured so the critical read path (browse articles, read cached
article body, view existing summaries) never touches the network and never
throws on absent network.  Only the write paths (poll, summarize) require
connectivity, and both are gated by the `isOffline` flag in the store.
