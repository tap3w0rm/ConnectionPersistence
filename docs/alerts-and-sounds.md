# Alerts And Sounds

## Visual Alerts

Visual alerts are custom app-owned popups, not Windows toast notifications.

Reason:

- Windows toast location is OS-controlled.
- The desired position is upper-right.
- Upper-right is less likely to conflict with common game HUD areas.

## Popup Behavior

- primary monitor only,
- upper-right work area,
- slide downward,
- stack downward,
- auto-dismiss,
- themed light/dark/system,
- never steal focus.

## Fullscreen Safety

Default:

```text
desktop or safe borderless fullscreen:
  show popup

risky fullscreen/game state:
  suppress popup
  play sound if enabled
  log event
  queue summary for later
```

Implementation must avoid:

- focus stealing,
- game minimization,
- resolution switching,
- display flicker.

## Popup Implementation Decision

Try egui viewports first.

If egui cannot guarantee non-activating behavior, use raw Win32 alert windows with no activation.

## Sound Events

Support separate sounds for:

- connection up,
- connection down,
- VPN went down,
- VPN came back up.

Default:

```text
standard Windows alert sound
```

Configuration supports selecting any `.wav` file on the system.

If custom sound playback fails:

- log the failure,
- fall back to the default Windows alert sound.
