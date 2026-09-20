**English** | [한국어](../ko/modules/06-movement-validator.md)

# Module 06 — Movement Validator

## Purpose
Server-side detection of speed hacks / teleport hacks / fly hacks. Consumes POSITION_TICK 32 Hz data.

## Interface

```cpp
namespace Security {

struct MovementLimits {
    float maxDashSpeedUPS    = 1200.f;  // legitimate max ~1500 (dash + butterfly spike)
    float speedTolerance     = 1.5f;    // 1200 × 1.5 = 1800 u/s cut
    float teleportDistPerSec = 5000.f;  // over 5000 u/s = teleport
    int   flyhackAirtimeMs   = 5000;    // 5 s or more of airtime
};

class MMovementValidator {
public:
    static MMovementValidator& Instance();

    void SetLimits(const MovementLimits& lim);

    // called on every POSITION_TICK
    void Validate(uint32_t uidHigh, uint32_t uidLow,
                  float x, float y, float z, long long tMs);

    void OnPlayerLeave(uint32_t uidHigh, uint32_t uidLow);

private:
    MMovementValidator();
    // per-player state map (uid → {lastSample, violationStreak, severity})
};

}
```

## Validation Items

### Speed
```
dt = tMs - lastTMs
dist = sqrt(dx² + dy² + dz²)
speed = dist / (dt / 1000)

if speed > limits.maxDashSpeedUPS × limits.speedTolerance:
    → SpeedHack signal, consecutive violations accumulated
```

### Teleport
```
if speed > limits.teleportDistPerSec:
    if !checkAlive (respawn/warp exemption):
        → TeleportSuspect signal
```

### FlyHack (heuristic)
```
if z stays above base for a sustained period (airtime > flyhackAirtimeMs):
    → FlyHack signal, severity capped at Medium (BSP not integrated → jump-map false positives possible)
```

## Severity Escalation

| Consecutive violations | severity |
|----------|---------|
| 1 | Low |
| 3 | Medium |
| 5 | High |
| 10 | Critical |

Reset condition: N minutes of normal behavior or match end.

## Integration Points
- `CSCommon/Security/MMovementValidator.{h,cpp}` new
- Called from the POSITION_TICK handler (1 line)
- Call `OnPlayerLeave` from `ObjectRemove`

## Configuration

| Environment variable | Default |
|---------|------|
| `MOVEMENT_MAX_SPEED_UPS` | 1200 |
| `MOVEMENT_SPEED_TOLERANCE` | 1.5 |
| `MOVEMENT_TELEPORT_DIST_PER_SEC` | 5000 |
| `MOVEMENT_FLYHACK_AIRTIME_MS` | 5000 |

## Failure Modes
| Condition | Result |
|------|------|
| Legitimate dash (~1500 u/s) | Below 1800 (1200×1.5) → passes |
| Position jump right after respawn | `CheckAlive` exemption |
| dt = 0 right before respawn | dt > 0 guard |
| Jump maps (successive high platforms) | FlyHack heuristic capped at Medium — operator review |

## Test Vectors

```
Normal dash:
  Record(0, 0, 0, 1000)
  Record(150, 0, 0, 1100)  // 1500 u/s
  → passes (1500 < 1800)

Speed hack:
  Record(0, 0, 0, 1000)
  Record(300, 0, 0, 1100)  // 3000 u/s
  → SpeedHack Low (1st)
  Record(600, 0, 0, 1200)  // 3000 u/s, 2nd accumulated (Low)
  Record(900, 0, 0, 1300)  // 3000 u/s, 3rd → escalates to Medium

Teleport:
  Record(0, 0, 0, 1000)
  Record(10000, 0, 0, 1100)  // 100,000 u/s
  → TeleportSuspect (separate severity policy)
```

## Limitations
- BSP terrain sampling not integrated → precise noclip not detected
- z-axis judgment is heuristic — jump-map false positives possible
- Exemption for legitimate teleports (warp, respawn) depends on `CheckAlive` — additional validation needed if exemption bypass is attempted

## Cross-References
- Infrastructure: [`05-position-history.md`](./05-position-history.md)
- Signal handling: [`08-signal-collector.md`](./08-signal-collector.md)
- Catalog: [`../04-anticheat-catalog.md`](../04-anticheat-catalog.md) §2.1
- Integration: [`../08-integration-guide.md`](../08-integration-guide.md) §2.6
