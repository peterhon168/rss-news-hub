# rss-news-hub — Requirements

Project: AI-powered personal news hub reviving the classic RSS reader.
Track: Kiroween 2026, Resurrection.

Requirements are written in EARS notation (Easy Approach to Requirements Syntax).
Template forms used:
- **Ubiquitous**: The \<system\> shall \<action\>.
- **Event-driven**: When \<trigger\>, the \<system\> shall \<action\>.
- **State-driven**: While \<state\>, the \<system\> shall \<action\>.
- **Conditional**: Where \<condition\>, the \<system\> shall \<action\>.
- **Optional feature**: Where \<feature is included\>, the \<system\> shall \<action\>.

---

## Pillar 1 — Feed Aggregation

### Feed Management

**REQ-AGG-01** The system shall allow a user to add an RSS or Atom feed by providing a URL.

**REQ-AGG-02** The system shall validate that a submitted URL resolves to a well-formed RSS 2.0 or Atom 1.0 document before persisting it.

**REQ-AGG-03** When a feed URL fails validation, the system shall reject the submission and display a human-readable error message identifying the reason for failure.

**REQ-AGG-04** The system shall allow a user to remove a previously added feed, permanently deleting it and all its cached articles from local storage.

**REQ-AGG-05** The system shall persist the list of user-configured feed sources across application restarts.

### Fetching and Polling

**REQ-AGG-06** The system shall poll each configured feed at a user-configurable interval, defaulting to every 15 minutes.

**REQ-AGG-07** When a feed fetch succeeds, the system shall parse and ingest all articles from the feed response.

**REQ-AGG-08** When a feed fetch fails (network error, HTTP error, or malformed response), the system shall display a visible warning on that feed's entry without removing the feed or silently discarding it.

**REQ-AGG-09** While a feed is in a failed-fetch state, the system shall retain all previously cached articles for that feed and continue displaying them.

**REQ-AGG-10** The system shall record the timestamp of the last successful fetch for each feed and display it in the feed list.

### Deduplication

**REQ-AGG-11** The system shall deduplicate articles using the article's GUID (RSS) or `id` (Atom); where no GUID or id is present, the system shall fall back to a hash of the article URL.

**REQ-AGG-12** When an article is already present in the local store (same feed + deduplicated key), the system shall update its metadata but shall not create a duplicate entry.

### Browsing

**REQ-AGG-13** The system shall provide a unified article list view combining articles from all feeds, sorted by publication date descending.

**REQ-AGG-14** The system shall provide a per-feed view showing only articles from a single selected feed, sorted by publication date descending.

**REQ-AGG-15** The system shall allow a user to mark individual articles as read, and shall visually distinguish read articles from unread ones.

**REQ-AGG-16** The system shall display each article entry with at minimum: title, source feed name, publication date, and read/unread status.

### v1 Out-of-Scope (Feed Aggregation)

- OPML import and export are deferred to v2.
- Fetching or caching content from paid sources is not supported.
- Bypassing paywalls or extracting full text via scraping is not supported.

---

## Pillar 2 — AI Summarization

### Single-Article Summary

**REQ-SUM-01** The system shall provide an on-demand "Summarize" action for any article that has cached body text.

**REQ-SUM-02** When a user invokes the Summarize action, the system shall call an AI language-model API and display the resulting summary within the article view.

**REQ-SUM-03** Every generated summary shall include a citation identifying the source feed name and article title from which it was produced.

**REQ-SUM-04** Where an article has no cached body text (title-only or empty-description feed), the system shall disable the Summarize action and display an explanatory tooltip; the system shall not fabricate a summary from zero content.

**REQ-SUM-05** When the AI summarization API call fails, the system shall surface an error message and display the original article text as a fallback; the system shall not leave the article view blank.

### Daily Digest

**REQ-SUM-06** The system shall provide a "Daily Digest" view that aggregates unread articles from the current calendar day across all feeds.

**REQ-SUM-07** When a user requests a Daily Digest summary, the system shall send the combined headlines and available descriptions to the AI API and return a single cohesive digest summary.

**REQ-SUM-08** The Daily Digest summary shall list, at minimum, the titles and source feed names of every article included in the digest as citations.

**REQ-SUM-09** Where no unread articles exist for the current day, the system shall display an empty-state message rather than calling the AI API.

**REQ-SUM-10** When the AI API returns a rate-limit or quota error, the system shall notify the user and shall not retry automatically within the same session.

### Summary Quality Constraints

**REQ-SUM-11** The system shall send only locally cached article text to the AI API; it shall not perform additional network fetches of article URLs at summarization time.

**REQ-SUM-12** The system shall not persist raw AI API keys in unencrypted local storage; keys shall be stored using the platform's secure credential store or supplied via environment variable.

---

## Pillar 3 — Offline Reading

### Cache Management

**REQ-OFF-01** The system shall cache, for each fetched article, the summary text (if generated) and the article body text as returned by the feed.

**REQ-OFF-02** The system shall not perform full-text offline sync (fetching full article HTML from the article's source URL); cached content is limited to what the feed provides plus generated summaries.

**REQ-OFF-03** The system shall store cached article data in a persistent local data store that survives application restarts.

**REQ-OFF-04** The system shall provide a user-configurable maximum cache size (number of articles per feed or total storage quota), defaulting to 500 articles total.

**REQ-OFF-05** When the cache exceeds the configured maximum, the system shall evict the oldest articles by publication date, preferring to keep unread articles over read ones.

### Offline State and UX

**REQ-OFF-06** While the device has no network connectivity, the system shall continue to display all cached articles and summaries without error.

**REQ-OFF-07** While the device has no network connectivity, the system shall display a persistent, clearly visible offline-mode indicator.

**REQ-OFF-08** While the device is offline, the system shall disable feed polling and the Summarize action, and shall show appropriate disabled-state UI to the user.

**REQ-OFF-09** When network connectivity is restored, the system shall automatically resume feed polling and remove the offline indicator.

**REQ-OFF-10** A cache write failure shall never discard or overwrite an article's existing read/unread state; the read state shall be stored independently and survive cache eviction.

---

## Submission Constraints (Non-Functional)

**REQ-SUBM-01** The application shall be runnable from a public GitHub repository with no code committed before 2026-10-30.

**REQ-SUBM-02** The repository shall contain a LICENSE file at the root.

**REQ-SUBM-03** The `.kiro` directory shall exist at the repository root and shall not appear in `.gitignore`.

**REQ-SUBM-04** A public demo video under 3 minutes shall accompany the submission.
