# Research: rhino-in-space, a cyberspace.online client for ESP32-S3 handhelds

Targets: Elecrow CrowPanel Advance 3.5", CrowPanel Advance 5.0", LilyGo T-Deck Plus.

Date: 2026-10-24

## 1. What Cyberspace is

[cyberspace.online](https://cyberspace.online/) is a small, text-only social
network built by a solo developer. It has no images in the feed (just ASCII
art), no likes, no algorithm and no ads. Its features are:

- **Entries**: long-form markdown posts with replies, topics and bookmarks.
- **Notes**: private notes with revisions.
- **C-Mail**: 1:1 direct messages.
- **cIRC**: multi-user chat rooms.
- **Guilds**, follows, notifications, search and "programs" (a gallery of
  user programs).

You need an account with a verified email to see anything. Accounts can only
be created on the website.

## 2. The API

**Source caveat.** This sandbox's network policy blocks `cyberspace.online`
and `api.cyberspace.online`. The reference below comes from the copy of the
official docs (**API v0.8.12**) mirrored in the open-source client
[ArmadilloBrillo/cyber-tui](https://github.com/ArmadilloBrillo/cyber-tui)
(`docs/00-latest-api-reference.md`, commit of 2026-10-07). It agrees with the
search-engine snippets of <https://api.cyberspace.online/docs>. Check it
against the live page before implementing.

### 2.1 Terms (these constrain the design)

- **Allowed:** a personal client (TUI, mobile, desktop) where *a real user
  drives each action*. This device qualifies.
- **Forbidden:** scraping or bulk collection, bots or automated posting, and
  feeding content to AI systems.
- **Consequence for us:** do no background prefetching of whole feeds and
  avoid aggressive polling. Every write happens on a user tap.

### 2.2 Transport and auth

| | |
|---|---|
| REST base | `https://api.cyberspace.online/v1` |
| Envelope | `{ "data": … }` or `{ "data": [...], "cursor": "…" }`; errors are `{ "error": { "code", "message" } }` |
| Login | `POST /v1/auth/login {email,password}` returns `idToken`, `refreshToken`, `rtdbToken`, `rtdbUrl` |
| Refresh | `POST /v1/auth/refresh {refreshToken}` returns `idToken`, `rtdbToken`, `rtdbUrl` |
| Auth header | `Authorization: Bearer <idToken>` (a Firebase ID token JWT, about 1 KB, which lives about 1 h) |
| Realtime | Firebase RTDB at `rtdbUrl` (currently `https://cyberspace-cyberspace-default-rtdb.europe-west1.firebasedatabase.app`), read over **Server-Sent Events** with `?auth=<idToken>` |

The device should store only the `refreshToken`, which is long-lived. The
password is used once at login and never persisted. On a `401`, or when an
SSE stream closes because the token expired, refresh the token and retry
once.

### 2.3 Endpoints worth supporting (in rough priority order)

| Area | Endpoints |
|---|---|
| Me | `GET /v1/users/me`, `GET /v1/users/:username`, `GET /v1/users/:username/posts` |
| Feed | `GET /v1/posts?limit&cursor`, `GET /v1/posts/:id`, `GET /v1/posts/:id/replies?limit&cursor` |
| Write | `POST /v1/posts`, `POST /v1/replies {postId, content, parentReplyId?}` |
| Notifications | `GET /v1/notifications`, unread count, mark read / mark all read |
| C-Mail | `GET /v1/cmail`, `POST /v1/cmail {recipientUsername}`, `GET /v1/cmail/:id?limit&before`, `POST /v1/cmail/:id {content}`, `POST /v1/cmail/:id/read`, `POST /v1/cmail/:id/typing` |
| cIRC | `GET /v1/circ`, `GET /v1/circ/:room` (history), `POST` message, `POST/DELETE /v1/circ/:room/presence`, `GET /v1/circ/:room/users`, mark read |
| Later | bookmarks, follows, topics, notes, guilds, search, watch thread, settings, programs |

### 2.4 Realtime (cIRC and C-Mail)

Each stream is a long-lived HTTPS GET with `Accept: text/event-stream`:

- `chat_messages/<roomId>.json?auth=…&orderBy="timestamp"&limitToLast=50`
- `chat_presence/<roomId>.json?auth=…`
- `dm_messages/<conversationId>.json?auth=…&orderBy="timestamp"&limitToLast=50`
- `user_conversations/<uid>.json?auth=…`: conversation list and unread counts
- `dm_presence/<conversationId>.json?auth=…`: typing indicator

The first event is a `put` at `/` carrying the whole window. After that, each
change arrives as a `put` or `patch` keyed by path. A `null` value means
deleted. A deleted cIRC message arrives as a `patch` that sets
`{content:"[DELETED]", deleted:true}`. Following the Firebase REST streaming
protocol, the client must also handle `keep-alive`, `cancel` and
`auth_revoked` events.

**Rules:**

- Always include `orderBy="timestamp"` and set `limitToLast` to 100 or less.
- Open one stream per room or conversation.
- Never reconnect in a tight loop.

**Presence:** to show up in a room's user list, heartbeat
`POST /v1/circ/:room/presence` every `heartbeatMs`. The server returns the
interval, currently 30 s. Send `lastActivity` (the time the user last touched
the screen) with each heartbeat.

### 2.5 Limits that shape the UI

- **Reads:** 30 to 45 requests per minute for each list endpoint, and 30 per
  minute for the unread count. Poll notifications no more than once every
  60 s, and only while the screen is on.
- **Writes:** entries 2 per minute and 24 per day; replies 3 per minute and
  48 per day; cIRC and C-Mail 15 per minute and 150 per hour.
- **Content:** chat messages up to 2,048 chars; entries and replies up to
  32,768 chars; up to 3 topics, which must be lowercase.
- **Images:** shown inline as markdown `![](https://bunker.cyberspace.online/…)`.
  They can't be posted from the API. Version 1 shows `[image]` and does not
  download or decode them.

### 2.6 Prior art (for reference only; no code is copied)

| Project | Language | Notes |
|---|---|---|
| [digital-grease/cs-tui](https://github.com/digital-grease/cs-tui) | Rust, ratatui | Covers most of the v0.8.10 surface, including cIRC and C-Mail |
| [ArmadilloBrillo/cyber-tui](https://github.com/ArmadilloBrillo/cyber-tui) | Go, Bubble Tea | Has the best docs: a mirrored API reference and captured JSON responses in `docs/api-reponses/` |
| [shighfield/tiespace](https://github.com/shighfield/tiespace) | Free Pascal, ncurses | Confirmed against a live account; keeps the refreshToken in a 0600 file |
| [johannesalke/cyberspacecli](https://github.com/johannesalke/cyberspacecli) | Go | A small CLI with a v0.3.6 doc snapshot |

I found no embedded or microcontroller client, so this would be the first.

## 3. Hardware: what camillia-mt already gives us

All three are ESP32-S3 parts with 16 MB flash, 8 MB octal PSRAM, WiFi and
BLE 5, and GT911 touch. The two CrowPanels have **no keyboard or button**; the
T-Deck Plus has a hardware QWERTY keyboard and a trackball. All three are
already ported in camillia-mt on espressif32@7.0.1 (Arduino core 2.0.17),
LovyanGFX 1.2.34 and LVGL 9.5.0.

| | CrowPanel 3.5" | CrowPanel 5.0" | T-Deck Plus |
|---|---|---|---|
| camillia files | `src/hal/hw_crowpanel_35.h`, `src/hal/display.h` (Panel_ILI9488) | `src/hal/hw_crowpanel_50.h`, `src/hal/crowpanel_50_display.h` (Bus_RGB / Panel_RGB + `CrowPanelBoardCtl`) | `src/hal/hw_tdeck.h`, `src/hal/display.h` (Panel_ST7789), `src/keyboard.*` |
| MCU | ESP32-S3-WROOM-1-N16R8 | ESP32-S3-WROOM-1-N16R8 | ESP32-S3FN16R8 |
| Panel | ILI9488 320×480 on SPI3 (SCK 42, MOSI 39, CS 40, DC 41, BL 38 PWM), run at 480×320 | ST7262 800×480 16-bit RGB from a PSRAM framebuffer, 14 MHz PCLK | 2.8" ST7789 240×320 on SPI2 (SCK 40, MISO 38, MOSI 41, CS 12, DC 11, BL 42 PWM), run at 320×240 |
| Touch | GT911 on I2C 15/16, INT 47, RST 48 | GT911 on I2C 15/16, INT 1; reset goes through the board controller | GT911 on I2C 18/8, INT 16 (same bus as the keyboard) |
| Keyboard / nav | none (on-screen, BLE keyboard) | none (on-screen, BLE keyboard) | I2C keyboard MCU at 0x55 (INT 46); trackball on GPIO 3/2/1/15, click GPIO0 |
| Backlight | GPIO38 PWM | V1.0: PCA9557 @0x18 IO1 (on/off). V1.1+: board MCU @0x30 (dimming below full makes the driver whine) | GPIO42 PWM |
| SD | Software SPI CLK5 / MISO4 / MOSI6 / CS7 (`sd_soft_spi.*`) | Shares pins with the LoRa slot through the DIP switch | Shared SPI2 bus, CS 39 |
| Power / battery | USB only, no battery ADC | USB only, no battery ADC | Peripheral rail on GPIO10 (hold HIGH at boot); battery on a 1:2 divider at GPIO4 (`BATT_CAL` 1.03) |
| PlatformIO env | `[env:crowpanel-35]` | `[env:crowpanel-50]` (extends the 3.5" env) | `[env:tdeck]` |

**T-Deck Plus notes.** camillia-mt has no separate Plus target. Its `tdeck`
env runs on the Plus: camillia's own GPS and wardriving field tests were done
on one (`src/gps.cpp`, `docs/WARDRIVING.md`). The Plus is the T-Deck with the
GPS fitted and a larger battery; the display, keyboard, touch and trackball
pins are the same. For this client:

- **Shared SPI bus.** The display, LoRa radio and SD card all sit on SPI2.
  Nothing drives the radio here, but its CS (GPIO9) must be held HIGH so it
  stays off the bus.
- **Peripheral rail.** GPIO10 must be driven HIGH early in boot, or the
  keyboard, display and radio have no power.
- **Keyboard.** The 0x55 keyboard is a small MCU that returns one ASCII byte
  per keypress. Copy camillia's `keyboard.cpp` (and the trackball ISR decoder
  in it) for the reading and for its key and modifier handling.
- **Screen wake.** camillia wakes the screen only on a trackball click,
  because the keyboard and touch panel trigger in a pocket. Keep that policy.
- **Battery.** This is the only target with a battery, so idle handling
  matters for power here: dim the screen, close the SSE streams after a
  timeout, and reconnect on wake. `battery_util.*` reads the gauge.
- **GPS.** Drop it, as on the CrowPanels.

What this client should take from camillia-mt:

- **Copy as-is:** `hw_crowpanel_35.h`, `hw_crowpanel_50.h`,
  `crowpanel_50_display.h`, the CrowPanel arms of `display.h` and `board.h`,
  the platformio env skeletons (board, flash and PSRAM settings, partition
  CSV) and `lv_conf.h`.
- **Copy and trim:** the LVGL display and touch glue in `main_lvgl.cpp`
  (flush callback, GT911 read, rotation). That file is 62 k lines, so extract
  only the CrowPanel paths. Also `storage.*`/`sd_soft_spi.*` (only the 3.5"
  needs the card) and `ble_keyboard.*`, which lets a BLE keyboard pair with a
  keyboard-less panel and is a big usability win for writing entries.
- **Drop:** RadioLib, mesh, MQTT, GPS, the spell dictionary and VNC.
  Dropping the radio also frees its pins:
  - On the 5.0", set the DIP switch to the SD position instead of the wireless
    module. The SD card then becomes available, but camillia-mt doesn't have
    the pins for that switch position yet, so they need to be taken from
    Elecrow's schematic.
  - On the 3.5", the slot-select line (IO45) can stay at its default.
- **Licensing:** camillia-mt is GPL-3.0. A private repository is fine. If you
  ever distribute binaries, the copied code has to stay GPL-compatible. You
  are the main author (94 of 121 commits), so you could relicense your own
  files, but not code from other contributors.

## 4. Proposed architecture

```
 ┌──────────── UI (LVGL 9, core 1) ────────────┐
 │ Feed · Entry+replies · Compose · Notifs     │
 │ C-Mail list/thread · cIRC rooms/room · Setup│
 └───────────────▲──────────────┬──────────────┘
          models │ (queue)      │ actions (queue)
 ┌───────────────┴──────────────▼──────────────┐
 │ cs_api   REST client (HTTPClient + WiFiClient│
 │          Secure, ArduinoJson, cursor paging) │
 │ cs_auth  login / refresh / NVS refreshToken  │
 │ cs_rtdb  SSE stream(s), put/patch merge      │
 │ cs_rate  client-side token buckets per §2.5  │
 └───────────────▲─────────────────────────────┘
                 │ WiFi (STA, SNTP, cert bundle)
```

- **Networking:** runs in its own FreeRTOS task, so no TLS call ever blocks
  LVGL. It talks to the UI over two queues.
- **TLS:**
  - Use the ESP-IDF x509 CA bundle (`setCACertBundle`). Don't pin certs: the
    API sits behind its host's certs and RTDB behind Google Trust Services.
  - Set the clock with SNTP before the first request.
  - Budget about 40 to 50 KB of internal RAM per open TLS session. Allow at
    most three at once: REST, plus one room stream, plus `user_conversations`.
- **JSON:** use ArduinoJson 7 with a PSRAM allocator and *filter* documents,
  so a 32 KB entry body never sits in internal RAM.
- **Markdown:** render a subset only: headings, bold and italic, code and
  preformatted blocks in a monospace font (needed for ASCII art), quotes,
  lists and links shown as text. Images become `[image]`.
- **Fonts:** a monospace font is needed for ASCII art, which camillia-mt
  doesn't have. Box-drawing glyphs and Unicode coverage need choosing; users
  post things like "ᑕ¥βєяรקค¢є". Reuse camillia's Montserrat for chrome and
  `emoji_font`.
- **Text entry:** on the CrowPanels, use the LVGL on-screen keyboard, plus a
  paired BLE keyboard. The 3.5" is cramped for long entries, so on it treat
  composing entries as secondary and chat as primary. On the T-Deck Plus, use
  its hardware keyboard as an LVGL keypad input device and the trackball as an
  encoder, so the whole UI works without touch.
- **Setup:**
  - WiFi credentials are entered on screen, or through a SoftAP captive
    portal adapted from camillia's `web_config`.
  - Email and password are entered once on screen. After that only the
    refreshToken is kept in NVS. NVS encryption needs flash encryption, so
    decide whether that matters.
- **Layout per board:**
  - 5.0" (800×480): two panes, a list on the left and the reader or chat on
    the right.
  - 3.5" (480×320): a single pane with a stack navigator.
  - T-Deck Plus (320×240): the 3.5"'s single pane, with denser fonts. Every
    screen needs a focus order (LVGL groups) so the trackball and keyboard
    can reach everything.

## 5. Risks and open questions

1. **Live API check.** None of the above has been tested against the live
   API, because the sandbox can't reach it. Check the live docs and confirm
   that the RTDB SSE endpoint accepts the ID token from an ESP32 client.
2. **5.0" PSRAM bandwidth.** The RGB panel scans out of PSRAM while WiFi, TLS
   and LVGL also use it. camillia already lowered PCLK to 14 MHz for this.
   Measure for tearing and drift with a long SSE stream open.
3. **ILI9488 over SPI** pushes 18-bit pixels, so full-screen redraws are slow.
   Use partial refresh and avoid full-screen animations on the 3.5".
4. **Token refresh and long-lived streams.** The ID token expires about every
   hour, which closes every stream. The client needs one coordinated refresh
   that reopens all streams with backoff, as the "no reconnect loops" rule
   requires.
5. **Power and idle.** The CrowPanels have no battery ADC and run on mains or
   USB, so on them screen dimming and stream teardown when idle matter more
   for the rate limits than for battery life. The T-Deck Plus runs on battery,
   so on it they matter for both.
6. **Hardware revision on the 5.0".** V1.0 and V1.1+ handle backlight and
   touch reset differently. camillia's `CrowPanelBoardCtl` already detects
   which one is present at runtime, and that detection carries over.
7. **T-Deck Plus screen size.** At 320×240, long entries and ASCII art wider
   than about 53 columns of a 6 px monospace font will wrap or need
   horizontal scrolling.

## 6. Suggested milestones

1. **Bring-up.**
   - A repository with `crowpanel-35`, `crowpanel-50` and `tdeck-plus` envs.
   - Display, touch and LVGL "hello" on each board (copied HAL), plus the
     keyboard and trackball on the T-Deck Plus.
   - WiFi setup and SNTP.
2. **Auth.** Login screen, refreshToken in NVS, `GET /v1/users/me` shown on
   screen.
3. **Read-only feed.** Feed list with cursor paging, entry view with the
   markdown subset and monospace text, and replies.
4. **cIRC.** Room list, room view on an SSE stream, sending messages,
   presence heartbeat and the user list.
5. **C-Mail.** Conversation list on a live `user_conversations` stream, a
   thread view and sending.
6. **Notifications.** Unread badge on a 60 s poll while the screen is on, and
   the notifications list.
7. **Writing.** Compose entries and replies, and pair a BLE keyboard.
8. **Later.** Bookmarks, follows, topics, notes, guilds, search and OTA.
