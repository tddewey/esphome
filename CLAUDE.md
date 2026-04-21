# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a collection of ESPHome firmware configs for ESP32/ESP8266 devices, managed locally and deployed via CLI. The goal is maintainable, version-controlled YAML that accommodates hardware variation without duplicating logic.

Configs are compiled and flashed from the local machine using the `esphome` CLI. The Home Assistant ESPHome add-on is used only for device monitoring and OTA triggering — not for editing or first-flash.

## Directory Structure

```
esphome/
├── <device>.yaml     # One file per physical device — entry point for compilation
├── packages/         # Shared YAML fragments included by device files
├── secrets.yaml      # Credentials — never committed
└── .esphome/         # Build cache — do not touch
```

Device files live at the root so `secrets.yaml` is found correctly (ESPHome resolves it relative to the config file being compiled).

## Commands

```bash
# Validate config (no compile)
esphome config <device>.yaml

# Compile without flashing
esphome compile <device>.yaml

# First flash over USB
esphome run <device>.yaml

# OTA update
esphome run <device>.yaml --device <ip>

# View logs
esphome logs <device>.yaml --device <ip>
```

## Authoring Rules

**Device files (root)** should be thin — logic lives in packages. Each file contains `substitutions`, `esphome`, a platform block, `packages` includes, and any device-specific overrides. All hardware-specific values (GPIO pins, board type, update intervals) go in `substitutions` at the top.

**Package files (`packages/`)** contain reusable logic grouped by functional role, not component type. Never hardcode pin numbers — reference `${substitution_name}` instead.

**secrets.yaml** — never read, modify, suggest changes to, or include contents in any output. Reference credentials in configs as `!secret key_name` only.

## Hardware Variation Pattern

```yaml
# my_device.yaml
substitutions:
  name: my-device
  friendly_name: "My Device"
  pir_pin: GPIO23
  led_pin: GPIO18
  led_count: "30"

packages:
  board: !include packages/board_esp32_wroom.yaml
  base: !include packages/base.yaml
  diagnostics: !include packages/diagnostics.yaml
  presence: !include packages/presence.yaml
```

To adapt a device to a different board or pin layout, only the `substitutions` block changes.

## Package Granularity

Extract to a package only when you'd otherwise copy a meaningful block to a second device.

**Good packages** — functional clusters with real internal logic:
- `base.yaml` — wifi, api, ota, logger (every device)
- `diagnostics.yaml` — uptime, wifi signal, IP, restart button
- `board_esp32_wroom.yaml`, `board_d1_mini.yaml` — platform block per board variant
- `environmental.yaml` — temperature + humidity sensors and automations together
- `addressable_leds.yaml` — neopixelbus config + light effects

**Bad packages** — single sensor wrappers, content that's mostly substitution placeholders, anything under ~20 lines with no internal logic.

## What to Avoid

- Hardcoding GPIO pins anywhere outside `substitutions`
- Editing configs via the HA add-on web UI
- Creating packages for single sensors with no internal logic
- Writing to `globals` with `restore_value: yes` on high-frequency update loops (flash wear)
- GPIO 6–11 on ESP32 (reserved for flash chip)
- GPIO 34–39 on ESP32 for output (input-only pins)
