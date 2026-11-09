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
- A **post** belongs to one campaign (and therefore one talent): platform, URL, post type (reel, story, video, image, other), posted date, optional label.
- Add by pasting the link on the campaign's **Posts** tab (several at once, one per line). The app detects the platform and external id from the URL, fetches the title/posted date where the API allows, and picks the collection mode:
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

## Where it appears
1. **Campaign page → Posts tab:** table of posts (platform, label, posted date, views, likes, comments, shares, saves, engagement rate, as-of), campaign totals, a views-over-time chart, top post, and "Add post links". The Overview tab gets a small summary (total views, engagement rate, posts tracked).
2. **Talent profile → Posts tab and Connected accounts card:** posts across the talent's campaigns, with the same columns.
3. **Company overview → new section "Content performance":** total views, engagements, engagement rate, posts tracked, views by platform, top 5 posts, and campaigns ranked by views and by CPM. It respects the date range (post date), talents and campaigns filters, and the compare-to option. Rows link to the post, campaign or talent.
4. **Campaigns table (overview section 11):** two new columns, Views and Engagement rate.

## Edge cases
- Post link malformed or unsupported platform: rejected with the list of supported platforms; "Add as manual" is offered.
- Same post added under two campaigns: allowed only with a confirmation (it will be counted in both).
- Metrics decrease between snapshots (platform recount): both snapshots are kept; the chart shows the real series.
- Campaign cancelled or completed: tracking continues until the 90-day cutoff; posts can still be added.
- Deactivated talent: posts keep refreshing while tokens are valid; connect links cannot be issued for new accounts.
- Rate limits: the job queues and spreads requests; the page never blocks on a platform call.
- Platform approval is pending (for example TikTok): those posts stay `manual` and the UI says why.

## Out of scope for v1
Follower growth, audience demographics, comment sentiment, paid-ads attribution, X/Twitter, third-party scrapers, sharing a branded performance report with the brand (a natural next step), and adding post links from WhatsApp.

## Acceptance criteria
- Pasting a YouTube link on a campaign shows its views, likes and comments within one minute and refreshes daily without action.
- Pasting an Instagram link for a talent with a connected account shows reach, plays, likes, comments, saves and shares; for a talent without one it prompts to connect or enter manually.
- A manually entered snapshot appears in the totals with a "manual" marker and its as-of date.
- A campaign's Posts totals equal the sum of its non-removed posts' latest snapshots.
- The overview's Content performance figures equal the sum of the matching campaign pages for the same filters.
- An expired token turns affected posts to manual and shows a Reconnect prompt; no post silently shows stale numbers without an as-of time.
- A deleted post is marked unavailable and keeps its last snapshot.
