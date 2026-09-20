**English** | [한국어](../ko/modules/10-position-lie-novel.md)

# Module 10 — Position Lie Detection (Novel Axis)

## Purpose
**Detection axis #2 unique to this package**. Verifies that the `srcPos` in HIT_TICK (the attacker-reported firing position) matches the most recent sample in the server's POSITION_TICK ring buffer. **POSITION × HIT cross-check**.

## Why Novel
- Typical hit validation focuses on `ImpossibleHit` (range) and `AutoAim` (hit rate)
- The path where the attacker tampers with their own position to bypass range/line-of-sight is a blind spot
- This check verifies the **temporal consistency** of two channels (POSITION_TICK at 32 Hz + HIT_TICK on hit)

## Approach

```
On HIT_TICK receipt:
    1. Look up the attacker's server-authoritative position
       atkSrvXYZ, atkSrvTMs = pAttacker->GetPositionHistory().GetLatest()
       if lookup fails (history empty): return  // avoid false positive right after connect

    2. Compare against the attacker-reported srcPos
       dist = sqrt(|srcPos - atkSrvXYZ|²)

    3. Threshold per weapon class
       Melee:    400 u
       Shotgun:  600 u
       Revolver: 600 u
       SMG:      800 u
       Rifle:    800 u
       Rocket:   1000 u
       (in world coordinates ~100u = 1m, actual game units)

    4. If threshold exceeded, PacketManipulation signal
       severity:
         dist > threshold × 2  →  High
         threshold < dist ≤ threshold × 2  →  Medium
       (single-shot detection, no escalation — POSITION_TICK at 32 Hz is dense enough)
```

## Why the Thresholds Differ

- **Melee (400)**: for melee weapons srcPos must be inside the character box. Small tolerance (latency compensation)
- **Shotgun/Revolver (600)**: mid-range, character-to-gun offset + latency tolerance
- **SMG/Rifle (800)**: stance changes + compensation margin
- **Rocket (1000)**: projectile weapon, large timing gap (position at fire time vs. hit time)

## Integration Points
- Inline in the `case MC_MATCH_HIT_TICK` handler in `MMatchServer_OnCommand.cpp`
- No new class — inline check *before* the `MCombatValidator::ValidateHit` call
- Depends on: [`05-position-history.md`](./05-position-history.md), [`07-combat-validator.md`](./07-combat-validator.md), [`08-signal-collector.md`](./08-signal-collector.md)

## Code Snippet (inline)

```cpp
case MC_MATCH_HIT_TICK:
{
    MUID atkUID;
    BYTE weaponType;
    float dmg;
    float srcPos[3];
    pCommand->GetParameter(&atkUID, 0, MPT_UID);
    pCommand->GetParameter(&weaponType, 1, MPT_UCHAR);
    pCommand->GetParameter(&dmg, 2, MPT_FLOAT);
    pCommand->GetParameter(srcPos, 3, MPT_FLOAT_ARRAY_3);

    auto* pAttacker = GetObject(atkUID);
    auto* pVictim = pObj;
    if (!pAttacker || !pVictim->CheckAlive()) break;

    auto wcls = Security::ClassifyMMatchWeapon(weaponType);

    // [INLINE] PositionLie cross-check
    float atkSrvXYZ[3];
    long long atkSrvTMs;
    if (pAttacker->GetPositionHistory().GetLatest(atkSrvXYZ, &atkSrvTMs)) {
        float dx = srcPos[0] - atkSrvXYZ[0];
        float dy = srcPos[1] - atkSrvXYZ[1];
        float dz = srcPos[2] - atkSrvXYZ[2];
        float dist = sqrtf(dx*dx + dy*dy + dz*dz);

        float threshold;
        switch (wcls) {
            case Security::WeaponClass::Melee:    threshold = 400.f; break;
            case Security::WeaponClass::Shotgun:  threshold = 600.f; break;
            case Security::WeaponClass::Revolver: threshold = 600.f; break;
            case Security::WeaponClass::SMG:      threshold = 800.f; break;
            case Security::WeaponClass::Rifle:    threshold = 800.f; break;
            case Security::WeaponClass::Rocket:   threshold = 1000.f; break;
            default:                              threshold = 800.f; break;
        }

        if (dist > threshold) {
            Security::CheatSignal sig;
            sig.type = Security::CheatSignalType::PacketManipulation;
            sig.severity = (dist > threshold * 2.f)
                ? Security::CheatSeverity::High
                : Security::CheatSeverity::Medium;
            sig.value = dist;
            sig.threshold = threshold;
            snprintf(sig.detail, sizeof(sig.detail),
                     "PositionLie wcls=%d dist=%.1f thr=%.1f",
                     (int)wcls, dist, threshold);
            Security::MSignalCollector::Instance().Push(
                pAttacker->GetUID().High, pAttacker->GetUID().Low, sig);
        }
    }
    // GetLatest failed → right after connect, before POSITION_TICK accumulates; skip (avoid false positive)

    // ValidateHit (range / hit-rate) — Module 07
    float vicXYZ[3]; long long vicTMs;
    if (pVictim->GetPositionHistory().GetLatest(vicXYZ, &vicTMs)) {
        Security::MCombatValidator::Instance().ValidateHit(
            pAttacker->GetUID().High, pAttacker->GetUID().Low,
            wcls, srcPos, vicXYZ, dmg);
    }
    break;
}
```

## Configuration

| Environment variable | Default |
|---------|------|
| `POSITIONLIE_MELEE_DIST` | 400 |
| `POSITIONLIE_SHOTGUN_DIST` | 600 |
| `POSITIONLIE_REVOLVER_DIST` | 600 |
| `POSITIONLIE_SMG_DIST` | 800 |
| `POSITIONLIE_RIFLE_DIST` | 800 |
| `POSITIONLIE_ROCKET_DIST` | 1000 |

## Failure Modes
| Condition | Result |
|------|------|
| Right after connect (no POSITION accumulated) | `GetLatest` fails → skipped |
| Normal latency / stance changes | Passes within threshold |
| Legitimate warp / jump (large position change) | POSITION_TICK is updated too → consistency preserved |
| Client clock vs. server clock gap (large RTT) | Threshold has ample margin (Melee 400u = 4m) |

## Test Vectors

```
Normal:
  attacker server pos = (1000, 0, 0), time t=2000
  HIT_TICK srcPos = (1010, 5, 0), wcls=SMG
  dist = sqrt(100+25+0) ≈ 11
  → 11 < 800, pass

PositionLie (Medium):
  attacker server pos = (1000, 0, 0)
  HIT_TICK srcPos = (1500, 0, 0), wcls=Melee
  dist = 500
  → 500 > 400 (Melee threshold), below 800 (=400×2) → Medium

PositionLie (High):
  attacker server pos = (1000, 0, 0)
  HIT_TICK srcPos = (3000, 0, 0), wcls=Melee
  dist = 2000
  → 2000 > 400×2 → High
```

## Limitations
- Per-weapon thresholds were measured in a self-built environment from the public (GitHub) source tree — re-tuning needed on live traffic (different latency distribution)
- If POSITION_TICK at 32 Hz drops samples (network jitter), the time gap to the last sample grows → legitimate movement can also produce false positives. A time-gap correction (skip when `atkSrvTMs - hitTMs > 100ms`) can be added
- Accuracy can be improved with time-aligned interpolation (`QueryAt(hitTMs)`) — future work

## Strengths
- Verifies the temporal consistency of the two channels POSITION_TICK and HIT_TICK — cannot be bypassed by tampering with a single channel
- To forge both channels, the client must maintain a consistent 32 Hz position time series and synthesize a matching position at every fire event → raises the cost of writing a hack

## Cross-References
- Infrastructure: [`05-position-history.md`](./05-position-history.md)
- HIT_TICK: [`07-combat-validator.md`](./07-combat-validator.md)
- Signals: [`08-signal-collector.md`](./08-signal-collector.md)
- Integration: [`../08-integration-guide.md`](../08-integration-guide.md) §2.8
