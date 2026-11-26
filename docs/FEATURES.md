# Literature Clock — feature reference

Everything the clock does, screen by screen and setting by setting. The main targets are the
**Waveshare ESP32-S3-Touch-LCD-7** (7", 800 × 480) and **Waveshare ESP32-S3-Touch-LCD-4.3B** (4.3",
800 × 480 with onboard PCF85063A RTC). Where a feature exists on those Waveshare panels only, it says so.

Contents

1. [What it is](#1-what-it-is)
2. [Boards and what each one runs](#2-boards-and-what-each-one-runs)
3. [Starting up](#3-starting-up)
4. [The quote page](#4-the-quote-page)
5. [Quotes and the library](#5-quotes-and-the-library)
6. [Paper palettes, brightness and night paper](#6-paper-palettes-brightness-and-night-paper)
7. [Type](#7-type)
8. [Clock faces](#8-clock-faces)
9. [Zen page](#9-zen-page)
10. [Start-up greeting](#10-start-up-greeting)
11. [Effects: falling things](#11-effects-falling-things)
12. [Sleep hours](#12-sleep-hours)
13. [Alarm](#13-alarm)
14. [Time, time zone and page turning](#14-time-time-zone-and-page-turning)
15. [Weather](#15-weather)
16. [Wi-Fi and the phone page](#16-wi-fi-and-the-phone-page)
17. [On-device screens](#17-on-device-screens)
18. [Phone page, tab by tab](#18-phone-page-tab-by-tab)
19. [Backup: export and import](#19-backup-export-and-import)
20. [Software updates](#20-software-updates)
21. [Power saving and offline mode](#21-power-saving-and-offline-mode)
22. [Web API](#22-web-api)
23. [Serial console](#23-serial-console)
24. [Stored settings and defaults](#24-stored-settings-and-defaults)
25. [Building, flashing and the preview server](#25-building-flashing-and-the-preview-server)
26. [Project layout](#26-project-layout)
27. [Known limits](#27-known-limits)
28. [Credits and licences](#28-credits-and-licences)

---

## 1. What it is

A wall or nightstand clock that tells the time with a line from a book. For each minute of the day
there are one or more passages that mention that minute in words ("twelve minutes past four"), and the
clock shows one of them, with the time phrase picked out in bold and in the paper's second colour at the **same font size** as the body text. It is
drawn to look like a printed page: paper colour, serif type, a header rule, a citation line.

No account, no cloud service and no phone app are needed. The quotes are stored on the clock. Wi-Fi is
used for the time (NTP), the optional weather line and the settings page you open from your phone.

## 2. Boards and what each one runs

| Board | Environment | Notes |
|---|---|---|
| Waveshare ESP32-S3-Touch-LCD-7 | `waveshare-lcd7` (default) | 7" 800 × 480 RGB, GT911 touch (`0x14`/`0x5D` auto-detect, I2C polling, native resolution auto-scaling), CH422G expander, `default_16MB.csv` dual-OTA layout. Flash over the USB-C port marked **UART**, not the OTG port. |
| Waveshare ESP32-S3-Touch-LCD-4.3B | `waveshare-lcd43` | 4.3" 800 × 480 RGB, GT911 touch (`0x14`/`0x5D` auto-detect, I2C polling, native resolution auto-scaling), CH422G expander, onboard **PCF85063A RTC** (`0x51`), `default_16MB.csv` dual-OTA layout. Flash and serial console over native USB-C (`ARDUINO_USB_CDC_ON_BOOT=1`). |
| M5Paper | `m5paper` | 4.7" e-ink with onboard BM8563 RTC and M5Power PMIC battery telemetry. Core page only. |
| ESP32-2432S028R "Cheap Yellow Display" | `cyd-28` | 2.8" ILI9341, resistive touch. Core page only. |

The core page (quote, themes, settings, Wi-Fi, weather, alarm) runs on all four targets. These are
**Waveshare-panel only** (7" & 4.3B): clock faces, the zen page's extras, capacitive swipe/long-press
gestures, the start-up greeting, falling things, sleep hours, the on-device Effects and Alarm-and-sleep
screens, the smooth outline type, the brightness panel and the on-screen Wi-Fi keyboard.

## 3. Starting up

Order of events on a cold start or restart:

1. **Hold BOOT** while powering on for about 2.5 seconds to clear the saved Wi-Fi. (On the Waveshare
   panel GPIO 0 is both BOOT and an LCD data line, so it is read before the screen is claimed.)
2. The quote library is read from flash, then the screen starts. The lamp comes up and stays lit for at
   least 30 seconds so you can reach the panel.
3. **The greeting** shows, if it is switched on (see [section 10](#10-start-up-greeting)). It is skipped
   when the chip wakes from deep sleep to turn a page.
4. The clock looks for the saved Wi-Fi. If there is none, it opens its own network,
   **LiteratureClock** (address `192.168.4.1`), and shows a QR code; you can also use the on-screen Wi-Fi
   screen (scan, pick, type the password).
5. If a valid hardware RTC (`PCF85063A`, `BM8563`, or `DS3231`) is present, it loads the time immediately; in **Battery saver** modes (`powerMode 2` on battery or `3` always), boot skips Wi‑Fi completely. Otherwise, once joined it asks NTP **once** to verify/update the hardware RTC (stopping SNTP via `esp_sntp_stop()` as soon as verified), saves a flash checkpoint, fetches the weather if enabled, and draws the first page.

While this happens, the screen that is already up (the greeting, or the single start-up notice) stays until
the first page is ready. Progress screens are not shown one after another. If the time takes more than four
seconds to arrive, "Waiting for the time…" is shown so the clock does not look dead. "Clearing Wi-Fi…" is
always shown.

If Wi-Fi is not set up, tapping **offline** on the QR screen (or `offline on` on the serial console) runs the
clock from its own internal time and the last time saved in flash.

## 4. The quote page

Top to bottom:

- **Header:** the date on the left (italic), the time on the right. When a battery sensor is detected
  ([section 21](#21-power-saving-and-offline-mode)), a compact **battery level indicator icon** is drawn just to
  the left of the time. With seconds switched on, the header time shows `HH:MM:SS`; only that time patch
  of the screen is redrawn each second (the battery icon sits outside the dirty rectangle so it never
  flickers or clips, and nothing drifting over the page is disturbed).
- **Weather line** under the date, when weather is on ([section 15](#15-weather)).
- **The quote:** wrapped and centred vertically. The time phrase is bold and in the accent colour. Type size
  drops one step at a time if a long passage does not fit.
- **Footer:** a rule, then title and author in italic, then five buttons.

Touch (including capacitive gestures on Waveshare 7" & 4.3B):

| You do | What happens |
|---|---|
| Tap the quote, or anywhere off the buttons | Another passage for this minute (over a clock face, shows/hides the footer buttons for 6 seconds) |
| **Swipe left / right** (Waveshare quote page, face, or Zen) | Cycles through clock faces (`NextFace` / `PrevFace`) |
| **Swipe up / down in centre** (Waveshare `>180 px` from edges) | Cycles through the 12 paper palettes (`NextTheme` / `PrevTheme`) |
| **Swipe up / down at left/right edge** (Waveshare `≤180 px` from edge) | Steps lamp and text brightness brighter / dimmer (`±35`) |
| **Long-press** (`≥700 ms` above footer buttons, Waveshare) | Enters **Zen mode** (tap anywhere to leave) |
| **Type size / Face button** | On Quote page (`Type: S`/`M`/`L`/`Auto`): cycles Small → Medium → Large → Auto. Over a clock face (`Classic >` / `Flip >` / `Nixie >` …): steps to the next face |
| **Quote / Quotes button** | On Quote page: shows another passage for this minute. Over a clock face (**Quotes**): returns directly to the quote page |
| **Paper / Face opts button** | On Quote page (shows current palette name): opens the 12-palette picker. Over a clock face (**Face opts** / **Style**): opens Clock face options and returns straight to the clock face |
| **Zen / Full screen button** | Enters full-screen Zen mode (hides footer/UI on quotes or stretches clock faces edge-to-edge) |
| **Setup** button | The Setup screen |

Over a clock face the buttons are hidden. Touch the face to show them (along with a compact swipe reminder
strip) for six seconds; touch away from the buttons to hide them again. On Waveshare 7" and 4.3B,
subtle edge-grip markers sit at the left and right bezels, every gesture shows a brief feedback pill (with
a 5-second swipe summary on boot), and **Setup → Swipe guide ?** opens a 1:1 visual map of all zones.

## 5. Quotes and the library

- The library is **Literature**, from Johannes Enevoldsen's collection, packed directly into internal flash
  as a compressed binary (`src/generated/quotes.pack`: **3,461 passages across 1,409 unique minutes**, about
  416 KB, built by `scripts/pack_device_quotes.py`). One hour is inflated at a time. The clock never
  fetches quotes over the network.
- **Flash-only for single-file OTA (no SD card required):** keeping the entire library in internal flash
  alongside `default_16MB.csv` partitions means firmware updates deploy cleanly as one GitHub OTA `.bin` file
  with no SD card preparation or card dependency. Gutenberg (`quotes/pd-times.txt`) and *A New Yorker*
  (`quotes/themes/new-yorker.txt`) remain in the repository for reference (see `NOTES.md`).
- For each minute the clock picks among the passages for that minute (up to 64), stepping round them each
  time the page turns or is tapped. If a minute has none, it looks back minute by minute until it finds
  one.
- If the library is missing, the page says "The books are still on the shelf" with the time.
- **Every quote is family-friendly.** When the library is packed, passages the collection marks NSFW
  (165 of about 3,600) are left out, so there is no content switch. Older versions had a "Safer quotes"
  setting; it is gone from every screen, and the stored value (`sfw`) is ignored.
- Quote text is cleaned for the fonts: HTML tags removed, curly punctuation reduced to plain forms.

## 6. Paper palettes, brightness and night paper

**Twelve palettes** — each has paper, grain, ink, muted, accent and rule colours:

| # | Name | Look |
|---|---|---|
| 0 | Paper | Warm cream, black ink |
| 1 | Snow | Cool white |
| 2 | Sepia | Aged book |
| 3 | Sage | Green-grey |
| 4 | Slate | Cool grey |
| 5 | Cinnabar | Cream with a red time phrase |
| 6 | Gallery | Warm, ochre accent |
| 7 | Midnight | Dark, cream text (Waveshare default) |
| 8 | Forest | Deep green |
| 9 | Plum | Aubergine |
| 10 | Ocean | Night blue |
| 11 | Honey | Yellow paper |

Choose one on the panel (**Paper** button, or Setup → Palette) or on the phone (Look tab). On the phone the
choice shows in the page preview straight away and is sent to the clock straight away.

**Brightness** has three sliders, 20–255 (about 8–100%):

- **Backlight (lamp)** — the lamp and the paper level only.
- **Text** — the letters only.
- **Both** — moves the two together.

On the panel: Setup → Brightness. On the phone: Look tab.

**Night paper:**

| Mode | Behaviour |
|---|---|
| Off | Nothing |
| Auto 21–06 | Between 21:00 and 06:00 the **Midnight** palette is used and lamp and text drop to 62% (never below 20) |
| Auto Sunset–Sunrise | Uses NOAA solar calculation (`computeSunTimes`) for today’s date and coordinates (`Weather` geocode, manual Latitude/Longitude, or timezone city default) to switch to **Midnight** and 62% dimming from local sunset until sunrise (handles Arctic midnight sun and polar night automatically; falls back to 21–06 when invalid) |
| Always dark | Midnight and the dimmer levels, all day |

When night paper is active it overrides the palette you chose and automatically dims the panel backlight
via `effectiveLampLevel()`. Choose between `Off`, `21-06`, `Sun`, and `Always` (`Dark`) on the Waveshare
brightness panel (showing today's computed sunrise and sunset), M5Paper Setup row 8, or the phone's
**Look** tab (with live preview).

## 7. Type

- **Size:** Small, Medium, Large. Cycles from the footer button or Setup → Type size.
- **Typeface:** Serif, Smooth, Bold, Italic.
- The clock draws the large lettering (clock digits, greeting) from smooth outlines rather than bitmap
  fonts, so it stays clean at any size. The greeting fonts are Caveat (handwriting) and Great Vibes
  (script), both under the SIL Open Font Licence (`scripts/fonts/OFL-*.txt`).

## 8. Clock faces

Choose **Clock face** in Setup, or on the phone's **Face** tab. Eight choices:

| Face | Description |
|---|---|
| Page | The quote page (default) |
| Classic | Black and white dial |
| 1950 | Rectangular 1950s dial |
| Flip | 4 or 6 individual split-flap digit cards (`HH MM` / `HH MM SS`) that flip when a digit changes |
| Digital | Large digital time |
| Split | High-contrast top and bottom dial halves, synced to paper palettes when selected |
| Square | 1970s retro rounded-chassis analog dial with 6× antialiased numerals, slash indices, burgundy hour dots, and beveled baton hands (`clockFace=7`; fills the screen in Zen mode) |
| Nixie | Glowing glass cathode vacuum tubes (`clockFace=8`, 4 or 6 digits) with neon colons and AM/PM indicator |

Options (each is a chip on the panel and a select on the phone):

- **Seconds** — show seconds (analog second hand or 6-digit Flip/Nixie/Digital seconds).
- **Numbers** — dial numerals shown or hidden. With Split, hidden numbers become solid hour points.
- **Date**, **Digital time**, **Weather** — small captions under the face (hidden in full-screen Zen mode). All off until you turn them on.
- **Minute marks** (Classic and 1950) and **Bar hands** (Classic, 1950, and Square) — pointed/beveled hands or straight bars.
- **Invert** (Classic, Flip, Digital, Square, and Nixie) — dark dial / brushed-metal palette swap.
- **Paper colors** — replace the face's own colours with the chosen palette (Split adds **From palette**). Choosing a new palette switches this on for you.
- **Blink :** (Flip, Digital, and Nixie) — the colon blinks each second.
- **Colours** — Split: top half, bottom half, hour points, second hand. 1950: numbers, hour points, minute ticks. Fourteen swatches on the panel; any colour on the phone.

Faces redraw once a second when seconds or a blinking colon are on.

## 9. Zen page

The words and their source, nothing else. Enter with the **Zen** footer button, by **long-pressing** (`≥700 ms`)
above the footer buttons on a Waveshare panel, from the phone's **Page** tab (**Zen on the clock**), or by
setting **Zen page on the clock** on the Look tab. Tap the screen to leave; while in Zen on Waveshare panels
you can swipe **left/right** to jump straight to a clock face, swipe **up/down in the centre** to cycle
palettes, or swipe **up/down along either edge** to adjust brightness. The zen page turns on the same
schedule as the quote page, and falling things can drift over it
([section 11](#11-effects-falling-things)).

## 10. Start-up greeting

A few words shown when the clock is switched on, before anything else, and optionally again when
[sleep hours](#12-sleep-hours) end.

**Words:** up to 100 letters, numbers and punctuation. Accents become plain letters (é → e). Pictures are
written as `:name:` and the phone page's picture buttons insert them:

`heart` ❤️ · `star` ⭐ · `smile` 😊 · `sun` ☀️ · `moon` 🌙 · `flower` 🌸 · `sparkle` ✨ · `hearteyes` 😍 · `kiss` 😘 ·
`hands` 🫶 · `canada` 🇨🇦 · `cake` 🎂 · `balloon` 🎈 · `tree` 🎄

Other emoji are dropped, and only the Canadian flag is drawn. The pictures are drawn as smooth vector shapes.

**Lettering:** Handwriting, Script or Serif italic. **Size:** 40–130% of the largest size that fits; long
words shrink to fit the screen anyway, and a long greeting wraps to a second line. **Word colours:** the
page's ink, one colour, two colours in turn, or a rainbow word by word.

**How it appears:**

| # | Name | What you see |
|---|---|---|
| 0 | Written across | Each line is written left to right |
| 1 | Typed letter by letter | Typewriter |
| 2 | All at once | Appears whole |
| 3 | Fireworks | Words, then fireworks |
| 4 | Heart and arrow | A red heart with an arrow drawn through it, then the words |
| 5 | Christmas | Words, with snow and a string of blinking lights |
| 6 | Birthday balloons | Words, with balloons rising |
| 7 | Snow | Words, with falling snow crystals |
| 8 | Autumn leaves | Words, with tumbling autumn leaves |
| 9 | Blossom petals | Words, with spinning pink petals |
| 10 | Fireflies | Words, with wandering golden glows |
| 11 | Night sky | Words, with twinkling stars and shooting stars |
| 12 | Gentle rain | Words, with thin slanting drops |
| 13 | Floating hearts | Words, with rising hearts |
| 14 | By the season | Words, with the current season's ambient effect |

Touch the screen at any point to skip. The greeting is held a moment after it finishes.

**Switches:**

- **Greeting on/off** — Setup → Greeting (a row), the Effects screen, or the phone's Greeting tab.
- **Greeting when sleep ends** — shows the greeting again in the morning after sleep hours. It works even
  when the start-up greeting is off. A touch that only wakes the clock for a minute, or the alarm waking it,
  does not show it.
- **Show it now** — the Effects screen has a button, and the phone's Greeting tab has "Save and show it on
  the clock".

The greeting settings are stored in flash (see [section 24](#24-stored-settings-and-defaults)).

## 11. Effects: falling things

Gentle things that drift over the quote page. Each piece is drawn on the panel from a clean copy of the
page held in memory, so the words never flicker.

**Kinds:**

| Kind | Description |
|---|---|
| Snow | Six-armed crystals of several sizes, slowly turning, with small round flakes far away |
| Autumn leaves | Maple leaves in four colours, tumbling |
| Blossom petals | Pink petals, spinning |
| Fireflies | Soft golden glows that wander and pulse |
| Night sky | Twinkling stars, with an occasional shooting star (over the words only) |
| Rain | Gentle rain (`7`): thin slanting drops |
| Hearts | Floating hearts (`8`): hearts rising |
| Balloons | Birthday balloons (`9`): colourful balloons rising |
| Fireworks | Fireworks (`10`): rockets bursting into fading colourful sparks |
| Christmas | Christmas (`11`): snow crystals with a top string of blinking coloured lights |
| By season | `5`: Snow in Dec–Feb, petals Mar–May, fireflies Jun–Aug, leaves Sep–Nov |

**Where:** *over the words*, or *in the margins only* (pieces stay in the strips beside the text).
**How many:** a few, some, plenty. **On the zen page** and **On the clock faces:** each can drift or stay
still.

Set on the phone's **Effects** tab, or on the panel: Setup → **Falling things** cycles the kind, and
Setup → **Effects** opens the full screen. Effects pause while any setup screen is open.

## 12. Sleep hours

Puts the display into **blackout / sleep mode** and pauses page turns:

- **Scheduled (`Sleep Sched`, `sleepMode=1`):** active between **From** and **Until** (default 23:00 to 07:00).
- **Auto Sunset–Sunrise (`Sleep Sun`, `sleepMode=2`):** blacks out automatically from calculated sunset until sunrise at your configured coordinates (midnight sun stays awake; polar night sleeps until touched).
- **Instant Blackout now:** tap **Blackout now** on Setup → Alarm and sleep, tap **Blackout / Sleep now** on the phone’s **Page** or **Time** tab (`POST /api/sleep`), or run `sleep now` / `blackout on` on the serial console to black out immediately; tap the glass or press **Wake screen** / `blackout off` / `sleep wake` to restore.

Hardware behaviour:
- **Waveshare 7" & 4.3B & 2.8" CYD:** switches the backlight completely off and clears the framebuffer to `0x0000` black so there is zero edge glow or bleed.
- **M5Paper e-ink:** performs one full refresh to dark paper with `"Sleeping - touch to wake"`.

Wake rules:
- **Touch** the screen to wake it for one minute during a scheduled/sun window (each touch extends the minute) or to exit a manual blackout. The touch that wakes it is swallowed for 700 ms so it does not press a button underneath.
- **When the window ends** (sunrise or morning time) the lamp comes back and the page turns. If **Greeting when sleep ends** is on, the greeting shows first.
- **The alarm's minute is never slept through**; the alarm wakes the clock.
- After a restart the clock stays lit for 30 seconds before sleep hours can begin.

Set on the phone's **Time** tab, or on the Waveshare panel: Setup → **Alarm and sleep**, where tapping Sleep
cycles **Off / Sched / Sun** and times move by half an hour.

## 13. Alarm

There is no sounder on these boards, so the alarm is the screen: at the alarm minute the whole glass pulses
with bright colours until you touch it. The alarm is not repeated within the same minute after you stop it.

Three ways to set it:

- **The phone page:** Time tab → **Alarm**: on/off and a time of day (to the minute).
- **The panel:** Setup → **Alarm and sleep**; the hour moves in steps of one and the minutes in steps of five.
- **The serial console:** `alarm 07:30` or `alarm off`.

A change from any of them takes effect at once, and an alarm set to a new time may ring again in that minute.

## 14. Time, time zone and page turning

- **Time source (NTP + hardware RTC + flash):**
  - **NTP:** asks `pool.ntp.org` / `time.nist.gov` whenever Wi-Fi connects and every six hours.
  - **Hardware RTC:** automatically detects **PCF85063A** (`0x51`, standard onboard the Waveshare 4.3B) and
    **DS3231** (`0x68`) over I2C (`SDA=GPIO 8, SCL=GPIO 9`), plus **BM8563** (`0x51`) on M5Paper. Reads the
    RTC on boot when its oscillator-stop flag (`OS`/`OSF`) is clear, and updates the RTC whenever NTP,
    browser sync, panel setup, or serial console sets the time.
  - **Flash backup:** writes the epoch to NVS every 15 minutes (or on manual/browser updates) so boards
    without a coin-cell RTC restart near the right minute after a power cut.
- **By hand:** set the date and time from the phone's Time tab, or on the panel: Setup → Time and Date.
  Useful without a network. Saving unrelated Time-tab settings will not overwrite the clock's running time
  unless you edit the date/time fields or press a browser Sync button.
- **From the browser (UTC, date, time & timezone sync):**
  - The **Page** tab provides a one-click **Sync time + TZ** quick button.
  - The **Time** tab shows a live **browser-vs-clock drift readout** and three granular buttons:
    **Sync time + TZ**, **Sync time only**, and **Sync TZ only**.
  - Sync sends the exact browser UTC Unix `epoch` (in seconds) plus a POSIX timezone string from
    `posixTzFromBrowser()`: it matches the built-in zone list first, otherwise builds a fixed offset such as
    `<+0545>-5:45` or synthesises a full DST rule (e.g. `IST-1GMT0,M10.5.0,M3.5.0/1` or `NST3:30NDT,M3.2.0,M11.1.0`)
    directly from your browser's calendar transitions.
  - When the clock boots with no valid time at all (no network, no valid hardware RTC, no saved epoch), opening
    the web app auto-syncs the browser's time and zone once.
- **Time zone:** 24 built-in global zones ordered West → East from `UTC-10` to `UTC+12` (Hawaii/Honolulu,
  Alaska/Anchorage, Pacific Los Angeles/Vancouver, Arizona/Phoenix, Mountain Denver/Calgary, Central
  Chicago/Winnipeg, Mexico City, Eastern Toronto/New York, Atlantic/Halifax, Newfoundland/St. John's,
  Brazil/Sao Paulo, UTC/Reykjavik/Greenwich, London/Dublin, Central Europe Paris/Berlin, Eastern Europe
  Athens/Helsinki, South Africa/Johannesburg, Gulf/Dubai, India Kolkata/Mumbai, Indochina Bangkok/Jakarta,
  China/SG Singapore/Hong Kong, Japan Tokyo/Seoul, Central Aus/Adelaide, Eastern Aus Sydney/Melbourne, and
  New Zealand/Auckland), each with its local DST rule and default coordinates for Sunrise/Sunset. Pick on
  the phone (live search filter + **Custom POSIX** field), on any panel (`Time zone` opens a paginated
  picker: 2 pages on Waveshare 7″/4.3B and M5Paper, 4 pages on CYD 2.8″), or via browser sync / serial
  (`tz <city|posix|list>`).
- **12- or 24-hour** clock.
- **Page turn:** every 1, 5, 15, 30 or 60 minutes.
- **Timing:** *on the mark* (12:00, 12:15, 12:30) or *any minute inside the window* — each page waits a
  random 1 to N minutes, so the same lines never come round together.
- **Turn the page now:** Setup → Turn the page, the phone's Page tab, or touch the quote.

## 15. Weather

An optional line under the date, such as `Halifax  14C  rain` on the panel (the phone page shows `Halifax · 14°C · rain`). Data comes from Open-Meteo (no key needed) in a background task (`litwx`) protected by a mutex so drawing and panel DMA never block on network waits. Geocoding (`lat`, `lon`, resolved place query) is cached in flash (`litwx` NVS namespace) and weather refreshes at most every 15 minutes. Labels are clear, cloud, fog, rain, snow and storm.

Set **Show weather**, **Units** (Celsius or Fahrenheit) and **Place** on the phone's Weather tab. The
default place is Halifax. On clock faces the weather is a caption that must be turned on separately.

## 16. Wi-Fi and the phone page

- **Setup from a phone:** join **LiteratureClock**, or scan the QR code on the clock; the page lists the
  networks in range, takes the password and joins.
- **Setup on the clock (Waveshare):** Setup → Wi-Fi opens a scan list and an on-screen keyboard.
- **The phone page** is served by the clock on port 80. Open it by scanning the QR from Setup → Phone page,
  or at `http://literature-clock.local/`, or at the clock's IP address shown in Setup.
- **Forget Wi-Fi** on the phone's Wi-Fi tab, or hold BOOT at power-on.
- The page is designed for phones and adapts up to a two-column layout on a desktop browser.

## 17. On-device screens

**Setup** is a two-column grid; tap a row to change it, tap the top to close.

| Row | Tap to |
|---|---|
| Type size | Cycle Small / Medium / Large |
| Brightness | Open the brightness panel |
| Palette | Open the palette picker |
| Time zone | Open the paginated time-zone picker (Waveshare 2 pp., M5Paper 2 pp., CYD 4 pp.) |
| Clock | Toggle 12 / 24-hour |
| Page turn | Cycle 1 / 5 / 15 / 30 / 60 minutes |
| Timing | On the mark / any minute |
| Date | Open the date-and-time panel |
| Clock face | Open the clock-face panel |
| Wi-Fi | Join a network |
| Phone page | Show the QR (or address) |
| Alarm and sleep | Open the alarm-and-sleep screen |
| Effects | Open the effects screen |
| Falling things | Cycle through the kinds |
| Greeting | Greeting on / off |
| Turn the page | Fetch another passage now |
| Restart clock | Restart |
| Time | Open the date-and-time panel |

**Brightness panel:** three sliders (lamp, text, both), a top-right **Power:** cycle chip (`Always on`, `Idle Wi-Fi`, `Auto bat`, `Bat save`), a reminder of the edge-swipe brightness gesture, and
four Night paper chips (`Off`, `21-06`, `Sun`, `Always`) along with today’s calculated sunrise and sunset
readout.

**Palette picker:** a grid of the twelve palettes with a swatch and name (plus a centre-swipe hint in the header).

**Clock-face panel:** seven face chips (`Page`, `Classic`, `1950`, `Flip`, `Digital`, `Split`, `Square`), five option chips, more chips and colour swatch rows depending on the
face (plus a horizontal-swipe hint in the header). Touch the top to go back.

**Screen orientation (`Setup` header `Rotate: 0` / `90` / `180` / `270`):** rotates display output, capacitive/resistive touch coordinates, PSRAM canvas, and screen layouts in 90° steps across Waveshare 7"/4.3B (`800×480` landscape & `480×800` portrait), M5Paper (`960×540` & `540×960`), and CYD 2.8" (`240×320` & `320×240`).

**Swipe guide (`Setup → Swipe guide ?`):** a 1:1 diagram of the screen highlighting the outer 180 px
left/right brightness edges, the centre swipe/hold actions, and the footer zone. Tap anywhere to return.

**Alarm and sleep:** alarm on/off with hour and minute steppers; sleep mode cycling (`Off`, `Sched`, `Sun`)
with from and until steppers plus today’s sun readout, the wake-greeting switch, and **Blackout now**.

**Effects:** a chip per falling thing; chips for where, how many, zen page and clock faces; greeting on/off,
how it appears (tap to cycle) and *Show it now*.

## 18. Phone page, tab by tab

| Tab | Contents |
|---|---|
| **Page** | Live copy of the page, quick **Sync time + TZ** button, **Blackout / Sleep now** (or **Wake screen**) button, Turn the page, Refresh, Zen on this screen, Zen on the clock, plus a collapsible **Touch & swipe gestures** diagram for the 7" & 4.3B panel; top header displays a live **battery icon** badge when a battery monitor is detected |
| **Wi-Fi** | Scan, pick a network, password (hide option), Join, Forget |
| **Time** | Searchable 24-zone dropdown (`UTC±HH · Label`) + **Custom POSIX TZ** input, automatic browser DST synthesis, live clock-vs-browser drift readout, **Sync time + TZ** / **Sync time only** / **Sync TZ only** buttons, manual date and time, hardware RTC indicator, 12/24-hour, page turn and timing, **Sleep / blackout mode** (`Off`, `Schedule`, `Auto Sunset–Sunrise`) with sun readout + **Blackout now**, alarm |
| **Face** | Clock face and every face option and colour |
| **Look** | Palette, backlight/text/both sliders, type size, typeface, **Night paper** (`Off`, `Auto 21–06`, `Auto Sunset–Sunrise`, `Always dark`) with today’s sun times + live preview, zen page, **Screen rotation** (`0°` / `90°` / `180°` / `270°`) |
| **Greeting** | On/off, how it appears, greeting after sleep, words, picture buttons, lettering, size, word colours, live preview, "Save and show it on the clock" |
| **Effects** | Falling things, where, how many, on the zen page, on the clock faces, live preview |
| **Weather** | On/off, units, place, plus **Latitude** and **Longitude** (auto-populated from place lookup or editable offline for Sunrise/Sunset) |
| **Quotes** | Flash-only onboard Literature library summary and single-file OTA story (no SD card required; information only) |
| **System** | Board name, IP address, firmware version, hardware **Clock RTC** chip (`verified via NTP` / `RTC-first`), **Power mode**, **Rotation**, today’s **Sun** sunrise/sunset, **Screen** awake/blackout & Night paper status, **Battery** telemetry (`%`, `mV`, charging, sensor source) when present; Restart clock |
| **Update** | Upload a firmware `.bin` file, install from an address (single-file GitHub `latest.txt` / `.bin` OTA), or opt into a **Daily check (03:00)** |
| **Advanced** | **Power & battery saver** (`0..3`: Always on, Sleep Wi-Fi when idle, Auto on battery, Battery saver always), export and import of settings, full `0x08..0x77` I2C device discovery (touch, expander, RTC, INA219) |

The preview at the top of the page follows the palette, brightness, typeface and clock face you choose. Every
tab except Page has a **Save on the clock** button; the palette is also sent as soon as you choose it.

## 19. Backup: export and import

Advanced → **Back up settings**:

- **Export settings** downloads `literature-clock-settings-YYYY-MM-DD.json`: every setting the phone page
  controls (including `night` `0..3`, `sleepMode` `0..2`, `lat`, and `lon`), in a small JSON file (format
  tag `literature-clock-settings`, version 1).
- **Import settings** reads such a file and sends it to the clock. Only known settings are used, colours are
  checked, legacy `sleep: "1"` backups without `sleepMode` import cleanly, the greeting is cut to 100
  characters, and the page reports how many settings it loaded.

The alarm is included. Wi-Fi passwords and the date and time are not.

## 20. Software updates

Because the Literature pack lives inside firmware flash and **Waveshare 7" & 4.3B use `default_16MB.csv`**
(two ~6.2 MB OTA app slots at ~29% used, leaving ~4.6 MB headroom), all features and updates deploy as a
**single-file OTA binary with no SD card required**. The **Update** tab supports:

- **Upload .bin** — choose the `firmware.bin` built for this board (`waveshare-lcd7`, `waveshare-lcd43`, `m5paper`, or `cyd-28`).
- **Check for update** — compares `FW_VERSION` (`1.2.9`, also shown in the Web UI top bar, on-device **Setup** header, and Serial `help`) against `firmware/<board>/latest.txt` without installing (`POST /api/update-check`, Web UI **Check for update**, or serial `checkupdate` / `update check`).
- **Install from address / on-device Setup → Check update** — fetches `latest.txt` and installs `latest/firmware.bin` if newer, showing a live animated **`0%–100%` progress bar** on the clock screen (also shown during `.bin` uploads and serial `update`).
- **Daily check (03:00)** — **Off** by default (opt-in on the **Update** tab, `/api/settings` `autoUpdate`,
  or serial `autoupdate on`). Checks `firmware/<board>/latest.txt` from
  [markusvankempen/literature-clock](https://github.com/markusvankempen/literature-clock) once per local calendar
  day between `03:00` and `03:04` when Wi-Fi is connected, and installs `latest/firmware.bin` if the remote
  version differs (skipped automatically when offline or in Battery saver).

Leave the clock powered; it restarts when the copy is stored. **Restart clock** is on the System tab and in
Setup.

## 21. Power saving and offline mode

- **RTC-first timekeeping (one-shot NTP verification):** When a hardware RTC (`PCF85063A`, `BM8563`, or external `DS3231`) is present and holds valid time, it serves as the primary clock. When Wi-Fi connects, the clock verifies/updates the RTC **once** via NTP (up to a `30 s` window) and immediately calls `esp_sntp_stop()` so no periodic 6-hour or background SNTP polling runs. Every hour the ESP32 internal clock re-aligns from the hardware RTC, and 15-minute NVS checkpoints (`clockTimeSaveFlashCheckpoint()`) never overwrite the hardware RTC with unverified internal ticks. Non-RTC builds (e.g., `cyd-28`) continue 6-hour NTP while Wi-Fi is connected.
- **Power & battery saver** (`powerMode` `0..3` in **Web UI → Advanced**, **Setup → Brightness** top-right chip on Waveshare, **Setup → Wi-Fi** `Power` chip on M5Paper, and `power 0..3` over serial):
  - **`0` — Always on:** Wi-Fi stays reachable; full user brightness/ink.
  - **`1` — Sleep Wi-Fi when idle:** Turns Wi-Fi off after `120 s` of phone-page inactivity (`WIFI_OFF` + `80 MHz` CPU except Waveshare RGB). Onboard flash quote turns **never** wake Wi-Fi; touching the phone/Wi-Fi icon wakes it (`30 s` grace).
  - **`2` — Auto on battery:** Activates automatically whenever discharging on battery (`M5Power` VBUS `< 4.3 V`, `INA219` `0x40`, or `-DBATTERY_ADC_PIN`) and restores normal power when USB/charging returns: scales backlight & ink to **55%** (on top of Night Paper), pauses ambient animations, skips weather fetches, sleeps Wi-Fi after **`45 s`** idle, and skips boot Wi-Fi completely when a valid hardware RTC is present.
  - **`3` — Battery saver always:** Same power-saving rules unconditionally (55% dim, ambient paused, 45 s Wi-Fi sleep, skip boot Wi-Fi on valid RTC) — useful on stock Waveshare 7″/4.3B boards running on a LiPo without an external voltage gauge.
- **Standalone / offline mode:** powers Wi-Fi completely off (`WIFI_OFF`) and runs strictly from the hardware RTC (`PCF85063A` on 4.3B, `BM8563` on M5Paper, optional external `PCF85063A`/`DS3231`), internal oscillator, flash quotes, and NVS settings. Toggle Standalone/Wi-Fi or start/stop the temporary **Clock AP** (`LiteratureClock`) directly from **Display (`Setup → More` or AP QR screen)**, **Web UI (`Wi-Fi` / `Advanced`)**, or **Serial (`standalone`, `wifi on|off`, `ap on|off`)**.
- **Battery indicator icon & telemetry (`src/battery.cpp`):** automatically reads and caches (10 s) battery
  level across supported hardware:
  - **M5Paper:** onboard `M5Power` PMIC (`M5.Power.getBatteryLevel()` / voltage / charging state).
  - **Waveshare (I2C auto-detect):** probes `INA219` (`0x40`) bus monitor at startup using thread-safe I2C
    transactions (`0x36` is skipped because CH422G occupies `0x20–0x27` and `0x30–0x3F`)
  - **Waveshare ADC (opt-in only):** enabled only when compiled with `-DBATTERY_ADC_PIN=<gpio>` (and optional
    `-DBATTERY_ADC_DIVIDER`, default `2.0`), mapping voltage via a LiPo discharge table; disabled by default
    since stock Waveshare 7"/4.3B boards do not wire VBAT to an MCU ADC pin.
  - Displays an SVG badge in the phone web UI header, detailed telemetry (`%`, `mV`, charging, sensor name) on
    the System tab, and a compact battery icon on the Waveshare header and M5Paper e-ink header (including a
    bolt/fill bar when charging and red accent at low charge).
- The lamp can be dimmed to 20/255, or off with sleep hours.

## 22. Web API

The phone page is a thin client over these routes, served by the clock. There is no authentication, so keep
the clock on a network you trust.

| Route | Method | Purpose |
|---|---|---|
| `/` | GET | The phone page |
| `/api/status` | GET | Current settings, quote, weather, network, time, `sun.{valid,polarDay,polarNight,sunrise,sunset,sunriseMin,sunsetMin}`, `nightActive`, `sleepMode`, `sleeping`, `powerMode`, `batterySaverActive`, `lat`, `lon`, `device.{rtc,hwRtc,rtcVerified,battery,batteryMv,charging,onBattery,batterySource}` (JSON) |
| `/api/lists` | GET | Time zones and palette names |
| `/api/settings` | POST | Change settings (form fields below) |
| `/api/scan` | GET | Wi-Fi networks in range |
| `/api/join` | POST | Join a network (`ssid`, `pass`) |
| `/api/forget` | POST | Forget the saved Wi-Fi |
| `/api/next` | POST | Turn the page |
| `/api/greet` | POST | Show the greeting now |
| `/api/sleep` | POST | Trigger/clear instant blackout (`on` or `mode` = `1`/`on`, `0`/`off`/`wake`, or `toggle`) |
| `/api/ap` | POST | Start, stop, or toggle the `LiteratureClock` setup AP (`on` or `mode` = `1`/`on`/`start`, `0`/`off`/`stop`, or `toggle`) |
| `/api/reboot` | POST | Restart |
| `/api/update` | POST | Upload a firmware file (shows live `0–100%` progress bar on screen) |
| `/api/update-url` | POST | Install firmware from an address (`url`, supports `.bin` or `latest.txt` → `latest/firmware.bin`, with `0–100%` progress bar) |
| `/api/update-check` | POST | Check `latest.txt` against running `FW_VERSION` without installing (`running`, `remote`, `updateAvailable`) |
| `/api/i2c` | GET | I2C device discovery |
| `/api/device` | GET | Name, IP and phone-page address |

`/api/settings` fields (send only what you change): `tz` (built-in or POSIX rule), `clock24`, `interval`, `random`, `power` (`0`–`3`, or `on`/`off`), `powerMode` (`0`–`3`), `theme`,
`lamp`, `ink`, `font`, `fontFace`, `night` (`0`–`3`), `zen`, `clockFace` (`0`, `2`–`8`), `rotation` (`0`, `90`, `180`, `270`), `seconds`, `numbers`, `faceTime`, `faceDate`,
`faceWeather`, `faceInvert`, `facePaper`, `blink`, `minutes`, `rectHands`, `faceBg`, `faceBg2`, `faceDot`,
`faceSec`, `vinNum`, `vinDot`, `vinMin`, `greetOn`, `greetOnWake`, `greetText`, `greetFont`, `greetAnim`,
`greetSize`, `greetColorMode`, `greetCol1`, `greetCol2`, `sleep` (`0`–`2`), `sleepMode` (`0`–`2`), `sleepFrom`, `sleepTo` (`HH:MM`), `alarm` (`0`/`1`), `alarmAt` (`HH:MM`), `ambient`,
`ambientWhere`, `ambientAmount`, `ambientZen`, `ambientFaces`, `weather`, `units`, `place`, `lat`, `lon`, `apPass` (blank or `8`–`63` chars), `autoUpdate` (`0`/`1`), `epoch` (UTC Unix
seconds, parsed via `strtoll` to set system clock, hardware RTC, and NVS directly), or
`date` with `tod` (and optionally `sec`, `0`–`59`). Colours are `#rrggbb`. The reply is the new status.

## 23. Serial console

115200 baud (use either the **UART** or **native** USB-C port on Waveshare 7", or the **native** USB-C port on 4.3B). Type `help` for the list.

| Command | Effect |
|---|---|
| `status` | Wi-Fi, time, settings, hardware RTC (`hwRtc`), sun times (`sun`), sleep mode, and battery reading |
| `tz <city\|posix\|list>` | Set time zone or list all 24 built-in zones |
| `font s\|m\|l\|auto` | Type size (`auto` maximizes quote text without scrolling/clipping) |
| `typeface <serif\|smooth\|bold\|italic\|0-3>` | Font family (`fontface` alias) |
| `bright\|lamp\|ink <20-255>` | Backlight/ink/combined brightness levels |
| `interval 1\|5\|15\|30\|60` | Page turn |
| `random on\|off` | Any minute inside the window |
| `power 0\|1\|2\|3\|always\|idle\|battery\|saver` | Power / Wi-Fi / battery saver mode (`0` always on, `1` sleep idle Wi-Fi, `2` auto on battery, `3` battery saver always) |
| `standalone\|offline on\|off`, `weather on\|off` | Toggle Standalone RTC mode (`WIFI_OFF`) / Weather |
| `units c\|f`, `place <city>`, `coords <lat> <lon>` | Weather & Sunrise/Sunset location |
| `theme 0-11`, `face <page\|classic\|1950\|flip\|digital\|split\|square\|nixie>`, `faceopt <opt> <on\|off>`, `rotate <0\|90\|180\|270>`, `night off\|auto\|sun\|on`, `clock 12\|24`, `datetime YYYY-MM-DD HH:MM` | Look, clock face/options, orientation, manual clock |
| `ambient [0-11\|where\|amount\|zen\|faces ...]` | Falling effects kind (`0..11`, incl. balloons/fireworks/christmas), placement, count, zen/faces toggles |
| `sleep off\|on\|sun\|now\|wake\|<from> <to>`, `blackout on\|off` | Scheduled/sun blackout or instant screen sleep/wake |
| `zen on\|off` | The words alone |
| `setup [done]`, `factoryreset confirm` | Re-open first-time setup wizard (or mark complete) / wipe NVS &amp; Wi-Fi |
| `greet [on\|off\|show\|wake on\|off\|font 0-2\|anim 0-14\|size 40-130\|color 0-3\|text <msg>]` | Greeting toggle, show now, style (`0..14`), or message |
| `alarm 07:30\|off` | Alarm |
| `wifi [on\|off\|<ssid> <pass>\|forget]`, `scan` | Control Wi-Fi radio, join a network, clear credentials, or scan |
| `ap [on\|off\|pass <pw>\|off]` | Start/stop Clock AP, check status, or set WPA2 password (`8`–`63` chars) |
| `autoupdate on\|off`, `checkupdate`, `update [check]` | Toggle daily `03:00` GitHub OTA check, check `latest.txt` only, or check and install if newer |
| `quote`, `url`, `i2c`, `reboot` | Actions (`i2c` scans `0x08..0x77` on Waveshare `GPIO 8/9`) |

Waveshare panel tuning (these reboot the clock): `panel` (show timing), `panel pclk 10-20`, `panel color 8|16`,
`panel porch <hbp> <hfp> <vbp> <vfp>`, `panel reset`; `freeze on|off` to stop drawing; `bl on|off`, `bl test`,
`bl pwm on|off`, `bl <percent>` for the lamp. The panel reads the whole picture out of PSRAM continuously, so
pixel clock and colour depth decide how steady the picture is. See `include/ws7_panel.h`.

## 24. Stored settings and defaults

Settings live in flash (NVS namespace `litclock`) and survive a restart or power cut. `settingsSave()` diffs
against the last committed snapshot (`gSaved`) so only changed keys write to NVS (reducing flash wear).
Weather place coordinates (`geoQ`, `geoLat`, `geoLon`) cache separately in NVS namespace `litwx`, and runtime
scheduling uses wrap-safe `millisReached` / `millisElapsed` helpers across the ~49.7-day `millis()` rollover.

| Setting | Key | Default |
|---|---|---|
| Time zone | `tz` | Eastern (`EST5EDT,M3.2.0,M11.1.0`) |
| Palette | `theme` | 7 Midnight on the Waveshare panel (0 elsewhere) |
| Lamp / text level | `lamp` / `ink` | 80 on the Waveshare panel (140 elsewhere) |
| Type size / typeface | `font` / `face` | Medium / Serif |
| Clock face / Rotation | `cface` / `rot` | Page (`0`) / `0°` (`0`) |
| Face options | `secs` `fnums` `ftime` `fdate` `fwx` `finv` `fblink` `fpap` `fmin` `frect` | seconds off, numbers on, others off, minute marks on |
| Split / 1950 colours | `fbg` `fbg2` `fdot` `fsec` `vnum` `vdot` `vmin` | white / blue, sky-blue points, near-black numbers |
| Greeting | `gron` `grtx` `grfont` `granim` `grsz` `grcm` `grc1` `grc2` `grwk` | off, empty, handwriting, written, 100%, page ink, red / blue, wake-greeting off |
| Falling things | `amk` `amw` `ama` `amz` `amf` | none, over the words, some, drift on zen and faces |
| Sleep / blackout | `slon` `slmd` `slfr` `slto` | off (`sleepMode=0`), 23:00–07:00 (`1`=schedule, `2`=sunset–sunrise) |
| Alarm | `almOn` `almHour` `almMin` | off, 07:00 |
| Safer quotes (retired) | `sfw` | ignored |
| Interval / timing | `interval` `rndTurn` | 1 minute, on the mark |
| 12/24-hour | `clock24` | 24-hour |
| Weather | `wxOn` `wxPlace` `wxUnit` `lat` `lon` | on, Halifax, metric |
| Night paper | `night` | off (`0`=off, `1`=auto 21–06, `2`=always dark, `3`=sunset–sunrise) |
| Zen | `zen` | off |
| Power & battery saver | `pwrMode` (`pwrSave`) | `0` Always on (`1` idle Wi-Fi 120s, `2` auto on battery, `3` battery saver always) |
| Daily auto-update | `autoUpd` | off (`0` manual only, `1` check `03:00` daily if connected) |
| Setup AP password | `appass` | empty (`open`; `8`–`63` chars enables WPA2 on `LiteratureClock`) |
| First-time setup | `setup` | false on factory reset (`1` once finished or skipped) |
| Offline | `offline` | off |

Out-of-range values from the phone, the serial console or an import are clamped when saved.

## 25. Building, flashing and the preview server

Install [PlatformIO](https://platformio.org/).

```bash
pio run -e waveshare-lcd7 -t upload   # Waveshare 7" (default)
pio run -e waveshare-lcd43 -t upload  # Waveshare 4.3B (onboard PCF85063A RTC)
pio run -e m5paper -t upload          # M5Paper
pio run -e cyd-28 -t upload           # Cheap Yellow Display
pio device monitor                    # serial console, 115200
```

The build runs `scripts/pio_pack_quotes.py`, which packs the Literature collection into
`src/generated/quotes.pack` and embeds it in the firmware. Optional local home Wi-Fi credentials can be
placed in an untracked `include/secrets.h`; public builds without `secrets.h` compile cleanly without
hardcoded fallback credentials.

**Automated tests (no hardware required):**

```bash
python3 scripts/run_tests.py          # full 7-stage check (C++, quotes, web JS/UI, HTTP, public build, docs, 4 Pio envs)
python3 scripts/run_tests.py --fast   # skip PlatformIO builds (~1 second)
python3 scripts/test_device_live.py   # live hardware test over HTTP /api/* + USB Serial CLI
node scripts/capture_manual_screenshots.mjs  # refresh Web UI PNGs in docs/images/manual/
```

**Preview server (no hardware):**

```bash
python3 preview/server.py
```

- `http://127.0.0.1:8765/device` — the phone page, backed by a simulated clock (including simulated RTC and
  battery telemetry). It serves the exact HTML from `include/web_app_html.h`.
- `http://127.0.0.1:8765/` — the product-UI prototype.
- `http://127.0.0.1:8765/preview/` — small-TFT bezel demo.

The server does not draw the panel's screens (falling things, greeting shows, faces on glass); those need
the hardware.

## 26. Project layout

| Path | What is there |
|---|---|
| `src/main.cpp` | Start-up, main loop, page turning, sleep hours, alarm, touch/gesture routing |
| `src/display_ws7.cpp` | Waveshare 7" & 4.3B drawing: page, battery icon, faces, greeting, effects, setup screens, gestures |
| `src/display_tft.cpp`, `src/display_epd.cpp` | The CYD and M5Paper (including e-ink battery icon) drawing |
| `src/settings.cpp`, `include/settings.h` | Settings, defaults, clamping, delta-diffed flash storage |
| `src/web_app.cpp`, `include/web_app_html.h` | The phone page, API, browser time/TZ sync, OTA upload/URL handlers |
| `src/quotes_offline.cpp`, `src/quote.cpp` | The onboard library and quote clean-up |
| `src/clock_time.cpp`, `include/clock_math.h` | NTP, PCF85063A/DS3231/BM8563 RTC codecs, wrap-safe timers, pure math helpers |
| `src/battery.cpp`, `include/battery.h` | M5Power / INA219 (`0x40`) / opt-in ADC battery telemetry |
| `src/weather.cpp` | Mutex-protected background Open-Meteo task (`litwx`) + NVS geocode cache |
| `src/wifi_portal.cpp`, `src/wifi_onscreen.cpp` | Phone setup (optional `include/secrets.h`) and the on-screen Wi-Fi screens |
| `src/serial_cli.cpp`, `src/i2c_scan.cpp` | Serial console and I2C bus discovery |
| `include/themes.h`, `include/timezones.h` | Palettes and time zones |
| `include/ws7_panel.h`, `boards/` | Waveshare panel timing, thread-safe I2C helpers, backlight, 16 MB board JSON |
| `include/digit_outlines.h`, `include/outline_fonts.h` | Generated outline digits and greeting fonts |
| `tests/`, `scripts/` | Host C++17 unit tests (`tests/test_firmware_logic.cpp`), `scripts/run_tests.py`, `scripts/test_device_live.py`, `scripts/capture_manual_screenshots.mjs`, quote/font generators |
| `quotes/` | Quote sources and statistics |
| `preview/`, `web/` | Preview server and the web prototype |
| `mcp/`, `alexa/`, `chrome/` | Related products with their own docs |

## 27. Known limits

- **The greeting is plain letters:** accents become plain letters, and only the pictures listed in
  [section 10](#10-start-up-greeting) are drawn.
- **Wi-Fi is 2.4 GHz only.** There is no sound output, so no read-aloud.
- **Effects run on the quote page, the zen page and the clock faces** (if switched on), and pause while a
  setup screen is open.
- **Faces that redraw each second** (seconds or a blinking colon) may show a brief flicker in falling
  pieces.
- **The `cyd-28` build is compile-tested only** (~88.6% of flash); it has not been run on a board since the
  font-include fix.
- **Waveshare flash layout:** both `waveshare-lcd7` and `waveshare-lcd43` use `default_16MB.csv` (~30.5% / ~30.3% of the
  6.55 MB app partition used, leaving ~4.56 MB free for new features and dual-slot OTA).
- **Waveshare battery monitoring:** stock Waveshare 7" and 4.3B boards do not route battery voltage to an MCU
  ADC pin (`GPIO 6` is only exposed on the external sensor header), and the CH422G IO expander occupies I2C
  `0x20–0x27` and `0x30–0x3F` (so `MAX17048` at `0x36` cannot share the bus); external battery monitoring on
  Waveshare uses an I2C `INA219` (`0x40`) sensor or an external resistor divider with `-DBATTERY_ADC_PIN`.
- The phone page has no login. Anyone on the same network can change the clock.

## 28. Credits and licences

- Quote collection and the original web clock: [Johannes Enevoldsen](https://github.com/JohsEnevoldsen/literature-clock),
  **CC BY-NC-SA 2.5**.
- Idea: [Jaap Meijers, E-reader clock](https://www.instructables.com/Literary-Clock-Made-From-E-reader/).
  Crowd-sourced corpus: The Guardian.
- Caveat and Great Vibes fonts: SIL Open Font Licence 1.1 (`scripts/fonts/`).
- This firmware is offered under the same **CC BY-NC-SA 2.5** terms; see `LICENCE.md`.
- Public documentation & prebuilt OTA firmware releases (no source code): https://github.com/markusvankempen/literature-clock
- Private source repository (all code, tools, and tests): https://github.com/markusvankempen/literature-clock-dev

## Author Markus van Kempen

[Research | Floor 7½ 🏢🤏](https://markusvankempen.github.io/)

No bug too small, no syntax too weird.
