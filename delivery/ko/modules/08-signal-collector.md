[English](../../modules/08-signal-collector.md) | **한국어**

# Module 08 — Signal Collector (16-shard)

## Purpose
Validator 들이 발행하는 시그널을 수집·집계. 동시 접속 N → 단일 mutex 컨텐션 → push 지연 문제를 16-shard 분산으로 해결.

## Interface

```cpp
namespace Security {

enum class CheatSeverity { Low, Medium, High, Critical };

enum class CheatSignalType : uint16_t {
    // Movement (100~199)
    SpeedHack         = 101,
    TeleportSuspect   = 102,
    FlyHack           = 103,

    // Combat (200~299)
    RapidFire         = 201,
    InfiniteSlash     = 202,
    AutoAim           = 203,
    ImpossibleHit     = 204,
    WeaponSpoof       = 205,  // Novel 축 (#09)

    // Network (500~599)
    PacketManipulation = 501, // Position Lie 등 — Novel 축 (#10)
    ReplayAttack       = 502,

    // ... (총 21 종)
};

const char* SignalTypeToString(CheatSignalType);
const char* SeverityToString(CheatSeverity);

struct CheatSignal {
    CheatSignalType type;
    CheatSeverity severity;
    uint32_t uidHigh, uidLow;
    float value;
    float threshold;
    char matchUUID[65];
    char detail[256];
    long long timestamp;
};

struct PlayerSignalState {
    uint32_t scoreLow, scoreMed, scoreHigh, scoreCrit;
    uint32_t WeightedScore() const {
        return scoreLow * 1 + scoreMed * 3 + scoreHigh * 7 + scoreCrit * 10;
    }
};

class MSignalCollector {
public:
    static MSignalCollector& Instance();

    void Push(uint32_t uidHigh, uint32_t uidLow, const CheatSignal& sig);

    PlayerSignalState GetPlayerState(uint32_t uidHigh, uint32_t uidLow) const;

    // 가중 점수 상위 N
    std::vector<std::pair<uint64_t, uint32_t>> GetTopSuspects(size_t n) const;

    // batch flush (callback 등록 시)
    using ReportCallback = std::function<void(const std::vector<CheatSignal>&)>;
    void SetReportCallback(ReportCallback cb);
    void FlushToAPI();  // 주기적 호출

    // match_uuid 라이프사이클
    void SetMatchUUID(uint32_t uidHigh, uint32_t uidLow, const char* uuid);
    void ClearMatchUUID(uint32_t uidHigh, uint32_t uidLow);

    void OnPlayerLeave(uint32_t uidHigh, uint32_t uidLow);
};

}
```

## 16-shard 설계

```
shard_idx = hash(uid) % 16

각 shard 가 독립 std::mutex + std::unordered_map<uid, PlayerSignalState>

→ 동시 push 시 16 개 lock 으로 분산
```

`hash(uid)` 는 uidHigh^uidLow 의 단순 XOR. 분포 편향 시 별도 해시.

## 가중 점수

```
WeightedScore = (Low 횟수) × 1
              + (Med 횟수) × 3
              + (High 횟수) × 7
              + (Crit 횟수) × 10

기본 보고 임계: 50
→ 예: Low 50, Med 17, High 8, Crit 5 등이 임계 도달
```

## match_uuid 자동 스탬핑

`Push` 시 `sig.matchUUID` 가 빈 값이면 shard 의 현재 매치 UUID 로 자동 스탬핑. Validator 코드는 매치 UUID 를 알 필요 없음.

## Integration Points
- `CSCommon/Security/MSignalCollector.{h,cpp}` 신규
- `MMatchServer::OnCreate` 에서 `SetReportCallback`
- `MMatchServer::OnStageStart` 에서 모든 플레이어 `SetMatchUUID`
- `MMatchServer::StageFinishGame` 에서 `ClearMatchUUID`
- `MMatchServer::ObjectRemove` 에서 `OnPlayerLeave`
- 주기적 `FlushToAPI` (별도 worker 또는 매치서버 메인 루프)

## Configuration

| 환경변수 | 기본 |
|---------|------|
| `SIGNAL_REPORT_THRESHOLD_SCORE` | 50 |
| `SIGNAL_FLUSH_INTERVAL_MS` | 5000 |
| `SIGNAL_BATCH_MAX_SIZE` | 100 |

## Failure Modes
| 조건 | 결과 |
|------|------|
| `MAPIClient` 미초기화 (env 미설정) | callback 미등록, 메모리-only 동작 |
| API 5xx 응답 | `MAPIClient` 재시도 큐 (MAX 100) |
| 16 shards 모두 컨텐션 | 32-shard 또는 NUMA-aware 향후 |
| 시그널 push 실패 (메모리 부족) | 무시 (fail-open) |

## Test Vectors

```
Push 100 회 (uid=A):
  Low × 50, Med × 30, High × 15, Crit × 5
  → WeightedScore = 50 + 90 + 105 + 50 = 295

GetTopSuspects(5):
  → uid=A score=295, uid=B score=..., ...
```

## Limitations
- 시그널 타입별 가중치 (현재 모두 동일) — 타입별 차등 가중 향후
- 매치 단위 reset 정책 미명시 (현재는 매치 종료 시 누적 유지)
- 분산 매치서버 환경에서 shard 간 일관성 (현재는 단일 프로세스)

## Cross-References
- 발행자 (Movement): [`06-movement-validator.md`](./06-movement-validator.md)
- 발행자 (Combat): [`07-combat-validator.md`](./07-combat-validator.md)
- 발행자 (Novel 2축): [`09-weapon-spoof-novel.md`](./09-weapon-spoof-novel.md), [`10-position-lie-novel.md`](./10-position-lie-novel.md)
- 보고: [`11-api-client-hmac.md`](./11-api-client-hmac.md)
- 라이프사이클: [`12-match-uuid-pipeline.md`](./12-match-uuid-pipeline.md)
- 통합: [`../08-integration-guide.md`](../08-integration-guide.md) §2.10, §2.11
