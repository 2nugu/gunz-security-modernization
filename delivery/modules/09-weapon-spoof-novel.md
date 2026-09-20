# Module 09 — Weapon Spoof Detection (Novel 축)

## Purpose
**본 패키지 고유 탐지 축 #1**. 공격자가 ATTACK_TICK 의 weaponType 필드를 변조하면, 장착하지 않은 무기 클래스로 공격 행세 가능. 이를 서버-권위 장착 데이터와 대조해 탐지.

## 왜 Novel 인가
- 오리지널 GunZ 는 서버측 발사 검증이 부재 → weaponType 자체가 클라 신고만으로 처리
- 일반적인 안티치트 (RapidFire, AutoAim 등) 는 발사 빈도 / 명중률에 집중 → 무기 종류 변조는 사각지대
- 본 검증은 서버가 이미 보유한 장비 권위 데이터(`m_EquipedItem`)를 활용 → **추가 데이터 0**, 추가 비용 ~O(1)

## Approach

```
ATTACK_TICK 수신 시:
    reportedClass = ClassifyMMatchWeapon(packet.weaponType)
    if reportedClass == Unknown or ItemKit:
        return  // 검증 불가, 스킵

    server_equipped = [
        ClassifyMMatchWeapon(GetCharInfo()->m_EquipedItem.GetItem(MMCIP_MELEE)),
        ClassifyMMatchWeapon(GetCharInfo()->m_EquipedItem.GetItem(MMCIP_PRIMARY)),
        ClassifyMMatchWeapon(GetCharInfo()->m_EquipedItem.GetItem(MMCIP_SECONDARY))
    ]

    if reportedClass not in server_equipped:
        if !pObj->IsEquipmentSeen():
            return  // 로딩 중 false-positive 회피

        Push(WeaponSpoof, severity=High,
             value=reportedClass,
             threshold=0,
             detail="reported=<X> slots=<Y,Z,W>")
```

## Severity 정책

장비 데이터는 **서버 완전 권위**. 일반 noise 가능성 없음 → **연속위반 승급 불필요**, 단발 즉시 High.

| 시나리오 | severity |
|---------|---------|
| 일반 weapon spoof | **High** |
| 반복 spoof (3회+) | Critical 자동 승급 (선택) |

## Integration Points
- `MMatchServer_OnCommand.cpp` 의 `case MC_MATCH_ATTACK_TICK` 핸들러 inline
- 별도 클래스 신설 없음 — `MCombatValidator::ValidateAttack` 호출 후 inline 검증

## Code Snippet (inline)

```cpp
case MC_MATCH_ATTACK_TICK:
{
    BYTE weaponType;
    pCommand->GetParameter(&weaponType, 0, MPT_UCHAR);

    auto reportedClass = Security::ClassifyMMatchWeapon(weaponType);
    if (reportedClass == Security::WeaponClass::Unknown
        || reportedClass == Security::WeaponClass::ItemKit) break;

    // ValidateAttack (RapidFire / InfiniteSlash) — Module 07
    Security::MCombatValidator::Instance().ValidateAttack(
        sid.High, sid.Low, reportedClass, NowMs());

    // [INLINE] Weapon Spoof check
    auto& items = pObj->GetCharInfo()->m_EquipedItem;
    bool match = false;
    for (auto slot : {MMCIP_MELEE, MMCIP_PRIMARY, MMCIP_SECONDARY}) {
        auto* item = items.GetItem(slot);
        if (!item) continue;
        if (Security::ClassifyMMatchWeapon(item->GetType()) == reportedClass) {
            match = true; break;
        }
    }

    if (!match && pObj->IsEquipmentSeen()) {
        Security::CheatSignal sig;
        sig.type = Security::CheatSignalType::WeaponSpoof;
        sig.severity = Security::CheatSeverity::High;
        sig.value = (float)weaponType;
        sig.threshold = 0;
        snprintf(sig.detail, sizeof(sig.detail),
                 "reported=%d eq=[%d,%d,%d]",
                 weaponType,
                 items.GetItem(MMCIP_MELEE) ? items.GetItem(MMCIP_MELEE)->GetType() : 0,
                 items.GetItem(MMCIP_PRIMARY) ? items.GetItem(MMCIP_PRIMARY)->GetType() : 0,
                 items.GetItem(MMCIP_SECONDARY) ? items.GetItem(MMCIP_SECONDARY)->GetType() : 0);
        Security::MSignalCollector::Instance().Push(sid.High, sid.Low, sig);
    }
    break;
}
```

## Configuration
- 별도 환경변수 없음 (서버 권위 데이터 기반)

## Failure Modes
| 조건 | 결과 |
|------|------|
| 로딩 중 (장비 정보 미수신) | `IsEquipmentSeen()` false → 스킵 (false-positive 회피) |
| 무기 교체 직후 (서버 동기화 race) | 일시적 mismatch 가능 — 향후 `lastEquipChangeMs + tolerance` 그레이스 윈도우 추가 |
| 클래스 매핑 누락 (신규 무기 추가 시) | `Unknown` 로 분류 → 스킵 (덜 잡되 false-positive 0) |

## Test Vectors

```
정상:
  장착 = [Katana(Melee), AK47(Rifle), Beretta(Revolver)]
  ATTACK_TICK weaponType = AK47 (Rifle 클래스)
  → reportedClass=Rifle, 슬롯 매치 → 통과

변조:
  장착 = [Katana, AK47, Beretta]
  ATTACK_TICK weaponType = RPG (Rocket 클래스)
  → reportedClass=Rocket, 슬롯 [Melee, Rifle, Revolver] 미매치
  → WeaponSpoof High, detail="reported=RPG eq=[Katana, AK47, Beretta]"

장비 미수신:
  플레이어 입장 직후
  → IsEquipmentSeen() false → 스킵 (정상)
```

## Limitations
- 같은 클래스의 다른 무기로 교체 시 (예: AK47 → M16, 둘 다 Rifle) 변조 미탐지 — 클래스 단위 비교로 의도된 한계
- 무기 클래스 분류가 부정확하면 (`ClassifyMMatchWeapon`) 우회 가능 — 분류 테이블 정확성이 검증 정확성을 결정

## Limitations vs Strengths
| 측면 | 평가 |
|------|------|
| 거짓 양성 | 매우 낮음 (로딩 중 케이스 외) |
| 거짓 음성 | 같은 클래스 무기 spoof 시 미탐지 |
| 추가 비용 | O(1) 장비 슬롯 3개 비교 |
| 추가 데이터 | 0 (서버가 이미 보유) |

## Cross-References
- ATTACK_TICK: [`07-combat-validator.md`](./07-combat-validator.md)
- 시그널 전달: [`08-signal-collector.md`](./08-signal-collector.md)
- 통합: [`../08-integration-guide.md`](../08-integration-guide.md) §2.7
