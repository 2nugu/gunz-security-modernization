[English](../../modules/06-movement-validator.md) | **한국어**

# Module 06 — Movement Validator

## Purpose
스피드핵 / 텔레포트핵 / 플라이핵의 서버측 탐지. POSITION_TICK 32 Hz 데이터 소비.

## Interface

```cpp
namespace Security {

struct MovementLimits {
    float maxDashSpeedUPS    = 1200.f;  // 합법 최고 ~1500 (대시+버터링 스파이크)
    float speedTolerance     = 1.5f;    // 1200 × 1.5 = 1800 u/s 컷
    float teleportDistPerSec = 5000.f;  // 5000 u/s 초과 = teleport
    int   flyhackAirtimeMs   = 5000;    // 5 초 이상 airtime
};

class MMovementValidator {
public:
    static MMovementValidator& Instance();

    void SetLimits(const MovementLimits& lim);

    // 매 POSITION_TICK 에서 호출
    void Validate(uint32_t uidHigh, uint32_t uidLow,
                  float x, float y, float z, long long tMs);

    void OnPlayerLeave(uint32_t uidHigh, uint32_t uidLow);

private:
    MMovementValidator();
    // per-player state map (uid → {lastSample, violationStreak, severity})
};

}
```

## 검증 항목

### Speed
```
dt = tMs - lastTMs
dist = sqrt(dx² + dy² + dz²)
speed = dist / (dt / 1000)

if speed > limits.maxDashSpeedUPS × limits.speedTolerance:
    → SpeedHack 시그널, 연속위반 누적
```

### Teleport
```
if speed > limits.teleportDistPerSec:
    if !checkAlive (리스폰/워프 면제):
        → TeleportSuspect 시그널
```

### FlyHack (heuristic)
```
if z 가 base 위에서 일정 시간 (airtime > flyhackAirtimeMs) 유지:
    → FlyHack 시그널, severity Medium 캡 (BSP 미통합 → 점프맵 false-positive 가능)
```

## Severity 승급

| 연속 위반 | severity |
|----------|---------|
| 1 | Low |
| 3 | Medium |
| 5 | High |
| 10 | Critical |

리셋 조건: 정상 행동 N 분 또는 매치 종료.

## Integration Points
- `CSCommon/Security/MMovementValidator.{h,cpp}` 신규
- POSITION_TICK 핸들러에서 호출 (1줄)
- `ObjectRemove` 에서 `OnPlayerLeave` 호출

## Configuration

| 환경변수 | 기본 |
|---------|------|
| `MOVEMENT_MAX_SPEED_UPS` | 1200 |
| `MOVEMENT_SPEED_TOLERANCE` | 1.5 |
| `MOVEMENT_TELEPORT_DIST_PER_SEC` | 5000 |
| `MOVEMENT_FLYHACK_AIRTIME_MS` | 5000 |

## Failure Modes
| 조건 | 결과 |
|------|------|
| 합법 대시 (~1500 u/s) | 1800 (1200×1.5) 미만 → 통과 |
| 리스폰 직후 좌표 점프 | `CheckAlive` 면제 |
| 리스폰 직전 dt = 0 | dt > 0 가드 |
| 점프맵 (높은 발판 연속) | FlyHack heuristic Medium 캡 — 운영자 리뷰 |

## Test Vectors

```
정상 대시:
  Record(0, 0, 0, 1000)
  Record(150, 0, 0, 1100)  // 1500 u/s
  → 통과 (1500 < 1800)

스피드핵:
  Record(0, 0, 0, 1000)
  Record(300, 0, 0, 1100)  // 3000 u/s
  → SpeedHack Low (1회)
  Record(600, 0, 0, 1200)  // 3000 u/s 누적 2회 (Low)
  Record(900, 0, 0, 1300)  // 3000 u/s 3회 → Medium 승급

텔레포트:
  Record(0, 0, 0, 1000)
  Record(10000, 0, 0, 1100)  // 100,000 u/s
  → TeleportSuspect (severity 별도 정책)
```

## Limitations
- BSP 지형 샘플링 미통합 → 정밀 noclip 미검출
- z-축 판정이 heuristic — 점프맵 false-positive 가능
- 합법 텔레포트 (워프, 리스폰) 면제는 `CheckAlive` 에 의존 — 면제 우회 시도 시 추가 검증 필요

## Cross-References
- 인프라: [`05-position-history.md`](./05-position-history.md)
- 시그널 처리: [`08-signal-collector.md`](./08-signal-collector.md)
- 카탈로그: [`../04-anticheat-catalog.md`](../04-anticheat-catalog.md) §2.1
- 통합: [`../08-integration-guide.md`](../08-integration-guide.md) §2.6
