**English** | [한국어](../ko/modules/08-signal-collector.md)

# Module 08 — Signal Collector (16-shard)

## Purpose
Collects and aggregates the signals emitted by the validators. Solves the N concurrent users → single-mutex contention → push latency problem by distributing across 16 shards.

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
    WeaponSpoof       = 205,  // Novel axis (#09)

    // Network (500~599)
    PacketManipulation = 501, // Position Lie etc. — Novel axis (#10)
    ReplayAttack       = 502,

    // ... (21 types total)
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

    // Top N by weighted score
    std::vector<std::pair<uint64_t, uint32_t>> GetTopSuspects(size_t n) const;

    // batch flush (when a callback is registered)
    using ReportCallback = std::function<void(const std::vector<CheatSignal>&)>;
    void SetReportCallback(ReportCallback cb);
    void FlushToAPI();  // called periodically

    // match_uuid lifecycle
    void SetMatchUUID(uint32_t uidHigh, uint32_t uidLow, const char* uuid);
    void ClearMatchUUID(uint32_t uidHigh, uint32_t uidLow);

    void OnPlayerLeave(uint32_t uidHigh, uint32_t uidLow);
};

}
```

## 16-shard design

```
shard_idx = hash(uid) % 16

Each shard owns an independent std::mutex + std::unordered_map<uid, PlayerSignalState>

→ concurrent pushes are spread across 16 locks
```

`hash(uid)` is a plain XOR of uidHigh^uidLow. Use a separate hash if the distribution skews.

## Weighted score

```
WeightedScore = (Low count)  × 1
              + (Med count)  × 3
              + (High count) × 7
              + (Crit count) × 10

Default report threshold: 50
→ e.g. Low 50, Med 17, High 8, Crit 5, etc. reach the threshold
```

## Automatic match_uuid stamping

On `Push`, if `sig.matchUUID` is empty it is automatically stamped with the shard's current match UUID. Validator code does not need to know the match UUID.

## Integration Points
- New `CSCommon/Security/MSignalCollector.{h,cpp}`
- `SetReportCallback` in `MMatchServer::OnCreate`
- `SetMatchUUID` for all players in `MMatchServer::OnStageStart`
- `ClearMatchUUID` in `MMatchServer::StageFinishGame`
- `OnPlayerLeave` in `MMatchServer::ObjectRemove`
- Periodic `FlushToAPI` (separate worker or match-server main loop)

## Configuration

| Env var | Default |
|---------|------|
| `SIGNAL_REPORT_THRESHOLD_SCORE` | 50 |
| `SIGNAL_FLUSH_INTERVAL_MS` | 5000 |
| `SIGNAL_BATCH_MAX_SIZE` | 100 |

## Failure Modes
| Condition | Result |
|------|------|
| `MAPIClient` not initialized (env not set) | Callback not registered, memory-only operation |
| API 5xx response | `MAPIClient` retry queue (MAX 100) |
| Contention on all 16 shards | 32-shard or NUMA-aware in the future |
| Signal push failure (out of memory) | Ignored (fail-open) |

## Test Vectors

```
Push 100 times (uid=A):
  Low × 50, Med × 30, High × 15, Crit × 5
  → WeightedScore = 50 + 90 + 105 + 50 = 295

GetTopSuspects(5):
  → uid=A score=295, uid=B score=..., ...
```

## Limitations
- Per-signal-type weights (currently all identical) — differentiated weights per type in the future
- Per-match reset policy unspecified (currently accumulation is kept across match end)
- Cross-shard consistency in a distributed match-server environment (currently single process)

## Cross-References
- Emitter (Movement): [`06-movement-validator.md`](./06-movement-validator.md)
- Emitter (Combat): [`07-combat-validator.md`](./07-combat-validator.md)
- Emitters (2 Novel axes): [`09-weapon-spoof-novel.md`](./09-weapon-spoof-novel.md), [`10-position-lie-novel.md`](./10-position-lie-novel.md)
- Reporting: [`11-api-client-hmac.md`](./11-api-client-hmac.md)
- Lifecycle: [`12-match-uuid-pipeline.md`](./12-match-uuid-pipeline.md)
- Integration: [`../08-integration-guide.md`](../08-integration-guide.md) §2.10, §2.11
