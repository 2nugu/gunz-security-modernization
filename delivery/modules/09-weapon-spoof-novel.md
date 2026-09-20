**English** | [한국어](../ko/modules/09-weapon-spoof-novel.md)

# Module 09 — Weapon Spoof Detection (Novel axis)

## Purpose
**Detection axis #1 unique to this package**. If an attacker tampers with the weaponType field of ATTACK_TICK, they can pose as attacking with a weapon class they have not equipped. This is detected by cross-checking against server-authoritative equipment data.

## Why this is Novel
- Original GunZ has no server-side fire validation → weaponType is processed purely on the client's report
- Conventional anti-cheat (RapidFire, AutoAim, etc.) focuses on fire rate / hit rate → weapon-type tampering is a blind spot
- This check uses equipment authority data the server already holds (`m_EquipedItem`) → **zero additional data**, additional cost ~O(1)

## Approach

```
On ATTACK_TICK receipt:
    reportedClass = ClassifyMMatchWeapon(packet.weaponType)
    if reportedClass == Unknown or ItemKit:
        return  // cannot validate, skip

    server_equipped = [
        ClassifyMMatchWeapon(GetCharInfo()->m_EquipedItem.GetItem(MMCIP_MELEE)),
        ClassifyMMatchWeapon(GetCharInfo()->m_EquipedItem.GetItem(MMCIP_PRIMARY)),
        ClassifyMMatchWeapon(GetCharInfo()->m_EquipedItem.GetItem(MMCIP_SECONDARY))
    ]

    if reportedClass not in server_equipped:
        if !pObj->IsEquipmentSeen():
            return  // avoid false positives during loading

        Push(WeaponSpoof, severity=High,
             value=reportedClass,
             threshold=0,
             detail="reported=<X> slots=<Y,Z,W>")
```

## Severity policy

Equipment data is **fully server-authoritative**. No possibility of ordinary noise → **no consecutive-violation escalation needed**; a single occurrence is immediately High.

| Scenario | severity |
|---------|---------|
| Ordinary weapon spoof | **High** |
| Repeated spoof (3+) | Automatic escalation to Critical (optional) |

## Integration Points
- Inline in the `case MC_MATCH_ATTACK_TICK` handler of `MMatchServer_OnCommand.cpp`
- No new class — inline check after the `MCombatValidator::ValidateAttack` call

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
- No dedicated env vars (based on server-authoritative data)

## Failure Modes
| Condition | Result |
|------|------|
| During loading (equipment info not yet received) | `IsEquipmentSeen()` false → skipped (false-positive avoidance) |
| Right after a weapon swap (server sync race) | Transient mismatch possible — add a `lastEquipChangeMs + tolerance` grace window in the future |
| Missing class mapping (when new weapons are added) | Classified as `Unknown` → skipped (catches less, but zero false positives) |

## Test Vectors

```
Normal:
  equipped = [Katana(Melee), AK47(Rifle), Beretta(Revolver)]
  ATTACK_TICK weaponType = AK47 (Rifle class)
  → reportedClass=Rifle, slot match → pass

Tampered:
  equipped = [Katana, AK47, Beretta]
  ATTACK_TICK weaponType = RPG (Rocket class)
  → reportedClass=Rocket, slots [Melee, Rifle, Revolver] no match
  → WeaponSpoof High, detail="reported=RPG eq=[Katana, AK47, Beretta]"

Equipment not received:
  right after the player joins
  → IsEquipmentSeen() false → skipped (normal)
```

## Limitations
- Swapping to a different weapon of the same class (e.g. AK47 → M16, both Rifle) is not detected as tampering — an intended limit of class-level comparison
- If weapon class classification is inaccurate (`ClassifyMMatchWeapon`), bypass is possible — the accuracy of the classification table determines validation accuracy

## Limitations vs Strengths
| Aspect | Assessment |
|------|------|
| False positives | Very low (apart from the loading case) |
| False negatives | Not detected when spoofing a same-class weapon |
| Additional cost | O(1), comparison of 3 equipment slots |
| Additional data | 0 (server already holds it) |

## Cross-References
- ATTACK_TICK: [`07-combat-validator.md`](./07-combat-validator.md)
- Signal delivery: [`08-signal-collector.md`](./08-signal-collector.md)
- Integration: [`../08-integration-guide.md`](../08-integration-guide.md) §2.7
