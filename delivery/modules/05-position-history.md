# Module 05 — Position History Ring Buffer

## Purpose
서버 권위 좌표 인프라. 안티치트 Validator 들이 소비할 좌표 시계열 보관. lag-comp / rewind 검증 기반.

## Interface

```cpp
namespace Security {

struct PositionSample {  // POD
    float x, y, z;
    long long tMs;
};

class MPositionHistory {
public:
    MPositionHistory();  // 32-slot ring

    void Record(float x, float y, float z, long long tMs);

    // 가장 최근 샘플 (없으면 false)
    bool GetLatest(float outXYZ[3], long long* outTMs) const;

    // tMs 시점의 보간 좌표 (선형 보간, 범위 밖이면 false)
    bool QueryAt(long long tMs, float outXYZ[3]) const;

private:
    PositionSample m_Buf[32];
    uint32_t m_Head;
    uint32_t m_Count;
};

}
```

**STL-free 헤더** — `Sample` POD 만 노출. ABI 호환성 위협 회피 (CSCommon 이 v143/v145 양쪽 빌드되는 환경 대응).

## Storage
- 32 슬롯 ring buffer
- 32 Hz 송신 기준 약 1 초 history
- per-player 메모리: 32 × 16 B = 512 B

## Integration Points
- `CSCommon/Security/MPositionHistory.{h,cpp}` 신규
- `MMatchObject` 에 `Security::MPositionHistory m_PositionHistory` 멤버 + `GetPositionHistory()` 접근자
- POSITION_TICK 핸들러에서 `Record()`
- HIT_TICK 핸들러에서 `GetLatest()` (피해자 권위 좌표)
- (향후) lag-comp hit validator 에서 `QueryAt(shotClientTMs - RTT/2)`

## 좌표 송신
- `MC_MATCH_POSITION_TICK = 2903` (MACHINE2MACHINE)
- 32 Hz, 기존 `MC_PEER_BASICINFO` 옆에 piggyback
- ZPACKEDBASICINFO blob 재사용 (첫 4B fTime + 다음 6B short XYZ)
- 리플레이 모드 화이트리스트에 미포함 → 리플레이 중엔 자동 미송신 (안전)

## Configuration
- `RING_SIZE = 32` (컴파일 상수)
- 시간 소스: 서버측 `std::chrono::steady_clock` ms

## Failure Modes
| 조건 | 결과 |
|------|------|
| 접속 직후 history 0 | `GetLatest` false → Validator 가 스킵 (false-positive 회피) |
| dt 역행 (monotonicity 위반) | Validator 측에서 dt > 0 가드 |
| ring 가득 후 wrap | head 이전 샘플 덮어씀 (의도된 동작) |

## Test Vectors

```cpp
MPositionHistory h;
h.Record(0, 0, 0, 1000);
h.Record(100, 0, 0, 1100);  // 100 units / 100 ms = 1000 u/s

float xyz[3]; long long t;
h.GetLatest(xyz, &t);  // → (100, 0, 0, 1100)

float interp[3];
h.QueryAt(1050, interp);  // → (50, 0, 0) (선형 보간)
h.QueryAt(900, interp);   // → false (범위 밖)
```

## Limitations
- 32 슬롯은 32 Hz 기준 1 초 이력. 더 긴 lag-comp 윈도우가 필요하면 슬롯 수 증가 필요
- 위치만 저장 — anim bone / weapon orientation / view angle 미포함 (정밀 hit validation 에 필요)
- 보간이 선형 — 가속 곡선이 큰 이동 (대시 시작점 등) 에서 오차 가능

## Cross-References
- 소비자 (Movement): [`06-movement-validator.md`](./06-movement-validator.md)
- 소비자 (Combat): [`07-combat-validator.md`](./07-combat-validator.md)
- 소비자 (PositionLie novel): [`10-position-lie-novel.md`](./10-position-lie-novel.md)
- 통합: [`../08-integration-guide.md`](../08-integration-guide.md) §2.6
