# Yoto Podcast Manager — Product Spec

Nov 24, 2026 · Paul Cho

## Overview

A small, self-hosted web app on the Mac mini that loads podcast episodes onto Make Your Own (MYO) Yoto cards. A parent can change any card's podcast in under 30 seconds from a phone.

**Problem.** Our 5-year-old listens to a handful of kids' podcasts, often several episodes in a row, including ones he has already heard. Today he asks a parent, who picks the show in Spotify on a phone and casts it to a Google Home. He can't do any of it himself.

**Goal.** Each MYO card stands for one podcast. He picks a card and plays it on his Yoto player himself. When he wants a different show or a different batch of episodes, he asks a parent, who makes the change in this app. The physical card stays the same; only its contents change.

**Key design insight.** The Yoto API lets a card's playlist hold *streaming* tracks: a track whose `trackUrl` is a plain web URL with `type: "stream"` ([Yoto: streaming tracks](https://yoto.dev/myo/streaming-tracks/)). Podcast RSS feeds already publish direct MP3 URLs. So the app never downloads or uploads audio. It rewrites a card's playlist JSON to point at episode URLs, and the player streams them straight from the podcast host.

**Users.** One parent (the admin), using a phone or laptop on the home network or Tailscale. The child never uses the web app; his interface is the physical cards.

## Scope

v1 is a parent-only admin tool with three jobs: remember feeds, remember cards, and choose what each card plays.

**In scope (v1)**

- Add a podcast by RSS URL and keep it permanently, with its episode list cached and refreshed.
- Register MYO cards (import an existing MYO playlist or create a new one) and keep them permanently, with a friendly name and color.
- Assign a feed to a card with an episode-selection mode (latest N, a numbered batch, random N, or hand-picked), then push it to Yoto.
- One-tap "Next batch" to advance a card through a show's back catalog.
- A daily job that re-pushes "latest N" cards when a feed publishes new episodes.
- Single Yoto account sign-in via OAuth, with tokens stored locally.

**Out of scope (v1)**

- Any child-facing UI.
- Downloading, transcoding, or uploading audio to Yoto (streaming only).
- Multiple users, roles, or app-level login (network access is the boundary).
- Spotify integration (Spotify's API doesn't expose podcast audio).
- Linking a physical card to a playlist via API. Yoto documents this only through the Yoto app or player ([Yoto: uploading to cards](https://yoto.dev/myo/uploading-to-cards/)), so it stays a one-time manual step per card.
- Custom pixel-art icons per episode (use one default icon; see Open questions).

**Nice to have (post-v1)**

- Paste an Apple Podcasts or Spotify show link and resolve it to an RSS URL.
- Upload the podcast's artwork as the card's 16×16 icon.
- A "mix" card that pulls episodes from several feeds.

## Functional requirements

Everything the parent sets up persists in SQLite and survives restarts; nothing is re-entered.

### Feeds

- **FR-F1 Add feed.** Paste an RSS URL. The server fetches and parses it, then shows title, artwork, and episode count before saving. Reject non-RSS or unreachable URLs with a clear message.
- **FR-F2 Persist.** A saved feed stays until deleted. The app stores the URL, title, author, artwork URL, and the full parsed episode list.
- **FR-F3 Episodes.** Each episode stores GUID, title, publish date, duration (if given), enclosure URL, MIME type, and a stable ordinal (1 = oldest), computed from publish date, then GUID as a tiebreaker.
- **FR-F4 Refresh.** Refresh all feeds every 6 hours, plus on demand. Use `ETag` / `Last-Modified` when the host supports them. New episodes get the next ordinals; existing ordinals never shift.
- **FR-F5 Delete.** Deleting a feed that is assigned to a card is blocked with a message naming the card(s).

### Cards

- **FR-C1 Import.** List the account's existing MYO playlists from Yoto (`GET /content/mine`) and let the parent register any of them.
- **FR-C2 Create.** Create a new empty playlist on Yoto (`POST /content` without a `cardId`). Show instructions to link a physical MYO card to it once in the Yoto app.
- **FR-C3 Persist.** Store the Yoto `cardId` (5 characters), a nickname ("Wild Kratts card"), a color tag matching the physical sticker, and a `linked` checkbox the parent ticks after linking.
- **FR-C4 Unregister.** Remove a card from the app without deleting the Yoto playlist (an explicit "also delete on Yoto" option is allowed).

### Assignments (what plays on each card)

- **FR-A1 Assign.** Pick a card, a feed, and a selection mode, preview the resulting episode list, and press **Push to card**.
- **FR-A2 Selection modes.**

| Mode | Parameters | Result |
| --- | --- | --- |
| Latest | N (default 15) | The N newest episodes, newest first |
| Batch | size N, batch index k | Ordinals N·(k−1)+1 … N·k, oldest first |
| Range | start ordinal, end ordinal | That range, oldest first |
| Random | N, seed | N episodes drawn from the whole catalog, stable until re-rolled |
| Picked | list of episode GUIDs | Exactly those, in chosen order |

- **FR-A3 Next / previous batch.** On a Batch card, one tap moves k forward or back and pushes immediately. At the end of the catalog it wraps to batch 1.
- **FR-A4 Re-roll.** On a Random card, one tap picks a new seed and pushes.
- **FR-A5 Auto-update.** Cards in Latest mode re-push automatically after a refresh finds new episodes. Other modes never change unless the parent acts.
- **FR-A6 Limits.** Cap any assignment at 100 episodes (the MYO track limit). Default N to 15 so the dial stays manageable for a 5-year-old.
- **FR-A7 Push result.** Show success or the Yoto error verbatim. Every push is written to a log with time, card, mode, episode count, and outcome.

### Dashboard

- **FR-D1** The home screen shows one tile per card: color, nickname, podcast artwork, mode summary ("Batch 3 of 9 · eps 31–45"), last pushed time, and quick actions (Next batch, Change podcast).

## Yoto API integration

The app needs only two Yoto API calls in steady state: list my playlists and create-or-update a playlist. Auth is OAuth 2.0 with PKCE.

### Auth

- Register an app at [dashboard.yoto.dev](https://dashboard.yoto.dev/) to get a `clientId`. Register the callback `http://localhost:<port>/auth/callback`, plus the Tailscale hostname if it differs.
- Use the Authorization Code flow with PKCE: redirect to `https://login.yotoplay.com/authorize` with `audience=https://api.yotoplay.com` and `offline_access` in scope, then exchange the code at `https://login.yotoplay.com/oauth/token` ([Yoto: headless/CLI auth](https://yoto.dev/authentication/headless-cli-auth/)). Don't build on the device-code flow; Yoto publishes a migration guide away from it ([Yoto: device-code migration](https://yoto.dev/authentication/device-code-migration/)).
- **Refresh tokens are single-use.** Persist the new refresh token from every refresh response atomically, in the same DB transaction, before using the new access token. Refresh lazily on a 401 or when expiry is less than 5 minutes away, behind a mutex so two requests never refresh at once.
- Scopes: the docs example uses `family:library:view offline_access`. Writing content needs a write-capable scope; confirm the exact name in the dashboard (see Open questions).

### Endpoints used

| Purpose | Call | Notes |
| --- | --- | --- |
| List MYO playlists | `GET https://api.yotoplay.com/content/mine` | Used by card import |
| Read one playlist | `GET /content/{cardId}` | Verify a push and show current contents ([docs](https://yoto.dev/api/getcontent)) |
| Create playlist | `POST /content` without `cardId` | Returns `card.cardId` ([docs](https://yoto.dev/api/createorupdatecontent)) |
| Update playlist | `POST /content` with `cardId` | Full replace of `content.chapters` |
| Delete playlist | `DELETE /content/{cardId}` | Only for the explicit "also delete on Yoto" option |

### Playlist payload

One chapter per episode, one streaming track per chapter, so the player's skip control moves episode by episode. `overlayLabel` is the 1-based position on the card, not the episode ordinal, so it stays small (1–15).

```json
{
  "cardId": "31yYU",
  "title": "Wild Kratts card",
  "content": {
    "playbackType": "linear",
    "config": { "resumeTimeout": 2592000 },
    "chapters": [
      {
        "key": "01",
        "title": "Ep 31: Octopus Escape",
        "overlayLabel": "1",
        "display": { "icon16x16": "yoto:#<defaultIconId>" },
        "tracks": [
          {
            "key": "01",
            "title": "Ep 31: Octopus Escape",
            "type": "stream",
            "format": "mp3",
            "trackUrl": "https://<enclosure url from RSS>",
            "overlayLabel": "1",
            "display": { "icon16x16": "yoto:#<defaultIconId>" }
          }
        ]
      }
    ]
  },
  "metadata": { "description": "Managed by Yoto Podcast Manager · Batch 3 (eps 31–45)" }
}
```

- `type: "stream"` plus a regular web `trackUrl` is the documented streaming pattern ([Yoto: streaming tracks](https://yoto.dev/myo/streaming-tracks/); [card content schema](https://yoto.dev/reference/card-content-schema/)).
- `resumeTimeout` of 2592000 s (30 days) matches the API's default in its example response. Keep it so he resumes mid-episode.
- Set `format` from the enclosure MIME type (`audio/mpeg` → `mp3`, `audio/mp4`/`audio/x-m4a` → `aac`; verify accepted values). Skip episodes with other types and report them in the preview.
- Keep the push builder a pure function `(card, episodes) → payload` with snapshot tests.

### Library choice

Call the REST API directly with `fetch`; the surface is five calls. If preferred, the official `@yotoplay/yoto-sdk` exists, as does the unofficial `yoto-nodejs-client`, which handles token refresh.

## Architecture and stack

One TypeScript monorepo, one process, one SQLite file. The Mac mini only rewrites playlists; audio never passes through it.

```
 Parent (phone/laptop)
        │  HTTPS over LAN / Tailscale
        ▼
 ┌─────────── Mac mini ───────────┐        fetch RSS      ┌───────────────┐
 │  React UI (Vite)               │ ────────────────────▶ │ Podcast hosts │
 │        │                       │                       │ RSS + MP3     │
 │        ▼                       │                       └───────▲───────┘
 │  Node API (Hono)               │   POST /content       ┌───────┴───────┐
 │  selection · push builder ·    │ ────────────────────▶ │   Yoto API    │
 │  cron                          │                       └───────┬───────┘
 │        │                       │                               │ playlist sync
 │        ▼                       │                               ▼
 │  SQLite · data/app.db          │       Child ──taps card──▶ Yoto player
 └────────────────────────────────┘                     (streams MP3 directly
                                                          from podcast hosts)
```

The parent edits a card in the UI. The API selects episodes, builds the playlist JSON, and pushes it to Yoto. The player syncs the playlist and streams each MP3 straight from the podcast host.

| Layer | Choice | Why |
| --- | --- | --- |
| Runtime | Node 22 LTS, TypeScript (strict) | Matches the preferred stack; good RSS and SQLite libraries |
| Server | Hono on `@hono/node-server` | Tiny, typed routes, serves the built UI too |
| Database | SQLite via `better-sqlite3` + Drizzle ORM | Zero-ops persistence, file-based backups |
| Validation | zod, shared between client and server | One source of truth for request types |
| RSS parsing | `rss-parser` (or `fast-xml-parser`) | Handles iTunes namespace fields like duration |
| Scheduler | `croner` in-process | No external cron to manage |
| Frontend | React + Vite + TanStack Query, Tailwind | Fast to build, mobile-friendly |
| Tests | Vitest | Unit tests for selection and payload builder |
| Process manager | launchd agent | Native on macOS, restarts on crash and boot |

Repo layout:

```
yoto-podcasts/
  server/   src/{routes,yoto,feeds,select,db,jobs}  drizzle/
  web/      src/{pages,components,api}
  shared/   zod schemas + types
  data/     app.db  (gitignored)
  ops/      com.paul.yoto-podcasts.plist, backup.sh
```

## Data model

Six tables in a single SQLite file (`data/app.db`) cover everything. Use WAL mode, Drizzle migrations, and ISO-8601 UTC timestamps.

| Table | Key columns | Notes |
| --- | --- | --- |
| `feeds` | `id`, `url` (unique), `title`, `author`, `artwork_url`, `etag`, `last_modified`, `last_fetched_at`, `last_error`, `created_at` | One row per saved podcast |
| `episodes` | `id`, `feed_id` → feeds, `guid`, `ordinal`, `title`, `published_at`, `duration_s`, `enclosure_url`, `mime_type`, `removed_at` | Unique (`feed_id`, `guid`). `ordinal` is assigned once, never reused. Episodes dropped from the feed get `removed_at`, not deleted |
| `cards` | `id`, `yoto_card_id` (unique, 5 chars), `nickname`, `color`, `linked` (bool), `created_at` | One row per physical MYO card |
| `assignments` | `card_id` (PK) → cards, `feed_id` → feeds, `mode` (`latest`/`batch`/`range`/`random`/`picked`), `params` (JSON), `pushed_episode_guids` (JSON), `pushed_at` | One current assignment per card; `params` holds N, k, range, seed, or GUID list |
| `push_log` | `id`, `card_id`, `feed_id`, `mode`, `params`, `episode_count`, `status` (`ok`/`error`), `error`, `created_at` | Append-only audit trail; keep the last 500 rows |
| `auth` | single row: `refresh_token`, `access_token`, `expires_at`, `updated_at` | Written in one transaction on every refresh |

Params shapes, validated with zod:

```ts
type Params =
  | { mode: "latest"; n: number }
  | { mode: "batch"; n: number; k: number }
  | { mode: "range"; start: number; end: number }
  | { mode: "random"; n: number; seed: number }
  | { mode: "picked"; guids: string[] };
```

Episode selection is a pure function `select(episodes, params) → Episode[]` with unit tests per mode, including empty feeds, N larger than the catalog, and batch wraparound.

## Web UI

Five screens, mobile-first. The common task ("he wants the next batch of Wild Kratts") must take two taps from the home screen.

1. **Dashboard** (`/`). A grid of card tiles in each card's sticker color: nickname, podcast artwork, mode summary, last pushed time. Primary action per tile: **Next batch** (Batch), **Re-roll** (Random), or **Refresh** (Latest). Secondary action: **Change**.
2. **Card editor** (`/cards/:id`). Choose a feed from saved feeds, choose a mode and its parameters, see a live preview of the exact episode list (position, title, date, duration), then **Push to card**. Also: rename, color, linked checkbox, unregister, and the last 10 push-log entries for this card.
3. **Feeds** (`/feeds`). Saved feeds with artwork, episode count, last refreshed time, any error, and the cards using each feed. **Add feed** opens a form that previews the parsed feed before saving. **Refresh now** per feed and for all.
4. **Feed detail** (`/feeds/:id`). A searchable episode list with ordinals. Multi-select episodes and "Put these on… [card]" to create a Picked assignment.
5. **Settings** (`/settings`). Yoto connection status (signed in as, token expiry), **Connect / Reconnect Yoto**, **Import cards from Yoto**, **Create new card**, and the global push log.

UI rules:

- Buttons that push to Yoto show a spinner, then a toast with the result. Never fail silently.
- Large touch targets (44 px minimum); it will mostly be used one-handed on a phone.
- No client-side state worth keeping: the server is the source of truth, and the client refetches after every mutation.

## Backend API

A JSON API under `/api`, consumed only by the bundled UI. Request and response bodies are validated with zod and shared with the client as types.

| Method | Route | Does |
| --- | --- | --- |
| GET | `/api/feeds` | List feeds with episode counts and usage |
| POST | `/api/feeds/preview` | `{url}` → parsed feed, not saved |
| POST | `/api/feeds` | `{url}` → save and fetch the feed |
| GET | `/api/feeds/:id/episodes` | Episodes, `?q=` search |
| POST | `/api/feeds/:id/refresh` | Refresh one feed now |
| POST | `/api/feeds/refresh` | Refresh all feeds now |
| DELETE | `/api/feeds/:id` | 409 if assigned to a card |
| GET | `/api/cards` | Cards with their assignment and last push |
| GET | `/api/yoto/playlists` | Proxy `GET /content/mine` for import |
| POST | `/api/cards/import` | `{yotoCardId, nickname, color}` |
| POST | `/api/cards` | Create a new Yoto playlist and register it |
| PATCH | `/api/cards/:id` | Nickname, color, linked |
| DELETE | `/api/cards/:id` | `?deleteOnYoto=true` optional |
| POST | `/api/cards/:id/preview` | `{feedId, params}` → episode list, no push |
| PUT | `/api/cards/:id/assignment` | `{feedId, params}` → save and push |
| POST | `/api/cards/:id/next` | Batch k+1 (or re-roll for Random), push |
| POST | `/api/cards/:id/prev` | Batch k−1, push |
| POST | `/api/cards/:id/push` | Re-push the current assignment |
| GET | `/api/log` | Push log, `?cardId=` filter |
| GET | `/auth/login` | Start PKCE, redirect to Yoto |
| GET | `/auth/callback` | Exchange the code, store tokens, redirect to `/settings` |
| GET | `/api/auth/status` | Connected?, expiry |
| GET | `/healthz` | Liveness for launchd and monitoring |

The assignment is saved only if the Yoto push succeeds, so the DB never claims a card holds something it doesn't. Pushes for the same card are serialized.

## Deployment, access, security

One Node process on the Mac mini, reachable only from the LAN and the tailnet, kept alive by launchd.

- **Process.** `npm run build` produces the server bundle plus the static UI. A launchd agent (`~/Library/LaunchAgents/com.paul.yoto-podcasts.plist`) runs it with `KeepAlive` and `RunAtLoad`, and logs to `~/Library/Logs/yoto-podcasts/`.
- **Binding.** Listen on `127.0.0.1:8787` by default. Expose it on the tailnet with `tailscale serve` (HTTPS on the Mac mini's tailnet name). An optional `BIND_LAN=true` binds `0.0.0.0` for LAN-only access. Never expose it to the public internet.
- **OAuth callback.** The redirect URI must exactly match what is registered on the Yoto dashboard. Support `PUBLIC_BASE_URL` (for example `https://mac-mini.<tailnet>.ts.net`) and register both that and `http://localhost:8787/auth/callback`.
- **No app login in v1.** Network reachability is the access control. Leave a middleware hook so basic auth can be added with one env var (`ADMIN_PASSWORD`) later.
- **Secrets.** `.env` holds `YOTO_CLIENT_ID` (and nothing else secret; PKCE needs no client secret). Tokens live only in SQLite. Set `data/` to mode 700 and add it to `.gitignore`.
- **Backups.** A nightly `sqlite3 .backup` to the Pi NAS share, keeping 14 copies. The DB is tiny, and everything except tokens can be rebuilt from feed URLs and card IDs.
- **Outbound only.** The server calls podcast hosts and Yoto. The Yoto player streams audio directly from podcast hosts, never through the Mac mini, so the Mac mini being asleep or off doesn't stop playback.
- **Scheduling.** An in-process scheduler (`croner`): feed refresh every 6 hours, then auto-push for Latest-mode cards with changes. Set the Mac mini to not sleep, or accept that jobs run on wake.

## Milestones and acceptance criteria

Build in five milestones, in order. M0 is a throwaway spike that de-risks the one assumption everything rests on: a podcast MP3 URL plays as a streaming track on a real card.

1. **M0 · Spike (script, no UI).** A CLI script runs the PKCE login, stores tokens, and pushes three episodes of one real feed as stream tracks to one existing MYO card.
   - [ ] The card plays all three episodes on the physical Yoto player.
   - [ ] Skip moves between episodes; position resumes after removing and reinserting the card.
   - [ ] A second push with different episodes changes what the card plays, without relinking.
   - [ ] Record how long the player takes to pick up an update, and whether it needs the card reinserted.
2. **M1 · Skeleton + feeds.** Server, DB, migrations, launchd, and the Feeds and Feed detail screens.
   - [ ] Adding a feed URL previews it, then persists it across restarts.
   - [ ] A refresh adds new episodes with the next ordinals and leaves existing ordinals unchanged.
3. **M2 · Yoto connect + cards.** OAuth in the web app, import, create, and the cards list.
   - [ ] Connect works over both localhost and the Tailscale URL.
   - [ ] Restarting the server and waiting past access-token expiry still allows a push (refresh-token rotation works).
   - [ ] Imported cards show their current Yoto title.
4. **M3 · Assignments.** The Card editor, all five modes, preview, push, Next/Prev, and the push log.
   - [ ] Preview and the pushed playlist match exactly (verified with `GET /content/{cardId}`).
   - [ ] Next batch from the dashboard takes two taps and plays the new batch on the player.
   - [ ] A failed push leaves the old assignment in the DB and shows the Yoto error.
5. **M4 · Automation + polish.** Scheduled refresh, Latest auto-push, backups, and mobile layout.
   - [ ] A new episode on a Latest feed appears on its card within 6 hours, untouched.
   - [ ] The dashboard is usable one-handed on an iPhone.

Testing: unit tests for `select()` and the payload builder; an integration test against a recorded RSS fixture; Yoto calls mocked behind one `YotoClient` interface.

## Open questions and risks

The two biggest risks are whether podcast-host URLs stream cleanly on the player and how fast the player picks up changes. M0 answers both before any UI is built.

| # | Question | Why it matters | How to resolve |
| --- | --- | --- | --- |
| 1 | Do podcast enclosure URLs with redirect chains (Podtrac, Chartable, and similar trackers) stream on the player? | Most kids' podcasts use tracking redirects | M0. If not, resolve redirects server-side at push time and store the final URL (it may expire; then re-resolve during refresh) |
| 2 | How quickly does a player see an updated playlist? Does it need Wi-Fi plus a card reinsert? | Sets the parent's expectations after Next batch | M0 timing test. Show a hint in the UI if a reinsert is needed |
| 3 | Exact OAuth scope name for writing content | Pushes fail without it | Check dashboard.yoto.dev during app registration |
| 4 | Accepted `format` values for stream tracks (mp3 only, or aac/m4a too?) | Some feeds publish m4a | Test with one m4a feed in M0; until then, filter to mp3 |
| 5 | Does the player stream offline? | No; stream tracks need Wi-Fi | Accept for v1. For car trips, a post-v1 "download mode" could use the upload/transcode API instead |
| 6 | Default 16×16 icon ID to use | Required in `display.icon16x16` | Pick one from Yoto's public icon library or upload one custom icon once and store its ID in settings |
| 7 | Rate limits on `POST /content` | Undocumented on the page used | Serialize pushes, back off on 429, log it |
| 8 | Does a content update reset resume position? | He may lose his place mid-episode | Observe in M0. Acceptable for v1 either way |
