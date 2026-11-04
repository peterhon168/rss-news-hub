# rss-news-hub — Implementation Tasks

Tasks are ordered by dependency. Each task lists which prior tasks it depends on.
"Spec-only" tasks (this file) carry no code; all tasks below are implementation work.

Legend:
- [ ] = not started
- [x] = done

---

## Phase 0 — Project Scaffolding

### T-00: Repository and toolchain setup
**Depends on**: nothing
- [ ] Initialize Vite + React 18 + TypeScript project at repo root
- [ ] Install core dependencies: `react`, `react-dom`, `zustand`, `idb`, `fast-xml-parser`, `openai`, `tailwindcss`
- [ ] Configure Tailwind CSS (`tailwind.config.ts`, `postcss.config.js`)
- [ ] Add `.env.example` documenting `VITE_AI_API_KEY`, `VITE_AI_BASE_URL`, `VITE_AI_MODEL`
- [ ] Verify `vite build` and `vite dev` run without errors
- [ ] Confirm `.kiro` is not in `.gitignore`

---

## Phase 1 — Data Layer

### T-01: TypeScript types
**Depends on**: T-00
- [ ] Create `src/types/index.ts` with `Feed`, `Article`, and `AppSettings` interfaces as specified in design.md §4

### T-02: IndexedDB schema
**Depends on**: T-01
- [ ] Create `src/db/schema.ts` defining `rss-news-hub-db` with object stores: `feeds`, `articles`, `settings`
- [ ] Implement versioned `upgrade` callback using `idb` library
- [ ] Export a `getDb()` helper that opens (and migrates) the database on first call

### T-03: Cache service
**Depends on**: T-02
- [ ] Create `src/services/cacheService.ts`
- [ ] Implement `upsertArticles(articles: Article[]): Promise<void>` — insert new, update existing (never duplicate)
- [ ] Implement `getArticlesByFeed(feedId: string): Promise<Article[]>`
- [ ] Implement `getAllArticles(): Promise<Article[]>`
- [ ] Implement `updateArticleSummary(articleId: string, summary: string): Promise<void>`
- [ ] Implement `setArticleRead(articleId: string, isRead: boolean): Promise<void>` — read state stored independently; survives eviction
- [ ] Implement `evictIfOverLimit(maxArticles: number): Promise<void>` using algorithm from design.md §7
- [ ] Implement `deleteArticlesByFeed(feedId: string): Promise<void>`

### T-04: Zustand store
**Depends on**: T-01, T-02
- [ ] Create `src/store/appStore.ts` with slices:
  - `feeds: Feed[]` + actions: `addFeed`, `removeFeed`, `updateFeed`
  - `articles: Article[]` + actions: `setArticles`, `setArticleSummary`, `markRead`
  - `settings: AppSettings` + action: `updateSettings`
  - `ui: { isOffline: boolean, activeView: 'all' | 'feed' | 'digest', selectedFeedId: string | null }` + actions: `setOffline`, `setView`, `selectFeed`
- [ ] On store init, hydrate `feeds`, `articles`, and `settings` from IndexedDB via cacheService

---

## Phase 2 — Feed Aggregation

### T-05: Feed parse service
**Depends on**: T-01
- [ ] Create `src/services/parseService.ts`
- [ ] Parse RSS 2.0 feeds: extract `channel` metadata and `item` entries
- [ ] Parse Atom 1.0 feeds: extract `feed` metadata and `entry` elements
- [ ] Return a normalized `{ feed: Partial<Feed>, articles: Article[] }` object
- [ ] Handle missing optional fields gracefully (empty string defaults, not exceptions)

### T-06: Dedup service
**Depends on**: T-01
- [ ] Create `src/services/dedupService.ts`
- [ ] Implement `dedupKey(feedId: string, rawGuid: string | undefined, url: string): string`
  - Use `feedId + ':' + rawGuid` if GUID present
  - Fall back to `feedId + ':hash:' + sha1(url)` using Web Crypto API

### T-07: Fetch service
**Depends on**: T-05, T-06
- [ ] Create `src/services/fetchService.ts`
- [ ] Implement `fetchFeed(feed: Feed): Promise<{ articles: Article[], feedMeta: Partial<Feed> }>`
- [ ] On network or HTTP error, throw a typed `FetchError` with a human-readable message
- [ ] On malformed XML (parseService throws), throw a `ParseError`
- [ ] Never silently swallow errors

### T-08: Feed service
**Depends on**: T-03, T-04, T-07
- [ ] Create `src/services/feedService.ts`
- [ ] Implement `addFeed(url: string): Promise<void>`
  - Validate URL format
  - Call `fetchService.fetchFeed` — on error, throw; UI shows validation error (REQ-AGG-02, REQ-AGG-03)
  - Persist feed config via cacheService; upsert articles; update store
- [ ] Implement `removeFeed(feedId: string): Promise<void>`
  - Delete feed config and all its articles from IndexedDB (REQ-AGG-04)
  - Update store
- [ ] Implement `setFetchError(feedId: string, msg: string): Promise<void>`
- [ ] Implement `setLastFetched(feedId: string, ts: number): Promise<void>`

### T-09: Poll service
**Depends on**: T-08
- [ ] Create `src/services/pollService.ts`
- [ ] Implement a per-feed polling scheduler using `setInterval`
- [ ] On each tick: call `fetchService.fetchFeed`; on success upsert articles + call eviction; on error call `feedService.setFetchError`
- [ ] Expose `pause()` and `resume()` methods for offline gating (REQ-OFF-08, REQ-OFF-09)
- [ ] On app init, start polling for all persisted feeds

---

## Phase 3 — UI: Feed Aggregation

### T-10: Feed list sidebar
**Depends on**: T-04, T-08
- [ ] Create `src/components/FeedList/` component
- [ ] Render list of feeds from store with: feed title, last-fetched timestamp, error badge if `fetchStatus === 'error'` (REQ-AGG-08, REQ-AGG-10)
- [ ] "Add Feed" input: URL text field + submit button; call `feedService.addFeed`; show inline validation error on failure (REQ-AGG-01, REQ-AGG-03)
- [ ] "Remove Feed" button per feed with confirmation dialog (REQ-AGG-04)

### T-11: Article list view
**Depends on**: T-04
- [ ] Create `src/components/ArticleList/` component
- [ ] Render unified article list (all feeds) sorted by `publishedAt` descending (REQ-AGG-13)
- [ ] Support filtering to a single feed when `ui.selectedFeedId` is set (REQ-AGG-14)
- [ ] Each row: title, feed name, publication date, read/unread indicator (REQ-AGG-16)
- [ ] Clicking an article: mark as read (`cacheService.setArticleRead` + store) and navigate to ArticleView (REQ-AGG-15)

### T-12: Article detail view
**Depends on**: T-04, T-11
- [ ] Create `src/components/ArticleView/` component
- [ ] Display: title, feed name, publication date, body (`description`), and link to original article
- [ ] Display existing `summary` if present
- [ ] Render "Summarize" button (disabled with tooltip if `description` is empty — REQ-SUM-04)
- [ ] Render "Summarize" button disabled when offline (REQ-OFF-08)

---

## Phase 4 — AI Summarization

### T-13: AI service
**Depends on**: T-01, T-04
- [ ] Create `src/services/aiService.ts`
- [ ] Read `VITE_AI_API_KEY`, `VITE_AI_BASE_URL`, `VITE_AI_MODEL` from `import.meta.env`
- [ ] Implement `summarizeArticle(article: Article): Promise<string>` — prompt includes citation header (REQ-SUM-03)
- [ ] Implement `summarizeDigest(articles: Article[]): Promise<string>` — prompt includes per-article citation list (REQ-SUM-08)
- [ ] On any API error, throw a typed `AiError` with `isRateLimit: boolean` flag (REQ-SUM-05, REQ-SUM-10)
- [ ] Expose `isEnabled(): boolean` — returns false when offline or API key not configured

### T-14: Summarize action (single article)
**Depends on**: T-12, T-13
- [ ] Wire "Summarize" button in ArticleView to `aiService.summarizeArticle`
- [ ] Show loading state on button during API call
- [ ] On success: call `cacheService.updateArticleSummary` + update store; display summary in ArticleView (REQ-SUM-02)
- [ ] On error: show error toast; keep original `description` visible as fallback (REQ-SUM-05)
- [ ] On rate-limit error: show quota notification; do not retry automatically (REQ-SUM-10)

### T-15: Daily Digest view
**Depends on**: T-04, T-13
- [ ] Create `src/components/DigestView/` component
- [ ] Compute today's unread articles from store; show empty-state message if none (REQ-SUM-09)
- [ ] "Generate Digest" button: call `aiService.summarizeDigest`; show loading state
- [ ] Render returned summary plus citation list of included article titles and feed names (REQ-SUM-07, REQ-SUM-08)
- [ ] On error: show error toast (REQ-SUM-10)
- [ ] Button disabled when offline (REQ-OFF-08)

---

## Phase 5 — Offline Reading

### T-16: Offline state management
**Depends on**: T-04, T-09
- [ ] In `src/main.tsx` (or store init), attach `window.addEventListener('offline', ...)` and `window.addEventListener('online', ...)`
- [ ] On 'offline': call `appStore.setOffline(true)` + `pollService.pause()` (REQ-AGG-06, REQ-OFF-08)
- [ ] On 'online': call `appStore.setOffline(false)` + `pollService.resume()` (REQ-OFF-09)

### T-17: Offline banner
**Depends on**: T-04, T-16
- [ ] Create `src/components/OfflineBanner/` component
- [ ] Renders a clearly visible persistent banner when `ui.isOffline === true` (REQ-OFF-07)
- [ ] Banner auto-hides when connectivity is restored (REQ-OFF-09)

### T-18: Offline read path verification
**Depends on**: T-11, T-12, T-16, T-17
- [ ] Confirm ArticleList and ArticleView render correctly with no network, using only IndexedDB data (REQ-OFF-06)
- [ ] Confirm no network errors surface to the user while offline and browsing cached content
- [ ] Confirm read/unread state persists after toggling offline (REQ-OFF-10)

---

## Phase 6 — Settings and Polish

### T-19: Settings panel
**Depends on**: T-04
- [ ] Create a settings panel (modal or route) for:
  - AI API key input (masked; persisted to env/credential store, not IndexedDB — REQ-SUM-12)
  - AI model and base URL configuration
  - Poll interval (per-feed or global default)
  - Max cached articles limit (REQ-OFF-04)

### T-20: Cache eviction integration
**Depends on**: T-03, T-09, T-19
- [ ] Call `cacheService.evictIfOverLimit(settings.maxCachedArticles)` after every successful poll cycle
- [ ] Confirm eviction respects read/unread preference and never clears read state (REQ-OFF-05, REQ-OFF-10)

### T-21: Accessibility and responsive layout
**Depends on**: T-10, T-11, T-12, T-15, T-17
- [ ] Ensure all interactive controls have accessible labels (`aria-label`, `aria-describedby`)
- [ ] Verify keyboard navigation for sidebar, article list, and article detail
- [ ] Verify layout is usable on mobile viewport (min 375 px wide)

### T-22: Production build and deployment
**Depends on**: T-00 through T-21
- [ ] Confirm `vite build` produces a dist with no TypeScript errors
- [ ] Deploy to a public URL (Vercel / Netlify / GitHub Pages)
- [ ] Smoke-test add feed, summarize, and offline banner on the deployed URL
- [ ] Record demo video (≤ 3 minutes) covering all three pillars

---

## Dependency Graph (summary)

```
T-00
 └─ T-01
     ├─ T-02 ─ T-03 ─┐
     │                ├─ T-08 ─ T-09 ─ T-16 ─ T-17
     ├─ T-04 ──────────────────────────────────────┤
     ├─ T-05 ─┐       └─ T-10 ─ T-11 ─ T-12
     ├─ T-06 ─┤                  └─ T-14 (+ T-13)
     └─ T-07 ←┘
               └─ T-08
T-13 ─ T-14
T-13 ─ T-15
T-19 ─ T-20 (+ T-03, T-09)
T-10,T-11,T-12,T-15,T-17 ─ T-21
All ─ T-22
```
