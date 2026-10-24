# Development plan

This plan turns [RESEARCH.md](RESEARCH.md) into work a session can pick up
cold. Read RESEARCH.md first: it has the API, the hardware tables and the
design decisions.

## Environment prerequisites

- **Network access** for the build environment needs these hosts:
  - PlatformIO: `api.registry.platformio.org`,
    `api.registry.nm1.platformio.org`, `dl.registry.platformio.org`,
    `collector.platformio.org`
  - Espressif: `dl.espressif.com`, `github.com` and its release-asset hosts
    (toolchains)
  - Cyberspace: `api.cyberspace.online` (API docs and smoke tests) and
    `*.firebasedatabase.app` (RTDB)
- **Tools:** `pip install platformio` (6.x). Use `platform = espressif32@7.0.1`,
  as camillia-mt does.
- **Hardware source:** camillia-mt (`oumike/camillia-mt`), cloned alongside
  this repo for copying the HAL.

## Conventions

- Every change goes through a pull request into `main`.
- CI builds all three envs on each PR push (`.github/workflows/build.yml`;
  add it in M1).
- Board-specific code is gated on `DEVICE_*` macros from `src/hal/board.h`,
  as in camillia-mt. Shared code doesn't test board names.
- Keep logic that needs no hardware in pure C++ (no Arduino headers), and
  test it on the host with `pio test -e native`. That covers the SSE parser,
  RTDB merge, markdown subset, rate limiter, JSON model parsing and the auth
  state machine.
- The API terms come first (RESEARCH §2.1): writes happen only on a user
  action, and there's no background crawling.

## Repository layout (target)

```
platformio.ini            envs: crowpanel-35, crowpanel-50, tdeck-plus, native
partitions_16mb.csv       app0/app1 (OTA-ready) + nvs + littlefs
src/
  main.cpp                setup()/loop(): board power, display, LVGL, tasks
  lv_conf.h               trimmed from camillia-mt
  hal/
    board.h               DEVICE_* selection, includes hw_*.h
    hw_crowpanel_35.h     copied from camillia-mt, radio/GPS sections kept as
    hw_crowpanel_50.h       documentation but unused
    hw_tdeck.h
    display.h             LGFX class: ILI9488 (3.5"), ST7789 (T-Deck)
    crowpanel_50_display.h  RGB panel + CrowPanelBoardCtl
    input.h/.cpp          LVGL indevs: GT911 touch; T-Deck keyboard (keypad)
                          + trackball (encoder); BLE keyboard (later)
    power.h/.cpp          backlight, idle dimming, T-Deck battery
  net/
    wifi_mgr.*            STA connect, retry/backoff, SNTP
    http.*                HTTPS wrapper over WiFiClientSecure + CA bundle
    cs_api.*              REST endpoints -> model structs
    cs_auth.*             login / refresh / NVS refreshToken
    rtdb_stream.*         SSE connection per path, reconnect on token refresh
    net_task.*            FreeRTOS task + request/response queues
  core/                   pure C++, host-tested
    sse_parser.*          text/event-stream -> (event, data)
    rtdb_merge.*          put/patch at path -> local message map
    rate_limit.*          token buckets mirroring RESEARCH §2.5
    markdown.*            subset -> styled spans for LVGL
    models.*              Entry, Reply, Notification, Room, ChatMsg, Conversation
  ui/
    theme.*               fonts (incl. monospace), colours, per-board metrics
    nav.*                 stack navigator (3.5"/T-Deck), two-pane (5.0")
    scr_setup.*           WiFi + login
    scr_feed.*  scr_entry.*  scr_compose.*
    scr_circ_rooms.*  scr_circ_room.*
    scr_cmail_list.*  scr_cmail_thread.*
    scr_notifications.*
test/                     PlatformIO unit tests (native env)
```

## Milestones

Each milestone ends in a pull request that builds all three envs. **Done
when** states what has to be shown before it counts as finished. "On
hardware" means a serial log or photo on the named board.

### M1: Bring-up (display, input, LVGL)

- `platformio.ini` with the three envs.
  - Board settings, flash size, PSRAM, partitions, LovyanGFX 1.2.34 and
    LVGL 9.5.0 all come from camillia's `crowpanel-35`, `crowpanel-50` and
    `tdeck` envs.
  - Drop RadioLib, TinyGPSPlus, PubSubClient and the Crypto deps. Keep NimBLE
    out until M7.
- Copy the HAL headers. In `main.cpp`:
  - On the T-Deck, drive `BOARD_POWERON` (GPIO10) HIGH first and park
    `LORA_CS` (GPIO9) HIGH.
  - On the 3.5", leave IO45 at its default.
  - On the 5.0", run `CrowPanelBoardCtl::begin()` to detect V1.0 or V1.1+.
- LVGL display driver: partial render buffers in PSRAM, with a flush that
  pushes through LovyanGFX. For the 5.0", reuse camillia's RGB framebuffer
  path.
- Input:
  - GT911 touch on all three boards.
  - On the T-Deck, the keyboard at 0x55 becomes an LVGL keypad and the
    trackball an encoder. Port the ISR decoder from camillia's `keyboard.cpp`
    and keep its mapping (right = DOWN, left = LEFT, up = RIGHT, down = UP).
  - Screen wake on the T-Deck: trackball click only.
- Demo screen: board name, resolution, free heap and PSRAM, a button and a
  text area. The text area proves keyboard input on the T-Deck and the
  on-screen keyboard on the CrowPanels.
- CI workflow to build the three envs. Host-test scaffold (`native` env with
  one trivial test).
- **Done when:** the three envs build in CI. On each board the demo screen
  draws the right way up, touch hits where it should, and on the T-Deck keys
  and the trackball move focus and type.

### M2: WiFi, time, TLS

- `wifi_mgr`: credentials in NVS, a scan-and-pick screen with the on-screen
  keyboard, reconnect with exponential backoff, and SNTP before any TLS.
- `http`: `WiFiClientSecure` with the ESP-IDF CA bundle (no pinning), keep-alive
  for REST, timeouts, a PSRAM response buffer and an `ArduinoJson` 7 PSRAM
  allocator. Log the internal heap low-water mark during each handshake.
- `net_task`: a single owner of all networking on core 0, talking to the UI
  over FreeRTOS queues. The UI never blocks on the network.
- **Done when:** on each board an HTTPS GET to `api.cyberspace.online` and to
  the RTDB host succeeds with certificate validation on, and the internal
  heap stays above 60 KB during it.

### M3: Auth

- `cs_auth` states: `NoAccount -> LoggingIn -> Authed <-> Refreshing ->
  (Authed | NeedsLogin)`.
  - Store only `refreshToken` and `rtdbUrl` in NVS, never the password.
  - Refresh proactively at 55 min, and on a 401 refresh and retry once.
  - Handle `EMAIL_NOT_VERIFIED`: offer "resend verification".
- Login screen: email and password fields, with the password masked.
- Header bar: `@username` from `GET /v1/users/me`, WiFi state and battery
  (T-Deck).
- Host tests: the state machine, including expiry, a 401 during refresh, and
  a revoked refresh token leading to `NeedsLogin`.
- **Done when:** after a reboot the device logs in again from NVS without the
  password, and shows `@username`.

### M4: Read-only feed

- `models` and the `cs_api` parsers for entries and replies, using filter
  docs so huge bodies are truncated to a cap (say 8 KB) for list views.
- `markdown`: headings, bold, italic, inline code, code fences and
  preformatted blocks (monospace, never wrapped), block quotes, lists, links
  as underlined text, and images as `[image]`. Host-tested against
  `docs/api-reponses/*.json` samples from cyber-tui as fixtures. Write our
  own fixtures; don't copy theirs.
- Fonts: choose and convert a monospace font with box-drawing and the Latin
  ranges (for example Terminus or Cozette, if the licence allows) at 2 or 3
  sizes.
- Screens: feed list with a "load more" cursor, entry view with replies
  (oldest first, threaded by `parentReplyId`) and pull to refresh.
  Navigation: back from the header, Esc and trackball-left on the T-Deck.
- `rate_limit`: buckets for every read endpoint. When a bucket is empty, show
  "slow down" in the UI rather than sending the request.
- **Done when:** you can scroll the feed, open an entry and read its replies
  on all three boards. ASCII art lines up on the 5.0" and the 3.5"; on the
  T-Deck it scrolls sideways.

### M5: cIRC

- `sse_parser` and `rtdb_merge` (host-tested):
  - Handle `put` at `/` (the whole window), `put` and `patch` at
    `/<id>[/field]`, `null` meaning delete, `keep-alive`, `cancel` and
    `auth_revoked`.
  - Cap each window at 100 messages.
- `rtdb_stream`: one open stream per path, with a coordinated reopen after a
  token refresh. Backoff starts at 2 s and caps at 5 min. Never reconnect in
  a tight loop.
- Screens: room list (`GET /v1/circ`, online counts) and room view (history
  from `GET /v1/circ/:room`, then live updates from
  `chat_messages/<room>`). Also the user list from `chat_presence/<room>`
  with stale and idle rules, a send box, mark-read, and deleted messages
  shown as `[deleted]`.
- Presence: heartbeat every `heartbeatMs` while the room is open, send
  `lastActivity` from the input timestamps, and `DELETE` presence when
  leaving.
- **Done when:** two-way chat with a browser user works on each board for
  more than 1 hour, which crosses one token refresh, with no duplicate or
  missing messages.

### M6: C-Mail and notifications

- C-Mail: conversation list, kept live from `user_conversations/<uid>`;
  thread view (history, then `dm_messages/<id>`); sending; mark-read; start
  a new conversation by username; and a typing indicator on
  `dm_presence/<id>` with the `staleAfterMs` rule.
- Notifications: unread badge polled every 60 s, but only while the screen
  is on and no stream gives the same information. Also a notification list
  with mark-one and mark-all read, and tapping through to the entry or reply.
- **Done when:** a DM sent from the browser appears within 2 s and the unread
  badge clears after reading.

### M7: Writing

- Compose an entry with title, content, up to 3 lowercase topics,
  `isPublic` and `isNSFW`, and character counters at the API limits.
  Reply and reply-to-reply.
- Show the write rate limits in the UI: disable the send button with a
  countdown when the per-minute bucket is empty, and show the daily remainder.
- BLE keyboard host on the CrowPanels: port camillia's `ble_keyboard.*` and
  its NimBLE 1.4.3 flags.
- Drafts autosaved to LittleFS.
- **Done when:** an entry and a reply posted from each board appear on the
  website.

### M8: Polish and extras

Ordered roughly by value:

- Idle handling: dim, then turn off the backlight, close streams after
  N minutes idle, and reopen on wake. This matters most on the T-Deck
  battery.
- Bookmarks, follows, a profile view, topics, search (60/hour cap) and
  watch or unwatch on a thread.
- Settings screen: brightness, rotation (all boards support both
  orientations) and font size.
- OTA from GitHub releases. This needs a signing scheme; see camillia's
  `ota_update.cpp`.
- SD card: the 3.5" uses soft SPI and the T-Deck the shared SPI bus. On the
  5.0", the DIP switch's SD position needs its pins taken from Elecrow's
  schematic.

## Open questions for the owner

1. Is the monospace font licence acceptable for distributing firmware
   (OFL, or similar)?
2. Should the refreshToken in NVS be protected with flash and NVS encryption?
   That complicates USB reflashing.
3. What licence does the repo use? It has to be GPL-3.0-compatible if
   camillia-derived HAL code ships in binaries.
4. Which 5.0" hardware revision is on the bench? The default build targets
   V1.2/V1.3.
