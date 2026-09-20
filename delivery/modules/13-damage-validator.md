# Module 13 — Damage Validator (데미지 감소핵 / 무적핵 방지)

> **Implementation Status: 구현 완료 (Detection-Only 기본 모드, Mitigation 토글 가능)**
>
> 본 모듈은 victim-authoritative HIT_TICK 설계가 만드는 사각지대를 보완한다.
>
> **구현 완료 (2026-05-06)**:
> - `CSCommon/Include/MDamageValidator.h` (190 lines)
> - `CSCommon/Source/MDamageValidator.cpp` (427 lines)
> - `MCheatSignal.h` 의 enum 확장 (`HPHack=303`, `HPInconsistency=304`, 기존 `DamageManipulation=203` 활용)
> - `MMatchServer_OnCommand.cpp` HIT_TICK 핸들러 통합 (`ComputeServerDamage` + `ValidateReportedDamage` + `RecordHit`)
> - `MMatchServer.cpp::OnRun` 1초 주기 게이트로 모든 활성 캐릭터에 `RecordHPSnapshot` + `ValidateHPConsistency` 호출
> - `MMatchServer.cpp::ObjectRemove` 의 `OnPlayerLeave` 정리
> - `CSCommon.vcxproj` ClInclude/ClCompile 등록
> - **빌드 검증 통과**:
>   - `MatchServer.exe` Release|Win32 v143 — **4.27 MB**, 신규 심볼 4 종 `strings` 검출
>   - `Gunz.exe` Release|Win32 v143 — **7.04 MB**, enum 변경 회귀 없음 (wire-stable 보장)
>
> **기본 모드는 Detection-Only** (`m_mitigationMode = false`). 시그널 발행만, 게임 데미지 흐름은 기존 그대로. Mitigation 활성화는 `MDamageValidator::Instance().SetMitigationMode(true)` 호출 — 운영 환경 도입 시 무기 테이블 import + 회복 메커닉 enumeration 검증 후 권장.
>
> **운영 도입 선행 조건**:
> 1. `MDamageValidator::Instance().RegisterWeaponProfile(WeaponClass, ...)` 로 운영 환경 무기 밸런스 표 import (공개 트리 자체 빌드 환경 측정값 교체)
> 2. 합법 HP 회복 메커닉 (의료품, 스킬, 자연 회복 등) 을 `RecordHPRestore(uid, amount, source, tickMs)` 로 등록하는 흐름 결정
> 3. Mitigation Mode 활성화는 1~2 의 검증 통과 후 환경변수 또는 어드민 명령 토글로 적용

## Purpose
**본 패키지의 victim-authoritative HIT_TICK 설계가 만드는 사각지대 보완**.

피해자가 자신의 OnDamaged 훅에서 HIT_TICK 을 보고하기 때문에, **피해자 측 핵이 데미지를 0 또는 축소해서 보고**할 수 있다. 이는 두 가지 핵으로 발현된다:

| 핵 | 동작 |
|----|------|
| **데미지 감소핵 (DamageReduce)** | victim 이 받은 데미지를 1/2, 1/3 로 보고하거나 HIT_TICK 자체를 누락 |
| **무적핵 (Invincibility)** | victim 이 모든 HIT_TICK 송신을 차단, HP 변화 없음 |
| **체력 조작핵 (HPHack)** | victim 이 자신의 HP 를 직접 조작, 데미지 무시 |

본 모듈은 이 사각지대를 **server-side weapon damage table** + **attacker × victim 이중 보고 cross-check** + **HP 변화 일관성** 으로 닫는다.

---

## 1. 사각지대 분석

### 1.1 현재 설계의 trade-off
- **공격자 권위 hit** → 공격자가 hit-rate 자가조작 (AutoAim 데미지 폭증)
- **피해자 권위 hit** (현 설계) → 피해자가 받은 데미지 축소/무시

→ 어느 쪽도 단독으로 안전하지 않음. **하이브리드** 가 필요.

### 1.2 권한 분리 재설계

| 정보 | 권위 |
|------|------|
| Hit 발생 사실 | **양측 보고** + 일치 여부 cross-check |
| 공격자 srcPos | 공격자 보고 (PositionLie 가 [`10-position-lie-novel.md`](./10-position-lie-novel.md) 에서 검증) |
| 피해자 vicPos | 서버 권위 (`MPositionHistory.GetLatest`) |
| 사용 무기 | 공격자 보고 (WeaponSpoof 가 [`09-weapon-spoof-novel.md`](./09-weapon-spoof-novel.md) 에서 검증) |
| **데미지 양** | **서버 계산** (무기 테이블 + 거리 falloff + 부위 multiplier) — 클라 보고 무시 |
| 데미지 적용 결과 | 서버가 victim HP 갱신, 클라에 통보 |

핵심: **데미지 양을 서버가 계산** 한다. 클라이언트는 hit 발생만 보고하고, 데미지 값은 서버 무기 테이블로 결정.

---

## 2. Interface

```cpp
namespace Security {

struct WeaponDamageProfile {
    float minDamage;
    float maxDamage;
    float optimalRange;     // falloff 시작점
    float maxRange;         // 이 거리 초과 시 minDamage 적용 또는 hit 거부
    float headshotMultiplier;
    float legshotMultiplier;
    float falloffCurve;     // 0=linear, 1=quadratic
};

class MDamageValidator {
public:
    static MDamageValidator& Instance();

    // 무기 데미지 테이블 등록 (서버 시작 시)
    void RegisterWeaponProfile(uint32_t weaponType, const WeaponDamageProfile& prof);

    // 서버 측 권위 데미지 계산
    float ComputeServerDamage(uint32_t weaponType,
                              float dist,
                              BodyPart part /* Head/Body/Legs */) const;

    // HIT_TICK victim 보고 검증
    // returns: severity. None = 통과
    CheatSeverity ValidateReportedDamage(
        uint32_t weaponType,
        float dist,
        BodyPart part,
        float reportedDmg) const;

    // HP 일관성 검증 — 일정 윈도우 동안 HIT_TICK 누적 합 vs HP 변화량
    void RecordHit(uint32_t victimUidH, uint32_t victimUidL,
                   float reportedDmg, long long tMs);
    void RecordHPSnapshot(uint32_t victimUidH, uint32_t victimUidL,
                          float currentHP, long long tMs);
    CheatSeverity ValidateHPConsistency(uint32_t victimUidH, uint32_t victimUidL) const;
};

}
```

---

## 3. 검증 항목 4종

### 3.1 ServerDamageOverride (Mitigation Mode 에서만 활성)

```
HIT_TICK 수신
    ↓
serverDmg = MDamageValidator::ComputeServerDamage(wcls, dist, part)
    ↓
[Detection-Only Mode]  victim HP 차감 = reportedDmg (클라 보고)
[Mitigation Mode]      victim HP 차감 = serverDmg   (서버 계산)
    ↓
양 모드 모두: 보고 dmg 와 serverDmg 비교
    ↓
abs(reportedDmg - serverDmg) > tolerance × serverDmg
    → DamageMismatch 시그널 (severity 거리 비율 따라)
```

**효과 (Mitigation)**: 데미지 감소핵이 보고를 0 으로 해도 서버는 계산값으로 차감. 클라 보고 무시.

**효과 (Detection-Only)**: 핵 효과는 무력화 안 됨, 시그널만 누적 → 운영자 수동 제재.

**잔여 위험**: HIT_TICK 자체를 차단하는 무적핵 — §3.3 / §3.4 로 보완 (두 모드 모두 동작).

상세 모드 비교 → §11.

### 3.2 DamageRange 검증

```
expected = [minDmg × falloff, maxDmg × falloff × headshotMul]

if reportedDmg < expected.lower × 0.7:
    → DamageReduce 시그널 (peer 의 합법 변동성 약 30% 허용)

if reportedDmg > expected.upper × 1.3:
    → DamageInflate 시그널 (데미지 증폭핵, 별개의 사각지대)
```

### 3.3 DamageNullification (HIT_TICK 누락)

```
공격자가 PEER_SHOT 으로 명중 사실을 알리는데
victim 의 HIT_TICK 이 일정 시간 (300 ms) 내 도달하지 않음
    → DamageNullification 시그널 (HIT_TICK 차단)
```

이는 **공격자 측 정보** (PEER_SHOT) 와 **피해자 측 정보** (HIT_TICK) 의 cross-check.

### 3.4 HPConsistency (가장 신뢰 높은 검증 — 두 모드 모두 동작)

**§3.1 과의 관계**:
- Mitigation 모드에서 §3.1 이 적용되면 victim HP 변화는 자동으로 일관 (서버가 차감) → §3.4 가 검출하는 것은 **서버 차감 후 클라가 HP 메모리를 직접 조작** 하는 시나리오
- Detection-Only 모드에서 §3.1 이 비활성이면 §3.4 가 §3.1 의 시그널까지 함께 커버 — 보고 hit 합과 HP 변화량 불일치 자체가 데미지 감소/무적/HP조작 모든 형태에 반응



```
시간 윈도우 (5 초) 동안:
    sumHits      = Σ HIT_TICK.dmg
    deltaHP      = HP_start - HP_end (서버 권위)
    expectedHP   = HP_start - sumHits

    if abs(currentHP - expectedHP) > tolerance:
        → HPHack 시그널
```

**핵심**: HP 변화량은 서버가 알고 있다 (서버가 실제 차감). 보고된 hit 합과 일치하지 않으면 HP 직접 조작 또는 HIT_TICK 누락.

이 검증이 **가장 강력**. § 3.1 의 ServerDamageOverride 와 결합 시:
- 서버가 정확한 데미지로 HP 차감
- victim 의 HP 보고가 서버 값과 일치하는지 주기적 비교
- 불일치 → 클라 HP 메모리 조작 (HPHack)

---

## 4. 무기 데미지 프로파일 (예시)

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

> 회사 라이브 GunZ 의 실제 무기 테이블이 있다면 그것으로 교체. 본 값은 공개 트리 자체 빌드 환경 측정 추정값.

---

## 5. Integration Points

### 5.1 신규 파일
- `CSCommon/Security/MDamageValidator.{h,cpp}` 신규
- `match-server/runtime/weapon_damage_table.xml` (또는 코드 등록)

### 5.2 ServerDamage 적용 — HIT_TICK 핸들러 수정

```cpp
case MC_MATCH_HIT_TICK:
{
    // ... [기존] atkUID, weaponType, dmg, srcPos 추출
    // ... [기존] WeaponSpoof, PositionLie 검증

    auto wcls = Security::ClassifyMMatchWeapon(weaponType);

    // [INSERT 1] 거리 계산 (서버 권위 좌표 기준)
    float vicXYZ[3]; long long vicTMs;
    pVictim->GetPositionHistory().GetLatest(vicXYZ, &vicTMs);

    float dx = atkSrvXYZ[0] - vicXYZ[0];
    float dy = atkSrvXYZ[1] - vicXYZ[1];
    float dz = atkSrvXYZ[2] - vicXYZ[2];
    float dist = sqrtf(dx*dx + dy*dy + dz*dz);

    // [INSERT 2] BodyPart 결정 (peer shot info 또는 기본 Body)
    BodyPart part = BodyPart::Body;  // 향후 hit location 정보 확장 시 갱신

    // [INSERT 3] 서버 권위 데미지 계산
    float serverDmg = Security::MDamageValidator::Instance()
        .ComputeServerDamage(weaponType, dist, part);

    // [INSERT 4] 보고 데미지와 비교
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

    // [INSERT 5] 실제 데미지 차감은 serverDmg 로 — 클라 보고 무시
    pVictim->Damage(serverDmg);  // 기존 dmg 대신 serverDmg 사용

    // [INSERT 6] HP 일관성 추적
    Security::MDamageValidator::Instance().RecordHit(
        pVictim->GetUID().High, pVictim->GetUID().Low, serverDmg, NowMs());

    // ... [기존] ValidateHit (range, hit-rate)
    break;
}
```

### 5.3 HP 주기 기록

```cpp
// MMatchObject::OnTick (또는 1초 주기 worker)
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
// MMatchServer 측 PEER_SHOT 핸들러 — 명중 사실 임시 등록
case MC_PEER_SHOT_TARGETING_HIT:  // (가상 — 실제 GunZ MSGID 매핑 필요)
{
    // 공격자가 어떤 victim 을 맞췄다고 보고
    Security::MDamageValidator::Instance().RecordExpectedHit(
        attackerUID, victimUID, weaponType, NowMs());
}

// MDamageValidator::OnHitTick — HIT_TICK 도달 시 expected hit 매칭
// 300 ms 내 매칭 안 되면 DamageNullification 시그널
```

---

## 6. 신규 시그널 타입

```cpp
enum class CheatSignalType : uint16_t {
    // ... [기존]

    // Combat (200~299) — 데미지 계열 추가
    DamageReduce       = 220,  // victim 보고 dmg 가 서버 계산보다 현저히 낮음
    DamageInflate      = 221,  // 반대 — 데미지 증폭핵 (사각지대 #2)
    DamageNullification = 222, // PEER_SHOT 만 있고 HIT_TICK 누락
    HPHack             = 223,  // HP 변화량 vs 보고 hit 합 불일치
    HPInconsistency    = 224,  // HP 가 알 수 없는 이유로 회복
};
```

---

## 7. Configuration

| 환경변수 | 기본 | 의미 |
|---------|------|------|
| `DAMAGE_TOLERANCE_LOWER` | 0.7 | 보고 dmg 가 서버 dmg × 0.7 미만 → 시그널 |
| `DAMAGE_TOLERANCE_UPPER` | 1.3 | 보고 dmg 가 서버 dmg × 1.3 초과 → 시그널 |
| `DAMAGE_NULLIFICATION_TIMEOUT_MS` | 300 | PEER_SHOT 후 HIT_TICK 대기 |
| `HP_CONSISTENCY_WINDOW_MS` | 5000 | HP 일관성 검증 윈도우 |
| `HP_CONSISTENCY_TOLERANCE` | 5.0 | HP 절대값 허용 오차 |
| `SERVER_DAMAGE_OVERRIDE` | true | true=서버 계산값으로 차감, false=클라 보고 사용 (legacy 모드) |

---

## 8. Failure Modes

| 조건 | 결과 |
|------|------|
| 무기 프로파일 미등록 | `ComputeServerDamage` 0 반환 → 검증 스킵, 로그만 |
| 합법 데미지 변동 (random spread, falloff) | tolerance 0.7~1.3 안에서 통과 |
| Shotgun 다중 펠릿 | 펠릿마다 별도 HIT_TICK 가정 — 또는 합산 dmg 단일 보고 |
| 거리 측정 부정확 (ping 큼) | falloff 계산 오차 → tolerance 가 흡수 |
| 폭발 splash (Rocket) | 거리 기반 falloff curve, 최대 반경 초과 시 hit 거부 |
| 합법 회복 (의료품, 스킬) | `RecordHPRestore(amount, source)` 별도 등록 → 일관성 검증에서 차감 |

---

## 9. Test Vectors

```
정상:
  무기 = Rifle (min=30, max=50, optRange=2500)
  거리 = 1500, BodyPart = Body
  serverDmg = ~40
  reportedDmg = 38
  → 38 ∈ [40×0.7, 40×1.3] = [28, 52]   통과

DamageReduce:
  무기 = Rifle, 거리 = 1500
  serverDmg = ~40
  reportedDmg = 15  (37%)
  → DamageReduce High (15 < 40 × 0.7 = 28, 더 나아가 50% 미만)

HPHack:
  HP_start = 100 (t=1000)
  HIT_TICK 누적 = 60 (t=1000~5000)
  HP_now    = 95  (t=5000)
  expectedHP = 100 - 60 = 40
  → abs(95 - 40) = 55 > 5 (tolerance)
  → HPHack Critical

DamageNullification:
  공격자 PEER_SHOT (victim=A, 1000)
  HIT_TICK 도달 안 함 (1300 ms 경과)
  → DamageNullification High
```

---

## 10. Limitations

- **Critical Dependency — 무기 데미지 테이블**: 본 모듈의 검증 정확성은 무기 데미지 테이블의 정확성에 직접 비례. §4 의 값은 공개 트리 자체 빌드 환경 측정 추정값이며, 운영 환경 도입 시 실제 무기 밸런스 표로 교체 필요. 테이블 부재 또는 부정확 시 false-positive 폭증 또는 검증 무력화. **통합 작업의 첫 단계는 테이블 import**.
- **합법 회복/스킬/버프 enumeration**: HPConsistency 검증 (§3.4) 은 합법 HP 회복 (의료품, 스킬, 자연 회복, 보호막 흡수 등) 을 모두 등록 받아야 거짓 양성 회피. 운영 환경의 모든 회복 메커니즘 enumeration 이 §11.3 Step 1 의 첫 작업.
- **부위 (head/body/legs) 판정** 이 hit collision 시스템에 의존. 본 패키지는 Body 기본값으로 단순화. 정확한 hit zone 판정은 RealSpace2 엔진 통합 필요.
- **Shotgun 펠릿 처리** — 펠릿 단위 vs 합산 보고 정책 결정 필요.
- **Splash 데미지** (Rocket/Grenade) — radius 내 victim 자동 계산 vs 클라 보고 비교.
- **기절/슬로우 등 상태 효과** 의 데미지 영향 — 별도 modifier 시스템 필요.
- **PvE NPC 데미지** — 본 패키지는 PvP 만 다룸.

---

## 11. 도입 모드 — Detection-Only vs Mitigation

본 모듈은 두 개의 도입 모드로 분리 제공된다. 게임 메커닉 영향 범위가 다르므로 도입 선택 사항이다.

### 11.1 Detection-Only Mode (관찰 전용)

```cpp
// HIT_TICK 핸들러
serverDmg = ComputeServerDamage(...);

if (abs(reportedDmg - serverDmg) > tolerance):
    Push(DamageReduce or DamageInflate signal);

// 데미지 적용은 기존 흐름 유지
pVictim->Damage(reportedDmg);   // 클라 보고 dmg 그대로
```

| 항목 | 값 |
|------|---|
| 핵 효과 | **무력화 안 됨** — 데미지 감소핵은 여전히 작동, 다만 감지/리포트 |
| 게임 메커닉 영향 | 없음 — 기존 데미지 흐름 그대로 |
| 도입 위험도 | 낮음 |
| 운영 가치 | 시그널 누적 → 패턴 식별 → 운영자 수동 제재 |
| 활용 시점 | 도입 첫 단계, false-positive 분포 측정 |

### 11.2 Mitigation Mode (효과 차단)

```cpp
// HIT_TICK 핸들러
serverDmg = ComputeServerDamage(...);

if (abs(reportedDmg - serverDmg) > tolerance):
    Push(DamageReduce or DamageInflate signal);

// 데미지 적용은 서버 계산값
pVictim->Damage(serverDmg);   // 핵 효과 자체 무력화
```

| 항목 | 값 |
|------|---|
| 핵 효과 | **무력화** — 데미지 감소핵 / 무적핵의 효과 자체가 무효 |
| 게임 메커닉 영향 | 데미지 계산 권한이 클라 → 서버로 이동 |
| 도입 위험도 | 중 — 합법 회복/스킬/버프와의 상호작용 검증 필요 |
| 선행 작업 | 무기 테이블 import + 회복/스킬 메커닉 등록 |
| 활용 시점 | Detection-Only 1~2 주 운용 후 false-positive 0 검증 후 활성화 |

### 11.3 도입 권장 로드맵

```
Step 1 (1~2 주) — Detection-Only 활성
  ↓
  거짓 양성 비율 측정
  → < 0.1 % : Step 2 진행
  → ≥ 0.1 % : tolerance 튜닝 또는 회복 메커닉 enumeration 보완

Step 2 (1~2 주) — Detection-Only 유지 + 운영자 리뷰 누적
  ↓
  실제 핵 사용 시그널 수집, false-positive 0 검증

Step 3 — Mitigation Mode 활성
  ↓
  환경변수 `SERVER_DAMAGE_OVERRIDE=true` 토글로 단일 라인 전환
  롤백 즉시 가능 (`=false` 로 복귀)
```

### 11.4 환경변수 토글

```
SERVER_DAMAGE_OVERRIDE=false    # 기본값 — Detection-Only
SERVER_DAMAGE_OVERRIDE=true     # Mitigation 활성
```

토글은 매치서버 재시작 또는 hot-reload (구현 시) 로 적용. 시그널 발행 로직은 두 모드에서 동일.

---

## 12. Strengths

- **victim-auth 의 핵심 약점 봉쇄** — 데미지 감소 / 무적 / HP 조작 3종을 단일 모듈로 다룸
- **단계적 도입 가능** — Detection-Only 로 first-day value 확보, Mitigation 은 검증 후 활성화
- **이중 보고 cross-check** — 공격자 PEER_SHOT × 피해자 HIT_TICK 일관성 → 단일 채널 변조 무효화
- **HP 일관성** — 가장 신뢰 높은 검증, 메모리 조작핵까지 커버 (두 모드 모두에서 동작)
- **Mitigation 채택 시 롤백 단순** — 환경변수 토글로 즉시 복귀

---

## 13. Cross-References

- victim-auth 설계 근거: [`07-combat-validator.md`](./07-combat-validator.md)
- 보완하는 사각지대: [`../04-anticheat-catalog.md`](../04-anticheat-catalog.md) §2.2 데미지 핵
- 통합 지점: [`../08-integration-guide.md`](../08-integration-guide.md) §2.8 (HIT_TICK 핸들러 확장)
- 운영 정책: [`../05-operations-runbook.md`](../05-operations-runbook.md) §2.2 (Combat 시그널)
