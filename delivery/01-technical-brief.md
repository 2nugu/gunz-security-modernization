# Technical Brief — GunZ Security Modernization

> **독자**: 서버 팀 리드, 보안 담당. 30 분 내 도입 가부 판단.

---

## 1. 위협 모델 (Threat Model)

오리지널 GunZ 프로토콜의 알려진 공격 표면 5종과 본 작업의 대응:

### T1 — XOR Keystream 복원

- **공격**: 동일 키로 암호화된 패킷 다수를 수집하면 known-plaintext (헤더 상수 등) 와의 XOR 로 키스트림 복원 가능. nonce 부재 → 동일 평문 → 동일 암호문.
- **대응**: AES-256-GCM AEAD. 12B nonce (`session_id ‖ direction ‖ sequence`) 로 재생성마다 다른 keystream. 16B GCM tag 로 변조 검출.
- **잔여 위험**: AAD 현재 NULL — `nSize` 평문 유지. 헤더 변조 검출은 GCM tag 가 커버하지만, 헤더-페이로드 바인딩은 향후 AAD 로 강화 가능.

### T2 — 정적 시드 키 → 키 유출 시 과거 세션 노출

- **공격**: 키 합의 시 서버가 정적 시드 송신 → 시드 유출 시 모든 과거 세션 복호화 가능 (forward secrecy 부재).
- **대응**: X25519 ECDHE 임시 키쌍. 세션마다 신규 키쌍, 합의 직후 비밀키 즉시 소거. **PFS 확보**.
- **핵심 결정**: KDF 정준 순서 — `memcmp(rx, tx, 32) ≤ 0` 기준 `min ‖ max` 정렬 후 BLAKE2b IKM. 서버/클라 분기(`#ifdef`) 금지. 서버·클라 IKM 갈라짐 → 첫 LOGIN 패킷 디크립트 실패 사고 방지.

### T3 — Peer IP 평문 노출

- **공격**: `MC_MATCH_RESPONSE_PEER_LIST` / `StageEnterBattle` 의 peer blob 에 실 IP/Port 평문. IP 수집 → DDoS 표적화 → 서비스 거부 → NAT punch-through 우회.
- **대응**: 두 호출 지점에서 `dwIP=0 / nPort=0` 무조건 마스킹. `ServerBased` 강제. 클라이언트 P2P 직결 실패 → `MC_MATCH_REQUEST_PEER_RELAY` 폴백 → 서버가 P2P 트래픽 중계.
- **잔여 위험**: 매치서버 자체 IP 는 노출됨 (필연). DDoS 보호는 외부 레이어 (CDN/스크러버) 에 의존.

### T4 — 서버 권위 좌표 부재

- **공격**: 위치/속도/공격 정보가 P2P 신뢰 기반. 클라가 임의 좌표 주장 가능 → 텔레포트핵, 스피드핵, AutoAim, ImpossibleHit, WeaponSpoof, PositionLie.
- **대응**: 32-슬롯 ring buffer (`MPositionHistory`) + 32 Hz piggyback (`MC_MATCH_POSITION_TICK = 2903`). 서버측 좌표 권위 확보. Validator 들이 이 데이터를 소비.
- **잔여 위험**: BSP 지형 샘플링 미통합 — 정밀 noclip 검출은 RealSpace2 엔진 링크 선행 필요. 현재는 absolute Z 검사 (FlyHack heuristic, Medium severity 캡).

### T5 — Accept-Flood / Packet-Flood / Bandwidth-Flood

- **공격**: TCP accept 무제한, 세션당 입력 패킷 수/대역폭 무제한 → 단일 IP 가 매치서버 다운 가능.
- **대응**: 3계층 게이트 — `MConnectRateLimit` (per-IP, accept 단계) + `MPacketRateLimit` (per-session, packet 수) + `MBandwidthThrottle` (per-session, bytes). IOCP 워커 진입 직후 게이팅 → 위반 시 alloc 전 컷.

---

## 2. 아키텍처

### 2.1 데이터 흐름 (전체)

```
클라이언트                 매치서버                    community-api          DB
   │                          │                              │                │
   │ ── ECDHE Challenge ───>  │                              │                │
   │ <── ECDHE Response ───   │                              │                │
   │ ── v2 Encrypted Cmd ─>   │                              │                │
   │  POSITION_TICK 32Hz   →  │ → MPositionHistory.Record    │                │
   │  ATTACK_TICK         →   │ → CombatValidator.Validate   │                │
   │  HIT_TICK            →   │ → CombatValidator.Validate   │                │
   │                          │   ↓                          │                │
   │                          │   MSignalCollector.Push      │                │
   │                          │   (16-shard)                 │                │
   │                          │   ↓                          │                │
   │                          │   MAPIClient.SendAsync ──>   │ HMAC verify    │
   │                          │   (WinHTTP+HMAC-SHA256)      │ ↓              │
   │                          │   재시도 큐 MAX 100          │ INSERT ─────>  │ anticheat_signals
   │                          │                              │                │
   │ ←── DDoS L1: per-IP   ── │                              │                │
   │     (Accept 단계)        │                              │                │
   │ ←── DDoS L2: per-sess ── │                              │                │
   │ ←── DDoS L3: bandwidth ─ │                              │                │
```

### 2.2 ECDHE 핸드셰이크 시퀀스

```
1. TCP Accept
2. Server: InitCryptCommObject() — v1 seed key + ECDHE keypair 생성
3. Server → Client: MC_MATCH_ECDHE_CHALLENGE (v1-encrypted, BLOB=서버 공개키 32B)
4. Server → Client: MC_MATCH_REPLYCONNECT (v1)
5. Client: ClientHandleChallenge → DeriveSymmetricKey (canonical min‖max)
6. Client: GetCrypterV2()->InitKey
7. Client → Server: MC_MATCH_ECDHE_RESPONSE (v1-encrypted, BLOB=클라 공개키 32B)
   ← 주의: SetV2Active(true) 는 RESPONSE 송신 *후* 에 flip
8. Server: ServerDeriveSessionKeys → DeriveSymmetricKey → InitKey → SetV2Active(true)
9. 이후 모든 패킷: MSGID_COMMAND_V2 (102), AES-256-GCM
```

타이밍 미묘함 — 클라가 `SetV2Active(true)` 를 RESPONSE 송신 전에 하면 RESPONSE 자체가 v2 프레임으로 나가는데 서버는 아직 v2 미적용 → 디크립트 실패. 클라 핸들러에서 Post 대신 동기 SendCommand 후 v2 플래그 flip.

### 2.3 안티치트 시그널 파이프라인

```
[클라] ZGame::Tick (32 Hz)
   ├── ZPostBasicInfo (peer broadcast, 기존)
   └── ZPOSTCMD1(MC_MATCH_POSITION_TICK, blob)  ← 신규, piggyback

[클라] ZPostShot / ZPostShotMelee
   └── ZPostAttackTick (CLOAK_CMD_ID factor=53817)

[클라] ZMyCharacter::OnDamaged  ← victim-authoritative
   └── ZPostHitTick (CLOAK_CMD_ID factor=49217)

[서버] MMatchServer_OnCommand
   ├── case MC_MATCH_POSITION_TICK → MMovementValidator.Validate
   ├── case MC_MATCH_ATTACK_TICK   → MCombatValidator.ValidateAttack
   └── case MC_MATCH_HIT_TICK      → MCombatValidator.ValidateHit
                                     + PositionLie cross-check

[서버] Validator → CheatSignal {type, severity, value, threshold, detail}
       → MSignalCollector.Push (16-shard, lock-distributed)
       → SetReportCallback 람다
       → MAPIClient.SendCheatReportsAsync (fire-and-forget + retry)

[community-api] POST /api/anticheat/report (X-Signature: hex(HMAC-SHA256(body)))
       → verify_signature
       → INSERT INTO anticheat_signals (... match_uuid ...)
```

---

## 3. 모듈 맵

| # | 모듈 | 위치 (제안) | LoC (≈) | 의존성 | 단독 도입 | 구현 상태 |
|---|------|-----------|--------|--------|----------|----------|
| 01 | `MPacketCrypterV2` | `CSCommon/{Include,Source}/MPacketCrypterV2.{h,cpp}` | 280 | libsodium | 02 와 함께 | 구현 완료 |
| 02 | `KeyExchange` | `CSCommon/{Include,Source}/KeyExchange.{h,cpp}` | 220 | libsodium | 01 과 함께 | 구현 완료 |
| 03a | `MConnectRateLimit` | `CSCommon/Security/MConnectRateLimit.{h,cpp}` | 130 | std::chrono | ✅ | 구현 완료 |
| 03b | `MPacketRateLimit` | `CSCommon/Security/MPacketRateLimit.{h,cpp}` | 110 | std::chrono | ✅ | 구현 완료 |
| 03c | `MBandwidthThrottle` | `CSCommon/Security/MBandwidthThrottle.{h,cpp}` | 110 | std::chrono | ✅ | 구현 완료 |
| 04 | IP Relay (peer mask) | `MMatchServer.cpp:2147` + `MMatchServer_Stage.cpp:467` | 30 | 없음 | ✅ | 구현 완료 |
| 05 | `MPositionHistory` | `CSCommon/Security/MPositionHistory.{h,cpp}` | 110 | 없음 (STL-free) | 06/07 의존성 | 구현 완료 |
| 06 | `MMovementValidator` | `CSCommon/Security/MMovementValidator.{h,cpp}` | 240 | 05, 08 | ✅ (05+08 후) | 구현 완료 |
| 07 | `MCombatValidator` | `CSCommon/Security/MCombatValidator.{h,cpp}` | 320 | 05, 08 | ✅ (05+08 후) | 구현 완료 |
| 08 | `MSignalCollector` | `CSCommon/Security/MSignalCollector.{h,cpp}` | 180 | 없음 | 06/07/09/10 인프라 | 구현 완료 |
| 09 | Weapon Spoof (inline) | `MMatchServer_OnCommand.cpp` ATTACK_TICK 핸들러 | 60 | 07, 08 | 07 + 08 후 | 구현 완료 |
| 10 | Position Lie (inline) | `MMatchServer_OnCommand.cpp` HIT_TICK 핸들러 | 80 | 05, 07, 08 | 05 + 07 + 08 후 | 구현 완료 |
| 11 | `MAPIClient` | `CSCommon/Security/MAPIClient.{h,cpp}` | 220 | WinHTTP, libsodium | 08 후 | 구현 완료 |
| 12 | match_uuid pipeline | `MMatchStage.{h,cpp}` + `MMatchServer_Stage.cpp` | 50 | 08 | 08 후 | 구현 완료 |
| 13 | `MDamageValidator` | `CSCommon/Include/MDamageValidator.{h,cpp}` | 350 | 05, 08, 무기 테이블 | 07 + 08 후 | **구현 완료** (Detection-Only) |

**Module 01~13**: 빌드 검증 통과 (CSCommon.lib / MatchServer.exe Release|Win32 v143, 바이너리 4.26 MB). 신규 심볼 (`DamageVal`, `HPHack`, `HPInconsistency`, `DamageManipulation`) `strings` 검출 확인. LoC 는 실제 코드 라인 수.

**Module 13 의 기본 모드**: Detection-Only (`SERVER_DAMAGE_OVERRIDE=false`). 시그널 발행만, victim HP 차감은 기존 흐름 그대로. Mitigation 모드 활성화는 환경변수 토글 (구현 완료, 라이브 데미지 흐름 검증 후 운영자 결정).

---

## 4. Wire Compatibility

### 4.1 패킷 포맷 v2

```
[ MPacketHeaderV2 (20B) ][ Ciphertext (N B) ][ GCM Tag (16B) ]

MPacketHeaderV2:
  uint16_t  nMagic;          // 패킷 매직
  uint16_t  nMsgID;          // 102 = MSGID_COMMAND_V2
  uint16_t  nSize;           // 전체 길이 (헤더+ct+tag), 평문
  uint16_t  nReserved;
  uint32_t  nSessionID;      // nonce 첫 4B
  uint32_t  nDirection;      // nonce 다음 4B (0=C2S, 1=S2C)
  uint32_t  nSequence;       // nonce 마지막 4B (replay 방지)

오버헤드: GZ_V2_OVERHEAD = 36 (header 20 + tag 16)
```

### 4.2 신규 MSGID

| ID | 이름 | 방향 | 용도 |
|----|------|------|------|
| 102 | `MSGID_COMMAND_V2` | C↔S | v2 패킷 프레이밍 |
| 2901 | `MC_MATCH_ECDHE_CHALLENGE` | S→C | ECDHE 서버 공개키 송신 |
| 2902 | `MC_MATCH_ECDHE_RESPONSE` | C→S | ECDHE 클라 공개키 송신 |
| 2903 | `MC_MATCH_POSITION_TICK` | C→S (MACHINE2MACHINE) | 32 Hz 위치 piggyback |
| 2904 | `MC_MATCH_ATTACK_TICK` | C→S (MACHINE2MACHINE) | 발사 이벤트 |
| 2905 | `MC_MATCH_HIT_TICK` | C→S (MACHINE2MACHINE) | 피격 보고 (victim-auth) |

### 4.3 v1 vs v2 Negotiation

현재 구현: **레거시 클라 호환 없음** — 핸드셰이크 4 패킷 구간만 v1, 이후 모든 패킷 v2 강제.

라이브 서비스 도입 시 권장:
- (Option A) 점진 도입 — v1/v2 negotiation 비트를 헤더에 추가, 클라 패치 배포 후 일정 기간 양립
- (Option B) 일괄 도입 — 클라 강제 업데이트 후 cutover

본 패키지는 Option B 가정 — sunset 시점 명시적 cutover 가 보안 분석을 단순화함.

---

## 5. 운영 영향

### 5.1 메모리 (per-session)

수치는 정적 분석 기반 **추정값** (런타임 측정 미실시). 실제 메모리는 컴파일러 padding / 할당자 오버헤드로 변동 가능.

| 모듈 | 추가 메모리 (≈) |
|------|--------------|
| `MPacketCrypterV2` | ≈ 700 B (AES context + nonce 카운터) |
| `KeyExchangeState` | ≈ 80 B (handshake 후 free) |
| `MPacketRateLimit` | ≈ 100 B (deque<ms> + 카운터) |
| `MBandwidthThrottle` | ≈ 80 B |
| `MPositionHistory` | 32 × 16 B = 512 B (sample POD) |
| `PlayerMovementState` | ≈ 80 B |
| `PlayerCombatState` | 20 × ≈ 24 B + 카운터 = ≈ 600 B |
| **합계 (≈)** | **≈ 2.2 KB / session** |

10,000 동시 접속 시 ≈ 22 MB 추가. 무시 가능 수준 (실측 권장).

### 5.2 CPU

| 모듈 | 비용 |
|------|------|
| AES-256-GCM (AES-NI on) | ~0.3 cycles/byte. 1500 B 패킷 ~450 cycles. 1 GHz 기준 0.45 µs/패킷 |
| ECDHE (per session, 1회) | X25519 점-스칼라 곱 ~70 µs |
| Validator 호출 | per-tick O(1) ~ O(20) — combat history 역스캔 최대 20개 |
| Signal Push | O(1), 16-shard lock으로 컨텐션 분산 |

### 5.3 대역폭

수치는 패킷 포맷 기반 **이론값**. 실측 미실시.

| 추가 패킷 | 빈도 | 크기 (헤더 포함, ≈) | per-player 추가 (≈) |
|----------|------|------------------|--------------------|
| `POSITION_TICK` | 32 Hz | ≈ 50 B (v2) | ≈ 1.6 KB/s |
| `ATTACK_TICK` | 발사 시 (≈ 5 Hz peak) | ≈ 30 B | ≈ 150 B/s peak |
| `HIT_TICK` | 피격 시 (≈ 2 Hz peak) | ≈ 50 B | ≈ 100 B/s peak |
| **합계 (peak, ≈)** | | | **≈ 1.85 KB/s/player** |

10,000 동접, peak 시 ≈ 18.5 MB/s. 매치서버 NIC 1 Gbps 기준 15 % 미만 (실측 권장).

### 5.4 신규 환경변수

| 환경변수 | 필수 | 용도 | 미설정 시 동작 |
|----------|------|------|--------------|
| `COMMUNITY_API_URL` | △ | community-api 엔드포인트 | callback 미등록, 메모리-only |
| `MATCH_RESULT_WEBHOOK_SECRET` | △ | HMAC-SHA256 secret (32 B 권장) | callback 미등록 |
| (Postgres 접속) | ✅ (Phase F 도입 시) | DB 연결 | community-api 기동 실패 |

---

## 6. Phasing 권장

| Phase | 단위 | 구현 상태 | 의존 | 작업 분량 (≈) |
|-------|------|----------|------|--------------|
| A | DDoS 3계층 + IP 마스킹 | 구현 완료 | — | 2~3 인일 (통합 + 튜닝) |
| B | AES-256-GCM + ECDHE + V2 패킷 프레이밍 | 구현 완료 | A | 2~3 인주 (통합 + 호환성 검증 + 클라 빌드) |
| C | Position History + Movement Validator | 구현 완료 | B | 1~2 인주 |
| D | Combat Validator + Weapon Spoof + Position Lie | 구현 완료 | C | 2~3 인주 |
| E | Signal Collector + API Client + community-api | 구현 완료 | D | 2~3 인주 (백엔드 인프라 포함) |
| F | match_uuid pipeline + Postgres 마이그레이션 | 구현 완료 | E | 1~2 인주 |
| **G** | **Damage Validator** (Detection-Only) | **구현 완료** | D, E | 1 인주 (통합 + 무기 테이블 import) + 1~2 주 운용 |
| **G2** | **Damage Validator** (Mitigation 활성) | **구현 완료, 토글 가능** | G | 1 인주 (검증 + `SetMitigationMode(true)`) + 회복 메커닉 등록 별도 |

총 (Phase A~F) 10~15 인주 — 본 패키지 구현 완료분 통합. **Phase G** 는 본 패키지에서 Detection-Only 코드 작성 완료 — 통합 시 무기 테이블 import + ZRule 측 회복 메커닉 등록 흐름 결정 (1 인주 추정). **G2** 는 Detection-Only 운용 후 false-positive 검증 통과 시 `SetMitigationMode(true)` 토글로 즉시 활성. 기존 데미지 흐름 침해 시 즉시 토글 false 로 롤백.

각 Phase 단독 출시 가능 (G 는 D, E 후).

---

## 7. 검증된 영역 / 미수행 영역

### 검증됨
- 빌드: CSCommon.lib / MatchServer.exe / Gunz.exe (Release|Win32, v143)
- 핸드셰이크: ECDHE Challenge/Response 흐름 코드 경로 완성
- HMAC E2E: community-api 3 시나리오 (200 valid / 409 dedup / 401 badsig)
- pytest: 57 passed / 12 skipped / 3 xfailed / 0 failed
- 통합: docker compose 풀스택 가동, Alembic 003 적용

### 검증 미수행
- 실기동 MatchServer ↔ Gunz 핸드셰이크 성공 로그 (다음 작업 항목)
- 프로덕션 트래픽 분포 기반 임계값 튜닝
- 10K+ 동시 접속 부하 테스트
- 라이브 GunZ 와의 wire 호환성 (별도 작업)

---

## 8. 주요 기술 결정

### 8.1 STL-free 헤더 (`MPositionHistory`)
- 헤더가 STL 을 끌어오면 ABI 호환성 위협. POD struct 만 노출. CSCommon 이 v143/v145 양쪽 빌드되는 환경에서 안전.

### 8.2 16-shard Lock (`MSignalCollector`)
- 동시 접속 N → 단일 mutex 컨텐션 → push 지연. shard 16개로 분산, `hash(uid) % 16` 으로 라우팅.
- AUDIT-05 에서 식별된 컨텐션 위협 대응.

### 8.3 victim-authoritative HIT_TICK + Server-Side Damage Override
- 공격자-권위 보고 → 공격자가 hit-rate 자가조작 가능
- 피해자-권위 보고 → **피해자가 받은 데미지를 축소/무시 가능** (데미지 감소핵)
- **하이브리드 채택**: hit 발생 사실은 양측 보고 cross-check, srcPos 는 공격자, vicPos 는 서버 권위, **데미지 양은 서버 계산**
- victim HP 차감은 클라 보고 dmg 가 아니라 서버 무기 테이블 기반 `serverDmg` 로 → 데미지 감소핵 / 무적핵 무력화
- HPConsistency: 시간 윈도우 동안 HIT_TICK 합 vs HP 변화량 비교 → HP 메모리 조작핵 검출 (Module 13)

### 8.4 자동 밴 금지
- 모든 시그널은 운영자 큐로. 거짓 양성 비용 (실 유저 부정 밴) > 거짓 음성 비용 (핵 유저 며칠 더 활동).
- 단, 시그널 누적 점수 (L×1 + M×3 + H×7 + C×10) 가 임계 초과 시 자동 ghost-mode 모니터링 큐 등록.

### 8.5 `#ifdef BUILD_MATCH_SERVER` 분기 금지
- ECDHE KDF 가 서버/클라 분기로 갈라지면 IKM 불일치 → 핸드셰이크 실패. canonical 순서로 통합.
- CSCommon, match-server, netprobe 세 사본 모두 동일 코드.

---

## 9. 참고 문서

- 한 페이지 비교 → [`02-before-after-comparison.md`](./02-before-after-comparison.md)
- 안티치트 카탈로그 → [`04-anticheat-catalog.md`](./04-anticheat-catalog.md)
- 모듈 상세 → [`modules/`](./modules/)
- 통합 의사코드 → [`08-integration-guide.md`](./08-integration-guide.md)
- 검증 재현 → [`06-validation-kit.md`](./06-validation-kit.md)
