[English](../06-validation-kit.md) | **한국어**

# Validation Kit

> **독자**: QA / 보안 검증팀
> **목적**: 본 패키지의 동작을 30 분 내에 자체 재현. "이게 진짜 동작하는가" 의 답.

---

## 1. 사전 요건

| 요소 | 버전 | 비고 |
|------|------|------|
| OS | Windows 10/11 또는 Linux | 백엔드만이면 Linux/macOS 가능 |
| Docker Desktop | 4.x+ | 필수 |
| Python | 3.11+ | pytest 실행용 |
| Visual Studio | 2022 (v143) 또는 호환 | 클라/서버 빌드 (선택) |
| `curl` | any | HMAC E2E 검증용 |

---

## 2. 검증 시나리오 4종

### 2.1 백엔드 풀스택 (10 분)

목표: community-api + Postgres + Redis 가동.

```bash
# 프로젝트 루트에서
docker compose up -d

# 검증
curl http://127.0.0.1:8100/docs
# → FastAPI Swagger UI 가 떠야 함

curl http://127.0.0.1:8100/health
# → {"status":"ok"} 응답
```

기대 결과:
- 컨테이너 3개 가동 (`gunz_api`, `gunz_postgres`, `gunz_redis`)
- Postgres alembic head = `003_add_match_uuid_to_signals`
- Redis PONG 응답

### 2.2 HMAC E2E — 3 시나리오 (5 분)

목표: 매치서버 ↔ API HMAC 검증 동작 확인.

```bash
# 시나리오 A — 정상 서명 (200)
SECRET="test-secret-32-bytes-min"
BODY='{"reports":[{"player_uid_high":1,"player_uid_low":2,"signal_type":"WeaponSpoof","severity":"High","value":1.0,"threshold":0,"detail":"test"}]}'
SIG=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$SECRET" -hex | awk '{print $2}')

curl -X POST http://127.0.0.1:8100/api/anticheat/report \
  -H "Content-Type: application/json" \
  -H "X-Signature: $SIG" \
  -d "$BODY"
# 기대: HTTP 200

# 시나리오 B — 중복 (409)
curl -X POST http://127.0.0.1:8100/api/anticheat/report \
  -H "Content-Type: application/json" \
  -H "X-Signature: $SIG" \
  -H "Idempotency-Key: same-key" \
  -d "$BODY"
curl -X POST http://127.0.0.1:8100/api/anticheat/report \
  -H "Content-Type: application/json" \
  -H "X-Signature: $SIG" \
  -H "Idempotency-Key: same-key" \
  -d "$BODY"
# 기대: 두번째 요청 HTTP 409

# 시나리오 C — 잘못된 서명 (401)
curl -X POST http://127.0.0.1:8100/api/anticheat/report \
  -H "Content-Type: application/json" \
  -H "X-Signature: deadbeefdeadbeef" \
  -d "$BODY"
# 기대: HTTP 401
```

### 2.3 pytest 회귀 (10 분)

목표: 백엔드 로직 회귀 검증.

```bash
docker compose exec backend pytest -v
# 또는 호스트에서:
cd community-api
pytest -v
```

기대 결과:
```
============== 57 passed, 12 skipped, 3 xfailed in N.NNs ==============
```

상세:
- `test_phase2_integration.py` — OpenAPI 라우터 등록 + E2E 여정
- `test_shop_race.py` — Postgres 동시성 (SQLite 자동 스킵)
- `test_anticheat_*.py` — HMAC + 중복 + 시그널 보고

### 2.4 매치서버 빌드 (선택, 30 분)

목표: 신규 보안 모듈 포함 매치서버 바이너리 생성.

```powershell
# Windows / Visual Studio 2022
cd match-server\build
msbuild MatchServer.sln /p:Configuration=Release /p:Platform=Win32

# 출력
ls bin\Release\MatchServer.exe
```

기대 크기: ~3.7 MB (Release|Win32)

빌드 검증:
```powershell
# strings 명령으로 신규 심볼 확인
strings bin\Release\MatchServer.exe | findstr "MAPIClient"
strings bin\Release\MatchServer.exe | findstr "X-Signature"
strings bin\Release\MatchServer.exe | findstr "/api/anticheat/report"
```

---

## 3. 시그널 동작 시연 (라이브)

### 3.1 환경 준비
```bash
export COMMUNITY_API_URL=http://127.0.0.1:8100
export MATCH_RESULT_WEBHOOK_SECRET="test-secret-32-bytes-min"
match-server\build\bin\Release\MatchServer.exe
```

### 3.2 WeaponSpoof 재현 (의도적 변조)

테스트 클라이언트가 서버에 ATTACK_TICK 패킷을 보낼 때, 보고 무기 타입을 장착하지 않은 무기 클래스로 위조:

```cpp
// 변조 클라 코드 (테스트 전용)
ZPostAttackTick(/*WeaponType=*/(BYTE)Rocket);  // 실제 장착은 Melee
```

기대 결과:
1. 매치서버 로그: `WeaponSpoof: uid=... reportedClass=Rocket equipped=Melee severity=Critical`
2. community-api 로그: `POST /api/anticheat/report 200`
3. DB 조회:
```sql
SELECT signal_type, severity, match_uuid, detail
FROM anticheat_signals
WHERE signal_type = 'WeaponSpoof'
ORDER BY created_at DESC LIMIT 1;
```
→ `match_uuid` 가 `stage-<H>-<L>-<epoch>` 포맷으로 채워짐

### 3.3 SpeedHack 재현

```cpp
// 변조 클라 — 32 Hz 위치 송신을 5 배 가속한 좌표로
ZPOSTCMD1(MC_MATCH_POSITION_TICK, x + 1000, y + 1000, z, ...);
```

기대 결과:
- 첫 위반: Low severity
- 3 회 누적: Medium 승급
- 5 회 누적: High 승급

### 3.4 DamageReduce / HPHack 재현 (Module 13)

```cpp
// 변조 클라 — HIT_TICK 보고 시 dmg 를 1/5 로 축소
ZPostHitTick(atkUID, weaponType, /*dmg=*/realDmg * 0.2f, srcPos);
```

기대 결과:
1. 매치서버 로그: `DamageVal: Reduce <uid> wc=<W> dist=<D> reported=<X> server=<Y> ratio=0.2 streak=N`
2. ratio 0.2 < tolerance 0.5 → **High** severity 즉시 발행
3. 1초 주기 HPConsistency 검증 누적: 5초 후 HP 변화량 vs hit 합 차이 → 추가 `HPHack` 시그널 가능
4. community-api 도착 (callback 등록 시):
```sql
SELECT signal_type, severity, value, threshold, detail
FROM anticheat_signals
WHERE signal_type IN ('DamageManipulation', 'HPHack')
ORDER BY created_at DESC LIMIT 5;
```

### 3.5 PositionLie 재현

```cpp
// 변조 클라 — HIT_TICK 의 srcPos 를 실제 위치에서 1500u 떨어진 좌표로 위조
ZPostHitTick(targetUID, weaponType, dmg, /*srcPos=*/{X+1500, Y+1500, Z});
```

기대 결과:
- 무기 클래스가 SMG 면 threshold 800u, 1500u 는 1.875x → Medium
- 무기 클래스가 Melee 면 threshold 400u, 1500u 는 3.75x → High

---

## 4. DDoS 게이트 동작 시연

### 4.1 L1 — Connect Flood
```bash
# 동일 IP 에서 11 번 연속 접속 시도
for i in {1..11}; do
  nc -z 127.0.0.1 7777 &
done
```

기대:
- 처음 10 회는 정상 (TCP accept 후 ECDHE 핸드셰이크)
- 11번째: `Disconnect` 즉시 — 매치서버 로그에 `MConnectRateLimit: blocked IP=127.0.0.1`

### 4.2 L2 — Packet Flood
```cpp
// 변조 클라 — 1 초 내 256 개 초과 패킷 송신
for (int i = 0; i < 300; ++i) ZPOSTCMD0(MC_MATCH_HEARTBEAT);
```
기대: 257번째 패킷부터 세션 disconnect.

### 4.3 L3 — Bandwidth
```cpp
// 변조 클라 — 1 초 내 256 KiB 초과 송신
for (int i = 0; i < 100; ++i) ZPOSTCMD1(LARGE_BLOB, 4096B);
```
기대: 누적 256 KiB 초과 시점에 disconnect.

---

## 5. 시연 영상 (제공 시)

| 영상 | 길이 | 내용 |
|------|------|------|
| 01-handshake.mp4 | 2~3 분 | ECDHE 핸드셰이크 + Wireshark 패킷 캡처. v1 평문 vs v2 ciphertext 비교 |
| 02-ddos-gate.mp4 | 1~2 분 | L1/L2/L3 동작. 의도적 flood → 게이트 로그 |
| 03-anticheat.mp4 | 3~5 분 | WeaponSpoof / PositionLie / SpeedHack 재현. 시그널 → community-api 대시보드 도착 |

영상은 **음성 없이 자막만** — 회사 내부 회람 시 음성 부담 회피.

---

## 6. 검증 결과 보고 양식

```
[프로젝트] GunZ Security Modernization Validation Report
[검증자]   ___
[검증일]   ___
[환경]     OS / Docker / Python 버전

[시나리오 결과 — 구현 완료 모듈 (#01~#12)]
- 백엔드 풀스택 가동:           [ Pass / Fail ]
- HMAC E2E 3 시나리오:          [ Pass / Fail ]   200/409/401 모두 응답 ___
- pytest 회귀:                  [ Pass / Fail ]   passed/skipped/xfailed = ___
- 매치서버 빌드:                [ Pass / Fail / N/A ]   바이너리 크기 ___
- WeaponSpoof 시그널 도달:      [ Pass / Fail / N/A ]
- SpeedHack 승급:               [ Pass / Fail / N/A ]
- PositionLie 무기별 임계:      [ Pass / Fail / N/A ]
- DDoS L1/L2/L3 차단:           [ Pass / Fail / N/A ]

[시나리오 결과 — Module 13 (Detection-Only 모드)]
- MDamageValidator 빌드 검증:        [ Pass / Fail ]   바이너리 strings 신규 심볼 확인
- 무기 프로파일 등록 (RegisterDefaultProfiles): [ Pass / Fail ]
- DamageReduce 시그널 (보고 dmg < server × 0.7): [ Pass / Fail / N/A ]
- DamageInflate 시그널 (보고 dmg > server × 1.3): [ Pass / Fail / N/A ]
- HPConsistency 윈도우 검증:         [ Pass / Fail / N/A ]
- HPHack 시그널 (HP 변화량 불일치):  [ Pass / Fail / N/A ]
- OnPlayerLeave 메모리 정리:         [ Pass / Fail / N/A ]
- Mitigation 토글 (SetMitigationMode): [ N/A — 라이브 데미지 흐름 검증 후 활성 ]
  → ZRule* 의 OnPeerDamage 경로에 Mitigation 분기 추가 + 회복 메커닉 등록 결정 후 활성화

[관찰 사항]
- ___

[질문]
- ___
```

---

## 7. 자주 발생하는 문제

### 7.1 Docker 컨테이너 기동 실패
- 호스트 8000 포트 점유 (Manager.exe 등) → docker-compose.yml 에서 8100 매핑 확인
- Postgres 데이터 디렉토리 권한 → `docker compose down -v` 후 재시작

### 7.2 pytest 실패
- `@pytest.mark.postgres` 가드 → SQLite 환경에서 자동 스킵 (정상)
- 의존성 누락 → `pip install -r requirements-dev.txt`

### 7.3 매치서버 빌드 실패
- v143 PlatformToolset → 커맨드라인 `/p:PlatformToolset=v143` 명시
- libsodium 헤더 누락 → `Directory.Build.props` 의 include 경로 확인
- WindowsTargetPlatformVersion → 10.0.26100.0

### 7.4 ECDHE 핸드셰이크 실패
- AES-NI 미지원 호스트 → `MPacketCrypterV2::InitKey` 가 false 반환, 로그에 명시적 메시지
- 서버/클라 KDF 분기 → canonical min‖max 순서 확인 (`#ifdef BUILD_MATCH_SERVER` 분기 금지)

---

## 8. 검증 후 다음 단계

검증 통과 시:
1. NDA 체결 (모듈 소스 / 통합 가이드 열람)
2. [`08-integration-guide.md`](./08-integration-guide.md) 검토
3. Phase A (DDoS) 부터 단계별 도입 결정
4. 라이선스 모드 협의 ([`03-license-inventory.md`](./03-license-inventory.md))

검증 실패 시:
1. 실패 시나리오 문서화
2. 환경 차이 식별 (OS / 빌드 도구 / 의존성 버전)
3. 본 패키지 작성자에게 환경 정보 + 로그 송부
