# Integration Guide

> **독자**: 통합 작업 엔지니어
> **목적**: 본 패키지 모듈을 회사 보유 트리에 통합할 때의 **삽입 위치 / 호출 시그니처 / 의사코드** 명세.
> **원칙**: 본 가이드는 회사 측 소스 파일을 직접 패치 형태로 제공하지 않는다. 회사 트리에서 동등 위치를 찾아 의사코드를 적용한다.

---

## 1. 통합 작업 흐름 (전체)

```
1. 신규 모듈 소스 통합 (modules/ 의 Layer A)
   ├─ CSCommon/ 또는 동등 공유 라이브러리에 모듈 추가
   ├─ libsodium 의존성 추가
   └─ 빌드 검증

2. Layer B 패치 사이트 통합 (본 가이드)
   ├─ 매치서버 측 IOCP 콜백
   ├─ 매치서버 측 OnCommand 핸들러
   ├─ 매치서버 측 스테이지 라이프사이클
   ├─ 클라이언트 측 SendCommand 분기
   └─ 클라이언트 측 PostShot/PostHit 매크로

3. 신규 패킷 ID 등록
   ├─ MSGID_COMMAND_V2 = 102
   └─ MC_MATCH_ECDHE_CHALLENGE..HIT_TICK = 2901..2905

4. 환경변수 / 설정 추가
   ├─ COMMUNITY_API_URL
   ├─ MATCH_RESULT_WEBHOOK_SECRET
   └─ DDoS / 안티치트 임계값 (선택)

5. community-api / DB 인프라 (Phase F)
```

---

## 2. 매치서버 통합 지점

### 2.1 IOCP Accept 콜백 — DDoS L1

**위치**: `MServer::RCPCallback` 의 `RCP_IO_ACCEPT` 핸들러

**삽입 시점**: MCommObject 할당 *직전*. 게이트 통과 못 하면 alloc 부담 0.

```cpp
case RCP_IO_ACCEPT:
{
    // [BEFORE] 기존 코드: szIP 추출 직후
    char szIP[16];
    GetClientIP(nKey, szIP);

    // [INSERT] DDoS L1 게이트
    if (!Security::MConnectRateLimit::Instance().CheckAndRecord(szIP)) {
        m_RealCPNet.Disconnect(nKey);
        return;
    }

    // [AFTER] 기존 MCommObject alloc + InitCryptCommObject
    ...
}
```

**호출 시그니처**: `bool CheckAndRecord(const char* szIP)` — true=통과, false=차단.

### 2.2 IOCP Read 콜백 — DDoS L2 / L3

**위치**: `MServer::RCPCallback` 의 `RCP_IO_READ` 핸들러

**삽입 시점**: `pCmdBuilder->Read` 호출 *직전*.

```cpp
case RCP_IO_READ:
{
    // [BEFORE] LockCommList + pCommObj 획득 직후
    LockCommList();
    MCommObject* pCommObj = GetCommObject(nKey);
    if (!pCommObj) { UnlockCommList(); return; }

    // [INSERT 1] DDoS L2 — packet rate
    if (!pCommObj->GetPacketRate().RecordAndCheck()) {
        pCommObj->SetAllowed(false);
        m_RealCPNet.Disconnect(nKey, true);
        UnlockCommList();
        return;
    }

    // [INSERT 2] DDoS L3 — bandwidth
    if (!pCommObj->GetBandwidth().RecordAndCheck(dwPacketLen)) {
        pCommObj->SetAllowed(false);
        m_RealCPNet.Disconnect(nKey, true);
        UnlockCommList();
        return;
    }

    // [AFTER] 기존 pCmdBuilder->Read 흐름
    ...
}
```

**호출 시그니처**:
- `bool MPacketRateLimit::RecordAndCheck()` — 패킷 1 개 카운트, true=통과
- `bool MBandwidthThrottle::RecordAndCheck(size_t bytes)` — 바이트 누적, true=통과

### 2.3 ECDHE 핸드셰이크 시작

**위치**: `MServer::InitCryptCommObject` 또는 동등 함수 (accept 직후)

```cpp
void MServer::InitCryptCommObject(MCommObject* pCommObj) {
    // [BEFORE] 기존 v1 seed key 생성
    pCommObj->GetCrypter()->Init(MMakeSeedKey(...));

    // [INSERT] ECDHE 키쌍 생성 + Challenge 송신
    auto* kxState = new Security::KeyExchangeState();
    kxState->ServerGenerateKeyPair();
    pCommObj->SetKxState(kxState);

    MCommand* pChallenge = new MCommand(...);
    pChallenge->SetID(MC_MATCH_ECDHE_CHALLENGE);
    pChallenge->AddParameter(new MCmdParamBlob(kxState->GetServerPublic(), 32));
    SendCommand(pCommObj, pChallenge);  // v1 으로 송신
    delete pChallenge;

    // [AFTER] 기존 ReplyConnect
    ...
}
```

### 2.4 ECDHE Response 수신

**위치**: `MMatchServer::OnCommand` 의 신규 case

```cpp
case MC_MATCH_ECDHE_RESPONSE:
{
    void* pBlob; int nLen;
    pCommand->GetParameter(&pBlob, 0, MPT_BLOB);

    auto* kxState = pCommObj->GetKxState();
    if (!kxState->ServerDeriveSessionKeys((BYTE*)pBlob)) {
        // 핸드셰이크 실패, 세션 종료
        Disconnect(uid);
        return;
    }

    BYTE symKey[32];
    Security::DeriveSymmetricKey(kxState, symKey);

    pCommObj->GetCrypterV2()->InitKey(symKey, 32);
    pCommObj->GetCommandBuilder()->InitCryptV2(pCommObj->GetCrypterV2());
    pCommObj->SetV2Active(true);

    // 임시 비밀키 즉시 소거
    kxState->WipePrivateKey();
    break;
}
```

### 2.5 IP 마스킹 — Layer B 가장 단순한 패치

**위치 1**: `MMatchServer::ResponsePeerList` (peer blob 빌드 부분)

```cpp
// [BEFORE]
pBlob->dwIP = pObj->GetIP();
pBlob->nPort = pObj->GetPort();

// [AFTER]
pBlob->dwIP = 0;
pBlob->nPort = 0;
```

**위치 2**: `MMatchServer::StageEnterBattle` 의 동일 패턴.

**기존 분기 제거**: admin / eventTeam / forcedNAT 조건부 마스킹 분기 전부 제거. **무조건** 마스킹.

### 2.6 안티치트 시그널 핸들러 — POSITION_TICK

**위치**: `MMatchServer::OnCommand` 의 신규 case

```cpp
case MC_MATCH_POSITION_TICK:
{
    void* pBlob; int nLen;
    pCommand->GetParameter(&pBlob, 0, MPT_BLOB);

    // ZPACKEDBASICINFO 의 첫 4B fTime + 다음 6B (short posX/Y/Z) 만 사용
    short sPosX = ((short*)pBlob)[2];
    short sPosY = ((short*)pBlob)[3];
    short sPosZ = ((short*)pBlob)[4];

    auto nowMs = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now().time_since_epoch()).count();

    if (!pObj->CheckAlive()) break;

    pObj->GetPositionHistory().Record((float)sPosX, (float)sPosY, (float)sPosZ, nowMs);

    // Validator 호출
    Security::MMovementValidator::Instance().Validate(
        sid.High, sid.Low, (float)sPosX, (float)sPosY, (float)sPosZ, nowMs);
    break;
}
```

### 2.7 안티치트 시그널 핸들러 — ATTACK_TICK

**위치**: `MMatchServer::OnCommand` 의 신규 case

```cpp
case MC_MATCH_ATTACK_TICK:
{
    BYTE weaponType;
    UINT clientTickMs;
    pCommand->GetParameter(&weaponType, 0, MPT_UCHAR);
    pCommand->GetParameter(&clientTickMs, 1, MPT_UINT);

    if (!pObj->CheckAlive()) break;

    auto wcls = Security::ClassifyMMatchWeapon(weaponType);
    if (wcls == Security::WeaponClass::Unknown) break;

    auto nowMs = NowMs();

    // (a) ATTACK timing validation
    Security::MCombatValidator::Instance().ValidateAttack(
        sid.High, sid.Low, wcls, nowMs);

    // (b) WeaponSpoof inline check (Novel 축)
    auto& items = pObj->GetCharInfo()->m_EquipedItem;
    bool match = false;
    for (auto slot : {MMCIP_MELEE, MMCIP_PRIMARY, MMCIP_SECONDARY}) {
        auto* item = items.GetItem(slot);
        if (item && Security::ClassifyMMatchWeapon(item->GetType()) == wcls) {
            match = true; break;
        }
    }
    if (!match && pObj->IsEquipmentSeen()) {
        Security::CheatSignal sig;
        sig.type = Security::CheatSignalType::WeaponSpoof;
        sig.severity = Security::CheatSeverity::High;
        sig.value = (float)weaponType;
        sig.threshold = 0;
        snprintf(sig.detail, sizeof(sig.detail), "reported=%d slots=...", weaponType);
        Security::MSignalCollector::Instance().Push(sid.High, sid.Low, sig);
    }
    break;
}
```

### 2.8 안티치트 시그널 핸들러 — HIT_TICK

**중요**: HIT_TICK 핸들러는 다음 검증을 **순서대로** 수행한다 — (1) PositionLie 크로스체크 (Novel #10), (2) Damage 검증 + Server-Side Override (Module 13), (3) ValidateHit (Module 07). Damage Validator 가 적용되면 victim HP 차감은 **클라 보고 dmg 가 아니라 서버 계산 serverDmg** 로 이루어진다 — victim-auth 사각지대 (데미지 감소 / 무적 / HP 조작) 봉쇄.

**위치**: `MMatchServer::OnCommand` 의 신규 case

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
    if (!pAttacker || !pObj /*victim*/->CheckAlive()) break;

    // 피해자 위치는 서버 링버퍼 (클라 위치 조작 봉쇄)
    float vicXYZ[3];
    long long vicTMs;
    if (!pObj->GetPositionHistory().GetLatest(vicXYZ, &vicTMs)) break;

    auto wcls = Security::ClassifyMMatchWeapon(weaponType);

    // (a) PositionLie 크로스체크 (Novel 축)
    float atkSrvXYZ[3]; long long atkSrvTMs;
    if (pAttacker->GetPositionHistory().GetLatest(atkSrvXYZ, &atkSrvTMs)) {
        float dx = srcPos[0] - atkSrvXYZ[0];
        float dy = srcPos[1] - atkSrvXYZ[1];
        float dz = srcPos[2] - atkSrvXYZ[2];
        float dist = sqrtf(dx*dx + dy*dy + dz*dz);

        float threshold = Security::PositionLieThreshold(wcls);  // Melee 400 ~ Rocket 1000
        if (dist > threshold) {
            Security::CheatSignal sig;
            sig.type = Security::CheatSignalType::PacketManipulation;
            sig.severity = (dist > threshold * 2)
                ? Security::CheatSeverity::High
                : Security::CheatSeverity::Medium;
            sig.value = dist;
            sig.threshold = threshold;
            Security::MSignalCollector::Instance().Push(
                pAttacker->GetUID().High, pAttacker->GetUID().Low, sig);
        }
    }

    // (b) Server-Side Damage Override + 데미지 감소핵 검증 (Module 13)
    float dx = atkSrvXYZ[0] - vicXYZ[0];
    float dy = atkSrvXYZ[1] - vicXYZ[1];
    float dz = atkSrvXYZ[2] - vicXYZ[2];
    float dist = sqrtf(dx*dx + dy*dy + dz*dz);
    Security::BodyPart part = Security::BodyPart::Body;  // 향후 hit zone 정밀화

    float serverDmg = Security::MDamageValidator::Instance()
        .ComputeServerDamage(weaponType, dist, part);

    auto dmgSev = Security::MDamageValidator::Instance()
        .ValidateReportedDamage(weaponType, dist, part, dmg);
    if (dmgSev != Security::CheatSeverity::None) {
        Security::CheatSignal sig;
        sig.type = (dmg < serverDmg * 0.7f)
            ? Security::CheatSignalType::DamageReduce
            : Security::CheatSignalType::DamageInflate;
        sig.severity = dmgSev;
        sig.value = dmg;
        sig.threshold = serverDmg;
        Security::MSignalCollector::Instance().Push(
            pVictim->GetUID().High, pVictim->GetUID().Low, sig);
    }

    // 실제 victim HP 차감 — 모드 분기 (Module 13 §11)
    bool mitigation = Security::MDamageValidator::Instance().IsMitigationMode();
    if (mitigation) {
        pVictim->Damage(serverDmg);   // Mitigation Mode: 핵 효과 자체 무력화
    } else {
        pVictim->Damage(dmg);          // Detection-Only Mode: 시그널만, 기존 흐름 유지
    }

    // HP 일관성 추적
    Security::MDamageValidator::Instance().RecordHit(
        pVictim->GetUID().High, pVictim->GetUID().Low, serverDmg, NowMs());

    // (c) HIT validation (range / hit-rate)
    Security::MCombatValidator::Instance().ValidateHit(
        pAttacker->GetUID().High, pAttacker->GetUID().Low,
        wcls, srcPos, vicXYZ, serverDmg);
    break;
}
```

### 2.9 스테이지 라이프사이클 — match_uuid

**위치**: `MMatchServer::OnStageStart` 의 `pStage->StartGame() == true` 분기

```cpp
if (pStage->StartGame() == true) {
    // [INSERT] match_uuid 발급 + 모든 플레이어에 주입
    pStage->GenerateMatchUUID();
    const char* szUUID = pStage->GetMatchUUID();

    auto end = pStage->GetObjEnd();
    for (auto it = pStage->GetObjBegin(); it != end; ++it) {
        MUID uid = (*it)->GetUID();
        Security::MSignalCollector::Instance().SetMatchUUID(uid.High, uid.Low, szUUID);
    }
}
```

**위치 2**: `MMatchServer::StageFinishGame`

```cpp
auto end = pStage->GetObjEnd();
for (auto it = pStage->GetObjBegin(); it != end; ++it) {
    MUID uid = (*it)->GetUID();
    Security::MSignalCollector::Instance().ClearMatchUUID(uid.High, uid.Low);
}
pStage->ClearMatchUUID();
```

### 2.9b HPConsistency 주기 호출 (Module 13)

**위치**: `MMatchServer::OnRun` 의 `for(MMatchObjectList::iterator ...)` 객체 순회 루프 안

```cpp
// 함수 시작 부 — 1초 게이트
static unsigned long int s_lastHPCheckMs = 0;
const bool runHPCheck = (nGlobalClock - s_lastHPCheckMs > 1000);
if (runHPCheck) s_lastHPCheckMs = nGlobalClock;

// 객체 순회 안
if (pObj->GetCharInfo()) {
    // ... [기존] DBCachingData 처리

    // [INSERT] HPConsistency 검증 — 활성 캐릭터만
    if (runHPCheck && pObj->GetCharInfo()->m_nHP > 0) {
        const MUID& uid = pObj->GetUID();
        Security::MDamageValidator::Instance().RecordHPSnapshot(
            uid.High, uid.Low,
            (float)pObj->GetCharInfo()->m_nHP,
            (uint64_t)nGlobalClock);
        Security::MDamageValidator::Instance().ValidateHPConsistency(
            uid.High, uid.Low, (uint64_t)nGlobalClock);
    }
}
```

**왜 1Hz 인가**: HPConsistency 의 윈도우가 5초 (`HP_CONSISTENCY_WINDOW_MS=5000`) 라 1Hz 면 평균 5개 샘플 누적. 더 빠른 주기는 CPU 비용만 증가하고 검출력 향상 미미. 더 느린 주기는 윈도우 부족 가드 (`hpSamples.size() < 2 || dt < window/2`) 에 걸려 검증 스킵.

**HP 0 (사망) 가드**: 사망 시점에 윈도우 종료. 다음 라이프부터 새 윈도우 누적. 리스폰 후 첫 샘플은 baseline.

### 2.10 ObjectRemove — 상태 누수 방지

**위치**: `MMatchServer::ObjectRemove` 의 player 제거 분기

```cpp
void MMatchServer::ObjectRemove(MUID& uid) {
    // [INSERT] per-player state cleanup
    Security::MMovementValidator::Instance().OnPlayerLeave(uid.High, uid.Low);
    Security::MCombatValidator::Instance().OnPlayerLeave(uid.High, uid.Low);
    Security::MSignalCollector::Instance().OnPlayerLeave(uid.High, uid.Low);

    // [AFTER] 기존 제거 흐름
    ...
}
```

### 2.11 MMatchServer::OnCreate — API 클라이언트 초기화

```cpp
bool MMatchServer::OnCreate() {
    // [INSERT] community-api 클라이언트 초기화
    const char* apiUrl = std::getenv("COMMUNITY_API_URL");
    const char* secret = std::getenv("MATCH_RESULT_WEBHOOK_SECRET");
    if (apiUrl && secret && apiUrl[0] && secret[0]) {
        Security::MAPIClient::Instance().Init(apiUrl, secret);

        Security::MSignalCollector::Instance().SetReportCallback(
            [](const std::vector<Security::CheatSignal>& batch) {
                std::vector<Security::CheatReport> reports;
                for (const auto& s : batch) {
                    Security::CheatReport r;
                    r.player_uid_high = s.uidHigh;
                    r.player_uid_low = s.uidLow;
                    r.signal_type = Security::SignalTypeToString(s.type);
                    r.severity = Security::SeverityToString(s.severity);
                    r.value = s.value;
                    r.threshold = s.threshold;
                    strncpy(r.match_uuid, s.matchUUID, sizeof(r.match_uuid));
                    strncpy(r.detail, s.detail, sizeof(r.detail));
                    r.timestamp = s.timestamp;
                    reports.push_back(r);
                }
                Security::MAPIClient::Instance().SendCheatReportsAsync(reports);
            });
    }
    // env 미설정 시 callback 미등록 → 메모리-only 동작 (로컬 개발 시 서버 기동 가능)

    // [AFTER] 기존 OnCreate 흐름
    ...
}
```

---

## 3. 클라이언트 통합 지점

### 3.1 SendCommand 분기 — v2 활성화 시

**위치**: `MClient::SendCommand` 또는 동등 함수

```cpp
void MClient::SendCommand(MCommand* pCommand) {
    // [INSERT] V2 활성화 여부 분기
    if (IsV2Active()) {
        // 버퍼를 GZ_V2_OVERHEAD(36) 만큼 더 할당
        // MakeCmdPacket 내부에서 MSGID_COMMAND_V2 + v2 Encrypt
        SendMsgCommandV2(pCommand);
    } else {
        // [EXISTING] v1 경로
        SendMsgCommandV1(pCommand);
    }
}
```

### 3.2 ECDHE Challenge 핸들러 — 클라

**위치**: `MMatchClient::OnCommand` 의 신규 case

```cpp
case MC_MATCH_ECDHE_CHALLENGE:
{
    void* pServerPub; int nLen;
    pCommand->GetParameter(&pServerPub, 0, MPT_BLOB);

    auto* kxState = new Security::KeyExchangeState();
    kxState->ClientHandleChallenge((BYTE*)pServerPub);

    BYTE symKey[32];
    Security::DeriveSymmetricKey(kxState, symKey);
    GetCrypterV2()->InitKey(symKey, 32);

    // RESPONSE 송신 — v1 경로로
    MCommand* pResponse = new MCommand(...);
    pResponse->SetID(MC_MATCH_ECDHE_RESPONSE);
    pResponse->AddParameter(new MCmdParamBlob(kxState->GetClientPublic(), 32));
    SendCommand(pResponse);  // v1
    delete pResponse;

    // 송신 *후* 에 v2 활성화
    GetCommandBuilder()->InitCryptV2(GetCrypterV2());
    SetV2Active(true);

    SetKxState(kxState);
    break;
}
```

**중요**: `SetV2Active(true)` 는 RESPONSE 송신 *후* 에 flip. 그렇지 않으면 RESPONSE 자체가 v2 프레임으로 나가서 서버 디크립트 실패.

### 3.3 PostShot 매크로에 ATTACK_TICK 추가

**위치**: `Gunz/ZPost.h` 의 `ZPostShot` / `ZPostShotMelee` 매크로

```cpp
#define ZPostShot(...) do { \
    /* [EXISTING] 기존 ZPOSTCMD1(MC_PEER_SHOT, ...) */ \
    ...; \
    /* [INSERT] ATTACK_TICK piggyback */ \
    ZPostAttackTick(); \
} while(0)

#define ZPostAttackTick() do { \
    auto* my = ZGetGame()->m_pMyCharacter; \
    if (!my) break; \
    auto* items = my->GetItems(); \
    auto* sel = items ? items->GetSelectedWeapon() : nullptr; \
    auto* desc = sel ? sel->GetDesc() : nullptr; \
    if (!desc) break; \
    BYTE wt = (BYTE)desc->m_nWeaponType.Ref(); \
    UINT tickMs = (UINT)timeGetTime(); \
    ZPOSTCMD2(MC_MATCH_ATTACK_TICK, wt, tickMs); /* CLOAK_CMD_ID factor=53817 */ \
} while(0)
```

### 3.4 OnDamaged 훅에 HIT_TICK 추가 — Victim-Authoritative

**위치**: `ZMyCharacter::OnDamaged`

```cpp
void ZMyCharacter::OnDamaged(MUID uidAttacker, ZObject* pAttacker, ...) {
    // [EXISTING] 기존 데미지 처리
    ...

    // [INSERT] HIT_TICK 보고 — 피해자 권위
    if (pAttacker && pAttacker != this  // self-damage 제외
        && /* NPC 제외 */
        && /* 자폭/낙사 제외 */
        && weaponType != MWT_NONE) {
        rvector srcPos = pAttacker->GetPosition();
        ZPostHitTick(uidAttacker, weaponType, fDmg, srcPos);
    }
}

#define ZPostHitTick(atkUID, wt, dmg, srcPos) \
    ZPOSTCMD4(MC_MATCH_HIT_TICK, atkUID, wt, dmg, srcPos)  /* CLOAK_CMD_ID factor=49217 */
```

### 3.5 ZGame::Tick 에 POSITION_TICK piggyback

**위치**: `ZGame.cpp` 의 32 Hz 위치 송신 부분

```cpp
void ZGame::Tick() {
    // [EXISTING] peer broadcast
    ZPOSTCMD1(MC_PEER_BASICINFO, blob);

    // [INSERT] 서버에도 동시 송신
    ZPOSTCMD1(MC_MATCH_POSITION_TICK, blob);  // ZPACKEDBASICINFO 그대로 재사용
    // ZNewCmd 가 MCDT_MACHINE2MACHINE 플래그로 자동 라우팅
}
```

---

## 4. 신규 패킷 ID 등록

**파일**: `MSharedCommandTable.h` 또는 동등

```cpp
// 패킷 프레이밍
#define MSGID_COMMAND_V2  102

// 매치서버 ↔ 클라 ECDHE
#define MC_MATCH_ECDHE_CHALLENGE  2901
#define MC_MATCH_ECDHE_RESPONSE   2902

// 매치서버 ↔ 클라 안티치트 (MACHINE2MACHINE)
#define MC_MATCH_POSITION_TICK    2903
#define MC_MATCH_ATTACK_TICK      2904
#define MC_MATCH_HIT_TICK         2905
```

**파일**: `MSharedCommandTable.cpp` 의 `MSCT_MATCHSERVER | MSCT_CLIENT` 블록

```cpp
ADD_COMMAND("Match.ECDHEChallenge", MC_MATCH_ECDHE_CHALLENGE, MCDT_MATCHSERVER, MCDT_CLIENT);
ADD_COMMAND_PARAM(MPT_BLOB, "ServerPublic");

ADD_COMMAND("Match.ECDHEResponse", MC_MATCH_ECDHE_RESPONSE, MCDT_CLIENT, MCDT_MATCHSERVER);
ADD_COMMAND_PARAM(MPT_BLOB, "ClientPublic");

ADD_COMMAND("Match.PositionTick", MC_MATCH_POSITION_TICK, MCDT_MACHINE2MACHINE);
ADD_COMMAND_PARAM(MPT_BLOB, "BasicInfo");

ADD_COMMAND("Match.AttackTick", MC_MATCH_ATTACK_TICK, MCDT_MACHINE2MACHINE);
ADD_COMMAND_PARAM(MPT_UCHAR, "WeaponType");
ADD_COMMAND_PARAM(MPT_UINT, "ClientTickMs");

ADD_COMMAND("Match.HitTick", MC_MATCH_HIT_TICK, MCDT_MACHINE2MACHINE);
ADD_COMMAND_PARAM(MPT_UID, "AttackerUID");
ADD_COMMAND_PARAM(MPT_UCHAR, "WeaponType");
ADD_COMMAND_PARAM(MPT_FLOAT, "Damage");
ADD_COMMAND_PARAM(MPT_FLOAT_ARRAY_3, "SrcPos");
```

---

## 5. 빌드 시스템 변경

### 5.1 libsodium 의존성

`Directory.Build.props` (또는 동등):
```xml
<PropertyGroup>
    <LibsodiumIncludeDir>$(SolutionDir)..\third_party\libsodium\include</LibsodiumIncludeDir>
    <LibsodiumLibDir>$(SolutionDir)..\third_party\libsodium\lib\Win32\Release</LibsodiumLibDir>
</PropertyGroup>

<ItemDefinitionGroup>
    <ClCompile>
        <AdditionalIncludeDirectories>$(LibsodiumIncludeDir);%(AdditionalIncludeDirectories)</AdditionalIncludeDirectories>
    </ClCompile>
    <Link>
        <AdditionalLibraryDirectories>$(LibsodiumLibDir);%(AdditionalLibraryDirectories)</AdditionalLibraryDirectories>
        <AdditionalDependencies>libsodium.lib;%(AdditionalDependencies)</AdditionalDependencies>
    </Link>
</ItemDefinitionGroup>
```

### 5.2 Windows SDK
- `WindowsTargetPlatformVersion=10.0.26100.0` (또는 회사 표준)

### 5.3 PlatformToolset
- v143 권장. v145 도 호환 (libsodium 정적 라이브러리는 v143 빌드 ABI-safe)

### 5.4 UTF-8 BOM
- 한글 주석 포함된 .h/.cpp 파일은 UTF-8 BOM 필수 (cp949 trail-byte 문제 회피)

---

## 6. 통합 검증 체크리스트

각 Phase 도입 후:

- [ ] 빌드 성공 (`Release|Win32`, v143)
- [ ] 신규 심볼 확인 (`strings <binary> | findstr <module>`)
- [ ] 매치서버 기동 시 신규 환경변수 인식 로그
- [ ] 클라 빌드 + 접속 테스트
- [ ] 핸드셰이크 성공 로그 (Phase B 도입 시)
- [ ] 의도적 위반 시나리오 → 시그널 발생 (Phase C/D 도입 시)
- [ ] community-api 도착 확인 (Phase E 도입 시)
- [ ] 정상 트래픽에서 1 주 운용 → false-positive 비율 측정

---

## 7. 통합 시 자주 발생하는 함정

### 7.1 ECDHE KDF 분기
**증상**: 첫 LOGIN 패킷 `DecryptAEAD FAILED`
**원인**: 서버/클라 IKM 갈라짐
**해결**: `#ifdef BUILD_MATCH_SERVER` 분기 절대 금지. canonical `min‖max` 순서 통일.

### 7.2 V2 플래그 flip 타이밍
**증상**: ECDHE RESPONSE 패킷 자체 디크립트 실패
**원인**: 클라가 송신 전에 `SetV2Active(true)` 호출
**해결**: 송신 *후* flip.

### 7.3 AES-NI 미지원 호스트
**증상**: 핸드셰이크 OK, 첫 v2 패킷부터 침묵 드롭
**원인**: `Encrypt/Decrypt` 가 AES-NI 의존
**해결**: `InitKey` 가 명시적 false 반환 → 핸드셰이크 단계에서 진단 가능.

### 7.4 stdafx.h 의 `_WIN32_WINNT=0x0501`
**증상**: `GetTickCount64` undeclared
**원인**: WinXP 호환을 위해 `_WIN32_WINNT` 가 낮게 고정
**해결**: `std::chrono::steady_clock` 사용.

### 7.5 ZPACKEDBASICINFO 직접 include
**증상**: CSCommon 빌드 시 `ZPost.h not found`
**원인**: CSCommon 은 클라/서버 양측에 링크되는 공용 라이브러리이므로, Gunz (클라 전용) 의 헤더에 의존하면 (a) 서버 빌드 실패 + (b) 의존성 그래프 사이클 발생. 의도적으로 차단된 의존 방향이며 우회 시도 시 빌드 시스템 전체가 깨짐.
**해결**: blob 의 첫 4B fTime + 다음 6B (short XYZ) 만 raw 로 언팩 (POD 가정 — 향후 ZPACKEDBASICINFO 레이아웃 변경 시 매치서버 핸들러도 동기 수정 필요).

---

## 8. 참고

- 모듈별 상세 → [`modules/`](./modules/)
- 검증 절차 → [`06-validation-kit.md`](./06-validation-kit.md)
- 운영 → [`05-operations-runbook.md`](./05-operations-runbook.md)
