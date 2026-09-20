**English** | [한국어](../ko/modules/07-combat-validator.md)

# Module 07 — Combat Validator

## Purpose
Server-side detection of RapidFire / **InfiniteSlash (infinite heavy slash)** / AutoAim / ImpossibleHit. These are the high-frequency hack categories in GunZ live-server play.

**Scope limit**: this module validates **fire rate and hit fact** only. **Damage amount / HP change** is covered in [`13-damage-validator.md`](./13-damage-validator.md) (complementing the victim-authoritative blind spot).

## Interface

```cpp
namespace Security {

enum class WeaponClass {
    Unknown, Melee, Shotgun, Revolver, SMG, Rifle, Rocket, ItemKit
};

WeaponClass ClassifyMMatchWeapon(uint32_t weaponType);

struct AttackRecord {
    long long tMs;
    WeaponClass wcls;
    bool hit;
};

struct PlayerCombatState {
    std::deque<AttackRecord> recent;   // MAX 20
    uint32_t shotsFired;
    uint32_t shotsHit;
    uint32_t consecutiveViolations;
};

class MCombatValidator {
public:
    static MCombatValidator& Instance();

    // ATTACK_TICK handler
    void ValidateAttack(uint32_t uidHigh, uint32_t uidLow,
                        WeaponClass wcls, long long nowMs);

    // HIT_TICK handler
    void ValidateHit(uint32_t uidHigh, uint32_t uidLow,
                     WeaponClass wcls,
                     const float srcPos[3],
                     const float vicPos[3],
                     float dmg);

    void OnPlayerLeave(uint32_t uidHigh, uint32_t uidLow);
};

}
```

## Fire interval per weapon class

```
Melee     : 450 ± 100 ms      ← key threshold for InfiniteSlash
Shotgun   : 1100 ± 150 ms
Revolver  : 300 ± 80 ms
SMG       : 70  ± 30 ms       ← most common RapidFire target
Rifle     : 100 ± 40 ms
Rocket    : 1800 ± 200 ms
```

**InfiniteSlash (infinite heavy slash)** is GunZ's signature hack. The distributions of a legitimate slash combo (350–550 ms) and hack use (50–200 ms mashing) are clearly separated, so validation confidence is high. In this package, one Melee interval under 350 ms (450 - 100) yields Low; 3 accumulated escalate to Medium.

`ValidateAttack` compares the interval between `nowMs` and the most recent 20 records (reverse scan of `recent`) against the weapon-class threshold.

## Severity escalation

| Consecutive violations | severity |
|----------|---------|
| 1 | Low |
| 3 | Medium |
| 5 | High |
| 10 | Critical |

## ValidateHit — two checks

### ImpossibleHit
```
range = WeaponMaxRange(wcls)  // Melee 200, SMG 1500, Rifle 3000, etc.
dist = sqrt(|srcPos - vicPos|²)
if dist > range × 1.3:
    → ImpossibleHit signal (severity by distance ratio)
```

### AutoAim (statistical)
```
shotsHit += 1
if shotsFired >= 50 and (shotsHit / shotsFired) > 0.95:
    → AutoAim Medium (never a standalone verdict; pattern accumulation required)
```

## Integration Points
- New `CSCommon/Security/MCombatValidator.{h,cpp}`
- `ValidateAttack` in the ATTACK_TICK handler
- `ValidateHit` in the HIT_TICK handler
- `OnPlayerLeave` in `ObjectRemove`

## Configuration

| Env var | Default |
|---------|------|
| `COMBAT_MELEE_INTERVAL_MS` | 450 |
| `COMBAT_SHOTGUN_INTERVAL_MS` | 1100 |
| `COMBAT_REVOLVER_INTERVAL_MS` | 300 |
| `COMBAT_SMG_INTERVAL_MS` | 70 |
| `COMBAT_RIFLE_INTERVAL_MS` | 100 |
| `COMBAT_ROCKET_INTERVAL_MS` | 1800 |
| `COMBAT_AUTOAIM_MIN_SHOTS` | 50 |
| `COMBAT_AUTOAIM_HIT_RATE` | 0.95 |
| `COMBAT_RANGE_MULTIPLIER` | 1.3 |

## Failure Modes
| Condition | Result |
|------|------|
| Weapon class Unknown / ItemKit | ValidateAttack skipped |
| `recent` empty | First shot passes (history accumulation begins) |
| Legitimate slash combo (exactly 450 ms within the 450 ms window) | Passes within tolerance |
| AutoAim statistics as a standalone verdict (operations) | **Forbidden** — pattern accumulation required |

## Test Vectors

```
RapidFire SMG:
  ValidateAttack(SMG, 1000)  → OK
  ValidateAttack(SMG, 1030)  → 30ms < 70ms - 30 = 40, violation
  ValidateAttack(SMG, 1060)  → 2 accumulated violations
  ValidateAttack(SMG, 1090)  → 3 accumulated violations → Medium

InfiniteSlash:
  ValidateAttack(Melee, 1000)
  ValidateAttack(Melee, 1100)  → 100ms < 350 (450-100), violation

ImpossibleHit:
  ValidateHit(SMG, srcPos=(0,0,0), vicPos=(2500,0,0), dmg=10)
  → dist=2500 > 1500×1.3=1950, ImpossibleHit
```

## Limitations
- AutoAim statistics are **never a standalone verdict** — pro players with precise aim produce the same signal. Match-level distribution analysis required
- Per-weapon max range table not included (separate data)
- Self-damage (fall death, self-explosion) must be excluded from HIT_TICK transmission (client-side filter)

## Cross-References
- Infrastructure: [`05-position-history.md`](./05-position-history.md)
- Signals: [`08-signal-collector.md`](./08-signal-collector.md)
- Novel axes: [`09-weapon-spoof-novel.md`](./09-weapon-spoof-novel.md), [`10-position-lie-novel.md`](./10-position-lie-novel.md)
- Integration: [`../08-integration-guide.md`](../08-integration-guide.md) §2.7, §2.8
