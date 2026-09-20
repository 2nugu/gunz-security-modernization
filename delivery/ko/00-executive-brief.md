[English](../00-executive-brief.md) | **한국어**

# Executive Brief — GunZ Security Modernization

> **독자**: 사업/기획 디렉터. 5 분 안에 의사결정 가능한 수준으로 압축.

---

## 한 줄 요약

오리지널 GunZ(2007 MAIET) 의 패킷 보안·DDoS 방어·서버측 안티치트 레이어를 현대 표준으로 교체한 작업물 — **12 개 신규 모듈 + 통합 지점 명세 + 검증 키트**. 라이브 GunZ 가 이미 동등 솔루션을 보유하고 있을 가능성을 전제하며, 본 패키지의 차별 가치는 GunZ 플레이 + 개발 경험 기반 **안티치트 시그널 카탈로그** 와 **신규 탐지 축 2종**(Weapon Spoof / Position Lie) 에 있다.

---

## 핵심 수치

| 영역 | 오리지널 | 본 작업 | 효과 |
|------|---------|--------|------|
| 패킷 암호 | XOR + 비트 시프트 + 0xF0 (32B 키) | AES-256-GCM AEAD + X25519 ECDHE | 인증된 무결성, PFS, keystream 복원 공격 봉쇄 |
| Peer IP 노출 | 평문 브로드캐스트 | `0.0.0.0:0` 마스킹 + 서버 릴레이 | DDoS 표적화 봉쇄, IP 수집 차단 |
| Connect-flood | 무제한 | per-IP 10 conn / 10 s | accept-flood 차단 (alloc 전 컷) |
| Packet-flood | 무제한 | per-session 256 packets / 1 s | 작은 패킷 플러드 차단 |
| Bandwidth-flood | 무제한 | per-session 256 KiB / 1 s | 큰 패킷 포화 차단 |
| 서버 권위 좌표 | 없음 (P2P 신뢰) | 32-슬롯 링버퍼 + 32 Hz piggyback | rewind 검증, 서버측 탐지 가능 |
| Speed/Teleport/Fly 검증 | 없음 | `MMovementValidator` | 4단계 severity 승급 |
| RapidFire/InfiniteSlash 검증 | 없음 | `MCombatValidator` 무기 클래스별 발사 간격 | GunZ 의 시그니처 핵 봉쇄 |
| AutoAim/ImpossibleHit 검증 | 없음 | `MCombatValidator` range × 1.3 + hit-rate 95%+ | 서버측 탐지 |
| Weapon spoof | 탐지 불가 | 서버-권위 장착 3슬롯 대조 | **본 작업 고유 축** |
| Position lie | 탐지 불가 | POSITION × HIT 크로스체크 | **본 작업 고유 축** |
| **데미지 감소핵 / 무적핵 / HPHack** | 탐지 불가, 차단 불가 | `MDamageValidator` Detection-Only 구현 완료 — 무기 테이블 + tolerance + HP 일관성 | **victim-auth 사각지대 봉쇄. Mitigation 토글로 핵 효과 자체 무력화** |
| 외부 보고 | 없음 | HMAC-SHA256 + 재시도 큐 | 운영자 대시보드 연동 |
| match_uuid 추적 | 없음 | 라이프사이클 전파 | 라운드 단위 분석 |
| DB | MS-SQL | PostgreSQL + Alembic | 마이그레이션 가능 인프라 |
| 배포 | 수동 | 멀티스테이지 Docker (210 MB, non-root) | 표준 컨테이너 배포 |

---

## 도입 시 기대 효과

1. **현대 보안 베이스라인 확보**. 2007 년 시점 비표준 XOR 변형은 현대 공격 모델 대응 불가. AEAD + ECDHE 도입으로 패킷 변조·재생·키 유출 시나리오에서 보호.
2. **서버 안정성**. accept-flood / packet-flood / bandwidth-flood 가 단일 장비를 다운시키는 시나리오를 3계층 게이트로 차단. IOCP 워커 진입 직후 게이팅으로 alloc 부담 최소화.
3. **운영 가능한 안티치트**. 자동 밴 없이 시그널 → 경중 분류 → 운영자 리뷰 흐름. 자동 밴이 만드는 거짓 양성 분쟁을 회피.
4. **GunZ 핵 카탈로그의 형식화**. 본서버 플레이 + 공개 트리 자체 빌드 환경에서 관찰된 핵 패턴을 시그널·튜닝값 형태로 정리.

---

## 도입 시 리스크

| 리스크 | 완화 방안 |
|--------|----------|
| 프로토콜 v1 → v2 전환 — 레거시 클라 호환성 | 핸드셰이크 4 패킷 구간만 v1, 이후 v2 강제. 점진적 도입 시 v1/v2 negotiation 추가 작업 필요 (현재 미구현) |
| libsodium 신규 의존성 추가 | ISC 라이선스, 정적 링크 가능, 사실상 업계 표준 |
| AES-NI 미지원 호스트 | InitKey 시점 명시적 실패 — 핸드셰이크 단계에서 진단 가능 |
| 안티치트 임계값 튜닝 부담 | 본 패키지 기본값은 공개 트리 자체 빌드 환경 측정 추정값. 라이브 트래픽 분포 측정 후 조정 권장 |
| Win32 IOCP 의존 | 현재 코드는 Win32 전제. 크로스플랫폼화는 별도 작업 |
| DB 변경 (MS-SQL → Postgres) | 백엔드 인프라만 영향, 클라이언트 무영향. 회사 운영 환경에 따라 채택 여부 분리 가능 |

---

## 도입 단위 (Phase 별 단독 가치)

| Phase | 단위 | 구현 상태 | 단독 도입 | 기대 효과 |
|-------|------|----------|----------|----------|
| **Phase A** | DDoS 3계층 게이트 + IP 마스킹 | 구현 완료 | ✅ | 서버 가용성 즉시 개선. 프로토콜 변경 없음 |
| **Phase B** | AES-256-GCM + ECDHE | 구현 완료 | ✅ | 패킷 보안. 클라/서버 양측 빌드 필요 |
| **Phase C** | Position History + Movement Validator | 구현 완료 | ✅ (B 의존) | 스피드핵/텔레포트핵 서버측 탐지 |
| **Phase D** | Combat Validator + Novel 2축 (WeaponSpoof, PositionLie) | 구현 완료 | ✅ (C 의존) | 무한강베기/RapidFire/AutoAim 탐지 |
| **Phase E** | Signal Collector + API 보고 | 구현 완료 | ✅ (D 의존) | 운영자 대시보드 연동 |
| **Phase F** | community-api + Postgres + match_uuid | 구현 완료 | △ (인프라) | 라운드 단위 분석, HMAC 내부 API |
| **Phase G** | **Damage Validator** (Detection-Only → Mitigation) | **구현 완료** (Detection-Only 기본) | 단계 도입 (D, E 의존) | 데미지 감소핵 / 무적핵 / HPHack 대응 |

각 Phase 는 이전 Phase 없이도 단독 도입 가능 (F 인프라 단계 별도). Phase A 만 도입해도 서버 안정성 가치 단독 실현.

**Phase G 의 단계 도입**:
- Step G1 — Detection-Only 활성 (1~2 주, false-positive 분포 측정)
- Step G2 — 운영자 리뷰 누적 (1~2 주, 무기 테이블 + 회복 메커닉 검증)
- Step G3 — Mitigation 활성 (환경변수 토글, 즉시 롤백 가능)

상세 → [`modules/13-damage-validator.md`](./modules/13-damage-validator.md) §11

---

## 라이선스 요약

| 의존성 | 라이선스 | 클라이언트 영향 | 서버 영향 |
|--------|----------|----------------|----------|
| libsodium | ISC | 정적 링크 가능 | 정적 링크 가능 |
| WinHTTP | Windows SDK | 무 | 무 |
| FastAPI / SQLAlchemy / Alembic | MIT | 무 | 백엔드 의존 |
| PostgreSQL | PostgreSQL License | 무 | 데이터베이스 |
| Redis | RSAL/SSPL (7.x) 또는 BSD (6.x/Valkey) | 무 | 운영 정책 검토 필요 |
| Docker | Apache 2.0 | 무 | 배포 인프라 |

상세 → [`03-license-inventory.md`](./03-license-inventory.md)

---

## 패키지 구성

- 문서 21 개 (Tier 1~3 + 부속)
- 신규 코드 모듈 12 개 (`Security::*` 네임스페이스, `~2,400 LoC`)
- 통합 지점 명세 (Layer B — 의사코드 + 호출 시그니처)
- 검증 키트 (`docker compose up` + pytest 57/0 + HMAC E2E 3 시나리오)

---

## 권장 다음 단계

1. 본 Brief 검토 (5 분)
2. [`01-technical-brief.md`](./01-technical-brief.md) + [`02-before-after-comparison.md`](./02-before-after-comparison.md) 검토 (30 분)
3. 검증 키트 자체 재현 ([`06-validation-kit.md`](./06-validation-kit.md), 30 분)
4. Phase A~G 중 선별 도입 의향 결정

---

## 작성 주체 / 사용 조건

- 작성자: 이홍구 <2nugu@naver.com>
- 본 패키지는 자유 공개이며 NDA / 라이선스 협상 / 보상 절차 없이 채택 가능
- 작성자 권리: 공개 자료 기반 후속 분석 진행 + 비상업 OSS 공개 (자세한 내용은 README 참조)
