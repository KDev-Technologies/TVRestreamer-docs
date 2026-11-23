# 14 · TimeShift integration (catch-up)

[← Index](00-README.md) · [Български](../bg/14-timeshift-integration.md)

TvRestream has no archive (DVR) of its own — see [01 · Vision](01-vision-prd.md). Catch-up / timeshift comes from the
separate **TimeShift Server**: it records the channels from TvRestream and serves the archive, and TvRestream puts its
address into the subscribers' playlists. This document describes the link from the TvRestream side (tvr-core). The
TimeShift side is in its own documentation: `docs/en/TVRESTREAM.md` in the TimeshiftServer repository (the portal
`index.html` → 05 "TvRestream integration").

| | |
|---|---|
| Tickets | TVR-132 (catch-up in the playlists, API for TimeShift), TVR-133 (recording subscriber), BUG-059…068 |
| In TimeShift | TSS-064…071 |
| Settings | Settings → **TimeShift (catch-up)**, group `TIMESHIFT` |
| Package | `io.tvr.core.timeshift` |
| Migration | V30 `package.catchup` |

## 1. Who does what

| | TvRestream | TimeShift |
|---|---|---|
| Live (HLS / HTTP-TS) | ✓ edge `/live/<slug>/`, `/ts/<slug>.ts` | – |
| The subscriber's playlist (`/pl/{token}.m3u`, `get.php`) | ✓ with `catchup`, `catchup-source`, `catchup-days` | – |
| Subscribers, packages, IP / country, MAC lock, last day | ✓ | asks TvRestream (`/api/timeshift/v1/access`) |
| Recording the channels | gives the streams to a service subscriber | ✓ ffmpeg per channel |
| Archive (catch-up playlist and segments) | – | ✓ the player fetches them directly |
| Which channels have an archive, how many days | reads it from TimeShift | ✓ `GET /api/integration/v1/channels` |

Archive video **never** goes through TvRestream — neither through tvr-core nor through the edge.

## 2. Diagram

```
 Player ──① /pl/{token}.m3u, get.php ─────────────▶ tvr-core :8080 ──(cache from ②)── catchup-source to TimeShift
 Player ──  live /live/<slug>/ ───────────────────▶ edge :8081
 Player ──③ catch-up /?ch=<id>&utc=…&token=…&mac=… ─────────────────────────────▶ TimeShift :13100

 tvr-core ──② GET  /api/integration/v1/channels (Bearer, If-None-Match) ─────────▶ TimeShift Java :18080
 tvr-core ◀─④ GET  /api/timeshift/v1/access?token|username+password&ip&mac (Bearer) ── TimeShift
 tvr-core ◀─⑤ POST /api/timeshift/v1/events {"type":"channels.changed"} (Bearer) ─── TimeShift
 edge     ◀─⑥ recording: /pl/<service subscriber token>.m3u → /live/<slug>/ on 127.0.0.1 ─ TimeShift's ffmpeg
```

1. The subscriber fetches their playlist as before. Channels that TimeShift records and whose package allows catch-up
   get the three attributes. A playlist **never waits** for TimeShift — it uses the cached list.
2. tvr-core reads the TimeShift list at start, every `timeshift.refresh` (5 min) and after event ⑤, with an ETag.
3. The player fetches the archive straight from TimeShift with the subscriber's credentials.
4. TimeShift asks tvr-core whether the subscriber may watch and which channels.
5. TimeShift reports that its list changed — tvr-core reads it at once in the background.
6. TimeShift records through the playlist of a service subscriber (TVR-133) — internal links, no device restrictions.

## 3. Connecting (from the TvRestream side)

The full procedure has steps in both panels. The TvRestream steps are here in detail; the TimeShift ones in short
(in detail in the TimeShift documentation).

### 3.1 Before you start

* Ports on one machine: tvr-core `127.0.0.1:8080`, edge `:8081`, TimeShift Java `127.0.0.1:18080`, TimeShift nginx
  `:13100`. TimeShift's Java **cannot** use 8080 (`install.sh --app-port 18080`).
* On separate machines: tvr-core must reach TimeShift's Java, TimeShift must reach tvr-core and the edge. Between
  machines: HTTPS or a private network.
* NTP on both machines.

### 3.2 The recording subscriber (TVR-133)

1. **Packages → New package**, e.g. "TimeShift recording": every channel to record.
2. **Subscribers → New subscriber**, e.g. username `timeshift`, no last day, with that package. Never a real customer.
3. Copy the token of its personal playlist (`/pl/<token>.m3u`) — it goes into TimeShift.
4. **Settings → TimeShift (catch-up):**

| Field | Value |
|---|---|
| TimeShift recording subscriber (`timeshift.recorder-subscriber`) | `timeshift` |
| Recording address (`timeshift.recorder-url`) | empty = the edge address with the host `127.0.0.1` (same scheme and port, e.g. `http://127.0.0.1:8081`); otherwise the edge's internal address when TimeShift is in the LAN |

What this changes for that subscriber (`TimeshiftRecorder`):

* the links of its playlist point to the recording address instead of the public edge address — no hairpin NAT;
* playback **skips** the allowed IPs, countries, MAC lock and connection limit (TimeShift keeps one connection per
  recorded channel); status, last day and packages apply — its packages are what TimeShift may record;
* its playlist has **no** catch-up attributes — it records, it does not watch;
* the username is resolved to the id once every 30 s, because the playback check runs on every edge request.

### 3.3 In TimeShift (short)

1. Page **TvRestream:** Enabled, tvr-core address `http://127.0.0.1:8080`, the token from 3.2, "Subscribers sign in
   with TvRestream" on → Save → **Check the connection**.
2. **Use TvRestream as the channel source** → **Channels → Sync** → turn **Record** on. New channels get id =
   `tvrSlug` = the stream slug.
3. **Integration key → New key** — copy it (shown once).

### 3.4 Settings → TimeShift (catch-up)

| Field | Key / env | Default | Value |
|---|---|---|---|
| TimeShift integration | `timeshift.enabled` / `TVR_TIMESHIFT_ENABLED` | false | on |
| Internal address of TimeShift | `timeshift.api-url` / `TVR_TIMESHIFT_API_URL` | – | `http://127.0.0.1:18080` (Java, not nginx) |
| Integration key | `timeshift.key` / `TVR_TIMESHIFT_KEY` | – | the key from 3.3 (16–256 characters) |
| Public address of TimeShift | `timeshift.public-url` / `TVR_TIMESHIFT_PUBLIC_URL` | – | empty = TimeShift's `publicBaseUrl`; otherwise e.g. `http://tv.example.com:13100` |
| Refresh | `timeshift.refresh` | 5m | 30 s … 1 h |
| Credentials in catch-up for username/password subscribers | `timeshift.xtream-auth` | `TOKEN` | `TOKEN` (advised) or `USERNAME_PASSWORD` |
| Keep the last list (expert) | `timeshift.stale-after` | 6h | 5 min … 7 days |
| Recording subscriber / Recording address | `timeshift.recorder-subscriber` / `timeshift.recorder-url` | – | see 3.2 |

Every value applies at once, without a restart. The list cache is the file `tvr.timeshift.cache-file`
(`TVR_TIMESHIFT_CACHE_FILE`, by default `timeshift-cache.json` in tvr-core's working directory).

**Check the connection** (`POST /api/v1/timeshift/actions/check`, ADMIN) reads the list again without
`If-None-Match` and shows "TimeShift answers: N channels, M with catch-up, K matched to streams". TimeShift channels
without a TvRestream stream are listed below ("no tvrSlug" — the channel was added in TimeShift before the integration).

### 3.5 Packages and streams

* **Packages:** the **Catch-up** check box (V30 `package.catchup`, on by default for all, existing ones included).
  Untick it for a package that should have no archive. A channel gets catch-up when at least one of the subscriber's
  active packages that contains it has the box ticked.
* **Streams:** the **Catch-up** column — archive days and the state from TimeShift (`RUNNING`, `RESTARTING`,
  `STOPPED`). The data comes from `GET /api/v1/timeshift` (every role).

### 3.6 Check

```bash
# TimeShift accepts the key
curl -s -H "Authorization: Bearer <key>" http://127.0.0.1:18080/api/integration/v1/ping

# a subscriber's playlist has catch-up
curl -s "http://127.0.0.1:8080/pl/<subscriber token>.m3u" | grep -m3 catchup-source

# get.php (m3u_plus) — token or username/password by timeshift.xtream-auth
curl -s "http://127.0.0.1:8080/get.php?username=<u>&password=<p>&type=m3u_plus" | grep -m1 catchup-source

# what TimeShift asks about the subscriber
curl -s -H "Authorization: Bearer <key>" \
     "http://127.0.0.1:8080/api/timeshift/v1/access?token=<subscriber token>&ip=203.0.113.5"

# the service subscriber: internal links, no catchup-source
curl -s "http://127.0.0.1:8080/pl/<timeshift token>.m3u" | head -4
```

Finally: TiviMate (or another player) with the old playlist address plays a programme from 2 hours ago from the EPG.

## 4. What the subscriber gets

`/pl/{token}.m3u` (and `type=ts`):

```
#EXTM3U url-tvg="http://tv.example.com:8081/pl/<token>.xml" x-tvg-url="…"
#EXTINF:-1 catchup="default" catchup-source="http://tv.example.com:13100?ch=btv&utc=${start}&lutc=${timestamp}&token=<token>" catchup-days="7" tvg-id="btv" tvg-name="bTV HD" group-title="National",bTV HD
http://tv.example.com:8081/live/btv/index.m3u8?token=<token>
```

| Case | `catchup-source` ends with |
|---|---|
| `/pl/{token}.m3u` | `&token=<token>` |
| `/pl/{token}.m3u?mac=00:1A:79:…` (MAC lock) | `&token=<token>&mac=00:1A:79:…` |
| `get.php`, `timeshift.xtream-auth = TOKEN` | `&token=<the subscriber's token>` |
| `get.php`, `USERNAME_PASSWORD` | `&username=<u>&password=<p>` (the password shows in the player and in logs — hence not the default) |

* `${start}` and `${timestamp}` stay literal — the player fills them in.
* The address is TimeShift's `catchupSource` (with `ch=<channel id in TimeShift>`); `timeshift.public-url` replaces
  only its beginning (`publicBaseUrl`).
* `catchup-days` comes from TimeShift: the configured days, but no more than the real archive in whole days.
* EPG: when TvRestream has no guide source of its own, the header carries TimeShift's guide (`/epg.xml.gz` with the
  subscriber's credentials).
* The Xtream API (`player_api.php`) does not announce an archive (`tv_archive` = 0): catch-up is in the M3U playlists
  (`/pl`, `get.php`).

## 5. How the cache works

`TimeshiftService` keeps the TimeShift list (`TimeshiftCatalog`):

| When | What |
|---|---|
| tvr-core starts | reads `timeshift-cache.json` — catch-up survives a restart |
| a 10 s tick, a read after `timeshift.refresh` | `GET /api/integration/v1/channels` with `If-None-Match` → 304 only confirms the list |
| a `channels.changed` event | 204 at once; when the `etag` differs from the cached one — a read in the background; a repeated event does nothing |
| "Check the connection" | a read without `If-None-Match`; waits up to 30 s for a read already running, to show its own answer (BUG-066) |
| 304 | the confirmation is written to the file too (BUG-067) |
| TimeShift does not answer | the last list is used for `timeshift.stale-after` (6 h) after the last successful contact; then playlists go out without catch-up |

Matching: a TimeShift channel matches the stream whose `slug` = its `tvrSlug`; without a `tvrSlug`, `slug` = `id`
(TVR-133). Channels with a `tvrSlug` win over an id match.

## 6. The TvRestream API for TimeShift

Only with `Authorization: Bearer <timeshift.key>`; 404 while the integration is off or has no key; 401 for a missing
or wrong key. The key and passwords are not logged. OpenAPI security: `timeshiftKey`.

### `GET /api/timeshift/v1/access`

| Parameter | Meaning |
|---|---|
| `token` | the personal playlist token (32 characters) |
| `username`, `password` | the Xtream login — the same lockout and Argon2 limit as `get.php` |
| `ip` | the player's IP: wrong passwords count against it, not against TimeShift; the allowed IPs and countries are checked — without it a subscriber with such a restriction is refused (BUG-061) |
| `mac` | the box's MAC from the catch-up link, for the MAC lock (BUG-061) |

| Answer | When |
|---|---|
| 200 | an ACTIVE subscriber whose last day has not passed; body below |
| 400 | neither a token nor a username and password |
| 403 `{"reason": "disabled"}` | suspended (SUSPENDED) or banned (BANNED) |
| 403 `expired` | expired, or not ACTIVE |
| 403 `ip` / `country` / `mac` | not from an allowed address, country or box |
| 404 `{"reason": "unknown"}` | no such subscriber (also a wrong password) |
| 503 + `Retry-After: 5` | too many password checks at once |

```json
{"subscriber": "ivan", "enabled": true, "expiresAt": "2026-12-31T21:59:59Z", "catchup": true,
 "channels": ["btv", "nova", "bnt1"], "catchupChannels": ["btv", "nova"]}
```

* `channels` — the enabled channels of the active packages with a public output (HLS or TS), the same as the playlist.
* `catchupChannels` — those of them whose packages allow catch-up; TimeShift plays the archive only for these.
* `expiresAt` — the end of the last day (the subscribers' time zone); `null` = no end.

### `POST /api/timeshift/v1/events`

```json
{"type": "channels.changed", "etag": "\"7f3a\"", "at": "2026-11-18T12:00:00Z"}
```

204 at once. Every type is taken as "the list may have changed" — an unknown future type costs one conditional request.

### For the panel

| Address | Role | Description |
|---|---|---|
| `GET /api/v1/timeshift` | every | state (`enabled`, `configured`, `usable`, last read, error) and the TimeShift channels with their match |
| `POST /api/v1/timeshift/actions/check` | ADMIN | "Check the connection" |

## 7. Failure handling

| Situation | Result |
|---|---|
| TimeShift down | playlists keep catch-up for 6 h from the last list, then go out without it; live is not affected |
| tvr-core restarted | the list is read from `timeshift-cache.json` |
| tvr-core does not answer `/access` | TimeShift lets known subscribers in for 30 more minutes (`graceMinutes`), new ones get 503 |
| The edge is down | recordings in TimeShift are `RESTARTING`, the archive gets a gap |
| A lost event | the 5 min refresh catches it |
| A suspended subscriber / a new restriction | TimeShift caches the answer ≤ 10 s (`tvrestream.authCacheSeconds`), then refuses |

## 8. Security

* One key for both directions, generated in TimeShift ("New key"); a new key stops the old one at once — enter it here.
* `timeshift.api-url` is an internal address; do not publish TimeShift's Java port.
* Xtream subscribers' passwords stay out of catch-up links by default (`TOKEN`, BUG-062).
* The service subscriber has no device restrictions — do not hand out its token and do not use it to watch.
* `/api/timeshift/v1/*` is not behind the panel login but only behind the key; a wrong key gets 401.

## 9. Troubleshooting

| Symptom | Cause / check |
|---|---|
| "TimeShift does not answer: … 404" | the integration in TimeShift is off or has no key there |
| "… 401" | the key here is not the latest "New key" from TimeShift |
| "… Connection refused" | `timeshift.api-url` points to the wrong port (nginx :13100 instead of Java :18080) or TimeShift is down |
| 0 channels matched | the TimeShift channels have no `tvrSlug` and their id is not a slug — sync TimeShift from the service playlist |
| The playlist has no `catchup-source` | the package has Catch-up unticked; TimeShift does not record the channel (`catchupDays` 0); the list is older than `stale-after`; it is the service subscriber |
| Player: 403 for catch-up | the reason is in the TimeShift log: `mac` (the player sends no MAC — the playlist must be loaded with `?mac=`), `ip` / `country`, `expired` / `disabled` |
| Every IP-restricted subscriber gets 403 | TimeShift does not send `ip` (an old version) or the player's IP does not reach it behind nginx |
| TimeShift stops recording every channel | the service subscriber is suspended / expired, its package changed or its token was renewed |
| Recording goes through the router | "Recording address" points to the public address; empty = the edge on `127.0.0.1` |
| 502 when recording a channel playlist | an old edge without the BUG-069 fix (`Range` from ffmpeg) |
