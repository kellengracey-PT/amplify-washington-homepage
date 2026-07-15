# Live & Latest — how it works (Amplify Washington)

This describes the Phase 1 "Live & Latest" foundation for the community
show. Sports Live & Latest, the highlight carousel, and the sports schedule
are later phases and aren't implemented yet — this doc will grow as those
land.

## Where configuration lives

Everything is in one object near the bottom of `index.html`:

```js
const SITE_CONFIG = { ... };
```

There is no build step and no separate config file — this repo is a single
static `index.html` plus images, deployed as-is by GitHub Pages from the
repo root on `main`. Edit `SITE_CONFIG` directly and commit.

| Field | Purpose |
|---|---|
| `youtubeChannelUrl` | The public channel link used in every "Watch on YouTube" / follow button |
| `youtubeChannelId` | **Blank — not yet confirmed.** Set this once the real channel ID is verified in YouTube Studio |
| `facebookUrl` | Facebook page link |
| `resolverUrl` | Blank today. Once the live-data resolver (see below) is deployed, set this to its base URL and the site switches from static fallback to live data automatically — no other code changes |
| `fallbackCommunityVideoId` | Manually-set "most recent episode" video ID, used only when the resolver is unavailable or not configured |
| `communityPlaylistId` | A YouTube playlist ID. When set, powers the Recent Episodes fallback embed (see below) |
| `recentCommunityShows` | Manually-curated array of `{ videoId, title, date }`, max 5 used. Optional — see Recent Episodes below |
| `sportsChannelUrl`, `fallbackSportsVideoId`, `sportsPlaylistId`, `sportsHighlightsPlaylistId` | Sports fields — present for Phase 2, not yet wired to any new logic |
| `appUrl` | Amplify Media Platform app link. Blank until the app exists |
| `communityScheduleText` | Short phrase used in the static fallback message once the show is airing (e.g. "every weekday") |
| `communityLaunchText` | **Currently set** — daily shows begin the week of July 20, 2026, and this message takes priority over the generic fallback until then. Clear it back to `""` once the show has actually launched, so the site switches to the normal "most recent episode" framing |
| `liveRefreshSeconds` | How often the page re-polls the resolver. Has no effect while `resolverUrl` is blank |

All blank fields have a `// TODO` comment explaining what real value belongs
there. Nothing in this file is a placeholder that looks like real data —
blank means blank.

## Important: pre-launch state

As of this writing, Amplify Washington's daily show has not started yet
(launch is the week of July 20, 2026). Until then:

- The Watch section will always show the static fallback ("Watch & Follow"
  badge + the `communityLaunchText` message), since there's no episode to
  resolve live/upcoming/replay status for
- Once the first episode airs, set `fallbackCommunityVideoId` (or deploy
  the resolver) and clear `communityLaunchText` back to `""`

## How current live content is resolved

Priority order, implemented in `initCommunityLiveAndLatest()`:

1. If `SITE_CONFIG.resolverUrl` is set, fetch it (4s timeout). A successful
   response's `community.status` drives the UI:
   - `live` → "Live Now" badge, primary player loads the live video
   - `upcoming` → "Next Broadcast" badge, scheduled time shown in Central
     Time, **no player embedded** (YouTube doesn't reliably support
     playback for a video that hasn't started)
   - `replay` → "Most Recent Show" badge, primary player loads that video
   - `unavailable`, a failed fetch, a timeout, or no `resolverUrl` at all →
     falls through to static fallback
2. Static fallback: badge reads "Most Recent Show" if
   `fallbackCommunityVideoId` is set, otherwise "Watch & Follow" with a
   designed empty state and a YouTube channel link. **This never shows
   "Live Now"** — that label only ever comes from a resolver's confirmed
   `status: "live"`.

The resolver itself doesn't exist yet. See
`/Amplify_Live_Data/README.md` (sibling directory, not part of this repo)
for the planned Cloudflare Worker architecture and what's needed to deploy
it — a Cloudflare account, a YouTube Data API key stored as an encrypted
Worker secret, and confirmed channel IDs. None of that has been created.

## Recent Episodes

`renderRecentEpisodes()` picks its data source in this order:

1. The resolver's `recentCommunity` array, if present and non-empty
2. `SITE_CONFIG.recentCommunityShows`, if manually curated
3. Neither → falls back to a single YouTube playlist embed
   (`communityPlaylistId`) showing the standard playlist UI
4. No playlist ID either → a designed empty state linking to the channel

When per-episode data is available, up to 5 episodes render as clickable
thumbnail cards (using YouTube's public thumbnail CDN, no API call — e.g.
`https://i.ytimg.com/vi/VIDEO_ID/hqdefault.jpg`). Clicking a card swaps the
primary player rather than opening its own embed, so the page never loads
more than one YouTube iframe at a time.

## Caching

Not applicable yet on the site side — there's no resolver to cache
responses from. When the Worker exists, caching happens server-side (see
`/Amplify_Live_Data/README.md`): a Cron Trigger polls YouTube on a fixed
schedule and writes to Workers KV, and the public `/api/washington` endpoint
just serves that cached JSON. Visitor traffic never triggers a YouTube API
call.

## Fallback behavior

The site is designed to degrade gracefully at every layer:

- JavaScript disabled → a `<noscript>` block inside the video player
  container links to the YouTube channel instead of an empty box
- Resolver not configured (`resolverUrl` blank) → static fallback, instant,
  no network request attempted
- Resolver configured but unreachable/slow → same static fallback after a
  4-second timeout
- No fallback video ID either → designed empty state, not a blank box or
  infinite spinner

## Testing locally

No build step. From this directory:

```bash
npx http-server -p 8080 -c-1
```

Then open `http://localhost:8080/index.html`. To exercise the live/upcoming/
replay states without a real resolver, open the browser console and call
the rendering functions directly, e.g.:

```js
renderCommunityFromResolver({ status: 'live', videoId: 'dQw4w9WgXcQ', title: 'Test', actualStart: new Date().toISOString(), youtubeUrl: '...' });
renderRecentEpisodes([{ videoId: 'dQw4w9WgXcQ', title: 'Episode 1', date: 'July 14, 2026' }]);
```

## Deploying safely

This repo deploys via GitHub Pages from the repo root on `main` — no
workflow file, no `.nojekyll`, no build step. Merging to `main` is live
within a minute or two. Always work on a feature branch, test locally
first, and only merge to `main` when explicitly asked to.
