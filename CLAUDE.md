# CLAUDE.md — Operating Instructions for Claude Code

This file tells Claude Code how to work on the `office_panel` project. Read this first before making any changes.

---

## Primary reference

**`SPEC.md` is the source of truth.** If `SPEC.md` and this file disagree, `SPEC.md` wins. If a user request conflicts with `SPEC.md`, stop and ask whether to update the spec before implementing.

Do not expand scope beyond what `SPEC.md` defines. The "Non-goals" section (§2) and "Out-of-scope future work" section (§14) are both binding. If a requested change falls into either, ask first.

---

## Project context

This is an ESPHome + Home Assistant project. The hardware is a Seeed reTerminal E1002 (color e-paper, battery-powered). The application is a room-booking status panel mounted in a hallway. Read `SPEC.md` §1 for the full summary before making design decisions.

The user (Tom) runs:

- Home Assistant as the backend
- The Home Assistant ESPHome app (formerly "ESPHome add-on") for firmware flashing and OTA
- Ubuntu 26.04 Beta on his development workstation
- UniFi networking with VLANs (IoT VLAN likely destination for this device — confirm before network-related changes)

---

## Build and deploy workflow

- **Firmware lives in `esphome/office_panel.yaml`.** Tom flashes and updates via the HA ESPHome app UI. Do not assume a local `esphome` CLI is installed on the workstation unless asked.
- **Home Assistant configuration lives in `homeassistant/packages/office_panel.yaml`.** This is a standard HA package include, loaded by adding `packages: !include_dir_named packages` to the user's HA `configuration.yaml` (if not already present).
- **Secrets are not in the repo.** `esphome/secrets.yaml.example` is a template. The real `secrets.yaml` is gitignored and lives on Tom's machine.
- **Voice intents live in `homeassistant/intents/office_intents.yaml`**, loaded via the HA intent_script integration.

## Working across repos

This project has a hard dependency on Tom's `esphome-device-library` (local path: `/home/tom/Projects/esphome-device-library`). The hardware-level E1002 package lives there; application logic lives here.

**Preferred session setup.** Claude Code sessions should be launched from `/home/tom/Projects/` so both repos sit in a single context tree. Cross-repo imports and reasoning work naturally. Alternatively, `claude --add-dir /home/tom/Projects/esphome-device-library` from within `office_panel` achieves the same access.

**Audit before authoring.** Before writing any E1002 hardware package in the device library, you must:

1. Inspect the current state of `esphome-device-library` for any existing E1002 work — branches, open PRs, draft files, related issues.
2. Survey external references: Clelia Rella's "Style over Substance" build log, tutoduino's reTerminal guide, Home Assistant community forum thread 970545, ESPHome issue #12709.
3. Draft a proposed package structure and present it to Tom for review.
4. Only after approval, write the package.

The audit is a good candidate for a research subagent (Task tool) — dispatch it in parallel while continuing other Phase 1 work. Do not let the main context author hardware abstractions cold without the audit complete.

**Commits cross both repos.** Changes to the device library and the app repo may need to land together for a working build. Coordinate commits and note cross-repo dependencies in commit messages.

---

## Code placement conventions

Tom prefers explicit, precise instructions on where new code goes. When adding or modifying code:

- State exactly which file is being modified at the top of the change.
- If creating a new file, say so explicitly and give the full path.
- Show bracket alignment and indentation clearly.
- When inserting into an existing structure (e.g., a YAML block inside another block), show enough surrounding context that the insertion point is unambiguous.
- Do not reformat code unrelated to the change.

---

## Naming conventions

Follow these exactly. The spec depends on them.

- Calendar entity: `calendar.office`
- Panel entities are prefixed `office_panel_` (e.g., `sensor.office_panel_battery`)
- Office state entities are prefixed `office_` (e.g., `sensor.office_mode`)
- Scripts are prefixed `script.` with a verb-first name: `script.book_office`, `script.end_office_booking`
- Automations use snake_case IDs matching their purpose: `office_panel_button_1_book_30`
- ESPHome events use dotted namespaces: `office_panel.button_pressed`, `office_panel.beep_conflict`

If a name is not defined here or in `SPEC.md` §8, default to the closest existing pattern and ask if unsure.

---

## Testing approach

- **HA side:** Use HA Developer Tools to manually fire events, set entity states, and invoke services. Confirm template sensors evaluate correctly by viewing them in the Template editor.
- **ESPHome side:** Tom flashes to the device and views logs via the HA ESPHome app. There is no simulator; iteration is physical.
- **End-to-end testing:** Create test calendar events via `script.book_office` with short durations. Confirm panel renders correctly, then clean up via `script.end_office_booking` or by deleting events in the calendar UI.
- **Do not write automated tests** unless explicitly asked. This is a home-scale project; unit tests add maintenance burden without proportionate value.

---

## Working in phases

`SPEC.md` §12 defines four phases. Do not work ahead. If Phase 1 is in progress, do not add Phase 3 voice intents. If you believe a phase boundary should move, raise it as a question, do not act unilaterally.

Each phase has a "Definition of done" — use it to evaluate whether you are ready to declare the phase complete.

---

## What not to do

- **Do not integrate with Google Calendar.** Rejected in the design. If a workflow would be easier with Google Calendar, propose the design change and wait for approval before implementing.
- **Do not add Teams/Zoom/Meet detection** for auto-booking. Explicitly rejected.
- **Do not add album art, Sendspin integration, weather widgets, F1 widgets, or any non-booking-related content** to the panel display. The panel does one job.
- **Do not expose the system outside the LAN.** No reverse proxies, no Nabu Casa configuration, no public webhooks.
- **Do not attempt partial-refresh optimizations** on the Spectra 6 panel. The hardware does not support it. If a "partial refresh" seems possible from an ESPHome doc, confirm for the `Seeed-reTerminal-E1002` model specifically before acting.
- **Do not assume grayscale or 24-bit color works.** Spectra 6 is a 6-color palette. Any image asset must be pre-quantized; any color value in code must be one of the six native colors.
- **Do not invoke NixOS-specific tooling.** Tom has moved off NixOS to Ubuntu. No `nixos-rebuild`, no flake references, no `home-manager`.
- **Do not write to any second-brain or note-capture tool automatically.** Tom previously ran an "Open Brain" MCP server with a dual-write convention; it is not currently active. If he asks to capture notes, just ask where he wants them.

---

## Communication style

Tom's preferences (from his user settings; apply throughout):

- Be opinionated. Lead with recommendations, not neutral pro/con lists.
- Stay conversational. No corporate filler. No "Great question!" or "I'd be happy to help."
- Be direct. No preamble that restates the question.
- On technical topics, start at intermediate-to-advanced. He knows ESPHome, Home Assistant, YAML, networking, and Linux. Do not over-explain basics.
- When iterating on config or code, match momentum: apply the change, show the result, save commentary for when something is ambiguous or about to go wrong.
- Follow explicit formatting instructions exactly on the first try.
- When he shares constraints mid-project, internalize them as guardrails. He should not have to repeat them.

---

## Committing and branching

- Work on a feature branch per phase: `phase-1-basic-status`, `phase-2-buttons`, etc.
- Commit messages should be conventional-style: `feat:`, `fix:`, `docs:`, `chore:`.
- Do not push to `main` without explicit approval.
- Keep commits small. A "Phase 2 complete" PR is harder to review than five focused commits.

---

## When in doubt

Ask Tom. He prefers a question over an assumption that has to be unwound later.
