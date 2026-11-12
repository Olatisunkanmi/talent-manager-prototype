# FRD: Campaign Post Insights

**PRD refs:** §6.10, §6.11, FR-33 to FR-38 · **Milestone:** 8

## Purpose
Attach the posts made for a campaign by link, pull their performance, and show it on the campaign, the talent and the company overview, so the agency can see how each campaign is performing, not only what it earned.

## Feasibility by platform (drives the design)
| Platform | What a link alone gives | What needs the talent's consent |
| --- | --- | --- |
| YouTube | Public views, likes, comments (API key) | Nothing; analytics such as watch time need consent and are out of scope |
| Instagram | Nothing usable (no metrics from a public link) | Business or Creator account connected once: reach, impressions, plays, likes, comments, saves, shares |
| TikTok | Nothing usable | Account connected once: views, likes, comments, shares (subject to platform app approval) |
| X | Paid API; not supported in v1 | n/a |
| Other | n/a | Manual entry only |

Because of this, every post has one of three **collection modes**: `api-public`, `api-connected`, or `manual`. Manual entry is always available as the fallback. Third-party scraping services are not used.

## Posts
- A **post** belongs to one campaign (and therefore one talent): platform, URL, post type (reel, story, video, image, other), posted date, optional label, and an optional **content style**.
- **Content style** is a free-text tag, for example "Money giveaway", "Quick tips (30s)", "Skit", "Vlog". The form suggests styles already used in the workspace so spellings stay consistent; a post with no style is grouped as "Untagged". It can be set when adding the link and edited later from any post row. Styles are compared across posts in the review (see "Quarter review").
- **Record-only use:** a post can be saved with just talent (via campaign), campaign, platform and link, with no figures at all. This is a supported use: the Insights page then works as a searchable record of every post the agency has made, so nobody has to scroll the talent's page to find a past post.
- Add by pasting the link on the campaign's **Posts** tab, or with **Add post links** on the Insights page (which first asks for the campaign; the talent follows from it). Several links at once, one per line. The app detects the platform and external id from the URL, fetches the title/posted date where the API allows, and picks the collection mode:
  - YouTube → `api-public`.
  - Instagram/TikTok → `api-connected` if the campaign's talent has a connected account on that platform **and** the post belongs to it; otherwise `manual` with a "Connect <talent>'s Instagram to pull numbers automatically" prompt.
  - Anything else → `manual`.
- A duplicate URL on the same campaign is rejected. A post whose account does not match the campaign's talent shows a warning (not a block).
- Posts are never hard-deleted; removing one marks it `removed` and excludes it from totals.

## Metric snapshots
- Each fetch or manual entry appends a **snapshot**: captured_at, views, reach, impressions, likes, comments, shares, saves (each nullable because platforms differ), `source` (api | manual). Snapshots are append-only; the **latest snapshot** is the current value and the series gives growth over time.
- Manual entry: the admin types the figures and the as-of date from the platform app or a screenshot (the screenshot is not stored).
- Metrics are shown with their as-of time on every screen ("as of 09 Nov 14:00").

## Refresh schedule
- A scheduled job refreshes `api-*` posts: **daily for the first 14 days** after posting, **weekly until 90 days**, then stops (a manual "Refresh" button still works, limited to once per hour per post).
- **Stories** disappear after 24 hours and their insights are only available during that window; the job captures them hourly for 24 hours, then stops.
- If a fetch fails, the post shows "last updated 3 days ago" and retries on the next run. A post that is deleted or private becomes `unavailable`: its last snapshot stays and it is flagged.

## Connected accounts (talent consent)
- A talent profile has a **Connected accounts** card: platform, handle, status (connected / expired / revoked), connected date.
- The admin sends the talent a one-time **connect link**. The talent opens it, signs in to the platform, grants read-only insights access, and sees a confirmation. This is a single-purpose consent page, not a talent portal or login.
- Tokens are encrypted at rest, refreshed before expiry (Instagram long-lived tokens last about 60 days), and can be disconnected by the admin or revoked by the talent. When a token expires, affected posts fall back to `manual` and the admin sees a "Reconnect" prompt.
- Only read access to insights is requested.

## Metric definitions
- **Views** use the platform's own definition (YouTube views, Instagram plays, TikTok views); they are not identical, so totals across platforms are labelled "views (mixed definitions)" and per-platform totals are shown first.
- **Engagements** = likes + comments + shares + saves (missing values count as 0).
- **Engagement rate** = engagements ÷ views, shown only for posts with views > 0.
- **Cost per 1,000 views (CPM)** = campaign invoiced (sent, non-void) ÷ total views × 1,000, shown only when views > 0 and labelled with the invoiced basis.
- Removed and unavailable-without-snapshot posts are excluded from totals; unavailable posts with a snapshot keep their last values.

## Linking performance to talents
Every campaign belongs to one talent, so every post's performance rolls up to that talent automatically; no separate linking step exists.
- A talent's figures are the sum of the latest snapshots of the non-removed posts across **all of their campaigns**.
- The Insights page shows **Performance by talent**: campaigns, posts, views, reach, likes, engagement rate and the talent's best content style, ranked by views. Each row opens that talent's Posts tab.
- The talent's **Posts tab** shows the same summary and review scoped to that talent only.
- Talent totals equal the sum of their campaigns' Posts totals for the same filters.
- If a campaign is reassigned to another talent, its posts move with it and past talent totals change accordingly.

## Date range
The Insights page filters posts by **posted date**: 7 days, 30 days, 90 days, this quarter, last quarter, this year, all time, or a custom from/to date (which also covers year-on-year comparisons by running two custom ranges). The active range is always printed under the buttons. Ranges apply to every figure on the page (totals, chart, review, talent ranking, post table, export). Figures remain each post's **latest snapshot**, not growth within the window; a post published in the range but refreshed later shows its current numbers.

## Quarter review
For the selected range (typically "Last quarter") the Insights page shows:
- **Content style comparison:** per style, posts, average views per post, reach, likes, comments and engagement rate, ranked by engagement rate. The top style is marked "best" when at least two styles are tagged. The purpose is to decide which style to make more of next quarter.
- **Reach by platform:** posts and total reach per platform, with a note of how many posts report reach (YouTube and TikTok public links do not).
- **Most liked / most commented:** top three posts each, showing campaign, platform and style.
- **Reach per post** is a column in the post table.

## Search
The post table has a search box matching talent name, campaign title, link, content style, post type and label. It composes with the platform filter and the date range.

## Export
**Export CSV** on the Insights page downloads the posts in the current view (range, platform and search applied): post, campaign, talent, platform, type, content style, posted date, views, reach, likes, comments, shares, saves, engagement rate, as-of date, source, plus a totals row. The filename carries the range (for example `post-insights_2026-07-01_to_2026-09-30.csv`). This lets the agency keep a cumulative quarter-end record outside the app. Branded or brand-facing reports remain out of scope.

## Where it appears
1. **Campaign page → Posts tab:** table of posts (platform, label, posted date, views, likes, comments, shares, saves, engagement rate, as-of), campaign totals, a views-over-time chart, top post, and "Add post links". The Overview tab gets a small summary (total views, engagement rate, posts tracked).
2. **Talent profile → Posts tab and Connected accounts card:** a summary (views, engagements, engagement rate, posts), the content style comparison, reach by platform and top posts for that talent, then posts across the talent's campaigns with the same columns.
3. **Insights page (Growth → Insights):** date range, export, search, quarter review, performance by talent, views over time, the post table and campaigns by views. This is the working page for quarter-end review.
4. **Company overview → new section "Content performance":** total views, engagements, engagement rate, posts tracked, views by platform, top 5 posts, and campaigns ranked by views and by CPM. It respects the date range (post date), talents and campaigns filters, and the compare-to option. Rows link to the post, campaign or talent.
5. **Campaigns table (overview section 11):** two new columns, Views and Engagement rate.

## Edge cases
- Post link malformed or unsupported platform: rejected with the list of supported platforms; "Add as manual" is offered.
- Same post added under two campaigns: allowed only with a confirmation (it will be counted in both).
- Metrics decrease between snapshots (platform recount): both snapshots are kept; the chart shows the real series.
- Campaign cancelled or completed: tracking continues until the 90-day cutoff; posts can still be added.
- Deactivated talent: posts keep refreshing while tokens are valid; connect links cannot be issued for new accounts.
- Rate limits: the job queues and spreads requests; the page never blocks on a platform call.
- Platform approval is pending (for example TikTok): those posts stay `manual` and the UI says why.

## Out of scope for v1
Automatic style detection from video content (styles are tagged by hand), growth measured inside a date window (figures are latest snapshots), follower growth, audience demographics, comment sentiment, paid-ads attribution, X/Twitter, third-party scrapers, sharing a branded performance report with the brand (a natural next step), and adding post links from WhatsApp.

## Acceptance criteria
- Choosing "Last quarter" shows only posts published in that calendar quarter, and Export CSV contains exactly those rows with a totals row equal to the on-screen totals.
- Tagging posts with styles produces a style table whose posts, likes and comments sum to the page totals (untagged posts appear as "Untagged").
- A talent's Posts tab totals equal the sum of their campaigns' Posts tab totals.
- A post saved with a link only (no snapshot) appears in search and in the CSV with blank figures and does not distort totals.
- Pasting a YouTube link on a campaign shows its views, likes and comments within one minute and refreshes daily without action.
- Pasting an Instagram link for a talent with a connected account shows reach, plays, likes, comments, saves and shares; for a talent without one it prompts to connect or enter manually.
- A manually entered snapshot appears in the totals with a "manual" marker and its as-of date.
- A campaign's Posts totals equal the sum of its non-removed posts' latest snapshots.
- The overview's Content performance figures equal the sum of the matching campaign pages for the same filters.
- An expired token turns affected posts to manual and shows a Reconnect prompt; no post silently shows stale numbers without an as-of time.
- A deleted post is marked unavailable and keeps its last snapshot.
