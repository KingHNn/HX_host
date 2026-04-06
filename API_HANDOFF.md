# API Handoff (2026-02-20)

This handoff reflects the current stable state after button-read, long-press, and runtime wiring fixes.

## 1) Workspace and build

- Project root: `c:\Users\X\AndroidStudioProjects\HRHostClone`
- Main implementation file: `app/src/main/java/com/example/hrhostclone/MainActivity.kt`
- USB serial dependency:
  - `app/build.gradle.kts` uses `com.github.mik3y:usb-serial-for-android:3.9.0`

Latest local build checks:

- `.\gradlew.bat :app:compileDebugKotlin -q` -> pass
- `.\gradlew.bat :app:assembleDebug -q` -> pass

## 2) Current user-verified runtime status

Verified in latest rounds:

- Device connection is stable.
- Draw actions (`画正方形` / `画圆形`) are working.
- Mouse buttons are detected and responsive.
- Long-press now works without early auto-off.
- Release now works (no permanent stuck-on state).

Current status from user: "很不错 目前没啥问题".

## 3) Major changes completed

### 3.1 MAKCU button parser hardening (serial + session)

Two parser stacks were kept in sync:

- `MakcuSerialEngine.parseButtonsMask(...)`
- `MakcuUsbSession.parseButtonsMask(...)`

Both now parse in layered order:

1. V2 frame parse first.
2. Text `buttons(...)` / `buttons:...`.
3. Per-button event text (`left/right/middle/x1/x2`).
4. Strict binary stream.
5. HID-like report fallback.

Critical fix for release handling:

- Previously many branches accepted only `1..0x1F`, which caused "press works but release never arrives".
- Now `0` mask (release) is accepted in V2/binary/HID branches with guard constraints to reduce noise.

Relevant anchors:

- `MakcuSerialEngine.parseBinaryButtonsFrameStrict`
- `MakcuSerialEngine.parseHidStyleButtonsReport`
- `MakcuSerialEngine.parseV2ButtonsFrame`
- `MakcuUsbSession` counterparts of the same parse branches

### 3.2 Long-press and polling behavior

`MakcuLinkRuntime` read loops were adjusted:

- Removed aggressive "idle few reads => force clear" behavior that broke long-press.
- While holding, use more frequent snapshot polling to keep state fresh.
- Re-prime button stream periodically on idle to recover from silent stream conditions.
- Release now depends on actual parsed zero-mask instead of premature timeout clearing.

Relevant anchors:

- `MakcuLinkRuntime.connect(...)` serial branch read loop
- `MakcuLinkRuntime.connect(...)` USB session branch read loop

### 3.3 Hotkey / control-parameter / runtime integration

Runtime wiring is now end-to-end:

- Hotkey trigger key is required for auto-aim activation (trigger-mask based).
- Category filtering and class-priority selection are applied in target pick logic.
- PD panel values are no longer local-only UI state.
- `AimPdConfig` now receives `x/y kp`, `x/y kd`, `x/y smooth`, `x/y deadzone`, `x/y maxOut`.
- Smoothing + deadzone are applied in move output path (`GameSurfaceView` aim loop).
- Input tabs include `热键3` page.

Relevant anchors:

- `RuntimeBridge.pickAimbotConfig(...)`
- `GameSurfaceView` aim selection and output section
- `AimPdConfig`
- `MainApp` + `InputControlScreen` + `ControlParamTab` parameter flow

### 3.4 Config persistence expansion

Import/export (`buildConfigJson` / `applyConfig`) now includes PD fields:

- `xKp`, `xKd`, `xSmooth`, `xDeadzone`, `xMaxOut`
- `yKp`, `yKd`, `ySmooth`, `yDeadzone`, `yMaxOut`

## 4) Known gaps / backlog (next stage)

No blocker right now, but these are still pending refinement:

1. Auto-fire fields are UI-complete but not fully executed as an end-to-end firing pipeline.
2. PD tuning UX can be improved with live telemetry (error, output, clamping indicator).
3. Optional safety fallback for rare stuck-hold scenarios can be added as a configurable policy.
4. Additional cleanup: parser logic is duplicated across serial/session paths and can be unified.

## 5) Suggested next tasks

1. Add control-parameter live diagnostics overlay:
   - current `errX/errY`, `pdOutX/pdOutY`, `moveX/moveY`, clamp flags.
2. Wire full auto-fire execution path:
   - trigger condition, initial delay, click count range, interval range, burst interval.
3. Add structured debug mode toggle:
   - show parser source and last RX snippet only when debug is enabled.
4. Refactor duplicated button parser code into shared utility functions.

## 6) Suggested continuation prompt

`Read API_HANDOFF.md first. Continue from app/src/main/java/com/example/hrhostclone/MainActivity.kt. Keep current button detection behavior unchanged, then focus on remaining control-parameter tuning UX and auto-fire end-to-end wiring.`
