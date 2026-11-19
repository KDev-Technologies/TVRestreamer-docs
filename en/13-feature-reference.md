# 13 · Feature Reference

[← Index](00-README.md) · [Български](../bg/13-feature-reference.md)

| Field | Value |
|---|---|
| Audience | Administrators, operators, support, integrators, developers |
| Code state | `main` @ TVR-105 (2026-11-19) — S0–S6, R2 and part of R3 |
| Layout | By module, in the order of the panel menu. For each feature: **what it does**, **how it works inside**, **where it is configured**, **API** and **limits** |

Document 02 describes the *requirements* (FR-*), document 08 and the [administrator guide](https://github.com/kocetod/TVRestreamer/blob/main/guides/admin-guide.md) describe *installation and daily work*. This document describes **what the software actually does today**, from the code. The tickets and requirements in brackets show where each part was built, so it can be found in the code and in the plan (document 12).

## Contents

1. [Overview](#1-overview)
2. [Channels](#2-channels)
3. [Inputs](#3-inputs)
4. [Failover](#4-failover)
5. [Stream processing](#5-stream-processing)
6. [Outputs](#6-outputs)
7. [Analyzer, charts and history](#7-analyzer-charts-and-history)
8. [Subscribers](#8-subscribers)
9. [Packages](#9-packages)
10. [Playback: playlists, Xtream, EPG, catch-up](#10-playback-playlists-xtream-epg-catch-up)
11. [Playback protection](#11-playback-protection)
12. [Sessions and devices](#12-sessions-and-devices)
13. [Servers and cluster](#13-servers-and-cluster)
14. [Alerts and incidents](#14-alerts-and-incidents)
15. [Dashboard and real-time events](#15-dashboard-and-real-time-events)
16. [Statistics and reports](#16-statistics-and-reports)
17. [Admins, roles and panel login](#17-admins-roles-and-panel-login)
18. [API keys and webhooks](#18-api-keys-and-webhooks)
19. [Audit log](#19-audit-log)
20. [Import and configuration archive](#20-import-and-configuration-archive)
21. [Settings and server health](#21-settings-and-server-health)
22. [Monitoring, backup and operations](#22-monitoring-backup-and-operations)
23. [Data security](#23-data-security)
24. [Reference: ports, retention, addresses](#24-reference-ports-retention-addresses)
25. [Not implemented yet](#25-not-implemented-yet)

---

## 1. Overview

### 1.1 Components

| Component | What it does | Technology |
|---|---|---|
| **tvr-core** | Control plane: REST API, admin panel, database, access check for the edge, playlists, Xtream API, EPG, alerts, reports. One core manages N agents. | Java 21, Spring Boot 4.1, PostgreSQL 16, Redis 7 |
| **tvr-agent** | Data plane: runs the channels of its server (FFmpeg processes, TS switcher, analyzer), HTTP-TS relay, RTMP ingest, UDP/SRT/RTMP outputs, DVB tuners, server metrics. | Java 21 (Spring Boot without a web server), Netty, FFmpeg 7+, TSDuck for DVB |
| **edge (nginx)** | Serves HLS and HTTP-TS to subscribers. Asks core about every playlist (`auth_request` → `/internal/auth`), signs the segments (njs) and checks them locally. | nginx + njs |
| **tvr-ui** | The admin panel, BG/EN, light and dark theme. | React, TypeScript, Ant Design |
| **tvr-common** | The agent ↔ core gRPC contract (`agent.proto`). | Protobuf |

### 1.2 The video path of one channel

```
source ──▶ input FFmpeg #1 ──UDP 127.0.0.1──▶ ┐
source ──▶ input FFmpeg #2 ──UDP 127.0.0.1──▶ ├─ TS switcher (Java, Netty) ──▶ HLS packager FFmpeg ──▶ /live/<slug>/index.m3u8
                                               │   · analyzer, CC, PCR         │
                                               │   · picks the active input    ├─▶ HTTP-TS relay   (/ts/<slug>.ts)
                                               │   · one continuous timeline   ├─▶ UDP multicast output
                                               └───────────────────────────────┴─▶ SRT / RTMP push (own FFmpeg)
```

- Every input has its own FFmpeg that only remuxes (`-c copy`, unless transcoding) and sends MPEG-TS to a loopback UDP port (range 20000–29999).
- The **TS switcher** receives all inputs, analyzes them and forwards only the **active** one to the packager. So an input switch does not restart packaging and players do not break (ADR-003).
- The HTTP-TS, UDP and SRT/RTMP outputs tap the switcher and get exactly what the HLS packager gets.
- Video never passes through core. Core only sends the agent the **desired state** and commands.

### 1.3 Core ↔ agent link

- The agent opens **one** bidirectional gRPC connection to core and keeps it open (it may be behind NAT). The first message is `Hello` (versions, running channels, capabilities). Core answers with `Welcome` (server id, heartbeat interval).
- The agent sends a heartbeat every 5 s, server metrics every 5 s, channel status on every change and a full report every 5 s.
- Core sends the **DesiredState** — the full list of the server's channels and the agent settings, with a growing version. The agent is a *reconciler*: it starts what is missing, stops what is no longer listed and restarts a channel **only** when something it runs changed. A change in the panel is sent at once; a resend every 30 s is the safety net.
- Commands: restart a channel, switch/pin an input, kick an HTTP-TS client, fetch a channel log, restart the agent.
- A server without a heartbeat for 15 s is `OFFLINE`.

### 1.4 Server roles

| Role | Channels | Serves viewers |
|---|---|---|
| `ALL_IN_ONE` | yes | yes |
| `INGEST` | yes | no |
| `EDGE` | no | yes (runs an agent for the HTTP-TS relay, metrics and balancing) |

---

## 2. Channels

Menu **Streams → Channels**. Rights: read — every role; change — Admin and Operator (see §17).

### 2.1 Channel list

- Columns: thumbnail, name, status, active input (`#1 primary` / `#2 backup`), bitrate, category, server, **Catch-up** (days of archive in TimeShift), actions.
- Search by name, slug or number (LCN); filters by status, category, server, package.
- States update in real time (SSE, §15) without reloading.
- Content badges: **Black screen**, **Frozen**, **Silence** (§7.4) and **Fallback slate** when the slate is on air (§4.6).
- A toggle **Always on ⇄ On demand** right in the row.
- Bulk actions on selected channels: **Start**, **Stop**, **Move…** (to another server), **Delete…**. On partial success the panel shows how many succeeded and the first error.

### 2.2 New channel / edit

The form has four sections: **Basic**, **Inputs**, **Processing**, **Outputs**.

| Field | Meaning |
|---|---|
| Name | Up to 120 characters; shown in playlists. |
| Slug | Lower-case Latin letters, digits and hyphen, 2–64 characters. Part of the playback address `/live/<slug>/index.m3u8`; unique. |
| Number (LCN) | 1–9999, optional; orders channels in playlists. Unique — the form says which channel already has it. |
| Category | `group-title` in the M3U; optional. |
| Server | The ingest/all-in-one server that runs the channel. "Automatic" = the only ingest server (with several servers the channel is pinned to the default server). |
| Mode | **Always on** or **On demand** (§2.4). |
| EPG ID | XMLTV id of the channel (`tvg-id`); empty = the slug. |
| Logo | An address (`http(s)://`) or an uploaded PNG/SVG up to 512 KB (§2.6). |
| Enabled | A disabled channel does not run. |
| Return to primary, fallback slate, failover triggers | §4. |
| Audio languages, subtitles, transcoding | §5. |

- **Concurrent edit protection:** every save increments the channel version. If another user changed it meanwhile, the panel says "The channel was changed by another user" and offers to reload it (HTTP 409).
- All invalid fields are marked at once (the server returns a list of errors per field).
- A new channel gets the defaults from **Settings** (HLS, input timeout, failover triggers, return to primary, slate, HTTP-TS) — see §21.
- API: `GET/POST /api/v1/streams`, `GET/PUT/DELETE /api/v1/streams/{id}`.

### 2.3 Actions and states

| Action | Effect |
|---|---|
| **Start / Stop** | Writes `enabled` to the database; the agent starts or stops the processes. |
| **Restart** | A one-off command to the agent, nothing changes in the database. |
| **Switch / pin input** | §4.5. |
| **Move** | §13.5. |
| **Delete** | The channel stops and disappears from the list, packages and playlists (soft delete — the audit history stays). One channel or a selection (`POST /api/v1/streams/actions/delete`). |

States reported by the agent:

| State | Meaning |
|---|---|
| `STOPPED` | The channel is disabled. |
| `STARTING` | Processes are starting, no data yet. |
| `ONLINE` | The active input delivers without problems. |
| `DEGRADED` | Running, but with a problem: CC errors, a lost input with a backup available, etc. |
| `OFFLINE` | No input delivers data (the fallback slate may be on air). |
| `ERROR` | Invalid configuration or the process cannot start. |

The state is kept in Redis with an expiry — an agent that stopped reporting does not leave a channel "Online" forever. Every state change is also stored as an event in the channel history (§7.6).

### 2.4 On-demand mode (TVR-063, FR-STR-04)

- The channel **does not run** while nobody watches. The first viewer request for the playlist starts it; it stops after **60 s** without a viewer (`TVR_ON_DEMAND_IDLE_TIMEOUT`, at least 20 s).
- While the channel starts, the edge asks the origin every 500 ms for up to 6 s. If there is no playlist yet, the player gets `503` with `Retry-After` and tries again.
- Good for rarely watched channels: saves CPU, provider bandwidth and provider connections. Not for channels with UDP/DVB outputs or external partners (they have no viewers through the edge).

### 2.5 Channel thumbnail (TVR-064)

Every 60 s the agent takes the first frame of the newest HLS segment and writes it as `thumb.jpg` (320 px wide) next to the playlist. The panel shows it in the list and the detail. There is no thumbnail when the channel is stopped, has no video or has just started. Agent settings: `tvr.agent.thumbnails.*` (`TVR_AGENT_THUMBNAILS_ENABLED`).

### 2.6 Logos (TVR-047)

- **Upload:** PNG or SVG up to 512 KB (`PUT /api/v1/streams/{id}/logo`). SVG files are checked: an SVG with scripts, event handlers or external resources is refused.
- The logo is stored in the database by content hash and served at `/logos/{id}.png|svg?v=<hash>` — a new logo changes the address, so players do not keep the old one in their cache.
- The channel gets the new logo only on **Save** of the form.
- In the M3U the logo is `tvg-logo`.

### 2.7 Categories (TVR-013)

CRUD and ordering (`/api/v1/categories`, `PUT /order`). The category is `group-title` in the M3U and the category in the Xtream API. Imports create missing categories automatically.

### 2.8 Channel detail

Tabs: **Overview** (state, since when, video/audio, last error, a player in the panel, thumbnail), **Inputs** (live state of every input, switch and pin), **Outputs** (playback addresses), **Analyzer** (§7), **Charts** (24 h, §7.5), **Events** (§7.6), **History** (audit of the channel's changes, §19), **Log** (last lines of FFmpeg and the agent, §7.7).

The panel player uses a temporary preview token of the admin (`GET /api/v1/auth/preview`), not a subscriber's.

---

## 3. Inputs

A channel has **1 to 8 inputs** with a priority (1 = primary). They are ordered by drag and drop or arrows.

### 3.1 Types and addresses

| Type | Address | Notes |
|---|---|---|
| **HTTP-TS** | `http(s)://…` | MPEG-TS over HTTP. |
| **HLS** | `http(s)://…/index.m3u8` | |
| **UDP / RTP** | `udp://[iface@]239.1.1.1:1234`, `rtp://…` | Multicast or unicast. A local address/interface binds the channel to its server (§13.6). |
| **SRT** (TVR-058) | `srt://host:port` | `mode`: **caller** (the server connects to the source) or **listener** (the source connects to `srt://0.0.0.0:port` — open the UDP port). Latency 20–8000 ms (empty = 120 ms), passphrase 10–79 characters (AES, `pbkeylen` 16/24/32), stream ID up to 512 characters. |
| **RTMP** (TVR-059) | pull: `rtmp(s)://host/app/stream`; push: `rtmp://0.0.0.0/<app>/<key>` | **pull** — the server plays the address. **push** — an encoder (OBS, vMix, FFmpeg) publishes to the agent's built-in RTMP ingest at `rtmp://<agent>:1935/<app>` with the stream key. The key is 8–128 characters; **New key** generates a random one. Whoever knows the key can broadcast to the channel. |
| **RTSP** (TVR-060) | `rtsp://user:password@camera:554/path` | Transport TCP (passes NAT/firewalls) or UDP (clean local network only). Audio is encoded to AAC by default. |
| **DVB-S/S2** (TVR-105) | `dvb://adapterN[/frontendM]` | A satellite card in the channel's server (§3.4). |

Only network protocols are allowed (`http`, `https`, `udp`, `rtp`, `srt`, `rtmp`, `rtmps`, `rtsp`, `dvb`). `file:`, `concat:`, `subfile:` and the other FFmpeg pseudo-protocols are refused by the API and by a database constraint (SEC-07) — otherwise an address could read local files. FFmpeg is always started with an argument list, never through a shell (SEC-06).

### 3.2 Common input options

| Option | Meaning |
|---|---|
| **Timeout** | 1–120 s (default 5). Seconds without data after which the channel moves to the next input. |
| **Hot standby** | The input is received all the time, even when not active (§4.2). |
| **Enabled** | A disabled input is skipped. |
| **Program (PNR)** | For MPTS — which program to keep (§3.3). |
| **Audio** | `copy` (default), `aac` (re-encode audio), `none` (no audio). RTSP defaults to `aac`. |
| User-Agent, request headers, reconnect, analyze duration | For HTTP/HLS. When the channel has no value of its own, the one from **Settings → Input defaults** is used. |

An M3U import carries over `#EXTVLCOPT:http-user-agent`, `http-referrer` and similar VLC/Kodi options.

### 3.3 MPTS: program selection (TVR-061)

When the source carries several programs in one stream (MPTS, e.g. a provider multicast or a satellite), the **Program (PNR)** field sets the program number (1–65535). Only that program stays in the channel. The **Test** button lists the programs found in the source with their SDT names, so the right one can be picked.

### 3.4 DVB-S/S2 input (TVR-105, FR-INP-09/10)

- Requires: a card in the server, TSDuck (`apt install tsduck`), the agent user in group `video`. The channel must be bound to the server with the card.
- Parameters: **DVB-S** or **DVB-S2** (default), transponder **frequency** 950–13000 MHz, **polarization** H/V/L/R, **symbol rate** 1000–45000 kSym/s, **modulation** QPSK (default), 8PSK, 16APSK, 32APSK (DVB-S is QPSK only), **FEC** auto or a rate (`3/4`…), **LNB** `universal` (9750/10600, switching at 11700) or frequencies in MHz (`10750` or `9750,10600,11700`), **DiSEqC** port 1–4.
- **Shared tuners:** a card receives one transponder, and a transponder carries many programs. All inputs on one transponder use **one** `tsp` process; the agent fans the stream out to the inputs and each keeps its own program by PNR. An input that wants another transponder of a busy card waits — it gets no data, so its channel fails over to a backup input.
- `tsp` is restarted with backoff when it exits or delivers nothing for 5 s (`TVR_AGENT_DVB_RECEIVE_TIMEOUT`).
- A DVB input cannot be tested from the panel: start the channel and read its log.

### 3.5 Input test (TVR-022, FR-INP-07)

The **Test** button next to each input runs `ffprobe` on the core server (up to 9 s, at most 4 at a time — more return 429). Result: container, programs (with SDT names), codecs, resolution, frame rate, bitrate, audio languages. Errors carry a clear code: no answer in time, connection refused, host not found, 404, access denied, not a video stream, no video/audio, protocol or address not allowed, no ffprobe on the server.

Cannot be tested: SRT listener, RTMP push, DVB (the source comes to the server or is hardware). Testing is Admin and Operator only, because it opens a connection from the server. With `TVR_OUTBOUND_DENY` set (e.g. `private`) the test cannot reach those networks (SSRF protection; redirects are checked too).

API: `POST /api/v1/inputs/probe`.

---

## 4. Failover

All failover happens in the agent's TS switcher — without restarting the packager. The player sees a continuous stream with a switch marker (§4.7).

### 4.1 Cold failover (TVR-027, FR-FOV-02)

A plain ("cold") input runs **only while it is active**. When no datagram arrives within the input timeout (5 s by default; plus 3 s at start), the agent starts the next input by priority, the switcher moves to it (sending PAT/PMT first so the packager learns the new PIDs) and the old input is stopped. After the last input comes the first again — a source that came back is found on the next round.

### 4.2 Hot standby (TVR-051, FR-FOV-03)

An input marked **standby** runs from the start of the channel and keeps running when it is not on air. The switcher always has its data, PAT/PMT and keyframes. When the active input goes silent, the channel does not wait for the full timeout: after **1 s** of silence (or the timeout, if shorter) it moves to the standby input at its next keyframe — the switch takes **under 3 s**. The price: one more provider connection and one more FFmpeg per standby input.

### 4.3 Quality triggers (TVR-054, FR-FOV-04)

An input that delivers data may still be broken. In the channel's **Advanced failover settings**:

| Trigger | Default |
|---|---|
| CC errors per minute above N (analyzer errors + corrupt packets FFmpeg sees in the source) | 0 = off |
| Minimum bitrate (kbit/s) | 0 = off |
| Switch when video is missing (the PMT has video but no packets arrive) | on |
| Switch when audio is missing | on |
| Switch on a scrambled stream | on |
| Hold — how long the problem must last | 5 s (1–300) |

Once a condition holds for the hold time, the channel moves to a standby input without a problem, otherwise to the next cold input. An input left for a quality problem is not tried again for 5 minutes. When no input is left, the channel stays where it is (logged once). The defaults for new channels are in **Settings → Failover**.

### 4.4 Return to primary (TVR-052, FR-FOV-05)

While a lower-priority input is on air, the agent watches the higher ones: standby inputs run anyway, cold inputs are **probed** — every 10 s one of them is started next to the active input. Once a higher-priority input has delivered without a gap for **30 s** (configurable 5–3600 s, "Stable for"), the channel switches back to it and a `RECOVERED` event is stored. It can be turned off per channel. An input with a quality problem does not count as stable.

### 4.5 Manual switch and pin (TVR-036, FR-FOV-06)

In the **Inputs** tab an operator can switch to a specific input and **pin** it. While pinned, automatic failover is suspended — the input stays active even without data — until a switch without pin. A pin lives as long as the running processes (a channel restart clears it). How many channels have a pinned input is also in Prometheus (`tvr_streams_input_pinned`).

API: `POST /api/v1/streams/{id}/actions/switch-input` (`inputId`, `pin`).

### 4.6 Fallback slate (TVR-099, FR-FOV-08)

When nothing has reached the packager for **8 s** and no input delivers, the agent starts an extra FFmpeg with a still picture and the text "Channel temporarily unavailable" (the text is set in **Settings → Failover**, up to 80 characters) and silent audio, in the channel's codecs. Players keep playing instead of stalling. Failover goes on behind the slate; the first input that delivers goes on air at its keyframe and the slate stops. The state stays `OFFLINE`, and the panel shows a **Fallback slate** badge. On/off per channel; on by default.

### 4.7 Continuous timeline and discontinuity (TVR-028, FR-PRC-04)

Every input FFmpeg has its own clock, so on a switch PTS/DTS/PCR would jump. The switcher puts the new input on air at its first keyframe with timestamps that continue the old ones plus one segment. The packager starts a new segment there, and the agent writes `EXT-X-DISCONTINUITY` before it in `index.m3u8` — players reset their decoder when codec, resolution or audio differ.

---

## 5. Stream processing

### 5.1 Audio languages and subtitles (TVR-062, FR-PRC-02)

- **Audio languages (keep):** ISO 639-2 codes (`bul`, `eng`…) in the order they go on air — most players start with the first. Empty = all tracks. A track without a language tag or without a match is dropped. When no track matches, the channel goes on without audio.
- **Drop subtitles/teletext:** removes DVB subtitles and teletext.
- Applies to every input of the channel (every input FFmpeg applies it), so a backup input carries the same tracks.

### 5.2 Video transcoding (TVR-106, FR-PRC-05, ADR-008)

| Profile | Height | Default bitrate |
|---|---|---|
| None | — | video copied as it comes (no load) |
| 1080p | 1080 | 5000 kb/s |
| 720p | 720 | 3000 kb/s |
| 480p | 480 | 1500 kb/s |

- **Encoder:** Automatic (the first available: NVIDIA NVENC → Intel Quick Sync → CPU), **NVIDIA NVENC** (`h264_nvenc`, needs `/dev/nvidia0`), **Intel QSV** (`h264_qsv`, needs `/dev/dri/renderD128`), **CPU** (`libx264`).
- **Video bitrate:** empty = the profile's.
- Output is H.264. Decoding, deinterlacing (`yadif`, only frames flagged interlaced) and scaling run on the CPU, so any source works (MPEG-2, H.264, HEVC). A smaller picture is not enlarged.
- A keyframe every 2 s (1 s for an odd segment length), as IDR — segments and switches start on a keyframe.
- Every input encodes, so a backup input gives the same format. The price: every hot standby input also takes a GPU session or CPU share.
- Audio follows the input's **Audio** option; subtitles are copied.

### 5.3 Service name (SDT)

The **Service name (SDT)** field is in the form but **not applied yet** (TVR-102). Outputs carry FFmpeg's default name.

---

## 6. Outputs

### 6.1 HLS (TVR-019, FR-OUT-01)

- The main output for subscribers: `/live/<slug>/index.m3u8`, served by nginx. Segments are in the agent's work directory — a tmpfs in production (`/run/tvr-agent/work`), so the disk does not wear out.
- **Segment** 1–30 s (default 4), **list** 2–60 segments (6), **delete threshold** 1–30 (2). Defaults for new channels and the **Apply to all channels** button are in **Settings → HLS and HTTP-TS**.
- Every 10 s the agent removes segments that are no longer in the playlist (and leftovers of a previous packager).

### 6.2 HTTP-TS (TVR-056, FR-OUT-02)

- A relay in the agent: `GET /ts/<slug>.ts` streams the channel's MPEG-TS (exactly what the packager gets) to every client, without FFmpeg and without an end. For VLC, Smart TVs, set-top boxes.
- A new client starts at the next keyframe with the latest PAT/PMT — a clean picture at once.
- A client that does not keep up loses data (counted as `bytes_dropped`) but does not slow down the others.
- Access: the viewer comes through the edge (which asks `/internal/auth` only once at the start). So the agent reports its open connections to core every 10 s; core checks the access of each again (the subscriber may have been suspended or expired) and kicks those no longer allowed. This also keeps the device's session alive.
- The relay port (8090) must be reachable from edge servers only; without an internal token it listens on 127.0.0.1 only.
- On per channel; **Settings → HTTP-TS output for new channels** and **Enable HTTP-TS on all channels** (no channel restart).

### 6.3 UDP multicast / unicast (TVR-057, FR-OUT-03)

- Address (IPv4/IPv6 or a name), port, TTL, optional **local address** and **interface** (e.g. `eth1`).
- Sent straight from the agent's switcher (no FFmpeg), in datagrams of 1316 bytes (7 TS packets).
- A local address or interface binds the channel to its server (§13.6).

### 6.4 SRT output (TVR-096, FR-OUT-04)

- **listener:** partners pull `srt://<agent address>:port` (one partner per port; port 1024–65535, open the UDP port). Binds the channel to its server.
- **caller:** the agent sends to the partner's `srt://host:port`.
- Latency (empty = 120 ms), passphrase, stream ID. One FFmpeg per output, fed from the switcher over loopback.

### 6.5 RTMP push to an external platform (TVR-096, FR-OUT-05)

To YouTube, Twitch, Facebook and others: the **RTMP server address** (`rtmp(s)://…/app`) and the **stream key** as the platform shows them. Video is copied; audio is **as is** or **→ AAC** (most platforms want AAC). The key is a secret: encrypted in the database and hidden in the audit.

---

## 7. Analyzer, charts and history

### 7.1 TS analyzer (TVR-026, TVR-053, FR-ANL-01..05)

The agent analyzes every running input in Java (per ISO/IEC 13818-1 and ETSI TR 101 290). The **Analyzer** tab shows for the active input:

- **Totals:** bitrate of the whole stream, CC errors over the last 5 s and since the start, transport errors, sync losses, PSI errors, **source errors** (corrupt packets the input FFmpeg sees — the remux hides them from the analyzer, so they are counted separately).
- **Services:** programs of the PAT with name and provider from the SDT, type (TV, radio, H.264 HD…), PMT PID, PCR PID, whether scrambled (CA descriptors in the PMT or `free_CA_mode` in the SDT).
- **PID table:** PID, type (video/audio/subtitle/data/PSI), codec (`h264`, `hevc`, `aac`, `mp2`, `ac3`…), bitrate, CC errors (now and total), scrambled, language.
- **PCR:** interval (max/average; above 40 ms violates TR 101 290), arrival jitter, clock drift (ppm, after the first 60 s), discontinuities.

The data comes with every status (every 5 s); API: `GET /api/v1/streams/{id}/analyzer`.

### 7.2 Video/audio in the overview

The **Overview** tab shows video codec, resolution and bitrate, audio codec and language — from the analyzer.

### 7.3 Lost UDP packets on the server (TVR-114)

A channel's processes talk over UDP on 127.0.0.1 and ask for a 4 MB buffer, but Linux silently cuts every request down to `net.core.rmem_max`. The agent checks the buffer the sockets really got and counts `RcvbufErrors` of `/proc/net/snmp` over the last minute. When a server loses packets, the **Servers** page shows "The server is losing UDP packets — the picture may stutter" with the fix command. The `.deb` package sets the sysctl values on installation.

### 7.4 Black screen, frozen picture, silence (TVR-101, FR-ANL-06)

- Every 15 s the agent decodes the newest HLS segment of every channel: keyframes as small grey pictures and audio as 8 kHz mono.
- **Black screen:** pixels at or below luma 32 (0..255) → reported after **15 s**.
- **Frozen picture:** frames differ on average less than the threshold → after **30 s**.
- **Silence:** audio at or below −60 dBFS → after **30 s**.
- The channel shows a badge, the history gets `BLACK_DETECTED`, `FREEZE_DETECTED`, `SILENCE_DETECTED`, `CONTENT_RESTORED` (with the duration), and the **Content** alert rule can send an alert (§14).
- Agent settings: `tvr.agent.content-check.*` (`TVR_AGENT_CONTENT_CHECK_ENABLED`).
- Content problems do **not** trigger failover — they only alert.

### 7.5 24-hour charts (TVR-055, FR-ANL-07)

Core adds up the agent reports (every 5 s) into one row per channel and minute: bitrate, CC and source errors, the worst state, the active input, viewers. The **Charts** tab shows bitrate, CC errors and viewers for a chosen range with markers for input switches. Minute data is kept **48 h**; for reports it is rolled up hourly (§16). API: `GET /api/v1/streams/{id}/metrics`.

### 7.6 Channel events (TVR-029, FR-FOV-07)

Everything the agent reports (`STARTED`, `STOPPED`, `INPUT_LOST`, `INPUT_RESTORED`, `FAILOVER` with from/to and trigger, `RECOVERED`, `PROCESS_EXITED` with exit code, `SCRAMBLED_DETECTED`, `CC_ERRORS_HIGH`, `ERROR`, the content events), plus every state change (`STATUS`). Tab **Events**, paged; kept **180 days** in monthly partitions. API: `GET /api/v1/streams/{id}/events`. The **History** tab is the audit of setting changes (§19).

### 7.7 Channel log (TVR-023, FR-LOG-01)

The last up to 1000 lines (configurable) of FFmpeg and the agent for the channel. The log lives only on the agent; core asks for it with a command and waits up to 5 s. The log of a stopped channel is kept under its slug. Admin and Operator only (lines may contain input addresses). API: `GET /api/v1/streams/{id}/logs?tail=N`.

---

## 8. Subscribers

Menu **Subscribers → Subscribers**. Rights: Admin, Operator — full; Support — create and change, no delete; Viewer — read only, without the token and link (§17).

### 8.1 Fields

| Field | Meaning |
|---|---|
| Username | 3–64 characters: Latin letters, digits and `. _ @ -`; unique. |
| Password | Optional, 6–128 characters — for the Xtream login (§10.2). Stored as an Argon2id hash. A new password **also changes the token** (the old link and the old Xtream login stop). Can be removed — then only the token link remains. |
| Full name, phone, e-mail, note | Free data; searchable. |
| Expires | Last day of access (Sofia time); empty = never. |
| Devices (max) | 1–100 (default from the settings, 1). |
| Over the limit | **Reject the new device** or **Kick the oldest session** (§12.2). |
| Packages | One or more packages (§9). |
| Allowed IPs (CIDR) | `10.0.0.5` or `10.0.0.0/24`; empty = any IP. A network with host bits is refused. |
| Countries | 2-letter ISO codes (`BG`…); empty = any country (§11.4). |
| MAC lock | MAC addresses of the MAG/Enigma2 boxes; empty = any device (§11.5). |
| Status | Active, Suspended, Expired, Banned. |

### 8.2 States and actions

| Action | Effect |
|---|---|
| **Suspend** (`SUSPENDED`) | Playback stops at once; **Resume** brings it back. |
| **Ban** (`BANNED`) | Access stops at once; only an administrator can restore it. |
| **Extend** | By N days after the current last day (or after today when it has passed) or until a given date; an expired subscriber becomes active again. Quick **+N d** buttons in the list. |
| **New token** | The old link stops at once; the subscriber needs the new one. |
| **Delete** | Admin only. |

Every change of a subscriber drops the access cache in Redis after the commit, so a suspended subscriber or an old token stops within seconds, not after 10 s.

### 8.3 Automatic expiry (TVR-046, BR-01)

Every day at 00:00 (Europe/Sofia) active subscribers whose last day has passed become **Expired** and their sessions end. It also runs at core start (in case core was down at midnight). **Expire now** runs it by hand (`POST /api/v1/subscribers/actions/expire`). Even before the nightly job the access check refuses a subscriber whose last day has passed (`EXPIRED`).

### 8.4 Bulk extend (TVR-083)

Selected subscribers are extended at once by N days or until a date — the same rule as for one subscriber (`POST /api/v1/subscribers/actions/extend`).

### 8.5 CSV import and export (TVR-083, FR-SUB-07)

- **Export** of the (filtered) list: username, name, phone, e-mail, packages, devices, policy, expiry, IPs, countries, note, status, creation date. The password is always empty; the token is never exported.
- **Template** `GET /api/v1/subscribers/import/template.csv`.
- **Import:** UTF-8 with or without a BOM, `,` or `;` separator, common alternative headers (`user`, `login`, `name`…). First a **preview** that says what happens to every row (new, existing, error), then the import of the chosen rows.
- Output format: UTF-8 with a BOM (Excel), every cell quoted, a cell that starts like a formula gets a leading `'` (CSV injection protection).

### 8.6 Subscriber detail

Tabs: **Basic**, **Playlist** (every access link: M3U for HLS and for HTTP-TS, EPG, Xtream data, an HTTP-TS link per channel; copy buttons), **Access** (IPs, countries, MAC), **Devices** (§12.3), **History** (audit, §19). The **Sessions** button opens who watches now.

---

## 9. Packages

Menu **Streams → Packages** (TVR-038, FR-PKG-01/02).

- A package is a named, **ordered** list of channels. A subscriber sees the union of the channels of their active packages.
- Fields: name, description, **Active** (an inactive package gives no access), **Catch-up** (channels TimeShift records get archive in the playlists of this package — §10.4).
- Channels are added by search (they go to the end), ordered by drag and drop and saved with **Save order**. Unsaved changes are guarded against leaving by accident.
- Deleting a package only removes the links — the channels stay.
- Playlists are built on every request, so a package change applies at the player's next refresh.
- API: `/api/v1/packages`, `GET/PUT /api/v1/packages/{id}/streams`.

---

## 10. Playback: playlists, Xtream, EPG, catch-up

### 10.1 Personal M3U playlist (TVR-041, FR-PLY-01/02/05)

- `GET /pl/{token}.m3u?type=hls|ts[&mac=…]` on the edge address. The token is 32 base62 characters (~190 bits) — it cannot be guessed.
- An extended M3U that VLC, TiviMate, Kodi, OTT Navigator read: `url-tvg`/`x-tvg-url` to the subscriber's EPG, per channel `tvg-id`, `tvg-name`, `tvg-logo`, `tvg-chno` (LCN), `group-title`, with catch-up `catchup`, `catchup-source`, `catchup-days`.
- `type=hls` — links `/live/<slug>/index.m3u8?token=…`; `type=ts` — `/ts/<slug>.ts?token=…` (only channels with an HTTP-TS output).
- `mac=` is copied into every link (for the MAC lock).
- Links point to the **least loaded edge** (§13.7), and the subscriber stays on it while it is available and not full.
- The playlist is built from the database on every request — a change of packages, channels or status applies at once. The edge checks access to the channel itself again (§11.1).

### 10.2 Xtream Codes API (TVR-076, FR-AUT-05)

For players that ask for host, username and password (TiviMate, IPTV Smarters, OTT Navigator…):

- `/player_api.php` (login, categories, channels), `/get.php` (M3U), `/xmltv.php` (EPG), `/live/{user}/{pass}/{id}.ts|m3u8` and the short forms.
- The same subscriber, packages and channels as the playlist; channels have numeric ids.
- The password is Argon2id (slow on purpose), and players call the API many times on start and on every channel switch. So a checked username/password pair is remembered for a short time (while the password is unchanged); wrong passwords are counted per client (IPv6 by its /64) and username, and above the limit the request is refused without checking; from many addresses for one username there is only a throttle, not a lock — so guessing from many addresses does not lock the subscriber's own devices out. Concurrent Argon2 checks are limited: under a flood the player gets 503 and retries. On the edge the redirect of Xtream addresses is cached by the SHA-256 of the URL, so username and password do not end up in the cache files.
- A subscriber without a password has no Xtream access.

### 10.3 EPG (TVR-077, FR-PLY-03)

- **Settings → SMTP, Telegram, EPG** sets one or more XMLTV sources (plain or `.gz`) and the refresh interval (default 6 h; on failure a retry after 15 min).
- Core downloads the sources (up to 120 s and 300 MB each), keeps only `<channel>` and `<programme>` of the installation's channels (by `tvg-id`/EPG ID), from yesterday to 8 days ahead, and keeps the last good copy on disk.
- Every subscriber gets **their own** XMLTV with their channels only: `/pl/{token}.xml` and `xmltv.php`.

### 10.4 Catch-up from a TimeShift server (TVR-132)

TvRestream has no archive (DVR) of its own. It integrates with a separate **TimeShift** server that records channels:

- **Settings → TimeShift (catch-up):** enabled, TimeShift API address, integration key, public address for players, refresh interval (5 min), Xtream login (username/password or playlist token), "stale after" (6 h, expert).
- Core reads the TimeShift channel list (`GET {api-url}/api/integration/v1/channels`, with the key and `If-None-Match`) at start, every 5 min and when TimeShift reports a change (`POST /api/timeshift/v1/events`). Channels are matched by `tvrSlug`.
- Playlists of packages with **Catch-up** get the `catchup` attributes, already with the subscriber's credentials. A playlist never waits for TimeShift — it uses the cached list; the last good answer is also kept in a file (`TVR_TIMESHIFT_CACHE_FILE`) and is used for 6 h after the last successful contact.
- TimeShift checks the subscriber's rights with `GET /api/timeshift/v1/access?token=…` (or `username`/`password`) — the same rules as the playlist.
- The panel: a **Catch-up** column in the channel list (days of archive / "not recorded"), the **Check connection** button (how many channels, how many with catch-up, how many matched, which have no stream in TvRestream).

---

## 11. Playback protection

### 11.1 The `/internal/auth` check (TVR-042, FR-AUT-01/03)

The nginx edge asks core about every playlist (`auth_request`). Core checks in order:

1. a valid token →
2. status (not suspended/banned) →
3. the last day has not passed →
4. the channel is in an active package of the subscriber →
5. the IP is in the allowed networks →
6. the country is allowed (GeoIP) →
7. MAC lock →
8. device limit.

On refusal the edge gets the reason in `X-Deny-Reason`:

| Reason | HTTP | When |
|---|---|---|
| `TOKEN` | 401 | No token, a malformed one, or the old one after a new token |
| `EXPIRED` | 403 | Expired or the last day has passed |
| `SUSPENDED` / `BANNED` | 403 | Suspended / banned |
| `NOT_IN_PACKAGE` | 403 | The channel is not in an active package, does not exist or is deleted |
| `IP` | 403 | IP outside the allowed list |
| `GEO` | 403 | The country is not allowed or cannot be told |
| `MAC` | 403 | The subscriber is MAC-locked and the request carries none of the addresses |
| `LIMIT` | 403 | A new device over the limit, or a device kicked by a newer one |

- A token's rights are cached in Redis for 10 s (**Access check cache**, 1 s–5 min); an unknown token is cached too, so a flood of bad tokens does not load the database. The edge caches the answer by a hash of the URI, IP, user agent and MAC cookie — without the token in the cache.
- Edge and core recognize each other by the **internal token** (`TVR_INTERNAL_TOKEN`, required outside dev); a wrong internal token raises the `TvrPlaybackAuthInternalToken` alarm.
- When Redis is down, playback is allowed (logged) — the limit is a business rule and must not stop playback.

### 11.2 Signed HLS segments (TVR-044, FR-AUT-02, ADR-005)

- The edge (njs) fetches the playlist from the origin and appends `?s=<signature>&e=<expires>&d=<session>` to every segment. The signature is HMAC-SHA256 with the **signing key** over "expires|slug|session".
- Segments are checked **locally**, without core: a missing/forged signature → 403, an expired one → 410.
- The signature covers the channel and the device (session), not one segment — a segment URL is worthless for another channel and after expiry. All copies of a republished playlist count against the segment rate of **one** device (`limit_req`), so one playlist cannot feed an unlimited number of viewers.
- **Signature lifetime:** 10 s–1 h (default 1 min).
- **Key rotation** from the panel: the previous key is accepted while its signatures are valid, then core removes it — viewers do not break.

### 11.3 Binding segments to the viewer's IP (TVR-098, FR-AUT-06)

Optional (**Bind segments to the viewer's IP**): the signature also covers the address that fetched the playlist (an IPv4 address as is, IPv6 its /64 prefix, because the privacy addresses of one device change within it). A copied URL from another address → 403. URLs signed before the setting changed stay valid until they expire. Not suitable for subscribers whose IP changes often (mobile networks with CGNAT).

### 11.4 IP and country restrictions (TVR-097, FR-SUB-06)

- **IP:** the subscriber's list of CIDR networks.
- **Country:** needs a MaxMind database (GeoLite2/GeoIP2 Country or City, `.mmdb`, `TVR_GEOIP_DATABASE`). `country` is used, otherwise `registered_country`. A newer file (e.g. from `geoipupdate`) is loaded without a restart (checked every 60 s). Without a database, subscribers with countries get `GEO`.

### 11.5 MAC lock (TVR-104, FR-DEV-02)

Only boxes with the listed MAC addresses can play. The box sends its MAC as the `mac` cookie (MAG) or as `&mac=` in the link (Enigma2; the playlist copies it into every link). Addresses are normalized to `00:1A:79:AB:CD:EF` however they are typed (`00-1a-79…`, `001A79…`, `001a.79ab.cdef`). Applies to HTTP-TS connections too. A MAC address can be spoofed — this is a barrier against casual sharing, not cryptographic protection.

---

## 12. Sessions and devices

### 12.1 Sessions (TVR-066, FR-SES-01/02/04)

- **Session = one device of the subscriber:** the key is a hash of subscriber + IP + user agent, so zapping on the same device is the same session.
- A device counts as watching for 30 s after its last request that reached core (**Session window**, 10 s–10 min). With the edge's 10 s cache, a watching device reaches core at least every ~10 s.
- Menu **Subscribers → Sessions**: who watches now (subscriber, channel(s), IP, device, since when), close one session or all of a subscriber (Admin and Support). The player stops within ~10 s, the device is refused for 2 min (**Block after kick**).
- **90-day history** (`TVR_SESSIONS_RETENTION`), synchronized every 10 s.
- API: `GET /api/v1/sessions`, `/sessions/history`, `DELETE /api/v1/sessions/{subscriberId}/{sessionKey}`, `POST /api/v1/subscribers/{id}/actions/kick`.

### 12.2 Device limit (TVR-045, TVR-067, BR-06)

- **Reject the new device** (`REJECT_NEW`): a new device over the limit gets `LIMIT`.
- **Kick the oldest session** (`KICK_OLDEST`): the session that started first is closed and that device is refused for 2 min — otherwise it would come back at its next request and kick the new one.
- The event is also sent as the `session.limit_exceeded` webhook (at most once per 5 min per subscriber).

### 12.3 Devices (TVR-067, FR-DEV-01)

The subscriber's **Devices** card lists the known devices (user agent, last IP, first and last seen, whether it watches now). Deleting a device also closes its session if it watches now; it is listed again when it plays a channel.

---

## 13. Servers and cluster

Menu **Platform → Servers**.

### 13.1 List and metrics (TVR-035, FR-NOD-03)

Per server: role, status (Online/Offline/Maintenance), last seen, agent and FFmpeg versions, CPU, memory, load average (with the core count), network in/out, free space in the work directory (tmpfs), number of channels and FFmpeg processes, lost UDP packets. Metrics are kept in Redis with an expiry — a silent agent shows no stale values.

### 13.2 Adding a server (TVR-070, FR-NOD-01, SEC-02)

1. **Add server** creates a one-time key valid for 1 h and shows the install command.
2. On the new server `install-agent.sh` (downloaded from core without a login) fetches `tvr-agent.jar` and the CA certificate, sends the key with a CSR and gets back an **agent certificate** signed by the cluster's private CA (valid 3 years).
3. The agent connects over **mTLS** on port 9443. Core accepts only the current certificate of an existing server.

The private CA is created on first use and kept in the database, so every core instance shares it. The certificate of the mTLS port can be replaced without restarting core. An alternative for simple installs: a shared token (`TVR_AGENT_ENROLL_TOKEN`) on the plain gRPC port 9090.

### 13.3 Editing a server (TVR-071/072/129)

- **Name** (letters, digits, `. _ -`).
- **Public address** — where players reach the edge (not the panel): host, `host:port`, `[ipv6]:port`, optionally with `https://`.
- **Outgoing traffic limit (Mbit/s)** — for balancing between edge servers (§13.7).

### 13.4 Maintenance (drain) (TVR-071, FR-NOD-04)

- **Ingest server:** its channels move to the least loaded online ingest servers (channels/maximum, then count, then name); no new ones start on it.
- **Edge server:** gets no new viewers in playlists; those watching now continue.
- **End maintenance:** the server takes channels and viewers again. Moved channels do **not** return by themselves.

### 13.5 Moving channels (TVR-071, FR-STR-09)

Selected channels → **Move…** → server. The channels stop on the current server and start on the chosen one; their settings do not change. API: `POST /api/v1/streams/actions/move`.

### 13.6 Automatic move when an ingest server fails (TVR-100, FR-NOD-05)

- When an ingest server with enabled channels has no heartbeat for **45 s** (`TVR_CLUSTER_AUTO_MOVE_AFTER`, at least 20 s), its channels go to the least loaded online ingest servers with room. A channel is back on air about 45 s + a few seconds after its server went silent.
- Channels that work only on that server **do not move**: an RTMP push input, an SRT listener input or output, a UDP input/output with a local address or interface, a DVB input. A manual move or a drain still moves them — there the administrator decides.
- A server that was never connected has not failed. Right after core starts nothing moves for 45 s (the agents must reconnect first).
- Channels do **not** move back when the server returns.
- What does not fit stays and is tried again. The result goes to the log and to the `node.channels_moved` webhook. Turned off with `TVR_CLUSTER_AUTO_MOVE=false`.

### 13.7 Balancing between edge servers (TVR-072, TVR-129, FR-NOD-06)

- Edges are EDGE and ALL_IN_ONE servers with a public address, online and not in maintenance. Without any, links use **Public address for subscribers** (single server).
- An edge with a limit is **full** when its outgoing traffic averaged over 30 s is at least the **threshold** (default 80 %, 10–100) of the limit. An edge without a limit is never full.
- A new subscriber gets: non-full edges first, then full ones, then those without a report; within a group the lowest load, then the fewest subscribers, then the name.
- **Stickiness:** a subscriber keeps their edge while it is available and not full; subscribers of a full edge move with their next playlist refresh.
- When every edge with a limit has been above the threshold for a minute, the `EDGE_SATURATED` alert fires (§14).
- Without Redis the choice is a stable hash of the subscriber.

### 13.8 Removing a server

The certificate is revoked and the agent stops connecting (also with the shared token). The server goes to the **Removed servers** list; **Allow again** lets it enroll again.

### 13.9 Restarting the agent and the channels of a server (TVR-082)

From **Settings**: **Restart channels** (of the server) and **Restart agent** — the agent stops its channels and exits with a non-zero code; systemd starts it again.

### 13.10 Overload protection (BUG-039)

- FFmpeg processes start gradually (a starts-per-second limit), not all at once when an agent with many channels starts.
- No new FFmpeg starts while free memory is below the threshold.
- Every FFmpeg gets a high `oom_score_adj`: under memory pressure the kernel kills one FFmpeg (that channel restarts) instead of the agent.
- The **Server memory** alert rule (§14).

---

## 14. Alerts and incidents

### 14.1 Rules (TVR-068, FR-ALR-01)

Menu **Platform → Alerts**. Rules are checked every 5 s.

| Rule | Threshold | Default |
|---|---|---|
| Channel offline (`STREAM_OFFLINE`) | seconds in OFFLINE/ERROR | 10 |
| Failover (`STREAM_FAILOVER`) | — (every input switch) | |
| CC errors (`STREAM_CC_ERRORS`) | errors per minute | 100 |
| Content (`STREAM_CONTENT`) | seconds beyond the agent's wait; 0 = at once | 0 |
| Server offline (`NODE_OFFLINE`) | seconds without a report | 30 |
| Server CPU (`NODE_CPU`) | % | 90 |
| Server network (`NODE_NIC`) | Mbit/s out | 900 |
| Server memory (`NODE_MEMORY`) | % RAM | 90 |
| Edges saturated (`EDGE_SATURATED`) | — (threshold from the settings) | |

A channel rule can watch all or selected channels; every rule sends to selected alert channels or to all enabled ones.

### 14.2 Alert channels (FR-ALR-02)

- **Telegram:** bot token (from @BotFather; never shown again) and chat ID (add the bot to the group first).
- **E-mail:** up to 20 recipients; the SMTP server is in **Settings → SMTP, Telegram, EPG** (STARTTLS, TLS or none).
- **Webhook:** `POST` with JSON `{source, title, text, at}`.
- **Quiet hours** (`HH:mm`–`HH:mm`, in `TVR_ALERTS_ZONE`): nothing is sent to the channel then — incidents stay in the panel.
- A **Test** button per channel. Alert texts are in Bulgarian.

### 14.3 Incidents (TVR-069, FR-ALR-03/04)

Menu **Platform → Incidents**.

- The first alert of a rule for a channel/server **opens an incident** and is sent.
- Repeats while it is open are counted and sent **grouped** at most every 10 min.
- A condition that still holds is **reminded** every 30 min until acknowledged.
- A condition clear for 30 s **resolves** the incident with a recovery notice; a failover incident resolves after 10 min without a new failover.
- **Acknowledge** stops grouped messages and reminders. **Mute** a channel/server for a period — no alerts and no recovery notice.
- Resolved incidents are kept 90 days.

---

## 15. Dashboard and real-time events

### 15.1 Dashboard (TVR-034, FR-MON-01)

Channels by state, viewers now, outgoing traffic of all servers, active subscribers, **problem channels**, servers (how many online, output, last seen, version), latest events of 24 h. API: `GET /api/v1/dashboard/summary`.

### 15.2 Real-time events (TVR-030, FR-MON-02)

`GET /api/v1/events` (Server-Sent Events) — channel states, events, servers. Every core instance gets them through Redis pub/sub, so it also works with several cores. Filter `?types=stream.status,node.*`. A slow client 1000 events behind is disconnected; a connection lasts at most 1 h (or until the token expires). Server events are not shown to Support. When the connection drops, the panel shows "Real-time connection lost, reconnecting…" and reconnects.

---

## 16. Statistics and reports

Menu **Subscribers → Statistics** (TVR-080, FR-STA-01..05). For a range of whole days in Sofia time:

| Report | What it shows |
|---|---|
| **Top channels** (`top-streams`) | Viewer-minutes (watch time), viewers, traffic per channel; deleted channels are marked |
| **Peak viewers** (`peak-viewers`) | Peak of concurrent viewers for the range and per day (with the time), watch time, traffic |
| **Uptime** (`uptime`) | Percent of time on air in total and per channel, number of incidents |
| **Subscribers** (`subscribers`) | Total and by status, new and expired per day, expiring soon |

Viewers and uptime are counted every minute; minute data is rolled up hourly every 10 min and kept **400 days**. Every report is also exported as CSV (`/api/v1/reports/{type}/export.csv`).

---

## 17. Admins, roles and panel login

### 17.1 Roles (TVR-048, FR-ADM-02)

| | Channels | Subscribers | Servers | Admins, API keys, webhooks, audit, archive, settings |
|---|---|---|---|---|
| **Admin** | full | full | full | yes |
| **Operator** | full | read | read | no |
| **Support** | read* | full, no delete | — | no |
| **Viewer** | read* | read** | read | no |

\* without the inputs — their addresses carry provider credentials (and without the channel log). \*\* without the subscriber's token and link — a Viewer could otherwise watch as the subscriber.

Menu **Platform → Admins**: create, role, active, new password, delete. An admin cannot change their own role, deactivate or delete themselves, and the last active Admin cannot be removed. A change of role, active flag or password revokes the user's refresh tokens; a change of role or deactivation also stops their current access token at the next request (the token is checked against the database) and ends their real-time streams.

### 17.2 Login and sessions (TVR-006, FR-ADM-01)

- An access token (JWT) for 15 min (1 min–1 h) and a refresh token in an httpOnly cookie for 7 days (1 h–90 days), rotated on every refresh. Expired refresh tokens are deleted every night.
- The initial admin is created from `TVR_ADMIN_EMAIL` / `TVR_ADMIN_PASSWORD`, only if the database has no users.

### 17.3 Two-factor login (TOTP) (TVR-073, FR-ADM-03)

- Account menu → **Two-factor login**: a QR code for Google Authenticator, Microsoft Authenticator, Authy, 1Password; turned on with the first code. Codes are 6 digits per 30 s and accepted one step before or after.
- **Required for Admin** (`TVR_ADMIN_2FA_REQUIRED`): an admin without 2FA can only set it up before using anything else.

### 17.4 Lockout after failed attempts (FR-ADM-04, BUG-012, BUG-042)

- 5 wrong passwords/codes in a row for one e-mail **from one address** lock that e-mail for that address only for 15 min.
- 50 failures from any addresses lock the account (then only password + 2FA code).
- 30 failures from one address for any e-mails lock the address.

---

## 18. API keys and webhooks

### 18.1 API keys (TVR-075, FR-API-02)

Menu **Platform → API keys** (Admin only, panel only). For integrations: billing, CRM, scripts.

- Name, scopes, expiry. The key (`tvr_…`) is shown **once**; only its SHA-256 is stored. Revocable at any time.
- Request: header `X-API-Key: tvr_…`. A wrong, expired or revoked key → 401 at once.
- Scopes: `streams:read|write`, `packages:read|write`, `subscribers:read|write`, `sessions:read|write`, `nodes:read`, `alerts:read|write`, `audit:read`. `:write` includes `:read`. A key can never manage admins, keys, webhooks, settings or the archive.
- A key has no role: it sees channel inputs only with `streams:write`, subscriber tokens only with `subscribers:write`.

### 18.2 Webhooks (TVR-095, FR-API-04)

Menu **Platform → Webhooks** (Admin only).

- Address and events: `stream.status_changed`, `stream.failover`, `subscriber.created`, `subscriber.updated`, `subscriber.expired`, `subscriber.deleted`, `session.limit_exceeded`, `node.status_changed`, `node.channels_moved`.
- Every delivery is a `POST` with `X-TVR-Event`, `X-TVR-Delivery`, `X-TVR-Timestamp` and **`X-TVR-Signature: sha256=<hex>`** — the HMAC-SHA256 of the raw body with the webhook's secret. Core generates the secret, shows it once, and **Rotate secret** replaces it.
- The receiver must answer 2xx within 5 s. Otherwise retries after 1 min, 5 min, 30 min, 2 h, 6 h; then the delivery is `FAILED`.
- A list of deliveries (status, attempts, answer) and **Redeliver**; a **Test** button. Deliveries are kept 7 days.
- Subscriber events come from the audit log — so they are sent for every change, whether from the panel, an API key, the CSV import, the bulk extend or the nightly expiry.
- `TVR_OUTBOUND_DENY` applies to webhook addresses too.

---

## 19. Audit log

Menu **Platform → Audit log** (TVR-074, FR-AUD-01/02; Admin only).

- Who (the admin's e-mail or the key; "system" for automatic actions), what (object and action), when, from which IP, with a **before/after diff** of the changed fields only.
- Secrets (passwords, SRT passphrases, stream keys, bot tokens) are **never** shown — only that they changed.
- Filters by range, admin, object type, action, text; CSV export (up to 100,000 rows).
- The admin's IP behind nginx is taken from `X-Forwarded-For`/`X-Real-IP` only from trusted proxies (`TVR_AUDIT_TRUSTED_PROXIES`).
- The **History** tab of a channel and a subscriber is the audit of that object.
- Kept **1 year** (`TVR_AUDIT_RETENTION`; 0 = forever).

---

## 20. Import and configuration archive

### 20.1 Channel import from M3U (TVR-049, FR-CFG-02)

Menu **Streams → Import**. The address (`http(s)`) or a file of the provider's playlist (M3U / M3U Plus): `#EXTINF` with the `tvg-*` attributes and `group-title`, `#EXTGRP`, the VLC/Kodi HTTP options. The **preview** says what happens to every channel (new, a backup input for an existing channel, skipped); the chosen ones are imported. The slug is generated from the name; missing categories are created. The download is limited to 20 MB, http(s) only, and goes through the SSRF protection (on every redirect too).

### 20.2 Import from Cesbo Astra (TVR-078, FR-CFG-03)

Astra's JSON configuration (`astra.conf` or the export of its web panel), the `make_stream` array. Every stream becomes a channel: the first readable input is the primary, the others backups in their order (priorities 1..8); UDP and HTTP outputs are carried over. Astra options after `#` are translated when there is an equivalent (`pnr` → PNR, `ua` → User-Agent, etc.). Preview and selection as for M3U.

### 20.3 Archive: export and import of the whole configuration (TVR-079, FR-CFG-04)

Menu **Platform → Archive** (Admin only).

- The whole configuration in **one JSON file**: categories, channels with inputs and outputs, packages, subscribers (optional, with their tokens and password hashes), alert rules and channels, the changed settings (without secrets and without env file values) — for safekeeping or moving to another installation.
- Deliberately **not** included: admins and their 2FA secrets, API keys, servers (they enroll again), sessions, metrics, audit and incidents. Rows refer to each other by natural keys, so the file imports into an empty database too.
- The file is **encrypted with a password** (it holds provider credentials and, optionally, subscribers).
- Import into the same or an empty installation: rows are matched by natural key (category, package and alert channel by name; channel by slug; subscriber by username); an existing row is skipped or overwritten, as chosen.
- This is **not** a database backup — see §22.2 for that.

---

## 21. Settings and server health

Menu **Platform → Settings** (TVR-082; Admin only). Every stream and network parameter — without editing the env file, `application.yml` or sysctl by hand.

### 21.1 How values work

- **Precedence:** a variable of the env file (`deploy/.env` or `/etc/tvrestreamer/tvrestreamer.env`) → the per-server value → the global panel value → the built-in default. A value from the env file is read-only in the panel ("set in the env file") — remove it there to change it from the panel.
- For agent settings, the agent's own `TVR_AGENT_*` variable on its server also wins.
- Every value shows **when it takes effect**: at once; at the next channel restart; needs an agent / core / edge restart; for new channels.
- Some values are set **per server** (the UDP chain, the agent, reconnect).
- Secrets are never shown — only "configured"; a **Generate** button for keys.
- Every change is audited. API: `GET /api/v1/settings`, `PUT|DELETE /api/v1/settings/{key}`, `POST /api/v1/settings/{key}/generate`.

### 21.2 Tabs and keys

| Tab | Keys (default) |
|---|---|
| **Network and addresses** | `playback.subscriber-url` public address for subscribers; `playback.base-url` (`http://localhost:8090`); `edge.port` (8081); `edge.bind`; `edge.load-threshold-percent` (80); `core.http-port` (8080); `core.http-bind`; `core.grpc-port` (9090); `cluster.grpc-address`; `cluster.public-url` |
| **Protected playback** | `playback.signing-key`; `playback.sign-ttl` (1m); `playback.sign-bind-ip` (false); `access.cache-ttl` (10s); `access.session-window` (30s); `access.kick-block` (2m); `subscriber.default-max-connections` (1); `subscriber.default-limit-policy` (`REJECT_NEW`); `access.internal-token` (env only) |
| **HLS and HTTP-TS defaults** | `hls.segment-sec` (4); `hls.list-size` (6); `hls.delete-threshold` (2); `output.http-ts` (false) |
| **Input defaults** | `input.timeout-sec` (5); `input.analyzeduration` (1.5); `input.reconnect` (false); `input.user-agent`; `input.headers` |
| **Failover** | `failover.cc-errors-per-min` (0); `failover.min-bitrate-kbps` (0); `failover.on-no-video`/`on-no-audio`/`on-scrambled` (true); `failover.hold-sec` (5); `failover.return-to-primary` (true); `failover.return-after-sec` (30); `failover.slate` (true); `failover.slate-text` |
| **UDP chain (expert)** | `agent.udp.switcher-rcvbuf`/`switcher-sndbuf` (4 MB); `agent.udp.write-low` (4 MB) / `write-high` (8 MB); `agent.udp.ffmpeg-buffer-size` (4 MB); `agent.udp.ffmpeg-fifo-size` (50000); `agent.udp.pkt-size` (1316); `agent.udp.switcher-threads` (2) |
| **Agent and processes** | `agent.process.backoff-min` (1s) / `backoff-max` (30s); `agent.process.stable-after` (30s); `agent.process.stop-timeout` (1500ms); `agent.process.log-lines` (1000); `agent.ffmpeg-path` (`ffmpeg`); `agent.shutdown-grace` (10s); `probe.ffprobe-path` (`ffprobe`) |
| **Core ↔ agent** | `link.heartbeat-interval` (5s); `link.offline-after` (15s); `link.desired-resync` (30s); `agent.link.reconnect-min` (1s) / `reconnect-max` (30s) |
| **Panel sessions** | `security.access-token-ttl` (15m); `security.refresh-token-ttl` (7d); `security.refresh-cookie-secure` (true); `security.jwt-secret` |
| **SMTP, Telegram, EPG** | `smtp.host`, `smtp.port` (587), `smtp.security` (STARTTLS), `smtp.username`, `smtp.password`, `smtp.from`; `telegram.api-url`; `epg.url`; `epg.refresh` (6h) |
| **TimeShift (catch-up)** | `timeshift.enabled`, `timeshift.api-url`, `timeshift.key`, `timeshift.public-url`, `timeshift.refresh` (5m), `timeshift.xtream-auth` (`USERNAME_PASSWORD`/`TOKEN`), `timeshift.stale-after` (6h) |

### 21.3 Buttons

| Button | What it does |
|---|---|
| **Apply to all channels** (HLS) | Sets the HLS values on every channel; their packaging restarts. |
| **Enable HTTP-TS on all channels** | No channel restart. |
| **Reload edge** / **Show generated config** | Core writes the edge's nginx include files (port, signing key and the previous key, lifetime, IP binding) into `TVR_EDGE_CONFIG_DIR`; nginx reloads (`.deb`: `tvr-edge-reload.path`; Docker: the container itself). Without `TVR_EDGE_CONFIG_DIR` core does not manage the edge. |
| **Reachable from the network?** | Checks that the public subscriber address answers (status, ms). |
| **Restart channels** / **Restart agent** | §13.9. |
| **Check connection** (TimeShift) | §10.4. |

### 21.4 Server health

Tab **Server health**: every agent reports every 30 s `net.ipv4.ip_forward`, `net.core.rmem_max`/`wmem_max`, free memory, swap, TCP ports in LISTEN, the work directory and the settings the agent really uses (with their source: env/core/default). Every check is OK, WARN or ERROR with the **exact fix command**. Core and the agent never change sysctl themselves — that needs root.

---

## 22. Monitoring, backup and operations

Details in the [README](https://github.com/kocetod/TVRestreamer/blob/main/README.md) (sections TVR-084…087), document 08 and the [administrator guide](https://github.com/kocetod/TVRestreamer/blob/main/guides/admin-guide.md). In short:

### 22.1 Prometheus metrics (TVR-081, FR-MON-03)

- **Core** `GET /actuator/prometheus`: `tvr_streams{state}`, `tvr_streams_input_pinned`, `tvr_nodes`, `tvr_sessions_active`, `tvr_subscribers`, `tvr_failovers_total`, `tvr_stream_events_total`, `tvr_playback_auth_seconds`, `tvr_admin_logins_total`, `tvr_state_last_refresh_timestamp_seconds` + JVM. Access from `TVR_PROMETHEUS_ALLOW` (localhost by default) or with `TVR_PROMETHEUS_TOKEN`.
- **Agent** `http://127.0.0.1:9464/metrics`: `tvr_agent_processes{state}`, `tvr_agent_process_restarts_total`, `tvr_agent_streams_running`, `tvr_agent_failovers_total`, `tvr_agent_relay_clients`, `tvr_agent_rtmp_connections`, `tvr_agent_core_connected`, `tvr_agent_core_connects_total`, `tvr_agent_udp_rcvbuf_*`, `tvr_agent_switcher_dropped_datagrams_total` + JVM. With a token it may listen beyond loopback.
- **Grafana** dashboards and **Alertmanager** rules (Telegram/e-mail) — `deploy/docker-compose.monitoring.yml`, `deploy/grafana`, `deploy/prometheus/rules` (TVR-086). Every alarm has a runbook in the administrator guide.

### 22.2 Database backup (TVR-087)

`scripts/tvr-backup.sh` / `tvr-restore.sh` and the systemd timer `tvr-backup.timer`: `pg_dump`, encryption with `age`, an off-site copy (rsync/rclone), a restore test. Alarms `TvrBackupFailed`, `TvrBackupTooOld`, `TvrBackupOffsiteFailed`.

### 22.3 Installation options

A `.deb` package for Ubuntu (one server, with the edge and a `deploy/.env` wizard), Docker images and `docker-compose.prod.yml` (TVR-084), Ansible roles for a cluster (TVR-085), `install-agent.sh` for an additional server (TVR-070).

### 22.4 Load and chaos tests (TVR-088/089/090/065)

2,000 virtual viewers, a 24-hour soak test with 250 channels, chaos tests for core/edge/ingest and failover — see [load-testing.md](https://github.com/kocetod/TVRestreamer/blob/main/guides/load-testing.md).

---

## 23. Data security

- **Secrets encrypted in the database** (BUG-041, SEC-05): input passwords and headers, SRT passphrases, RTMP stream keys, bot tokens, webhook secrets and more are stored with AES-256-GCM and the installation key `TVR_DATA_KEY` (64 hex characters or base64 of 32 bytes; or a file — a Docker secret). **Keep a copy of the key outside the server** — without it this data cannot be read. Old plain-text values are encrypted at the first start with a key.
- **ProductionGuard** (BUG-029, SEC-03): a server without `SPRING_PROFILES_ACTIVE` is treated as production. Outside `dev` core refuses to start without `TVR_JWT_SECRET` (≥ 32 characters), `TVR_INTERNAL_TOKEN` and `TVR_DATA_KEY`/`TVR_DATA_KEY_FILE` — the error is the first and only thing in the log.
- **SSRF:** addresses users type (input test, M3U import, webhooks, webhook alert channels) go through `TVR_OUTBOUND_DENY`; redirects are followed by hand and checked.
- **Audit without secrets**, CSV without formulas, logos without scripts — see §19, §8.5, §2.6.

---

## 24. Reference: ports, retention, addresses

### 24.1 Ports

| Port | Who | What for |
|---|---|---|
| 8080 | core | REST API and panel (behind nginx) |
| 9090 | core | gRPC for agents (shared token) |
| 9443 | core | gRPC for agents with mTLS |
| 8081 | edge | HLS/HTTP-TS for players (`edge.port`) |
| 8090 | agent | HTTP-TS relay — from edge servers only |
| 1935 | agent | RTMP ingest (push inputs) — for encoders only |
| 9464 | agent | Prometheus metrics (127.0.0.1 by default) |
| 20000–29999 | agent | UDP/TCP loopback between channel processes |
| chosen | agent | SRT listener inputs and outputs (UDP) |

### 24.2 Retention

| Data | How long | Variable |
|---|---|---|
| Per-minute channel metrics | 48 h | `TVR_METRICS_RETENTION` |
| Hourly statistics for reports | 400 days | `TVR_STATS_RETENTION` |
| Channel events | 180 days | `tvr.events.retention` |
| Session history | 90 days | `TVR_SESSIONS_RETENTION` |
| Resolved incidents | 90 days | `TVR_INCIDENTS_RETENTION` |
| Webhook deliveries | 7 days | `TVR_WEBHOOKS_RETENTION` |
| Audit log | 1 year | `TVR_AUDIT_RETENTION` |
| Channel log | last 1000 lines, on the agent only | `agent.process.log-lines` |

### 24.3 Addresses for subscribers and integrations

| Address | What for |
|---|---|
| `/pl/{token}.m3u?type=hls\|ts[&mac=]` | Personal playlist |
| `/pl/{token}.xml` | Personal EPG |
| `/live/<slug>/index.m3u8?token=` | HLS of a channel |
| `/ts/<slug>.ts?token=` | HTTP-TS of a channel |
| `/live/<slug>/thumb.jpg` | Thumbnail (for the panel) |
| `/logos/{id}.png\|svg?v=` | Logo |
| `/player_api.php`, `/get.php`, `/xmltv.php`, `/live/{user}/{pass}/{id}.ts` | Xtream Codes API |
| `/api/timeshift/v1/access`, `/api/timeshift/v1/events` | For the TimeShift server (with the key) |
| `/api/v1/agent/install.sh`, `/ca.pem`, `/tvr-agent.jar`, `/enroll` | For a new server |
| `/internal/auth` | For the edge only (with the internal token) |
| `/api/v1/...` | REST API — see document 05 and [openapi.yaml](https://github.com/kocetod/TVRestreamer/blob/main/api/openapi.yaml) |

---

## 25. Not implemented yet

| Ticket | What | Note |
|---|---|---|
| TVR-102 | SDT/PMT rewriting (service name, provider) | The **Service name** field is in the form but not applied |
| TVR-103 | Control plane high availability | Several core instances already share the Redis events and the CA, but there is no automatic takeover |
| TVR-107 | Billing integration | Meanwhile: API keys and webhooks |
| TVR-108 | Reseller panel | |
| TVR-109 | CAS/BISS descrambling | A scrambled stream is detected and can trigger failover, but is not descrambled |
| TVR-110/111 | License server and license check | |
| TVR-116–118 | Expiry notifications to subscribers | |
| TVR-130 | Edge router | Balancing today works through the address in the playlist (§13.7) |
| — | DVR / own archive | Out of scope; catch-up through an external TimeShift (§10.4) |
