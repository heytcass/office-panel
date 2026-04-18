# Office Panel — Technical Specification

**Status:** Ready for implementation
**Target:** Claude Code handoff
**Last updated:** 2026-04-18

---

## 1. Project summary

`office-panel` is a battery-powered, wall-mounted e-paper display that shows the real-time booking status of the home office (the primary Teams call location in Tom and Christen Cassady's home). It is mounted in the hallway adjacent to the office door, facing approaching foot traffic, and serves as a hard-signal indicator of whether the room is currently in use.

The system consists of three components:

1. **Hardware:** A Seeed Studio reTerminal E1002 running ESPHome firmware.
2. **Backend:** A Home Assistant Local Calendar instance plus a set of helpers, template sensors, scripts, and automations that expose booking state to the panel and accept booking actions from it.
3. **Input surfaces:** A Home Assistant Lovelace dashboard card (phone/desktop), four physical buttons on the panel itself, and HA Voice PE voice intents.

The panel answers one question, visible from six feet away: **can I knock on this door right now?**

---

## 2. Goals and non-goals

### Goals

- Provide glanceable, unambiguous room status (in use / open / starting soon) to anyone in the hallway.
- Allow either household member to book the room from their phone, from a voice assistant in-room, or with a single button press at the panel itself.
- Detect and refuse booking conflicts before they are created.
- Operate on battery power for months between charges, with proactive low-battery alerts.
- Be reliable when internet connectivity is degraded (LAN-only operation).

### Non-goals (do not implement)

- **No album art or Sendspin integration.** Evaluated and rejected due to refresh cadence.
- **No Teams/Zoom/Meet auto-detection** of calls to trigger bookings. The panel does not attempt to infer call importance.
- **No Google Calendar integration.** Household Google Calendars remain separate and are not mirrored to or from this calendar.
- **No weather, F1, network health, package tracking, or other "dashboard" content.** The panel has one job.
- **No remote booking.** The system is not exposed outside the LAN. No Nabu Casa, no reverse-proxied access.
- **No touch input.** The E1002 has no touchscreen; all interaction is via buttons, voice, or HA UI.
- **No individual authentication.** Booking owner is inferred from input source (per-voice-device mapping, dashboard-user mapping, button press = "whoever is at the panel"). This is a household system, not a multi-tenant one.
- **No secondary-office panel.** Out of scope for this project. A second instance may be built later as a separate project.

---

## 3. Hardware

### Device

- **Seeed Studio reTerminal E1002**
  - 7.3" color E Ink Spectra 6 ACeP panel (800×480, 6 native pigments: black, white, yellow, red, green, blue)
  - ESP32-S3R8 (8 MB PSRAM, 32 MB flash)
  - 2.4 GHz Wi-Fi + BLE 5.0
  - 2000 mAh LiPo battery, USB-C charging
  - 3 GPIO buttons (left `GPIO5`, right `GPIO4`, green/top `GPIO3`), piezo buzzer, SHT40 temperature/humidity sensor, microphone, microSD slot
  - No partial refresh; full refresh ≈ 15–30 seconds

### Physical install

- Wall-mounted next to the office door, facing the hallway. Not visible from inside the office.
- Battery-only power. No wired power available at install location.
- Mounting hardware selection is out of scope for this spec.

### Firmware

- **ESPHome 2025.11 or newer** (required for native `Seeed-reTerminal-E1002` model in the `epaper_spi` platform).
- Deployed via the Home Assistant ESPHome app (the former "ESPHome add-on"), installed on Tom's production HA instance.
- OTA updates via HA ESPHome app once initial flash is complete.

### Relationship to `esphome-device-library`

This project is its own repository. However, the hardware-level E1002 configuration (pin definitions, display model declaration, sensor/button/buzzer component setup) should be authored as a reusable ESPHome package in `heytcass/esphome-device-library` (local path: `/home/tom/Projects/esphome-device-library`) under a device path such as `devices/seeed/reterminal_e1002/`. `office-panel` then imports that package via ESPHome's `packages:` include syntax.

Application-specific logic (display modes, calendar integration, button action handling, mode state machine, voice intent hooks) stays in `office-panel` and does not belong in the device library.

**Working across both repos.** Claude Code sessions should be launched from `/home/tom/Projects/` (the parent directory) so both repos share a single context tree, or alternatively invoked with `--add-dir /home/tom/Projects/esphome-device-library` when scoped to `office-panel`.

**Audit existing work before authoring new abstractions.** Before writing any E1002 hardware package, the implementer must:

1. Audit the existing state of `esphome-device-library` for any prior E1002 work — branches, draft PRs, issues, partial files.
2. Survey the broader ESPHome community for reference configurations. Known starting points: Clelia Rella's "Style over Substance" build log (the OHF-amplified reference build), tutoduino's reTerminal-to-Home-Assistant guide, Home Assistant community forum thread 970545, and ESPHome GitHub issue #12709 (Spectra 6 driver dithering discussion).
3. Propose a package structure that reconciles library conventions with what has proven to work in the wild.
4. Obtain Tom's review of the proposed structure before writing the package.

This audit is an appropriate task to dispatch to a Claude Code research subagent (Task tool), running in parallel while other Phase 1 work proceeds.

---

## 4. Calendar and data model

### Backend

- **Home Assistant Local Calendar integration.** No external calendar services.
- Calendar entity: `calendar.office`.
- The calendar is a dedicated entity for this system. It is not merged with or synchronized to any household Google Calendar, work Outlook calendar, or shared family calendar.

### Event schema

All bookings are events on `calendar.office` with the following structure:

| Field | Value |
|---|---|
| `summary` | Owner display name. Values: `"Tom"`, `"Christen"`, `"Occupant"`. |
| `description` | JSON-serialized metadata: `{"source": "phone" \| "button" \| "voice", "created_at": ISO8601}`. Source is for debugging and future UI affordances; do not surface in Phase 1 UI. |
| `start` | ISO8601 datetime, always aligned to a 15-minute boundary. |
| `end` | ISO8601 datetime, always aligned to a 15-minute boundary, strictly greater than `start`. |
| `location` | Empty. |
| `uid` | Auto-assigned by Local Calendar. |

### Booking constraints

- All `start` and `end` times are aligned to `:00`, `:15`, `:30`, or `:45` of the hour. Voice and button inputs that would create off-boundary events must round to the nearest 15-minute boundary (round down for start, round up for end if duration would otherwise be reduced).
- Minimum booking duration: 15 minutes.
- Maximum booking duration: 4 hours (prevents accidental "all day" bookings via voice misparse).
- Overlapping bookings are not permitted. Any creation attempt that would overlap an existing event must be rejected before the event is created.

---

## 5. Display modes and state machine

### Mode definitions

The panel renders one of five modes at any given time. Mode is computed by a template sensor in HA (`sensor.office_mode`) that the firmware subscribes to.

#### Mode: BOOKED

- Trigger: current time falls within a `calendar.office` event.
- Background: full red (Spectra 6 red).
- Primary text: `IN USE` (huge, white, centered upper third).
- Secondary text: owner name (large, white, centered middle).
- Tertiary text: `Until HH:MM` (medium, white, centered lower third).
- No other content.

#### Mode: STARTING_SOON

- Trigger: no current event, but next event starts within 15 minutes.
- Background: full amber (Spectra 6 yellow).
- Primary text: owner name + start time (huge, black, centered upper half).
- Secondary text: `Starting in N minutes` (medium, black, centered lower half).
- Counts down in 1-minute increments but refreshes only on mode transitions and wake cycles (see §10).

#### Mode: AVAILABLE

- Trigger: no current event, and no event starting within 15 minutes.
- Background: full green (Spectra 6 green).
- Primary text: `OPEN` (huge, white, centered upper third).
- Secondary text: if a booking exists within the next 60 minutes, display `Next: HH:MM — Owner` (medium, white, centered middle). Otherwise, omit.
- Tertiary text: nothing.

#### Mode: WEEKEND

- Trigger: day is Saturday or Sunday, regardless of calendar state.
- Background: full green.
- Primary text: `OPEN` (huge, white, centered upper third).
- Secondary text: day name + date, e.g. `Saturday, April 18` (medium, white, centered middle).
- Bookings made on weekends are honored and override weekend mode with BOOKED while active. Weekend mode resumes on booking end.

#### Mode: OVERNIGHT

- Trigger: time is between 22:00 and 07:00 local, any day, and no active booking.
- Background: full white.
- Primary text: current time `HH:MM` (medium, black, centered).
- Secondary text: nothing.
- Purpose: minimize visual weight in a dark hallway while maintaining a "the panel is alive" signal.

### State machine

Transitions evaluated on every wake and on every calendar state change:

```
                   ┌──────────────┐
                   │   OVERNIGHT  │
                   └──────┬───────┘
                          │ 07:00 local
                          ▼
  ┌────────────┐   ┌──────────────┐   ┌──────────────┐
  │  WEEKEND   │◄──┤  AVAILABLE   ├──►│STARTING_SOON │
  └─────┬──────┘   └──────┬───────┘   └──────┬───────┘
        │                 │                   │
        │ booking active  │ booking active    │ booking starts
        ▼                 ▼                   ▼
                   ┌──────────────┐
                   │    BOOKED    │
                   └──────┬───────┘
                          │ booking ends
                          ▼
                   (return to applicable mode)
                          │ 22:00 local
                          ▼
                   ┌──────────────┐
                   │   OVERNIGHT  │
                   └──────────────┘
```

Precedence when multiple modes could apply:

1. BOOKED (active event always wins)
2. OVERNIGHT (22:00–07:00)
3. WEEKEND (Sat/Sun, 07:00–22:00)
4. STARTING_SOON
5. AVAILABLE (default weekday)

---

## 6. Button behavior

### Button layout

Three physical buttons on the E1002, hardware-fixed to the pins below:

| Button | Pin | Short press | Long press (≥1s) |
|---|---|---|---|
| Left | `GPIO5` | Book 30 minutes from now | — |
| Right | `GPIO4` | Book 1 hour from now | — |
| Green / top | `GPIO3` | Book until next event (or 2 hours if no next event within 4 hours) | End current booking now |

Rationale for the green-button long-press: the green button is semantically the "primary" button (it is the deep-sleep wake pin in most reference builds for this hardware), which makes it the natural home for the two thematically opposite actions — starting and stopping a booking. Placing "end booking" behind a long-press also makes it hard to trigger accidentally.

### Interaction model

- **Commit-not-compose.** Each button press fires a complete action. No multi-press sequences, no modifier buttons, no time-adjustment navigation. The one exception is the green button's short-vs-long distinction described above; the firmware implements this as a press-duration discriminator, not a multi-step composition.
- **Immediate buzzer feedback.** A short (~50 ms) buzzer beep confirms press reception before the display refresh begins. This fills the latency gap introduced by e-paper refresh time. A distinct second beep fires at the 1 s threshold to confirm long-press recognition before the action commits.
- **All three buttons wake the device from deep sleep.** Implemented via ESP32-S3 EXT1 wake with a bitmask covering GPIO3, GPIO4, and GPIO5; the firmware uses `esp_sleep_get_ext1_wakeup_status()` to determine which button woke the device. Wake-to-beep latency target: under 500 ms. Wake-to-display-update latency target: under 30 seconds.

### Conflict handling

If a button press would create an event that overlaps any existing `calendar.office` event (including events starting within the requested window):

1. The booking is not created.
2. The buzzer emits a double-beep (two short tones) instead of a single beep.
3. The display transitions to a CONFLICT mode for one refresh cycle:
   - Background: full amber
   - Primary text: `CONFLICT` (huge, black)
   - Secondary text: the conflicting booking owner + time, e.g. `Christen at 14:30`
4. On the next wake (or after ~60 seconds, whichever comes first), the panel returns to its computed mode.

The green-button long-press (End current booking) has no conflict case — if there is no current booking, the press is a no-op with a single short error beep (distinguishable from confirm beep by pitch or pattern, TBD in Phase 2).

### Button-triggered booking owner

Button-initiated bookings are attributed to `"Occupant"` unless a more specific identity-resolution mechanism is added in a later phase. Rationale: the panel cannot identify who pressed the button, and arbitrarily assigning "Tom" or "Christen" would cause confusion. "Occupant" is a neutral label that makes no false identity claim. Either household member can rename the event from the HA dashboard if attribution matters for a given booking.

---

## 7. Voice intents

### Platform

Home Assistant Voice PE devices running the HA Assist pipeline. Intents are defined as HA intent scripts, invoked via standard Assist phrasing.

### Intent: `BookOffice`

Phrasings to match:

- "Book the office"
- "Book the office for {duration}"
- "Book the office until {time}"
- "Book the office all afternoon" (interpreted as now until 17:00)
- "Book the office all morning" (interpreted as now until 12:00)

Slots:
- `duration` (optional, default 30 minutes; max 4 hours)
- `end_time` (optional, mutually exclusive with `duration`)

Owner resolution:
- Each Voice PE device is assigned to a household member in HA configuration.
- The office Voice PE device maps to Tom by default (as the primary office user); other devices map per their primary user.
- Conflict: same behavior as button conflict — do not create event, speak the conflict back: `"Conflict — Christen has it at 2:30."`

Success response: `"Office booked until HH:MM."`

### Intent: `ExtendOfficeBooking`

Phrasings:

- "Extend my booking"
- "Extend the office by {duration}"
- "Keep the office for another {duration}"

Behavior:
- Find the current active `calendar.office` event.
- Extend `end_time` by `duration` (default 15 minutes).
- Reject if extension would overlap a subsequent booking. Response: `"Can't extend — Christen has it at HH:MM."`
- Reject if no active booking. Response: `"The office isn't booked right now."`

Success response: `"Extended until HH:MM."`

### Intent: `ReleaseOffice`

Phrasings:

- "I'm done"
- "Release the office"
- "End my booking"
- "I'm done with the office"

Behavior: End the current active booking now by setting its `end_time` to the current clock time (rounded down to the nearest 15-minute boundary, or to `start_time + 15min` if current time is less than 15 minutes after start).

Success response: `"Office released."`
No active booking response: `"The office wasn't booked."`

### Intent: `OfficeStatus`

Phrasings:

- "Is the office free?"
- "Is the office booked?"
- "Who has the office?"
- "What's the office status?"

Responses:
- BOOKED: `"{Owner} has it until HH:MM."`
- AVAILABLE (no upcoming): `"The office is open."`
- AVAILABLE (with upcoming within 60 min): `"Open until {Owner} at HH:MM."`
- STARTING_SOON: `"{Owner} has it starting in N minutes."`

### Intent: `OfficeNextFree`

Phrasings:

- "When is the office free?"
- "When is the next opening?"

Response: `"Free after HH:MM"` or `"Open now"` if currently available.

---

## 8. Home Assistant contract

This section defines the full set of HA entities, helpers, scripts, and automations that the firmware and input surfaces depend on. These are the contract; the firmware should rely only on what is listed here.

**Naming note.** The project / repo / directory is `office-panel` (hyphenated), but HA entity IDs, automation IDs, and ESPHome event names use `office_panel_*` / `office_panel.*` (underscored). Home Assistant requires entity IDs to be `[a-z0-9_]+`, which forbids hyphens. Keep this in mind when reading code: `office-panel.yaml` is a filename; `office_panel_battery` is an entity ID.

### Entities and helpers

| Entity ID | Type | Purpose |
|---|---|---|
| `calendar.office` | Local Calendar | Source of truth for all bookings. |
| `sensor.office_mode` | Template sensor | One of `BOOKED`, `STARTING_SOON`, `AVAILABLE`, `WEEKEND`, `OVERNIGHT`, `CONFLICT`. |
| `sensor.office_current_booking_owner` | Template sensor | Owner name of active booking, or `unknown` if none. |
| `sensor.office_current_booking_ends` | Template sensor | ISO8601 end time of active booking, or `unknown`. |
| `sensor.office_next_booking_owner` | Template sensor | Owner name of next upcoming booking in the next 24 hours, or `unknown`. |
| `sensor.office_next_booking_starts` | Template sensor | ISO8601 start time of next upcoming booking, or `unknown`. |
| `sensor.office_panel_battery` | Published by ESPHome | Battery level 0–100. |
| `sensor.office_panel_last_refresh` | Published by ESPHome | Timestamp of last successful render. |
| `binary_sensor.office_panel_charging` | Published by ESPHome | True when USB-C connected. **Deferred to Phase 2**: no USB-C charge-detect GPIO is documented in any primary reference for the E1002. Pin to be probed during hardware bring-up. Phase 1 low-battery automations treat "unknown" as "not charging". |

### Scripts

| Script | Parameters | Behavior |
|---|---|---|
| `script.book_office` | `duration_minutes` (int), `owner` (string), `source` (string: phone/button/voice) | Create event on `calendar.office` starting now, rounded to 15-min boundary. Reject on conflict (raise error for caller to handle). |
| `script.book_office_until_next` | `owner` (string), `source` (string) | Book from now until the next scheduled event, or 2 hours if none exist within 4 hours. Reject if conflict. |
| `script.end_office_booking` | None | End active booking now (round down to 15-min). No-op if no active booking. |
| `script.extend_office_booking` | `duration_minutes` (int, default 15) | Extend active booking. Reject on subsequent-booking overlap. |

### Automations

| Automation | Trigger | Action |
|---|---|---|
| `office_panel_button_left_book_30` | ESPHome event: `office_panel.button_pressed` with `button: "left"` | Call `script.book_office(30, "Occupant", "button")`. On error, publish `office_panel.beep_conflict` service call. |
| `office_panel_button_right_book_60` | Same, `button: "right"` | Call `script.book_office(60, "Occupant", "button")`. |
| `office_panel_button_green_book_until_next` | Same, `button: "green"` | Call `script.book_office_until_next("Occupant", "button")`. |
| `office_panel_button_green_long_end` | ESPHome event: `office_panel.button_long_pressed` with `button: "green"` | Call `script.end_office_booking`. |
| `office_panel_low_battery` | `sensor.office_panel_battery` < 20 and not charging | Send mobile notification to Tom and Christen. Throttled to once per 24 hours. |
| `office_panel_stale` | `sensor.office_panel_last_refresh` older than 2 hours during active hours | Send mobile notification to Tom only. Throttled to once per 6 hours. |

### Intent scripts

One intent script per voice intent defined in §7. All intent scripts call the corresponding `script.*` entity above and handle the speech response.

### Dashboard card

A dedicated Lovelace view named `Office` containing:

- Current mode at-a-glance (color-coded entity card)
- Current booking owner, start time, end time
- Upcoming bookings (next 24 hours, scrollable)
- Four quick-book buttons mirroring the physical panel buttons, plus a custom-duration picker for non-standard durations
- Calendar view for creating/editing/deleting events manually
- Battery level and last-refresh timestamp for the panel
- Link to ESPHome device page for diagnostics

---

## 9. ESPHome firmware requirements

### High-level behavior

The firmware is a thin client. It subscribes to HA state, renders the current mode, and forwards button presses as events. It does not own any booking logic.

### Core components

- `wifi` with credentials in `secrets.yaml`, configured for the IoT VLAN (VLAN 4) to match existing smart home device placement on Tom's UniFi network.
- `api` with encryption key in `secrets.yaml`, registered with HA.
- `ota` for updates via HA ESPHome app.
- `time` with Home Assistant time source.
- `deep_sleep` (see §10 for schedule).
- `binary_sensor` × 3 for buttons, all three wired as EXT1 deep-sleep wake sources (bitmask over GPIO3/GPIO4/GPIO5). Each button publishes `office_panel.button_pressed` with `{button: "left" | "right" | "green"}` on short press. The green button additionally publishes `office_panel.button_long_pressed` with `{button: "green"}` when held ≥ 1 s. Firmware suppresses the short-press event if a long-press is recognized (the two events are mutually exclusive for a single hold).
- `output` + `rtttl` for the piezo buzzer with named tones: `beep_confirm`, `beep_conflict` (double), `beep_error` (low single).
- `sensor` for battery voltage → percentage, published as `sensor.office_panel_battery`.
- `binary_sensor` for USB-C charging state, published as `binary_sensor.office_panel_charging`.
- `display` using `epaper_spi` platform with `model: Seeed-reTerminal-E1002`.
- `sensor.office_mode` subscribed via HA API for mode transitions.
- Corresponding subscriptions for owner, start, end, next-owner, next-start sensors.

### Display rendering

- One `lambda` display callback per mode, selected based on current value of `sensor.office_mode`.
- Fonts: a large display font (approximately 96–128 pt for primary mode text), a medium font (approximately 32–48 pt for owner/time), a small font (approximately 18–24 pt for metadata). Specific font files and sizes to be tuned in Phase 4 visual review.
- Color palette: use only the six native Spectra 6 colors (black, white, yellow, red, green, blue). Do not attempt to render intermediate colors or grayscale.

### Wake behavior

On each wake:

1. Connect to Wi-Fi and HA API.
2. Check elapsed time since last render.
3. Fetch current values for mode sensor and associated metadata.
4. If mode or displayed values have changed, re-render. If unchanged, skip the refresh (saves the ~15-second refresh cost and extends panel life).
5. Publish `sensor.office_panel_last_refresh` with current timestamp.
6. Publish current battery level.
7. Return to deep sleep.

On button-triggered wake:

1. Immediately fire buzzer confirm tone (before Wi-Fi connection completes).
2. Connect to Wi-Fi and HA API.
3. Publish `office_panel.button_pressed` event.
4. Wait up to 3 seconds for HA to acknowledge (success or conflict).
5. On conflict, fire conflict buzzer pattern.
6. Fetch latest mode and re-render.
7. Return to deep sleep.

---

## 10. Battery and refresh strategy

### Wake schedule

| Period | Cadence |
|---|---|
| Weekday active hours (07:00–22:00 Mon–Fri) | Every 15 minutes, aligned to `:00`, `:15`, `:30`, `:45` |
| Weekend active hours (07:00–22:00 Sat–Sun) | Every 60 minutes, aligned to `:00` |
| Overnight (22:00–07:00 any day) | Every 2 hours |
| Any time: button press | Immediate wake |

15-minute alignment during active hours ensures booking start/end transitions (which always occur on 15-minute boundaries) are reflected within one wake cycle worst case.

### No-op refresh optimization

A refresh cycle completes without calling the display driver if all subscribed state values match the values used in the last render. This is the single most important battery optimization and should be implemented in Phase 1.

### Expected battery life

Rough estimate based on the above schedule and assuming ~10 display refreshes per weekday, ~2 per weekend day, and ~1 per overnight period:

- ~60 refreshes per week active
- Spectra 6 refresh draws significant current for ~15 seconds
- Realistic estimate: **3–6 weeks between charges.** Actual figure to be measured in Phase 1 and revised.

### Low-battery behavior

- 20% threshold: HA notification to both household members.
- 10% threshold: panel renders a small battery icon in the corner of the display on next refresh (all modes).
- 5% threshold: panel enters emergency mode — renders a full-screen "LOW BATTERY — CHARGE" message and extends wake cadence to hourly to reserve remaining charge for the display of this warning.

---

## 11. Repository layout

```
office-panel/
├── README.md                    # Project overview, setup instructions
├── SPEC.md                      # This document
├── CLAUDE.md                    # Claude Code operating instructions
├── LICENSE
├── esphome/
│   ├── office-panel.yaml        # Main firmware config
│   ├── secrets.yaml.example     # Template; real secrets gitignored
│   └── fonts/                   # Font files used by display
├── homeassistant/
│   ├── packages/
│   │   └── office-panel.yaml    # HA package: helpers, sensors, scripts, automations
│   ├── dashboards/
│   │   └── office.yaml          # Lovelace view definition
│   └── intents/
│       └── office_intents.yaml  # Voice intent scripts
└── docs/
    ├── install.md               # Physical install, wiring (N/A), mounting notes
    ├── flashing.md              # Initial flash procedure
    └── development.md           # How to iterate on firmware and HA side
```

---

## 12. Phased build plan

### Phase 1: Basic room status (target: 1 weekend)

- HA Local Calendar setup with `calendar.office` entity
- Template sensors for mode, current/next booking metadata
- ESPHome firmware with BOOKED / AVAILABLE / STARTING_SOON / WEEKEND / OVERNIGHT modes
- Dashboard card for manual booking via HA UI only
- Wake schedule implemented
- No-op refresh optimization

**Definition of done:** Booking created from HA dashboard is reflected on panel within one wake cycle. All five modes render correctly.

### Phase 2: Physical buttons (target: 1 weekend)

- Button wake sources and event publishing
- Buzzer component with confirm/conflict/error tones
- HA scripts and automations for button actions
- Conflict detection and CONFLICT mode rendering
- All three buttons (plus green-button long-press) implemented simultaneously (they share ~90% of the code path; iterating one-at-a-time adds integration overhead without benefit)

**Definition of done:** Each of the three buttons performs its designated short-press action, and the green long-press ends the active booking. Conflicts produce double-beep and CONFLICT display. All ESPHome-published events reach HA reliably within 3 seconds.

### Phase 3: Voice intents (target: 1 evening)

- Five intent scripts (`BookOffice`, `ExtendOfficeBooking`, `ReleaseOffice`, `OfficeStatus`, `OfficeNextFree`)
- Per-device owner mapping for Voice PE devices
- Voice responses tuned for natural speech

**Definition of done:** Each intent works from the office Voice PE device with the documented phrasings, returns correct speech responses, and creates valid calendar events.

### Phase 4: Polish (ongoing)

- Typography and color tuning after living with the panel for a week
- Battery life measurement; adjust wake cadence if needed
- Low-battery notification testing
- Optional: booking-source icons
- Optional: refinement of STARTING_SOON threshold (currently 15 min — may want shorter)
- Extract reusable E1002 hardware config and submit PR to `heytcass/esphome-device-library`

---

## 13. Open questions (resolve during implementation)

The following items are intentionally deferred to hardware bring-up. None of them block initial implementation.

1. **Font selection.** Specific font files (permissively-licensed, supports large-display legibility at the target sizes in §9). Evaluate during Phase 1 rendering by trying 2–3 candidates on the physical display before committing.
2. **Exact RGB hex values for Spectra 6 palette targets.** The six native pigments do not render as pure hex values; some tuning may be needed so that code-level color constants match the panel's actual pigment output under typical hallway lighting.
3. **Green long-press duration.** The spec sets the short/long threshold at 1 second. Tune during Phase 2 hardware bring-up if users accidentally commit to the wrong action — may need to push to 1.2–1.5 s, or shorten if it feels sluggish.
4. **Buzzer tone definitions.** Specific RTTTL tone strings for `beep_confirm`, `beep_conflict`, and `beep_error`. Iterate during Phase 2 until the three patterns are clearly distinguishable from across the hallway.

---

## 14. Out-of-scope future work (do not implement without updating this spec)

These ideas have been discussed and explicitly deferred:

- Secondary office panel (different hardware install, same pattern)
- Auto-book triggered by work laptop mic/camera state
- Booking analytics (who uses the room when, weekly patterns)
- Integration of household Google Calendars as read-only busy-time overlays
- "Quiet hours" indicator tied to Kieran or Maeve's sleep schedule
- Integration with UACC doorbell events
- Public API for booking from third-party tools

Any of these becoming a priority should trigger an update to this spec before implementation begins.
