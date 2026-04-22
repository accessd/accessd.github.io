---
layout: post
title: "A Flipper Zero That Buzzes When My AI Agents Need Me"
description: "flipper-agent-indicator — a physical notification surface for Claude / Codex / OpenCode, with an ACK button that clears the tmux pane"
category: blog
blog: true
tags: ["tmux", "ai", "claude", "codex", "agents", "flipper", "hardware"]
lang: en
---

A few days ago I wrote about [tmux-agent-indicator and agent-indicator]({% post_url 2026-04-19-tmux-agent-indicator-en %}) — tmux pane borders, terminal title tints, sound, desktop popups, push to phone. That covered the screen, the speakers, and the pocket.

One gap was still nagging me: the desk itself.

When I step away from the laptop for two minutes, the laptop is closed. No pane borders. No desktop popups. Sound is off because I was on a call. Push lands on the phone, but the phone is in the other room. I come back twenty minutes later and three agents have been sitting on `needs-input` the whole time.

I wanted a dumb little brick on the desk that lights up and vibrates. Preferably one that already lived there.

## Why the Flipper Zero

The Flipper Zero was already on my desk. It has a screen, a D-pad, Bluetooth LE, a vibra motor, and an RGB LED. It also has a custom firmware scene (Unleashed, Momentum, RogueMaster) with a well-documented `.fap` format — you can drop a binary onto the SD card and it shows up in the menu.

Which means it is basically a cheap, programmable, battery-powered desk buzzer. That I already own. And the hardware-hack toy-to-tool pipeline is exactly the kind of thing that is fun to build on a weekend.

So: [accessd/flipper-agent-indicator](https://github.com/accessd/flipper-agent-indicator).

<img width="476" alt="flipper showing claude needs-input" src="https://github.com/user-attachments/assets/e4347404-8686-4cb8-b76c-f879e956621e" />

## Demo

<video controls preload="metadata" width="100%"><source src="/assets/blog/flipper-agent-indicator/demo.mp4" type="video/mp4"></video>

Claude hits `needs-input`. The Flipper screen shows the agent label, the LED blinks, the vibra fires. I press OK on the device. The Flipper screen clears, and in the same motion the tmux pane decoration on the laptop clears too — the ACK travels back.

## How it fits together

Same hook contract as the other two projects. The state is produced by the agent. The delivery is new.

```
agent hook
    |
    v
flipper-indicator notify  --(UDS line)-->  flipper-indicator serve
                                                     |
                                                     | BLE Serial (bleak)
                                                     v
                                           Flipper Zero custom .fap
                                                     |
                                         OK-press = ACK frame
                                                     |
                                                     v
                             tmux-agent-indicator/scripts/agent-state.sh --state off
```

Four pieces.

**1. The CLI (`flipper-indicator notify`).** This is what the agent hooks call. It opens a Unix domain socket, writes one line of JSON, and exits. Fire-and-forget. It runs in under 50ms even if the daemon is down, so it never blocks the agent. If the socket does not exist, it exits silently — a missing Flipper must never break your Claude session.

**2. The daemon (`flipper-indicator serve`).** A Python asyncio process that owns the BLE connection. It listens on the UDS, dispatches each state to the Flipper via BLE Serial (using `bleak`), and reconnects with exponential backoff when the Flipper sleeps or walks out of range. Single-event overwrite: if ten notifications queue up while the Flipper is asleep, the freshest wins. Staleness is worse than loss here.

**3. The Flipper `.fap`.** A custom Flipper application written against Unleashed firmware. It listens on the BLE Serial characteristic, parses incoming frames, renders the agent name and state, blinks the LED, and vibrates on the right states. Pressing OK sends an ACK frame back the other way. The firmware owns its own patterns — the host just says "Claude is in `needs-input`" and the device picks the LED color and vibra sequence.

**4. The tmux bridge.** The daemon watches for ACK frames. When one arrives, it calls `tmux-agent-indicator`'s `agent-state.sh --state off`, which clears the pane border, the window title, and the status bar icon. The loop closes.

This last piece is the part I like most. It turns a one-way notification into a two-way acknowledgment. The Flipper is not just a speaker — it is also an input. Press a button on the desk, clear the state on the laptop. Physical dismiss, software effect.

## Setup

```sh
./install.sh
flipper-indicator pair    # scan, pick your Flipper, save its MAC
flipper-indicator serve   # run the daemon
```

Hook registration mirrors the other two projects. Claude gets a hooks JSON. Codex gets a `notify` script. OpenCode gets a JS plugin dropped in `~/.config/opencode/plugins/`. All three coexist with tmux-agent-indicator and agent-indicator — they are independent hook consumers that happen to share the same state vocabulary.

Config lives at `~/.config/flipper-agent-indicator/config.toml` and looks like this:

```toml
flipper_mac = "AA:BB:CC:DD:EE:FF"
tmux_bridge_enabled = true
tmux_bridge_script = "~/.tmux/plugins/tmux-agent-indicator/scripts/agent-state.sh"
log_path = "~/.local/state/flipper-agent-indicator/daemon.log"
socket_path = "/tmp/flipper-indicator-501.sock"
```

On macOS, a `launchd` plist runs the daemon on login. The first BLE connect triggers the usual macOS Bluetooth permission dialog for the Python interpreter — grant it once under System Settings and you are set.

## What I learned

Three things from building this.

First, fire-and-forget over a UDS is the right primitive for agent hooks. Hooks run often and must be cheap. Anything that does DNS, loads a big TOML file, or opens a TCP socket is a liability. A local socket write is cheap enough to run on every prompt submit.

Second, BLE reconnection is the only hard problem. The Flipper sleeps. You walk out of range. Your Mac's Bluetooth stack randomly drops the peripheral. Everything else is plumbing. Put all the retry logic in one place (the daemon) so the rest of the system can assume a working connection.

Third, the ACK loop matters more than I expected. The moment the button press on the physical device cleared the pane border on the laptop, the setup stopped feeling like a gimmick and started feeling like a second screen. Dismissal is a first-class part of notification UX, and hardware makes dismissal tactile.

## Where this sits

Three projects now, stacked by delivery surface:

- [tmux-agent-indicator](https://github.com/accessd/tmux-agent-indicator) — inside tmux. Pane borders, window titles, status bar.
- [agent-indicator](https://github.com/accessd/agent-indicator) — inside the terminal itself, plus sound, desktop, and push.
- [flipper-agent-indicator](https://github.com/accessd/flipper-agent-indicator) — outside the laptop, on a physical device you can touch.

Overkill? Absolutely. But when I have five agents working in parallel and I step away to refill coffee, one of them is going to need me before I sit back down. The laptop is closed, the phone is in the other room, and the Flipper on the desk lights up.

That is the whole product.

- [flipper-agent-indicator on GitHub](https://github.com/accessd/flipper-agent-indicator)
