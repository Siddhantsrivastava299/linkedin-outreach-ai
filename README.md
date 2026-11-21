# LinkedIn Outreach AI: improvements write-up

**Goal:** make the existing extension better without rebuilding it. I used it as a user first, read the code, then picked the problems that most damage trust and message quality.

## What I found
| # | Problem | Why it matters | Fixed? |
|---|---------|----------------|--------|
| 1 | Deep Mode claimed "OpenAI Profile Parser / Web Enrichment / searches company news", but it was a local string builder with `companyNews: []` and a fake 300 ms delay (and the app uses Anthropic) | Misleading UI destroys trust | Yes |
| 2 | Prompt had no system prompt, no grounding rule, no cliché ban; asked for 600 chars though connection notes are ~300; profile text unfenced (prompt-injection surface) | Generic, sometimes invented messages | Yes |
| 3 | Resume upload was stored (up to 4 MB) but never used | A feature that does nothing | Yes |
| 4 | No timeout/retry; raw API errors shown | Fragile on 429/5xx | Yes |
| 5 | Closing the popup (any click on the page) lost the in-flight message; edits weren't saved to Recent | Lost work | Yes |
| 6 | Free Mode said "I enjoyed your recent post…" though the scraper can't tell posts from reposts | Unverified claim | Yes |
| 7 | Privacy note said no data leaves the device, untrue in AI mode; extra `tabs`/`activeTab` permissions; content script on all of linkedin.com | Trust / least privilege | Yes |
| 8 | Scraper depends on English headings, 3-line grouping and hardcoded city names | Breaks on other locales/layouts | **No**: needs a design pass |
| 9 | No streaming; no one-click insert into LinkedIn's box | Slower perceived speed, extra steps | **No**: scope, and auto-inserting into LinkedIn adds ToS/automation risk |
| 10 | Scraping LinkedIn conflicts with LinkedIn's terms | Product/legal risk | **No**: flagged, not a code fix |

## What I changed (one commit each)
1. **Least-privilege manifest, dead code removed, accurate privacy copy.**
2. **AI quality:** structured system prompt (facts-only, one hook, no clichés, injection-safe), per-type character limits that are *enforced* (shorten pass + sentence-safe trim), a visible "Personalised using: …" line so users can verify the hook, retry with backoff + 30 s timeout + human-readable errors, resume summarised once and only the summary stored.
3. **Deep Mode made honest and real:** one AI call ranks the 3 best verifiable hooks with a reason; falls back to on-device rules and says so.
4. **Reliability:** results are saved by the service worker, so closing the popup never loses a message; the popup restores it; edits persist to Recent.

## Before / after (fill in from your own test run)
| Metric (10 profiles, same settings) | Before | After |
|---|---|---|
| Messages within the character limit | _x/10_ | _x/10_ |
| Messages containing a cliché opener | _x/10_ | _x/10_ |
| Claims not present in the profile | _x/10_ | _x/10_ |
| Message survives closing the popup | No | Yes |
| Permissions | 4 + all of linkedin.com | 2 + /in/* only |

## What I'd do next
- Stream tokens and show the message as it types (perceived speed).
- Harden the scraper (language-independent selectors, "what we couldn't read" indicator).
- An evaluation set of saved profiles with automatic checks for limit, clichés and grounding.
- Optional one-click *fill* (never send) into LinkedIn's note box, after a ToS review.

## Test it
`chrome://extensions` → Developer mode → Load unpacked → select this folder.
