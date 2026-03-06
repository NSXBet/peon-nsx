# Peon Avatar Desktop App — Design

**Date:** 2026-02-18
**Status:** Approved

## Summary

An Electron desktop app that renders the Peon character as an always-on-top corner widget using Three.js. The avatar reacts visually to peon-ping events by polling `.state.json` and playing corresponding sprite animations + WebGL shader effects.

## Architecture

Two-process Electron app:

```
Main process (Node.js)
  ├── Creates transparent always-on-top BrowserWindow
  ├── File watcher: polls ~/.claude/hooks/peon-ping/.state.json every 200ms
  └── Sends events to renderer via ipcMain → ipcRenderer

Renderer process (Chromium)
  ├── Three.js WebGL scene on transparent canvas
  ├── Sprite atlas animation state machine
  └── Shader effects layer (flash, glow, particles)
```

## Window

- Frameless, transparent, always-on-top BrowserWindow
- ~200x200px, positioned bottom-right corner
- `setIgnoreMouseEvents(true)` — clicks pass through to apps below
- No taskbar entry, no dock icon

## Event Detection

Poll `~/.claude/hooks/peon-ping/.state.json` every 200ms. Compare `last_event` timestamp on each poll. If changed, fire the corresponding animation. No changes to `peon.sh` required.

| CESP Event | Animation | Effect |
|---|---|---|
| `task.complete` | celebrate (jump) | Gold screen flash + particles |
| `input.required` | alarmed (wave arms) | Red pulse border |
| `task.error` | facepalm | Red screen flash |
| `session.start` | wave | Subtle glow |
| `user.spam` | annoyed (arms crossed) | Screen shake |
| idle | idle (breathing) | None |

## Three.js Scene

- `OrthographicCamera` — flat 2D view, no perspective distortion
- `PlaneGeometry` with sprite atlas texture, UV-animated per frame
- Separate `PlaneGeometry` for particle effects (sparks, confetti)
- Post-processing: `UnrealBloomPass` for glow effects
- Transparent renderer (`alpha: true`) so desktop shows through

## Sprite Art

Sprite atlas PNG: 6 animation rows x ~6 frames each (36 frames total).

Options:
1. AI-generated pixel art (recommended for v1)
2. Placeholder colored rectangles to build app first, swap art later
3. Commission pixel artist

## Repo Structure

New repo: `peonping-repos/peon-ping-avatar/`

```
peon-ping-avatar/
  main.js           ← Electron main process (window creation, state polling, IPC)
  preload.js        ← IPC bridge (contextBridge exposes events to renderer)
  renderer/
    index.html
    app.js          ← Three.js scene, animation state machine
    shaders/        ← GLSL vertex/fragment shader files
    assets/
      peon-atlas.png
  package.json
```

## Future Characters

The animation state machine and event mapping are character-agnostic. Swapping in a new sprite atlas = new character. Peon is character #1.
