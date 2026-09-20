# GunZ Security & Anti-Cheat Modernization Package

본 패키지는 GunZ 에 대한 개인적 애정에서 시작된 연구 작업물입니다.

GitHub 에 공개된 GunZ 트리 (GunZ-The-Duel, FGunZ, RefinedGunz, Gunz1.5-main 등) 를 베이스로 보안·안티치트 레이어 개선 컨셉을 잡았고, 실제로 구현해 공개 버전 기준 빌드가 동작하는 것까지 확인했습니다.

사측에서 본 작업이 도움이 된다고 판단하시면 그대로 채택하셔도 됩니다. 별도의 NDA, 라이선스 협상, 보상 같은 절차는 필요하지 않습니다.

다만 작업 도중 커피는 좀 마셨습니다 ☕

― 작성자가 보유하는 권리

1. 공개 자료 기반 후속 분석의 자유 진행
2. 비상업 OSS 라이선스 (MIT/ISC 등) 로 GitHub 공개

사측 채택 여부와 무관하게 위 권리는 유지됩니다.
GunZ 가 오래 살아남기를 바라며.

― 작성자: 이홍구 <2nugu@naver.com>
― 작성일: 2026-05-06

---

> **목적**: 오리지널 GunZ(2007 MAIET) 의 서버 보안·안정성·안티치트 레이어를 현대 표준으로 교체한 작업물의 전달용 패키지.
> **베이스**: 공개 GitHub 트리 (`source-reference1` GunZ-The-Duel 외) 위에서 신규 작성한 보안 레이어.

---

## 1. 이 패키지가 무엇이고 무엇이 아닌가

**이 패키지는**:
- 오리지널 프로토콜의 알려진 5개 공격 표면을 닫는 **현대 보안 레이어**
- DDoS 3계층 게이트, AES-256-GCM AEAD, X25519 ECDHE, 서버 권위 좌표 링버퍼
- GunZ 본서버 플레이 + 공개 트리 빌드 테스트에 기반한 **특화 안티치트 시그널 카탈로그**
- 의사결정자/엔지니어/운영자 각 독자별로 분리된 문서 트리

**이 패키지가 아닌**:
- 클라이언트 안티치트 (메모리 보호, DLL injection 방어 등은 다루지 않음)
- 라이브 GunZ 서비스에 직접 머지 가능한 patch (호환성 확인 + 회사 내부 트리 통합 작업 별도 필요)
- 게임 로직, UI, 그래픽 변경 (오리지널 게임플레이 보존)
- BSP 정밀 noclip 탐지, x64 빌드, TLS 1.3 로그인 채널 (계획 단계)

전체 미포함 항목은 [`07-whats-not-included.md`](./07-whats-not-included.md) 참조.

---

## 2. 독자별 진입점

| 역할 | 먼저 읽을 문서 | 시간 |
|------|-------------|------|
| **사업/기획 디렉터** | [`00-executive-brief.md`](./00-executive-brief.md) | 5 분 |
| **서버 팀 리드, 보안 담당** | [`01-technical-brief.md`](./01-technical-brief.md) → [`02-before-after-comparison.md`](./02-before-after-comparison.md) | 30 분 |
| **법무 / 컴플라이언스** | [`03-license-inventory.md`](./03-license-inventory.md) | 10 분 |
| **운영자 / GM 팀** | [`04-anticheat-catalog.md`](./04-anticheat-catalog.md) → [`05-operations-runbook.md`](./05-operations-runbook.md) | 20 분 |
| **통합 작업 엔지니어** | [`08-integration-guide.md`](./08-integration-guide.md) → [`modules/`](./modules/) | 모듈당 10~15 분 |
| **QA / 보안 검증팀** | [`06-validation-kit.md`](./06-validation-kit.md) | 30 분 (재현 포함) |

---

## 3. 디렉토리 인덱스

```
delivery/
├── README.md                          이 문서
├── 00-executive-brief.md              Tier 1 — 의사결정자용 (1~2 페이지)
├── 01-technical-brief.md              Tier 2 — 기술 디렉터용
├── 02-before-after-comparison.md      한 페이지 비교표
├── 03-license-inventory.md            의존성 라이선스 인벤토리
├── 04-anticheat-catalog.md            GunZ 특화 핵 분류 + 탐지 매핑
├── 05-operations-runbook.md           운영자 의사결정 흐름
├── 06-validation-kit.md               검증 재현 가이드
├── 07-whats-not-included.md           다루지 않은 영역 (정직성 신호)
├── 08-integration-guide.md            통합 지점 의사코드
└── modules/                           Tier 3 — 모듈별 레퍼런스 (13개)
    ├── 01-aes-gcm-crypter.md
    ├── 02-ecdhe-key-exchange.md
    ├── 03-ddos-rate-limit.md
    ├── 04-ip-relay.md
    ├── 05-position-history.md
    ├── 06-movement-validator.md
    ├── 07-combat-validator.md             (발사 빈도 / 명중률)
    ├── 08-signal-collector.md
    ├── 09-weapon-spoof-novel.md           (Novel 축 #1)
    ├── 10-position-lie-novel.md           (Novel 축 #2)
    ├── 11-api-client-hmac.md
    ├── 12-match-uuid-pipeline.md
    └── 13-damage-validator.md             (데미지 감소핵 / 무적핵 / HPHack)
```

---

## 4. 모듈 13개 한눈에

| # | 모듈 | 카테고리 | 구현 상태 | 위험도 | LoC (≈) |
|---|------|---------|----------|--------|--------|
| 01 | AES-256-GCM Crypter V2 | 암호화 | **구현 완료, 빌드 검증** | 중 (프로토콜 변경) | 280 |
| 02 | X25519 ECDHE Key Exchange | 키 교환 | **구현 완료, 빌드 검증** | 중 | 220 |
| 03 | DDoS 3계층 게이트 | 안정성 | **구현 완료, 빌드 검증** | 저 | 380 (3 모듈 합) |
| 04 | IP Relay (Peer Masking) | 토폴로지 | **구현 완료** | 저 | 30 (수정만) |
| 05 | Position History Ring Buffer | 안티치트 인프라 | **구현 완료, 빌드 검증** | 저 | 110 |
| 06 | Movement Validator | 안티치트 (이동) | **구현 완료, 빌드 검증** | 중 (튜닝 필요) | 240 |
| 07 | Combat Validator | 안티치트 (발사/명중) | **구현 완료, 빌드 검증** | 중 | 320 |
| 08 | Signal Collector (16-shard) | 안티치트 인프라 | **구현 완료, 빌드 검증** | 저 | 180 |
| 09 | **Weapon Spoof Detection** | 안티치트 (novel) | **구현 완료, 빌드 검증** | 저 | 60 |
| 10 | **Position Lie Detection** | 안티치트 (novel) | **구현 완료, 빌드 검증** | 저 | 80 |
| 11 | API Client (WinHTTP + HMAC) | 보고 파이프 | **구현 완료, 빌드 검증** | 저 | 220 |
| 12 | match_uuid Lifecycle | 운영성 | **구현 완료, 빌드 검증** | 저 | 50 |
| 13 | **Damage Validator** | 안티치트 (데미지/HP) | **구현 완료, 빌드 검증 (Detection-Only 모드)** | 중 (게임 메커닉 검토) | 617 (.h 190 + .cpp 427) |

**Novel 항목** (#09, #10) 은 본 프로젝트 외에 공개 ref 트리에 존재하지 않는 **고유 탐지 축**.

**Module 13** 은 victim-authoritative HIT_TICK 의 사각지대 (데미지 감소핵 / 무적핵 / HPHack) 를 닫는다. **Detection-Only 모드** 로 구현 완료 (빌드 검증 통과). **Mitigation 모드** (`SERVER_DAMAGE_OVERRIDE=true`) 활성화는 운영 환경의 무기 데미지 테이블 import + 회복/스킬 메커닉 enumeration 후 환경변수 토글로 즉시 적용 가능. 본 패키지의 기본 무기 프로파일은 공개 트리 자체 빌드 환경 측정값으로 초기화되어 있으며 라이브 도입 시 교체 권장.

---

## 5. 출처 / 베이스 트리 명시

본 작업은 다음 공개 자료를 베이스로 한다:

| 트리 | 공개 위치 | 용도 |
|------|----------|------|
| GunZ-The-Duel | GitHub 공개 | 커뮤니티 모드 / 클라이언트 베이스 |
| RefinedGunz | GitHub 공개 | CMake/C++14 구조 참조 |
| Gunz1.5-main (`source-reference8`) | GitHub 공개 | 1.5 버전 자산/소스 |
| `source-reference11` | GitHub 공개 | UI 베이스 |

본 패키지에 포함된 **신규 코드 (`Security::*` 네임스페이스)** 는 위 트리에 존재하지 않는다. 라이브 GunZ 서비스가 동등하거나 더 발전된 자체 솔루션을 이미 운용 중일 가능성을 인정하며, 이 경우 본 패키지의 가치는 (a) GunZ 본서버 플레이 + 공개 트리 빌드 테스트 기반 핵 패턴 카탈로그와 (b) 공개 트리 자체 빌드 환경에서 측정된 튜닝 값에 있다.

---

## 6. 연락 / 사용 조건

본 패키지의 모든 자료 (문서 + 코드 + 통합 가이드 + 검증 키트) 는 **자유 공개**입니다. NDA 단계, 라이선스 협상, 보상 절차 없습니다.

사측이 채택을 결정하시면 그대로 사용하시면 되고, 채택하지 않으셔도 무방합니다. 두 경우 모두 본 README 첫머리의 작성자 권리 2 항목은 유지됩니다.

문의: 이홍구 <2nugu@naver.com>

---

## 7. 변경 이력

| 일자 | 변경 |
|------|------|
| 2026-05-06 | 초판 |
