[English](../02-before-after-comparison.md) | **한국어**

# Before / After — 한 페이지 비교

> 오리지널 GunZ(2007 MAIET) 프로토콜 동작과 본 작업 산출물의 전면 비교.

---

## 1. 보안 / 암호화

| 축 | Before | After | 차이의 의미 |
|----|--------|-------|-----------|
| 대칭 암호 알고리즘 | XOR + rot8 + 0xF0 (32B 키, 비표준) | AES-256-GCM (libsodium) | 표준 AEAD, AES-NI 가속 |
| 무결성 | nCheckSum 6B (단순 합) | 16B GCM tag | 인증된 무결성 |
| Nonce / IV | 없음 (동일 평문 → 동일 암호문) | 12B (`session ‖ direction ‖ seq`) | replay·known-plaintext 차단 |
| 키 합의 | 정적 시드 송신 | X25519 ECDHE 임시 키쌍 | Forward secrecy |
| 키 유출 시 | 모든 과거 세션 복호화 가능 | 과거 세션 보호 | PFS 확보 |
| AAD 바인딩 | 없음 | NULL (향후 헤더 바인딩 여지) | 헤더 변조 검출은 GCM tag 가 커버 |
| HW 가속 폴백 | — | AES-NI 미지원 시 핸드셰이크 단계 명시적 실패 | 침묵 드롭 사고 방지 |

---

## 2. 네트워크 토폴로지

| 축 | Before | After | 차이의 의미 |
|----|--------|-------|-----------|
| Peer IP/Port 노출 | 평문 브로드캐스트 (`ResponsePeerList`, `StageEnterBattle`) | `0.0.0.0:0` 마스킹 | DDoS 표적화 봉쇄, IP 수집 차단 |
| P2P 연결 | 직결 시도 | 서버 릴레이 폴백 | NAT punch-through 우회 차단 |
| Admin/Event 예외 분기 | 존재 | 제거 (모두 마스킹) | 권한 상승 시나리오 봉쇄 |

---

## 3. 서버 안정성 (DDoS)

| 축 | Before | After | 차이의 의미 |
|----|--------|-------|-----------|
| Connect-flood | 무제한 | per-IP 10 conn / 10 s | accept 단계 컷, alloc 부담 0 |
| Packet-flood | 무제한 | per-session 256 packets / 1 s | 작은 패킷 플러드 차단 |
| Bandwidth-flood | 무제한 | per-session 256 KiB / 1 s | 큰 패킷 포화 차단 |
| 게이팅 위치 | — | IOCP 워커 진입 직후 | 최소 비용 컷 |
| 게이트 비활성화 | — | window=0 또는 max=0 (운영 튜닝) | 환경별 조정 가능 |
| 메모리 GC | — | 60 s 윈도, deque + lastSeen | 장기간 운영 누수 방지 |

---

## 4. 안티치트 (서버측 검증)

| 축 | Before | After | 차이의 의미 |
|----|--------|-------|-----------|
| 서버 권위 좌표 | 없음 (P2P 신뢰) | 32-슬롯 ring buffer | rewind / 크로스체크 가능 |
| 좌표 송신 | peer broadcast 만 | `MC_MATCH_POSITION_TICK` 32 Hz piggyback | 서버에도 동시 송신 |
| Speed/Teleport/Fly 검증 | 없음 | `MMovementValidator` 4단계 severity 승급 | 서버측 탐지 |
| RapidFire/InfiniteSlash (무한 강베기) | 없음 | 무기 클래스별 발사 간격 검증 | 서버측 탐지 |
| AutoAim/ImpossibleHit | 없음 | range×1.3 + 50발 이후 hit-rate 95%+ | 서버측 탐지 |
| Weapon Spoof | 탐지 불가 | 서버-권위 장착 3슬롯 vs 보고 무기타입 | **본 작업 고유 축** |
| Position Lie | 탐지 불가 | POSITION × HIT 크로스체크 (Euclidean) | **본 작업 고유 축** |
| **데미지 감소핵 (DamageReduce)** | 탐지 불가, 차단 불가 | Server-Side Damage Override | **victim-auth 사각지대 봉쇄** |
| **무적핵 (Invincibility)** | 탐지 불가 | DamageNullification (PEER_SHOT × HIT_TICK) | 본 작업 |
| **HP 메모리 조작핵 (HPHack)** | 탐지 불가 | HPConsistency (HP 변화량 vs hit 합) | 본 작업 |
| **데미지 증폭핵 (DamageInflate)** | 탐지 불가 | 보고 dmg > 서버 dmg × 1.3 | 본 작업 |
| 무기별 threshold | — | Melee 400u ~ Rocket 1000u | 클래스별 차등 적용 |
| Hit 보고 권위 | 공격자 | 피해자 (`ZMyCharacter::OnDamaged`) | 자가조작 봉쇄 |

---

## 5. 시그널 처리 / 운영

| 축 | Before | After | 차이의 의미 |
|----|--------|-------|-----------|
| 시그널 수집 | 없음 | `MSignalCollector` (16-shard) | lock 컨텐션 분산 |
| 가중 점수 | 없음 | L×1 + M×3 + H×7 + C×10 | 누적 평가 |
| 자동 제재 | (없음) | **금지** (정책) | 거짓 양성 분쟁 회피 |
| 운영자 보고 | 없음 | HMAC-SHA256 + 재시도 큐 (MAX 100) | 외부 시스템 연동 |
| 라운드 식별 | 없음 | `match_uuid = stage-<H>-<L>-<epochSec>` 라이프사이클 전파 | 라운드 단위 분석 |
| ghost-mode | 없음 | 설계 (관리자 투명 접속) | 실시간 모니터링 |

---

## 6. 백엔드 인프라

| 축 | Before | After | 차이의 의미 |
|----|--------|-------|-----------|
| DB | MS-SQL 단일 | PostgreSQL + Alembic | 마이그레이션 가능 |
| 캐시 | 없음 | Redis (선택) | 핫 데이터 분리 |
| 내부 API | 없음 | FastAPI + HMAC-SHA256 | 서명된 내부 통신 |
| 라우터 | — | game_accounts / leaderboard / shop / admin_game | 모듈화 |
| 테스트 | — | pytest 57 passed / 0 failed | 회귀 검증 |
| 배포 | 수동 | 멀티스테이지 Docker (210 MB, non-root uid 1000) | 표준 컨테이너 |
| 비동기 처리 | — | MatchServer → API HMAC + 로컬 재시도 큐 | 일시 장애 복원 |

---

## 7. 프로토콜 / 호환성

| 축 | Before | After |
|----|--------|-------|
| 패킷 ID 단일성 | `MSGID_COMMAND` | `MSGID_COMMAND` (v1, 핸드셰이크만) + `MSGID_COMMAND_V2 = 102` |
| 헤더 크기 | 6 B (`MPacketHeader`) | 20 B (`MPacketHeaderV2`) |
| 신규 MSGID | — | 2901, 2902, 2903, 2904, 2905 |
| Negotiation | — | 현재 없음 (cutover 가정). 점진 도입 시 별도 작업 |

---

## 8. 코드 / 빌드 환경

| 축 | Before | After |
|----|--------|-------|
| 컴파일러 | VC6 / VS9 | VS18 v143/v145 |
| 표준 | C++03 | C++14 (limited) |
| 외부 의존성 | 없음 (in-tree) | libsodium (ISC) |
| 인코딩 | EUC-KR/cp949 | UTF-8 BOM (한글 주석 .h/.cpp) |
| Windows SDK | 6.0 | 10.0.26100 |
| `#ifdef BUILD_MATCH_SERVER` 분기 | 사용 | **금지** (canonical 순서로 통합) |

---

## 9. 한눈 요약

```
Before                                     After
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[패킷 보안]
  XOR + rot8                          →    AES-256-GCM AEAD              [구현]
  정적 시드 키                        →    X25519 ECDHE PFS              [구현]
  무결성 없음                          →    16B GCM tag                   [구현]
  nonce 없음                          →    12B per-direction nonce       [구현]

[토폴로지]
  Peer IP 평문                        →    0.0.0.0:0 + 서버 릴레이       [구현]

[안정성]
  무제한 connect/packet/bandwidth    →    3계층 게이트                  [구현]

[안티치트 — 이동/전투]
  P2P 신뢰                            →    서버 권위 좌표 + Validator    [구현]
  탐지 인프라 없음                   →    16-shard Collector + HMAC API [구현]
  WeaponSpoof / PositionLie 불가    →    Novel 2축                     [구현]
  무한강베기/RapidFire 탐지 없음     →    무기 클래스별 발사 간격       [구현]

[안티치트 — 데미지/HP] (Module 13, Detection-Only 기본)
  데미지 감소핵 차단 불가            →    무기 테이블 + tolerance       [구현]
  데미지 증폭핵 (1-shot)             →    serverDmg × 1.3 초과 검출     [구현]
  HP 메모리 조작핵 탐지 불가         →    HPConsistency 윈도우 검증     [구현]
  Mitigation 토글로 효과 무력화     →    SetMitigationMode(true)        [구현]

[운영]
  자동 제재 / 수동                   →    시그널 → 경중 분류 → 운영자   [구현]
  라운드 식별 없음                   →    match_uuid 라이프사이클       [구현]
```

`[구현]` = 본 패키지 코드 작성 + 빌드 검증 통과
`[설계]` = 본 패키지 설계 명세만, 도입 결정 후 코드 작성
