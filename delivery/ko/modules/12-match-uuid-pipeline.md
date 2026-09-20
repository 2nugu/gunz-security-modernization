[English](../../modules/12-match-uuid-pipeline.md) | **한국어**

# Module 12 — match_uuid Lifecycle Pipeline

## Purpose
모든 안티치트 시그널을 라운드 단위로 식별 가능하게 만드는 운영성 인프라. **라운드 단위 분석** 의 기반.

## 형식

```
match_uuid = "stage-<UID_HIGH>-<UID_LOW>-<EPOCH_SEC>"

예: "stage-12345-67890-1714867200"
```

64 자 미만 (community-api `match_uuid` 컬럼 65 자 — null terminator 포함).

## Interface

### MMatchStage 측

```cpp
class MMatchStage {
    char m_szMatchUUID[65];

public:
    const char* GetMatchUUID() const { return m_szMatchUUID; }

    void GenerateMatchUUID() {
        MUID uid = GetUID();
        time_t epoch = time(nullptr);
        snprintf(m_szMatchUUID, sizeof(m_szMatchUUID),
                 "stage-%u-%u-%lu",
                 uid.High, uid.Low, (unsigned long)epoch);
    }

    void ClearMatchUUID() {
        m_szMatchUUID[0] = '\0';
    }
};
```

### MSignalCollector 측 — 자동 스탬핑

```cpp
class MSignalCollector {
    // shard 마다 unordered_map<uid, char[65]> 매치 UUID
    void SetMatchUUID(uint32_t uidHigh, uint32_t uidLow, const char* uuid);
    void ClearMatchUUID(uint32_t uidHigh, uint32_t uidLow);

    void Push(uint32_t uidHigh, uint32_t uidLow, CheatSignal& sig) {
        // sig.matchUUID 가 빈 값이면 shard 의 매치 UUID 로 자동 스탬핑
        if (sig.matchUUID[0] == '\0') {
            auto& shard = GetShard(uidHigh, uidLow);
            std::lock_guard lk(shard.mutex);
            auto it = shard.matchUUIDMap.find(MakeKey(uidHigh, uidLow));
            if (it != shard.matchUUIDMap.end()) {
                strncpy(sig.matchUUID, it->second.c_str(), sizeof(sig.matchUUID));
            }
        }
        // ... shard 에 시그널 누적
    }
};
```

## 라이프사이클

```
1. OnStageStart (pStage->StartGame() == true 분기)
   ├─ pStage->GenerateMatchUUID()
   └─ 모든 플레이어 순회:
        Security::MSignalCollector::Instance().SetMatchUUID(uid.H, uid.L, uuid)

2. 라운드 진행 중
   └─ Validator → CheatSignal {matchUUID=""} → Collector.Push
        → Push 시 자동으로 shard 의 UUID 로 스탬핑

3. StageFinishGame
   ├─ 모든 플레이어 순회:
   │    Security::MSignalCollector::Instance().ClearMatchUUID(uid.H, uid.L)
   └─ pStage->ClearMatchUUID()

4. ObjectRemove (player 이탈)
   └─ Security::MSignalCollector::Instance().OnPlayerLeave(uid.H, uid.L)
        → match_uuid 자동 정리

5. (필요 시) 다음 라운드 시작 → 1 부터 반복
```

## 이중 정리 (Why)

플레이어가 게임 종료 후 같은 스테이지에 남아 대기하는 시나리오:
- `ObjectRemove` 가 호출되지 않음 (스테이지 이탈 아님)
- 다음 라운드 시작 시 새 match_uuid 가 `SetMatchUUID` 로 덮어써짐
- 그러나 라운드 종료 ~ 다음 라운드 시작 사이의 시그널이 **이전 UUID** 로 스탬핑되는 문제

→ `StageFinishGame` 의 명시적 `ClearMatchUUID` 가 이 윈도우를 닫음.

## Integration Points
- `MMatchStage.{h,cpp}` 에 `m_szMatchUUID` + 메서드 3개
- `MMatchServer_Stage.cpp::OnStageStart` 의 `StartGame()==true` 분기 말미에 `GetObjBegin/End` 순회 + `SetMatchUUID`
- `MMatchServer::StageFinishGame` 동일 순회 + `ClearMatchUUID`
- `MMatchServer::ObjectRemove` 의 `OnPlayerLeave`

상세 → [`../08-integration-guide.md`](../08-integration-guide.md) §2.9, §2.10

## DB 스키마

`community-api` 측 `anticheat_signals` 테이블에 `match_uuid VARCHAR(65)` 컬럼.

Alembic migration `003_add_match_uuid_to_signals.py`:
```python
def upgrade():
    op.add_column('anticheat_signals',
                  sa.Column('match_uuid', sa.String(65), nullable=True))
    op.create_index('ix_anticheat_signals_match_uuid',
                    'anticheat_signals', ['match_uuid'])

def downgrade():
    op.drop_index('ix_anticheat_signals_match_uuid')
    op.drop_column('anticheat_signals', 'match_uuid')
```

## 운영 활용

### 라운드 내 누적 시그널 조회
```sql
SELECT player_uid_high, player_uid_low,
       signal_type, severity, COUNT(*) as cnt
FROM anticheat_signals
WHERE match_uuid = 'stage-12345-67890-1714867200'
GROUP BY player_uid_high, player_uid_low, signal_type, severity
ORDER BY cnt DESC;
```

### 가중 점수 라운드 단위 산출
```sql
SELECT player_uid_high, player_uid_low,
       SUM(CASE severity
           WHEN 'Low' THEN 1
           WHEN 'Medium' THEN 3
           WHEN 'High' THEN 7
           WHEN 'Critical' THEN 10
           END) as score
FROM anticheat_signals
WHERE match_uuid = 'stage-...'
GROUP BY player_uid_high, player_uid_low
HAVING score > 50
ORDER BY score DESC;
```

### 의심 라운드 → 리플레이 매칭
```
match_uuid = "stage-12345-67890-1714867200"
→ stage UID High = 12345, Low = 67890, epoch = 1714867200
→ 리플레이 파일 검색 (저장 시 stage UID 기록되어 있어야 함)
```

## Configuration
- 별도 환경변수 없음 (스테이지 시작/종료 자동 발생)

## Failure Modes
| 조건 | 결과 |
|------|------|
| `OnStageStart` 호출 누락 | 시그널의 match_uuid 빈 값 — 라운드 단위 분석 불가 (다른 동작 정상) |
| `StageFinishGame` Clear 누락 | 다음 라운드 시그널이 이전 UUID 로 스탬핑 |
| `ObjectRemove` 누락 | shard 의 매치 UUID map 메모리 누수 |
| 시계 후행 (epoch 동일) | match_uuid 충돌 가능 — stage UID 가 보통 충분히 unique 함 |

## Test Vectors

```
스테이지 시작:
  pStage UID = (12345, 67890), epoch = 1714867200
  GenerateMatchUUID → "stage-12345-67890-1714867200"
  SetMatchUUID(playerA, "stage-...")
  SetMatchUUID(playerB, "stage-...")

시그널 발생:
  Push(playerA, sig{type=WeaponSpoof, matchUUID=""})
  → 자동 스탬핑 → sig.matchUUID = "stage-12345-67890-1714867200"
  → community-api 보고 시 동일 값

스테이지 종료:
  ClearMatchUUID(playerA), ClearMatchUUID(playerB)
  pStage->ClearMatchUUID()

이후 시그널:
  Push(playerA, sig{matchUUID=""})
  → 자동 스탬핑 → sig.matchUUID = "" (정상, 라운드 외부)
```

## Limitations
- match_uuid 충돌 가능성 — stage UID + epoch 가 충돌하지 않는다는 가정. 분산 매치서버 환경에서는 서버 ID 추가 권장 (`stage-<server>-<H>-<L>-<epoch>`)
- 라운드 외부 (대기실 등) 시그널은 match_uuid 가 빈 값 — 별도 분석 카테고리

## Cross-References
- 시그널: [`08-signal-collector.md`](./08-signal-collector.md)
- 보고: [`11-api-client-hmac.md`](./11-api-client-hmac.md)
- 운영: [`../05-operations-runbook.md`](../05-operations-runbook.md) §2
- 통합: [`../08-integration-guide.md`](../08-integration-guide.md) §2.9
