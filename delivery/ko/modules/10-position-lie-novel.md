[English](../../modules/10-position-lie-novel.md) | **한국어**

# Module 10 — Position Lie Detection (Novel 축)

## Purpose
**본 패키지 고유 탐지 축 #2**. HIT_TICK 의 `srcPos` (공격자 보고 발사 좌표) 가 서버 POSITION_TICK 링버퍼의 최근 샘플과 일치하는지 검증. **POSITION × HIT 크로스체크**.

## 왜 Novel 인가
- 일반적인 hit validation 은 `ImpossibleHit` (사거리) 와 `AutoAim` (명중률) 에 집중
- 공격자가 자신의 좌표를 변조해 사거리/시야를 우회하는 경로는 사각지대
- 본 검증은 두 채널 (POSITION_TICK 32 Hz + HIT_TICK 피격 시) 의 **시간적 정합성** 확인

## Approach

```
HIT_TICK 수신 시:
    1. 공격자의 서버 권위 좌표 조회
       atkSrvXYZ, atkSrvTMs = pAttacker->GetPositionHistory().GetLatest()
       if 조회 실패 (history 비어있음): return  // 접속 직후 false-positive 회피

    2. 공격자가 보고한 srcPos 와 비교
       dist = sqrt(|srcPos - atkSrvXYZ|²)

    3. 무기 클래스별 threshold
       Melee:    400 u
       Shotgun:  600 u
       Revolver: 600 u
       SMG:      800 u
       Rifle:    800 u
       Rocket:   1000 u
       (월드 좌표 기준 ~100u = 1m, 실제 게임 단위)

    4. threshold 초과 시 PacketManipulation 시그널
       severity:
         dist > threshold × 2  →  High
         threshold < dist ≤ threshold × 2  →  Medium
       (단발 탐지, 에스컬레이션 없음 — POSITION_TICK 32 Hz 가 충분히 dense)
```

## Why threshold 다른가

- **Melee (400)**: 근접 무기는 srcPos 가 캐릭터 박스 안에 있어야 함. 작은 허용 폭 (latency 보정)
- **Shotgun/Revolver (600)**: 중거리, 캐릭터-총기 오프셋 + latency 허용
- **SMG/Rifle (800)**: 자세 변화 + 보정 여유
- **Rocket (1000)**: 발사체 무기, 시점 차이가 큼 (발사 시점 vs 명중 시점 좌표)

## Integration Points
- `MMatchServer_OnCommand.cpp` 의 `case MC_MATCH_HIT_TICK` 핸들러 inline
- 별도 클래스 신설 없음 — `MCombatValidator::ValidateHit` 호출 *전* 에 inline 검증
- 의존: [`05-position-history.md`](./05-position-history.md), [`07-combat-validator.md`](./07-combat-validator.md), [`08-signal-collector.md`](./08-signal-collector.md)

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
    // GetLatest 실패 → 접속 직후 POSITION_TICK 누적 전, 스킵 (false-positive 회피)

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

| 환경변수 | 기본 |
|---------|------|
| `POSITIONLIE_MELEE_DIST` | 400 |
| `POSITIONLIE_SHOTGUN_DIST` | 600 |
| `POSITIONLIE_REVOLVER_DIST` | 600 |
| `POSITIONLIE_SMG_DIST` | 800 |
| `POSITIONLIE_RIFLE_DIST` | 800 |
| `POSITIONLIE_ROCKET_DIST` | 1000 |

## Failure Modes
| 조건 | 결과 |
|------|------|
| 접속 직후 (POSITION 미누적) | `GetLatest` 실패 → 스킵 |
| 정상 latency / 자세 변화 | threshold 내 통과 |
| 합법 워프 / 점프 (큰 좌표 변화) | POSITION_TICK 도 함께 갱신 → 일관성 유지 |
| 클라 시계 vs 서버 시계 차이 (RTT 큼) | threshold 가 충분히 여유 (Melee 400u = 4m) |

## Test Vectors

```
정상:
  공격자 server pos = (1000, 0, 0), 시점 t=2000
  HIT_TICK srcPos = (1010, 5, 0), wcls=SMG
  dist = sqrt(100+25+0) ≈ 11
  → 11 < 800, 통과

PositionLie (Medium):
  공격자 server pos = (1000, 0, 0)
  HIT_TICK srcPos = (1500, 0, 0), wcls=Melee
  dist = 500
  → 500 > 400 (Melee threshold), 800 (=400×2) 미만 → Medium

PositionLie (High):
  공격자 server pos = (1000, 0, 0)
  HIT_TICK srcPos = (3000, 0, 0), wcls=Melee
  dist = 2000
  → 2000 > 400×2 → High
```

## Limitations
- 무기별 threshold 는 공개 트리 자체 빌드 환경 측정값 — 라이브 트래픽 (latency 분포 다름) 에서 재튜닝 필요
- POSITION_TICK 32 Hz 누락 시 (네트워크 jitter) 마지막 샘플과 시점 차이 → 합법 이동도 false-positive 가능. 시간차 보정 (`atkSrvTMs - hitTMs > 100ms` 시 스킵) 추가 가능
- 시점-맞춤 보간 (`QueryAt(hitTMs)`) 으로 정확도 향상 가능 — 향후 작업

## Strengths
- POSITION_TICK 과 HIT_TICK 두 채널의 시간적 정합성 검증 — 단일 채널 변조로는 우회 불가
- 두 채널 모두 위조하려면 클라가 32 Hz 수준 좌표 시계열 일관성 유지 + 발사 시점 마다 정합 좌표 합성 필요 → 핵 작성 비용 증가

## Cross-References
- 인프라: [`05-position-history.md`](./05-position-history.md)
- HIT_TICK: [`07-combat-validator.md`](./07-combat-validator.md)
- 시그널: [`08-signal-collector.md`](./08-signal-collector.md)
- 통합: [`../08-integration-guide.md`](../08-integration-guide.md) §2.8
