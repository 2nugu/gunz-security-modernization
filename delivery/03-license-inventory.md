# License Inventory

> **독자**: 법무 / 컴플라이언스 / 보안 거버넌스. 의존성 도입 가부 판단.
> **원칙**: 클라이언트(상용 게임 바이너리) 와 서버(백엔드 인프라) 의 라이선스 영향을 분리해서 평가.

---

## 1. 의존성 일람

### 1.1 클라이언트 (Gunz.exe) 측

| 라이브러리 | 라이선스 | SPDX | 정적/동적 | 영향 |
|----------|---------|------|---------|------|
| libsodium 1.0.20 | ISC | `ISC` | 정적 가능 | **상업 배포 가능, 라이선스 전파 없음** |
| WinHTTP | Windows SDK EULA | (proprietary) | 동적 (system) | Windows 플랫폼 의존, 추가 영향 없음 |
| Microsoft Detours | MIT | `MIT` | (사용 시) 정적 | 본 패키지 미사용 |
| 기존 GunZ 의존성 | (불변) | — | — | 본 작업이 추가하지 않음 |

**클라이언트 측 신규 라이선스 의무**:
- libsodium ISC 라이선스 텍스트를 게임 EULA / Credits / About 에 포함
- 그 외 의무 없음 (GPL, LGPL 같은 copyleft 의무 0)

### 1.2 서버 (MatchServer.exe) 측

| 라이브러리 | 라이선스 | SPDX | 정적/동적 | 영향 |
|----------|---------|------|---------|------|
| libsodium 1.0.20 | ISC | `ISC` | 정적 | 동일 |
| WinHTTP | Windows SDK EULA | (proprietary) | 동적 | 동일 |

서버는 사내 인프라이므로 외부 배포 라이선스 의무 없음.

### 1.3 백엔드 (community-api) 측

| 라이브러리 | 라이선스 | SPDX | 영향 |
|----------|---------|------|------|
| Python | PSF License | `PSF-2.0` | 무영향 |
| FastAPI | MIT | `MIT` | 무영향 |
| Starlette | BSD-3-Clause | `BSD-3-Clause` | 무영향 |
| SQLAlchemy | MIT | `MIT` | 무영향 |
| Alembic | MIT | `MIT` | 무영향 |
| pydantic | MIT | `MIT` | 무영향 |
| psycopg2 | LGPL with exception | `LGPL-3.0-or-later WITH PSPCOPG-EXCEPTION` | LGPL 이지만 동적 링크 + 예외 조항 → 사내 운영 무영향 |
| pytest | MIT | `MIT` | 테스트 전용 |
| uvicorn | BSD-3-Clause | `BSD-3-Clause` | 무영향 |

### 1.4 인프라

| 컴포넌트 | 라이선스 | 영향 |
|---------|---------|------|
| PostgreSQL 16 | PostgreSQL License | 무영향 (BSD-like) |
| Redis 6.x | BSD-3-Clause | 무영향 |
| Redis 7.x+ | RSAL/SSPL (dual) | **운영 정책 검토 필요** — SaaS 재판매 시 영향. 사내 운영은 영향 없음 |
| Valkey (Redis 7 fork) | BSD-3-Clause | Redis 7.x 대안, 영향 없음 |
| Docker Engine | Apache 2.0 | 무영향 |
| Python (3.11+) | PSF License | 무영향 |

**Redis 권장**:
- 사내 운영만이면 Redis 7.x 그대로 사용 가능
- SaaS 형태로 재판매할 가능성이 있다면 **Redis 6.x 또는 Valkey** 권장
- 본 패키지는 Redis 의존성을 캐시 계층으로만 사용 (필수 아님, 비활성화 가능)

---

## 2. 라이선스 호환성 매트릭스

```
도입 라이브러리  →  본 작업 코드  →  GunZ 클라이언트(상용)
   (ISC)              (회사 정책)        (회사 IP)
   호환 ✓               호환 ✓              영향 없음 ✓

도입 라이브러리  →  본 작업 코드  →  GunZ 서버(사내)
   (ISC/MIT)            (회사 정책)        (회사 IP)
   호환 ✓               호환 ✓              영향 없음 ✓
```

본 패키지 도입 시 발생하는 **유일한 의무**: libsodium ISC 라이선스 텍스트 포함.

---

## 3. libsodium ISC 라이선스 전문

```
ISC License

Copyright (c) 2013-2024
Frank Denis <j at pureftpd dot org>

Permission to use, copy, modify, and/or distribute this software for any
purpose with or without fee is hereby granted, provided that the above
copyright notice and this permission notice appear in all copies.

THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
```

---

## 4. 본 패키지 코드의 라이선스

본 패키지에 포함된 신규 코드 (`Security::*` 네임스페이스, FastAPI 라우터 등) 는 **사측 자유 사용**. NDA, 라이선스 협상, 양도 계약 등의 절차 없이 채택 가능.

작성자가 보유하는 권리는 README 첫머리에 명시된 2 항목 (공개 자료 기반 후속 분석 + 비상업 OSS 공개) 이며, 사측 채택 여부와 무관하게 유지됩니다.

추가 외부 의존성 라이선스 의무는 §3 의 libsodium ISC 텍스트 포함 한 가지뿐.

---

## 5. 출처 / 베이스 트리 라이선스

본 작업은 다음 공개 트리를 참조했다:

| 트리 | 공개 위치 | 라이선스 표기 |
|------|----------|--------------|
| GunZ-The-Duel | GitHub | (저장소 LICENSE 파일 참조) |
| RefinedGunz | GitHub | (저장소 LICENSE 파일 참조) |
| Gunz1.5-main | GitHub | (저장소 LICENSE 파일 참조) |

본 패키지에 포함된 신규 코드 (`Security::*`) 는 위 트리에 존재하지 않는다. 통합 가이드 ([`08-integration-guide.md`](./08-integration-guide.md)) 가 참조하는 패치 사이트 (`MServer.cpp`, `MMatchServer.cpp` 등) 의 파일 자체는 회사가 보유하는 동등 위치에 통합된다 — **본 패키지가 회사 측 소스를 재배포하지 않음**.

---

## 6. 컴플라이언스 체크리스트

회사가 본 패키지 도입 시 진행할 항목:

- [ ] libsodium ISC 라이선스 텍스트를 게임 EULA / Credits / 시작 화면 라이선스 표시에 포함
- [ ] Redis 7.x 사용 시 SaaS 재판매 가능성 검토 → 필요 시 Redis 6.x 또는 Valkey 로 다운그레이드
- [ ] 신규 의존성을 회사 OSS 검토 시스템에 등록
- [ ] (선택) 정적 분석 / 보안 감사 — 사내 보안팀 검토

---

## 7. 회사가 자주 묻는 질문 (선제 답변)

**Q. libsodium 대신 OpenSSL 도입 가능한가?**
A. 가능. AES-256-GCM 은 OpenSSL `EVP_aead_aes_256_gcm`, X25519 는 `EVP_PKEY_X25519` 로 동등 구현. 다만 OpenSSL 은 Apache 2.0 (3.0+) 또는 dual SSLeay+OpenSSL (1.1.1) 로 의존성 분량 큼. libsodium 이 ISC 단일이라 컴플라이언스 단순.

**Q. AES-NI 미지원 호스트는?**
A. `MPacketCrypterV2::InitKey` 가 즉시 false 반환 → 핸드셰이크 단계에서 명시적 실패. 이전 침묵 드롭 회귀 방지. 라이브 서비스 도입 시 모든 매치서버 호스트에 AES-NI 보유 확인 필수.

**Q. WinHTTP 외 대안은?**
A. cURL (MIT/X derivative, 무영향) 또는 회사 사내 HTTP 클라이언트 사용 가능. WinHTTP 선택 이유는 외부 의존성 0 (Windows SDK 기본).

**Q. 본 패키지의 코드를 회사 자체 트리에 머지하면 작성자가 나중에 권리 주장 가능한가?**
A. 본 패키지는 자유 사용으로 공개되어 있으며 사측은 별도 계약 없이 채택 가능. 작성자는 (a) 공개 자료 기반 후속 분석 진행 (b) 비상업 OSS 라이선스로 GitHub 공개 두 권리만 보유하며, 이는 사측 채택과 무관하게 유지됨. 사측이 본 패키지를 채택하든 안 하든 그 외의 권리 주장 의도는 없음.
