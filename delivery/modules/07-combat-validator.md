# Module 07 — Combat Validator

## Purpose
RapidFire / **InfiniteSlash (무한 강베기)** / AutoAim / ImpossibleHit 의 서버측 탐지. GunZ 본서버 플레이에서 빈도 높은 핵 카테고리.

**범위 한계**: 본 모듈은 **발사 빈도와 명중 사실** 만 검증한다. **데미지 양 / HP 변화** 는 [`13-damage-validator.md`](./13-damage-validator.md) 에서 다룬다 (victim-authoritative 사각지대 보완).

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

    // ATTACK_TICK 핸들러
    void ValidateAttack(uint32_t uidHigh, uint32_t uidLow,
                        WeaponClass wcls, long long nowMs);

    // HIT_TICK 핸들러
    void ValidateHit(uint32_t uidHigh, uint32_t uidLow,
                     WeaponClass wcls,
                     const float srcPos[3],
                     const float vicPos[3],
                     float dmg);

    void OnPlayerLeave(uint32_t uidHigh, uint32_t uidLow);
};

}
```

## 무기 클래스별 발사 간격

```
Melee     : 450 ± 100 ms      ← 무한 강베기 핵심 임계
Shotgun   : 1100 ± 150 ms
Revolver  : 300 ± 80 ms
SMG       : 70  ± 30 ms       ← RapidFire 가장 흔한 표적
Rifle     : 100 ± 40 ms
Rocket    : 1800 ± 200 ms
```

**InfiniteSlash (무한 강베기)** 는 GunZ 의 시그니처 핵. 합법 슬래시 콤보 (350~550 ms) 와 핵 사용 (50~200 ms 연타) 의 분포가 명확히 분리되어 검증 신뢰도가 높음. 본 패키지는 Melee 350 ms (450 - 100) 미만이 1 회 발생 시 Low, 3 회 누적 시 Medium 으로 승급.

`ValidateAttack` 은 최근 20 개 (`recent` 역스캔) 와 `nowMs` 의 간격을 무기 클래스 임계와 비교.

## Severity 승급

| 연속 위반 | severity |
|----------|---------|
| 1 | Low |
| 3 | Medium |
| 5 | High |
| 10 | Critical |

## ValidateHit — 두 검증

### ImpossibleHit
```
range = WeaponMaxRange(wcls)  // Melee 200, SMG 1500, Rifle 3000, etc.
dist = sqrt(|srcPos - vicPos|²)
if dist > range × 1.3:
    → ImpossibleHit 시그널 (severity 거리 비율 따라)
```

### AutoAim (통계)
```
shotsHit += 1
if shotsFired >= 50 and (shotsHit / shotsFired) > 0.95:
    → AutoAim Medium (단독 판정 금지, 패턴 누적 필요)
```

## Integration Points
- `CSCommon/Security/MCombatValidator.{h,cpp}` 신규
- ATTACK_TICK 핸들러에서 `ValidateAttack`
- HIT_TICK 핸들러에서 `ValidateHit`
- `ObjectRemove` 에서 `OnPlayerLeave`

## Configuration

| 환경변수 | 기본 |
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
| 조건 | 결과 |
|------|------|
| 무기 클래스 Unknown / ItemKit | ValidateAttack 스킵 |
| `recent` 비어있음 | 첫 발 통과 (history 누적 시작) |
| 합법 슬래시 콤보 (450 ms 내 정확히 450 ms) | tolerance 내 통과 |
| AutoAim 통계 단독 판정 (운영) | **금지** — 패턴 누적 필요 |

## Test Vectors

```
RapidFire SMG:
  ValidateAttack(SMG, 1000)  → OK
  ValidateAttack(SMG, 1030)  → 30ms < 70ms - 30 = 40, 위반
  ValidateAttack(SMG, 1060)  → 위반 누적 2회
  ValidateAttack(SMG, 1090)  → 위반 누적 3회 → Medium

InfiniteSlash:
  ValidateAttack(Melee, 1000)
  ValidateAttack(Melee, 1100)  → 100ms < 350 (450-100), 위반

ImpossibleHit:
  ValidateHit(SMG, srcPos=(0,0,0), vicPos=(2500,0,0), dmg=10)
  → dist=2500 > 1500×1.3=1950, ImpossibleHit
```

## Limitations
- AutoAim 통계는 **단독 판정 금지** — 정밀 조준 가능한 프로 유저도 동일 신호. 매치 단위 분포 분석 필요
- 무기별 max range 테이블 미포함 (별도 데이터)
- 자가 데미지 (낙사, 자폭) 의 HIT_TICK 송신 제외 필요 (클라 측 필터)

## Cross-References
- 인프라: [`05-position-history.md`](./05-position-history.md)
- 시그널: [`08-signal-collector.md`](./08-signal-collector.md)
- Novel 축: [`09-weapon-spoof-novel.md`](./09-weapon-spoof-novel.md), [`10-position-lie-novel.md`](./10-position-lie-novel.md)
- 통합: [`../08-integration-guide.md`](../08-integration-guide.md) §2.7, §2.8
