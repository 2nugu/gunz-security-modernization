**English** | [한국어](../ko/modules/13-damage-validator.md)

# Module 13 — Damage Validator (Damage-Reduction Hack / Invincibility Hack Prevention)

> **Implementation Status: Implementation complete (Detection-Only default mode, Mitigation toggleable)**
>
> This module closes the blind spot created by the victim-authoritative HIT_TICK design.
>
> **Implementation complete (2026-05-06)**:
> - `CSCommon/Include/MDamageValidator.h` (190 lines)
> - `CSCommon/Source/MDamageValidator.cpp` (427 lines)
> - enum extension in `MCheatSignal.h` (`HPHack=303`, `HPInconsistency=304`, reusing the existing `DamageManipulation=203`)
> - `MMatchServer_OnCommand.cpp` HIT_TICK handler integration (`ComputeServerDamage` + `ValidateReportedDamage` + `RecordHit`)
> - `MMatchServer.cpp::OnRun` calls `RecordHPSnapshot` + `ValidateHPConsistency` on every active character behind a 1-second gate
> - `OnPlayerLeave` cleanup in `MMatchServer.cpp::ObjectRemove`
> - `CSCommon.vcxproj` ClInclude/ClCompile registration
> - **Build verification passed**:
>   - `MatchServer.exe` Release|Win32 v143 — **4.27 MB**, 4 new symbols detected via `strings`
>   - `Gunz.exe` Release|Win32 v143 — **7.04 MB**, no regression from the enum change (wire-stable guaranteed)
>
> **Default mode is Detection-Only** (`m_mitigationMode = false`). Signals are emitted only; the game damage flow stays as it was. Mitigation is enabled by calling `MDamageValidator::Instance().SetMitigationMode(true)` — recommended for the production environment only after importing the weapon table and verifying the enumeration of recovery mechanics.
>
> **Prerequisites for production deployment**:
> 1. Import the production weapon balance table via `MDamageValidator::Instance().RegisterWeaponProfile(WeaponClass, ...)` (replacing the values measured in the public-tree self-build environment)
> 2. Decide the flow for registering legitimate HP recovery mechanics (medkits, skills, natural regeneration, etc.) via `RecordHPRestore(uid, amount, source, tickMs)`
> 3. Enable Mitigation Mode via an environment variable or admin command toggle only after 1 and 2 have passed verification

## Purpose
**Closes the blind spot created by this package's victim-authoritative HIT_TICK design**.

Because the victim reports HIT_TICK from its own OnDamaged hook, **a hack on the victim side can report damage as 0 or reduced**. This manifests as two hacks:

| Hack | Behavior |
|----|------|
| **Damage-reduction hack (DamageReduce)** | victim reports received damage at 1/2 or 1/3, or omits HIT_TICK entirely |
| **Invincibility hack (Invincibility)** | victim blocks all HIT_TICK transmission, no HP change |
| **HP hack (HPHack)** | victim manipulates its own HP directly, ignoring damage |

This module closes the blind spot with a **server-side weapon damage table** + **attacker × victim dual-report cross-check** + **HP change consistency**.

---

## 1. Blind Spot Analysis

### 1.1 Trade-off of the current design
- **Attacker-authoritative hit** → attacker self-manipulates hit-rate (AutoAim damage explosion)
- **Victim-authoritative hit** (current design) → victim reduces/ignores received damage

→ Neither is safe on its own. A **hybrid** is needed.

### 1.2 Authority separation redesign

| Information | Authority |
|------|------|
| Fact that a hit occurred | **Reported by both sides** + cross-check for agreement |
| Attacker srcPos | Attacker report (PositionLie verified in [`10-position-lie-novel.md`](./10-position-lie-novel.md)) |
| Victim vicPos | Server-authoritative (`MPositionHistory.GetLatest`) |
| Weapon used | Attacker report (WeaponSpoof verified in [`09-weapon-spoof-novel.md`](./09-weapon-spoof-novel.md)) |
| **Damage amount** | **Server-computed** (weapon table + distance falloff + body-part multiplier) — client report ignored |
| Damage application result | Server updates victim HP, notifies client |

Key point: **the server computes the damage amount**. The client reports only that a hit occurred; the damage value is determined by the server weapon table.

---

## 2. Interface

```cpp
namespace Security {

struct WeaponDamageProfile {
    float minDamage;
    float maxDamage;
    float optimalRange;     // falloff start point
    float maxRange;         // beyond this distance apply minDamage or reject hit
    float headshotMultiplier;
    float legshotMultiplier;
    float falloffCurve;     // 0=linear, 1=quadratic
};

class MDamageValidator {
public:
    static MDamageValidator& Instance();

    // Register weapon damage table (at server startup)
    void RegisterWeaponProfile(uint32_t weaponType, const WeaponDamageProfile& prof);

    // Server-side authoritative damage computation
    float ComputeServerDamage(uint32_t weaponType,
                              float dist,
                              BodyPart part /* Head/Body/Legs */) const;

    // Validate HIT_TICK victim report
    // returns: severity. None = pass
    CheatSeverity ValidateReportedDamage(
        uint32_t weaponType,
        float dist,
        BodyPart part,
        float reportedDmg) const;

    // HP consistency validation — cumulative HIT_TICK sum vs HP delta over a window
    void RecordHit(uint32_t victimUidH, uint32_t victimUidL,
                   float reportedDmg, long long tMs);
    void RecordHPSnapshot(uint32_t victimUidH, uint32_t victimUidL,
                          float currentHP, long long tMs);
    CheatSeverity ValidateHPConsistency(uint32_t victimUidH, uint32_t victimUidL) const;
};

}
```

---

## 3. Four Validation Checks

### 3.1 ServerDamageOverride (active only in Mitigation Mode)

```
HIT_TICK received
    ↓
serverDmg = MDamageValidator::ComputeServerDamage(wcls, dist, part)
    ↓
[Detection-Only Mode]  victim HP deduction = reportedDmg (client report)
[Mitigation Mode]      victim HP deduction = serverDmg   (server-computed)
    ↓
Both modes: compare reported dmg with serverDmg
    ↓
abs(reportedDmg - serverDmg) > tolerance × serverDmg
    → DamageMismatch signal (severity by distance ratio)
```

**Effect (Mitigation)**: even if the damage-reduction hack reports 0, the server deducts the computed value. Client report ignored.

**Effect (Detection-Only)**: the hack's effect is not neutralized; signals only accumulate → manual sanction by operators.

**Residual risk**: an invincibility hack that blocks HIT_TICK itself — covered by §3.3 / §3.4 (both operate in both modes).

Detailed mode comparison → §11.

### 3.2 DamageRange validation

```
expected = [minDmg × falloff, maxDmg × falloff × headshotMul]

if reportedDmg < expected.lower × 0.7:
    → DamageReduce signal (allows ~30% legitimate peer variance)

if reportedDmg > expected.upper × 1.3:
    → DamageInflate signal (damage-amplification hack, a separate blind spot)
```

### 3.3 DamageNullification (HIT_TICK omission)

```
Attacker announces a hit via PEER_SHOT but
victim's HIT_TICK does not arrive within a set time (300 ms)
    → DamageNullification signal (HIT_TICK blocked)
```

This is a cross-check between **attacker-side information** (PEER_SHOT) and **victim-side information** (HIT_TICK).

### 3.4 HPConsistency (most reliable check — operates in both modes)

**Relationship to §3.1**:
- When §3.1 is applied in Mitigation mode, victim HP change is automatically consistent (server deducts) → what §3.4 detects is the scenario where **the client directly manipulates HP memory after the server deduction**
- When §3.1 is inactive in Detection-Only mode, §3.4 also covers §3.1's signals — the mismatch between reported hit sum and HP delta itself reacts to every form of damage reduction / invincibility / HP manipulation



```
Over a time window (5 s):
    sumHits      = Σ HIT_TICK.dmg
    deltaHP      = HP_start - HP_end (server-authoritative)
    expectedHP   = HP_start - sumHits

    if abs(currentHP - expectedHP) > tolerance:
        → HPHack signal
```

**Key point**: the server knows the HP delta (the server performs the actual deduction). If it does not match the reported hit sum, either HP was manipulated directly or HIT_TICK was omitted.

This check is **the strongest**. Combined with the ServerDamageOverride of § 3.1:
- Server deducts HP by the exact damage
- Periodically compares whether the victim's HP report matches the server value
- Mismatch → client HP memory manipulation (HPHack)

---

## 4. Weapon Damage Profiles (example)

```
Katana   (Melee):    min=30, max=50, headMul=1.0, legMul=1.0  (no falloff)
Dagger   (Melee):    min=15, max=25, headMul=1.0, legMul=1.0
Pistol   (Pistol):   min=15, max=25, optRange=1500, maxRange=3000, headMul=2.0, legMul=0.7
Revolver (Pistol):   min=50, max=80, optRange=2000, maxRange=4000, headMul=2.5, legMul=0.7
SMG      (SMG):      min=12, max=20, optRange=1000, maxRange=2500, headMul=1.5, legMul=0.7
Rifle    (Rifle):    min=30, max=50, optRange=2500, maxRange=5000, headMul=2.0, legMul=0.7
Shotgun  (Shotgun):  min=8 per pellet, 6 pellets, optRange=500, maxRange=1500
Rocket   (Rocket):   min=80, max=150 (splash), radius=300, falloff quadratic
```

> If the company's live GunZ has an actual weapon table, replace these with it. These values are estimates measured in the public-tree self-build environment.

---

## 5. Integration Points

### 5.1 New files
- `CSCommon/Security/MDamageValidator.{h,cpp}` new
- `match-server/runtime/weapon_damage_table.xml` (or code registration)

### 5.2 Applying ServerDamage — HIT_TICK handler modification

```cpp
case MC_MATCH_HIT_TICK:
{
    // ... [existing] extract atkUID, weaponType, dmg, srcPos
    // ... [existing] WeaponSpoof, PositionLie validation

    auto wcls = Security::ClassifyMMatchWeapon(weaponType);

    // [INSERT 1] distance computation (based on server-authoritative coordinates)
    float vicXYZ[3]; long long vicTMs;
    pVictim->GetPositionHistory().GetLatest(vicXYZ, &vicTMs);

    float dx = atkSrvXYZ[0] - vicXYZ[0];
    float dy = atkSrvXYZ[1] - vicXYZ[1];
    float dz = atkSrvXYZ[2] - vicXYZ[2];
    float dist = sqrtf(dx*dx + dy*dy + dz*dz);

    // [INSERT 2] BodyPart determination (peer shot info or default Body)
    BodyPart part = BodyPart::Body;  // update when hit location info is extended in the future

    // [INSERT 3] server-authoritative damage computation
    float serverDmg = Security::MDamageValidator::Instance()
        .ComputeServerDamage(weaponType, dist, part);

    // [INSERT 4] compare with reported damage
    auto reportSev = Security::MDamageValidator::Instance()
        .ValidateReportedDamage(weaponType, dist, part, dmg);

    if (reportSev != Security::CheatSeverity::None) {
        Security::CheatSignal sig;
        if (dmg < serverDmg * 0.7f)
            sig.type = Security::CheatSignalType::DamageReduce;
        else if (dmg > serverDmg * 1.3f)
            sig.type = Security::CheatSignalType::DamageInflate;
        sig.severity = reportSev;
        sig.value = dmg;
        sig.threshold = serverDmg;
        snprintf(sig.detail, sizeof(sig.detail),
                 "reported=%.1f server=%.1f wcls=%d dist=%.1f",
                 dmg, serverDmg, (int)wcls, dist);
        Security::MSignalCollector::Instance().Push(
            pVictim->GetUID().High, pVictim->GetUID().Low, sig);
    }

    // [INSERT 5] actual damage deduction uses serverDmg — client report ignored
    pVictim->Damage(serverDmg);  // use serverDmg instead of the existing dmg

    // [INSERT 6] HP consistency tracking
    Security::MDamageValidator::Instance().RecordHit(
        pVictim->GetUID().High, pVictim->GetUID().Low, serverDmg, NowMs());

    // ... [existing] ValidateHit (range, hit-rate)
    break;
}
```

### 5.3 Periodic HP recording

```cpp
// MMatchObject::OnTick (or a 1-second periodic worker)
void MMatchObject::OnTick(long long nowMs) {
    Security::MDamageValidator::Instance().RecordHPSnapshot(
        GetUID().High, GetUID().Low, GetHP(), nowMs);

    auto sev = Security::MDamageValidator::Instance()
        .ValidateHPConsistency(GetUID().High, GetUID().Low);
    if (sev != Security::CheatSeverity::None) {
        Security::CheatSignal sig;
        sig.type = Security::CheatSignalType::HPHack;
        sig.severity = sev;
        Security::MSignalCollector::Instance().Push(
            GetUID().High, GetUID().Low, sig);
    }
}
```

### 5.4 PEER_SHOT × HIT_TICK pairing — DamageNullification

```cpp
// MMatchServer-side PEER_SHOT handler — temporarily register the hit
case MC_PEER_SHOT_TARGETING_HIT:  // (hypothetical — actual GunZ MSGID mapping required)
{
    // attacker reports having hit some victim
    Security::MDamageValidator::Instance().RecordExpectedHit(
        attackerUID, victimUID, weaponType, NowMs());
}

// MDamageValidator::OnHitTick — match expected hit when HIT_TICK arrives
// DamageNullification signal if not matched within 300 ms
```

---

## 6. New Signal Types

```cpp
enum class CheatSignalType : uint16_t {
    // ... [existing]

    // Combat (200~299) — damage family additions
    DamageReduce       = 220,  // victim-reported dmg markedly lower than server computation
    DamageInflate      = 221,  // the opposite — damage-amplification hack (blind spot #2)
    DamageNullification = 222, // PEER_SHOT only, HIT_TICK omitted
    HPHack             = 223,  // HP delta vs reported hit sum mismatch
    HPInconsistency    = 224,  // HP recovered for an unknown reason
};
```

---

## 7. Configuration

| Environment variable | Default | Meaning |
|---------|------|------|
| `DAMAGE_TOLERANCE_LOWER` | 0.7 | reported dmg below server dmg × 0.7 → signal |
| `DAMAGE_TOLERANCE_UPPER` | 1.3 | reported dmg above server dmg × 1.3 → signal |
| `DAMAGE_NULLIFICATION_TIMEOUT_MS` | 300 | HIT_TICK wait after PEER_SHOT |
| `HP_CONSISTENCY_WINDOW_MS` | 5000 | HP consistency validation window |
| `HP_CONSISTENCY_TOLERANCE` | 5.0 | HP absolute tolerance |
| `SERVER_DAMAGE_OVERRIDE` | true | true=deduct by server-computed value, false=use client report (legacy mode) |

---

## 8. Failure Modes

| Condition | Result |
|------|------|
| Weapon profile not registered | `ComputeServerDamage` returns 0 → validation skipped, log only |
| Legitimate damage variance (random spread, falloff) | passes within tolerance 0.7~1.3 |
| Shotgun multiple pellets | assumes a separate HIT_TICK per pellet — or a single report of summed dmg |
| Inaccurate distance measurement (high ping) | falloff computation error → absorbed by tolerance |
| Explosion splash (Rocket) | distance-based falloff curve, hit rejected beyond max radius |
| Legitimate recovery (medkits, skills) | registered separately via `RecordHPRestore(amount, source)` → subtracted in consistency validation |

---

## 9. Test Vectors

```
Normal:
  weapon = Rifle (min=30, max=50, optRange=2500)
  distance = 1500, BodyPart = Body
  serverDmg = ~40
  reportedDmg = 38
  → 38 ∈ [40×0.7, 40×1.3] = [28, 52]   pass

DamageReduce:
  weapon = Rifle, distance = 1500
  serverDmg = ~40
  reportedDmg = 15  (37%)
  → DamageReduce High (15 < 40 × 0.7 = 28, and further below 50%)

HPHack:
  HP_start = 100 (t=1000)
  cumulative HIT_TICK = 60 (t=1000~5000)
  HP_now    = 95  (t=5000)
  expectedHP = 100 - 60 = 40
  → abs(95 - 40) = 55 > 5 (tolerance)
  → HPHack Critical

DamageNullification:
  attacker PEER_SHOT (victim=A, 1000)
  HIT_TICK does not arrive (1300 ms elapsed)
  → DamageNullification High
```

---

## 10. Limitations

- **Critical Dependency — weapon damage table**: this module's validation accuracy is directly proportional to the accuracy of the weapon damage table. The values in §4 are estimates measured in the public-tree self-build environment and must be replaced with the actual weapon balance table for production deployment. A missing or inaccurate table means a false-positive explosion or neutralized validation. **The first step of integration work is the table import**.
- **Enumeration of legitimate recovery/skills/buffs**: HPConsistency validation (§3.4) must have every legitimate HP recovery (medkits, skills, natural regeneration, shield absorption, etc.) registered to avoid false positives. Enumerating every recovery mechanism in the production environment is the first task of §11.3 Step 1.
- **Body-part (head/body/legs) determination** depends on the hit collision system. This package simplifies to the Body default. Accurate hit zone determination requires RealSpace2 engine integration.
- **Shotgun pellet handling** — per-pellet vs summed reporting policy must be decided.
- **Splash damage** (Rocket/Grenade) — automatic computation of victims within radius vs comparison with client report.
- **Status effects such as stun/slow** and their effect on damage — a separate modifier system is required.
- **PvE NPC damage** — this package covers PvP only.

---

## 11. Deployment Modes — Detection-Only vs Mitigation

This module is provided in two separate deployment modes. Because the scope of impact on game mechanics differs, the choice is a deployment decision.

### 11.1 Detection-Only Mode (observation only)

```cpp
// HIT_TICK handler
serverDmg = ComputeServerDamage(...);

if (abs(reportedDmg - serverDmg) > tolerance):
    Push(DamageReduce or DamageInflate signal);

// damage application keeps the existing flow
pVictim->Damage(reportedDmg);   // client-reported dmg as-is
```

| Item | Value |
|------|---|
| Hack effect | **Not neutralized** — the damage-reduction hack still works, but is detected/reported |
| Game mechanics impact | None — existing damage flow unchanged |
| Deployment risk | Low |
| Operational value | Signal accumulation → pattern identification → manual sanction by operators |
| When to use | First deployment stage, measuring false-positive distribution |

### 11.2 Mitigation Mode (effect blocking)

```cpp
// HIT_TICK handler
serverDmg = ComputeServerDamage(...);

if (abs(reportedDmg - serverDmg) > tolerance):
    Push(DamageReduce or DamageInflate signal);

// damage application uses the server-computed value
pVictim->Damage(serverDmg);   // neutralizes the hack effect itself
```

| Item | Value |
|------|---|
| Hack effect | **Neutralized** — the effect of the damage-reduction / invincibility hack itself is voided |
| Game mechanics impact | Damage computation authority moves from client → server |
| Deployment risk | Medium — interaction with legitimate recovery/skills/buffs must be verified |
| Prerequisite work | Weapon table import + registration of recovery/skill mechanics |
| When to use | Enable after 1~2 weeks of Detection-Only operation and verifying zero false positives |

### 11.3 Recommended deployment roadmap

```
Step 1 (1~2 weeks) — Detection-Only active
  ↓
  measure false-positive rate
  → < 0.1 % : proceed to Step 2
  → ≥ 0.1 % : tune tolerance or complete recovery mechanic enumeration

Step 2 (1~2 weeks) — keep Detection-Only + accumulate operator reviews
  ↓
  collect signals from actual hack use, verify zero false positives

Step 3 — Mitigation Mode active
  ↓
  single-line switch via environment variable toggle `SERVER_DAMAGE_OVERRIDE=true`
  immediate rollback possible (revert with `=false`)
```

### 11.4 Environment variable toggle

```
SERVER_DAMAGE_OVERRIDE=false    # default — Detection-Only
SERVER_DAMAGE_OVERRIDE=true     # Mitigation active
```

The toggle takes effect on match server restart or hot-reload (when implemented). Signal emission logic is identical in both modes.

---

## 12. Strengths

- **Seals the core weakness of victim-auth** — handles all three of damage reduction / invincibility / HP manipulation in a single module
- **Staged deployment possible** — Detection-Only secures first-day value, Mitigation is enabled after verification
- **Dual-report cross-check** — attacker PEER_SHOT × victim HIT_TICK consistency → single-channel tampering nullified
- **HP consistency** — the most reliable check, covers even memory-manipulation hacks (operates in both modes)
- **Simple rollback if Mitigation is adopted** — immediate revert via environment variable toggle

---

## 13. Cross-References

- victim-auth design rationale: [`07-combat-validator.md`](./07-combat-validator.md)
- Blind spot addressed: [`../04-anticheat-catalog.md`](../04-anticheat-catalog.md) §2.2 Damage hacks
- Integration point: [`../08-integration-guide.md`](../08-integration-guide.md) §2.8 (HIT_TICK handler extension)
- Operations policy: [`../05-operations-runbook.md`](../05-operations-runbook.md) §2.2 (Combat signals)
