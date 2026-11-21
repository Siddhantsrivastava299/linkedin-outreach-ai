# LinkedIn Outreach AI: improvements write-up

**Brief:** improve the existing Chrome extension without rebuilding it. What to improve was my call.

**Approach:** I used the original (v3.1.0) as a real user, read all the code, then picked the problems that most damage user trust and message quality. I kept the architecture and file structure as they were.

> **What I tested, and what I didn't.** I tested everything end to end in **Free Mode (no API key)**. The AI-mode changes (the new prompt, enforced length limits on AI output, the AI hook-ranker, resume summarisation, API retry/timeout) are implemented and syntax-checked, but **I have not run them against the live API**. I mark each one below so nothing is overclaimed.

## What I found
| # | Problem in v3.1.0 | Why it matters | Status |
|---|---|---|---|
| 1 | Deep Mode claimed "OpenAI Profile Parser" and "Web Enrichment: searches company news". It was local string building with `companyNews: []` and a fake 300 ms delay (and the app uses Anthropic, not OpenAI) | A misleading UI in a tool that writes in the user's name | **Fixed and tested** |
| 2 | Free template claimed "I enjoyed your recent post about…" even though the scraper cannot tell a post from a repost or reaction | An unverifiable claim sent under the user's name | **Fixed and tested** |
| 3 | No visible length limit, although LinkedIn caps connection notes (~300 chars) | Messages can be silently too long | **Fixed and tested** (live counter, red when over, bio auto-dropped if needed) |
| 4 | Closing the popup (any click on the page) lost the message; edits were not saved to Recent | Lost work | **Fixed and tested** (message restored on reopen; edits persist) |
| 5 | Resume upload said "saved ✓" but was never used anywhere | A feature that silently does nothing | **Partly fixed, tested** (now says clearly it needs an API key). **Resume summarising itself is untested** |
| 6 | Privacy note said no data leaves the device; untrue in AI mode. Extra `tabs`/`activeTab` permissions; content script on all of linkedin.com | Trust and least privilege | **Fixed and tested** (2 permissions, profile pages only, accurate copy) |
| 7 | AI prompt: no system prompt, no grounding rule, no cliché ban, asked for 600 chars, scraped text unfenced (prompt-injection surface) | Generic or invented messages | **Fixed in code, not tested** (needs API key) |
| 8 | No timeout/retry on API calls; raw API errors shown | Fragile on 429/5xx | **Fixed in code, not tested** |
| 9 | Deep Mode's second step did nothing real | Feature was cosmetic | **Real AI hook-ranker written, not tested**. In Free Mode it shows on-device hooks and is labelled as such |
| 10 | Scraper depends on English headings, 3-line grouping and hardcoded city names | Breaks on other locales/layouts | **Not fixed.** Needs a design pass |
| 11 | No streaming; no one-click insert into LinkedIn's note box | Slower perceived speed, extra steps | **Not fixed.** Out of scope; auto-inserting into LinkedIn adds terms-of-service/automation risk |
| 12 | Scraping LinkedIn conflicts with LinkedIn's terms | Product/legal risk | **Not fixed.** Flagged, not a code fix |

## What I changed (one commit each)
1. **Least-privilege manifest, dead code removed, accurate privacy copy.** *(tested)*
2. **AI quality:** structured system prompt (facts only, one hook, no clichés, injection-safe), per-type character limits, a "Personalised using: …" line, retry with backoff and 30 s timeout, resume summarised once. *(counter, note line and Free Mode limits tested; AI parts untested)*
3. **Deep Mode made honest:** removed the false "OpenAI" claims, relabelled the steps, added an AI hook-ranker that falls back to on-device rules and says which ran. *(labels and on-device path tested; AI path untested)*
4. **Reliability:** results are saved by the service worker, so closing the popup does not lose a generated message; the popup restores it; edits persist to Recent. *(tested)*

## Before / after (Free Mode, N profiles, Networking / Neutral)
| Metric | Before (v3.1.0) | After (v3.2.0) |
|---|---|---|
| Messages within the 300-character limit | _x/N_ | _x/N_ |
| Messages with a cliché opener ("came across your profile", "impressed by") | _x/N_ | _x/N_ |
| Messages claiming a "recent post" I could not verify | _x/N_ | _x/N_ |
| Message survives closing the popup | No | Yes |
| Resume upload | Says "saved ✓", never used | Says clearly it needs an API key |
| Permissions | 4, content script on all of linkedin.com | 2, profile pages only |

_Small sample, same profiles, message type and tone in both versions._

## Known limitations of this submission
- **Deep Mode in Free Mode** shows on-device hooks but the free template does not use them, so it adds little without an API key. It is labelled honestly, but it is a real gap.
- The page **scrape still runs through the popup**; only the generation step survives closing it.
- The 300/500 character limits are my estimates for LinkedIn's caps and may differ by account type (constant: `CHAR_LIMITS` in `background.js` and `popup.js`).

## What I'd do next
1. Test the AI path end to end with an API key and measure it on a saved set of profiles (limit pass rate, clichés, claims not in the profile).
2. Stream tokens so the message appears as it is written.
3. Move the scrape into the background script, and harden it (language-independent selectors, a "what we could not read" indicator).
4. Let the free template use the hook chosen in Deep Mode.
5. Optional one-click *fill* (never send) into LinkedIn's note box, after a terms-of-service review.

## Try it
`chrome://extensions` → Developer mode → Load unpacked → select this folder. Works without any API key.
