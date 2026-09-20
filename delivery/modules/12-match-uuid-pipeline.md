**English** | [한국어](../ko/modules/12-match-uuid-pipeline.md)

# Module 12 — match_uuid Lifecycle Pipeline

## Purpose
Operational infrastructure that makes every anti-cheat signal identifiable per round. The foundation of **per-round analysis**.

## Format

```
match_uuid = "stage-<UID_HIGH>-<UID_LOW>-<EPOCH_SEC>"

e.g.: "stage-12345-67890-1714867200"
```

Under 64 characters (the community-api `match_uuid` column is 65 characters — including the null terminator).

## Interface

### MMatchStage side

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

### MSignalCollector side — auto-stamping

```cpp
class MSignalCollector {
    // per shard: unordered_map<uid, char[65]> match UUID
    void SetMatchUUID(uint32_t uidHigh, uint32_t uidLow, const char* uuid);
    void ClearMatchUUID(uint32_t uidHigh, uint32_t uidLow);

    void Push(uint32_t uidHigh, uint32_t uidLow, CheatSignal& sig) {
        // if sig.matchUUID is empty, auto-stamp with the shard's match UUID
        if (sig.matchUUID[0] == '\0') {
            auto& shard = GetShard(uidHigh, uidLow);
            std::lock_guard lk(shard.mutex);
            auto it = shard.matchUUIDMap.find(MakeKey(uidHigh, uidLow));
            if (it != shard.matchUUIDMap.end()) {
                strncpy(sig.matchUUID, it->second.c_str(), sizeof(sig.matchUUID));
            }
        }
        // ... accumulate the signal in the shard
    }
};
```

## Lifecycle

```
1. OnStageStart (pStage->StartGame() == true branch)
   ├─ pStage->GenerateMatchUUID()
   └─ iterate over all players:
        Security::MSignalCollector::Instance().SetMatchUUID(uid.H, uid.L, uuid)

2. During the round
   └─ Validator → CheatSignal {matchUUID=""} → Collector.Push
        → auto-stamped with the shard's UUID on Push

3. StageFinishGame
   ├─ iterate over all players:
   │    Security::MSignalCollector::Instance().ClearMatchUUID(uid.H, uid.L)
   └─ pStage->ClearMatchUUID()

4. ObjectRemove (player leaves)
   └─ Security::MSignalCollector::Instance().OnPlayerLeave(uid.H, uid.L)
        → match_uuid cleaned up automatically

5. (If needed) next round starts → repeat from 1
```

## Double Cleanup (Why)

Scenario where players stay in the same stage and wait after the game ends:
- `ObjectRemove` is not called (they did not leave the stage)
- On the next round start, the new match_uuid is overwritten via `SetMatchUUID`
- But signals between round end and next round start would be stamped with the **previous UUID**

→ The explicit `ClearMatchUUID` in `StageFinishGame` closes this window.

## Integration Points
- `m_szMatchUUID` + 3 methods in `MMatchStage.{h,cpp}`
- `GetObjBegin/End` iteration + `SetMatchUUID` at the end of the `StartGame()==true` branch in `MMatchServer_Stage.cpp::OnStageStart`
- Same iteration + `ClearMatchUUID` in `MMatchServer::StageFinishGame`
- `OnPlayerLeave` in `MMatchServer::ObjectRemove`

Details → [`../08-integration-guide.md`](../08-integration-guide.md) §2.9, §2.10

## DB Schema

`match_uuid VARCHAR(65)` column in the `anticheat_signals` table on the `community-api` side.

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

## Operational Use

### Query accumulated signals within a round
```sql
SELECT player_uid_high, player_uid_low,
       signal_type, severity, COUNT(*) as cnt
FROM anticheat_signals
WHERE match_uuid = 'stage-12345-67890-1714867200'
GROUP BY player_uid_high, player_uid_low, signal_type, severity
ORDER BY cnt DESC;
```

### Compute weighted score per round
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

### Suspicious round → replay matching
```
match_uuid = "stage-12345-67890-1714867200"
→ stage UID High = 12345, Low = 67890, epoch = 1714867200
→ search replay files (stage UID must be recorded at save time)
```

## Configuration
- No dedicated environment variables (triggered automatically on stage start/end)

## Failure Modes
| Condition | Result |
|------|------|
| `OnStageStart` call missing | Signals have empty match_uuid — per-round analysis impossible (other behavior normal) |
| `StageFinishGame` Clear missing | Next round's signals stamped with the previous UUID |
| `ObjectRemove` missing | Memory leak in the shard's match UUID map |
| Clock going backwards (same epoch) | Possible match_uuid collision — stage UID is usually unique enough |

## Test Vectors

```
Stage start:
  pStage UID = (12345, 67890), epoch = 1714867200
  GenerateMatchUUID → "stage-12345-67890-1714867200"
  SetMatchUUID(playerA, "stage-...")
  SetMatchUUID(playerB, "stage-...")

Signal raised:
  Push(playerA, sig{type=WeaponSpoof, matchUUID=""})
  → auto-stamped → sig.matchUUID = "stage-12345-67890-1714867200"
  → same value when reported to community-api

Stage end:
  ClearMatchUUID(playerA), ClearMatchUUID(playerB)
  pStage->ClearMatchUUID()

Subsequent signal:
  Push(playerA, sig{matchUUID=""})
  → auto-stamped → sig.matchUUID = "" (normal, outside a round)
```

## Limitations
- Possible match_uuid collision — assumes stage UID + epoch do not collide. In a distributed match-server environment, adding a server ID is recommended (`stage-<server>-<H>-<L>-<epoch>`)
- Signals outside a round (lobby, etc.) have an empty match_uuid — separate analysis category

## Cross-References
- Signals: [`08-signal-collector.md`](./08-signal-collector.md)
- Reporting: [`11-api-client-hmac.md`](./11-api-client-hmac.md)
- Operations: [`../05-operations-runbook.md`](../05-operations-runbook.md) §2
- Integration: [`../08-integration-guide.md`](../08-integration-guide.md) §2.9
