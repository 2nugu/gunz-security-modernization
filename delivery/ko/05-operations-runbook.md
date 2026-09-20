[English](../05-operations-runbook.md) | **한국어**

# Operations Runbook

> **독자**: 운영자 / GM 팀 / 라이브 운영 엔지니어
> **목적**: 시그널 수신 → 판단 → 제재의 일상 운영 흐름.

---

## 1. 일상 작업 흐름 (Daily Loop)

```
┌─────────────────────────────────────────────────┐
│ 1. 운영자 큐 확인 (시작 시 / 4 시간 단위)        │
│    └─ community-api 대시보드 / 슬랙 알림        │
└─────────────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────┐
│ 2. 우선순위 정렬                                 │
│    Critical > High > Medium                     │
│    가중 점수 ≥ 임계 (기본 50) 항목 우선          │
└─────────────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────┐
│ 3. 케이스별 처리                                 │
│    a. 시그널 상세 검토 (type/value/threshold)   │
│    b. match_uuid 로 라운드 단위 집계 확인       │
│    c. 필요 시 ghost-mode 로 모니터링            │
│    d. 필요 시 리플레이 검토 (Phase 6 도입 시)   │
└─────────────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────┐
│ 4. 판정                                          │
│    ├─ 정상 (false positive) → 노이즈 레이블링   │
│    ├─ 핵 의심 (단발) → 경고 발송                │
│    ├─ 핵 확정 (반복) → 일시 정지                │
│    └─ 핵 확정 (악성) → 영구 정지                │
└─────────────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────┐
│ 5. 사후 기록                                     │
│    - 판정 근거 (signal_type, severity, value)   │
│    - 처리 결과                                   │
│    - 임계 조정 데이터 (월별 주기로 튜닝팀 전달) │
└─────────────────────────────────────────────────┘
```

---

## 2. 시그널 타입별 응답 가이드

### 2.1 Movement 계열 (100~199)

| 시그널 | 첫 응답 | 비고 |
|--------|--------|------|
| `SpeedHack` (101) | High 이상 → ghost-mode 30 분 | 합법 대시 ~1500 u/s 통과, 1800+ 컷 |
| `TeleportSuspect` (102) | Medium 이상 → match_uuid 단위 누적 확인 | 리스폰/워프는 면제됨. 면제 우회 의심 시 검토 |
| `FlyHack` (103) | Medium 캡 → 단독 판정 금지 | airtime heuristic, 점프맵 false-positive 가능 |

### 2.2 Combat 계열 (200~299)

| 시그널 | 첫 응답 | 비고 |
|--------|--------|------|
| `RapidFire` (201) | High → ghost-mode | 무기 클래스별 임계 |
| `InfiniteSlash` (202) | High → ghost-mode | Melee 450±100 ms |
| `AutoAim` (203) | Medium → 매치 통계 누적 | 50발 이후 hit-rate > 95% 단독 판정 금지, 패턴 누적 필요 |
| `ImpossibleHit` (204) | High → 단독 판정 가능 | range × 1.3 초과 |
| `WeaponSpoof` (205) | **Critical → 즉시 ghost-mode + 알림** | 서버-권위 데이터 기반, 거의 false-positive 없음 |
| `DamageReduce` (220) | High → ghost-mode | victim 보고 dmg < 서버 계산 × 0.7. 합법 회복/스킬 등록 누락 시 false-positive 가능 — 회복 메커닉 enum 점검 |
| `DamageInflate` (221) | High → 단독 판정 가능 | victim 보고 dmg > 서버 계산 × 1.3. 데미지 증폭핵 (1-shot 의심) |
| `DamageNullification` (222) | High → ghost-mode | 공격자 PEER_SHOT 후 HIT_TICK 300 ms 내 미도달. 무적핵/HIT_TICK 차단 의심 |
| `HPHack` (223) | **Critical → 즉시 ghost-mode + 알림** | HP 변화량 vs 보고 hit 합 불일치 5+ 차이. HP 메모리 조작핵, false-positive 거의 없음 |
| `HPInconsistency` (224) | Medium → 매치 단위 누적 | HP 가 알 수 없는 이유로 회복. 합법 회복 메커닉 등록 후 재발생 시 High 승급 |

**Damage 계열 (220~224) 운용 주의**:
- Module 13 의 도입 모드에 따라 의미 다름
  - **Detection-Only Mode**: 시그널만 발행, 핵 효과 무력화 안 됨. 운영자 수동 제재로 대응
  - **Mitigation Mode**: 시그널 발행 + victim HP 차감을 서버 계산값으로 강제 → 핵 효과 자체 무력화
- Module 13 도입 첫 1~2 주는 Detection-Only 로 운용 → false-positive 분포 측정 → tolerance 튜닝 → Mitigation 활성

### 2.3 Network 계열 (500~599)

| 시그널 | 첫 응답 | 비고 |
|--------|--------|------|
| `PacketManipulation` (501) | High → ghost-mode | PositionLie. 무기별 threshold 2 배 초과 시 High |
| `ReplayAttack` (502) | (자동 차단됨, AEAD nonce) | 알림만, 운영 조치 불필요 |

---

## 3. Ghost Mode (관리자 투명 접속) 운용

### 3.1 진입 방법
- 매치서버 콘솔 명령 `/ghost <player_uid>` (구현 예정)
- 또는 community-api 어드민 라우터 호출

### 3.2 ghost 상태에서 보이는 것
- 의심 플레이어의 1인칭 시야 (선택적)
- 시그널 실시간 스트림 (해당 플레이어만 필터)
- 매치 내 모든 플레이어 위치 (디버그 뷰)

### 3.3 ghost 상태에서 *보이지 않는* 것
- 의심 플레이어에게 ghost 자체가 노출되지 않음 (peer broadcast 제외)
- 게임 결과에 영향을 주지 않음 (사망 처리, 점수 등)

### 3.4 운영 원칙
- ghost 진입 사실 자체를 로그 보존 (악용 방지)
- ghost 시간 한도 (기본 60 분), 갱신 가능
- ghost 결과를 케이스 노트에 첨부

---

## 4. 임계값 튜닝 가이드

### 4.1 튜닝 주기
- 월 1 회 정기 검토
- 거짓 양성 보고 누적 시 임시 검토
- 신규 핵 출현 시 이벤트 검토

### 4.2 튜닝 가능 환경변수

| 환경변수 | 기본값 | 의미 |
|---------|-------|------|
| `MOVEMENT_MAX_SPEED_UPS` | 1800 | Speed 임계 (u/s) |
| `MOVEMENT_TELEPORT_DIST_PER_SEC` | 5000 | Teleport 임계 (u/s) |
| `MOVEMENT_FLYHACK_AIRTIME_MS` | 5000 | FlyHack airtime 임계 |
| `COMBAT_MELEE_INTERVAL_MS` | 450 | Melee 발사 간격 |
| `COMBAT_AUTOAIM_MIN_SHOTS` | 50 | AutoAim 최소 발사 수 |
| `COMBAT_AUTOAIM_HIT_RATE` | 0.95 | AutoAim hit-rate 임계 |
| `POSITIONLIE_MELEE_DIST` | 400 | PositionLie Melee threshold (units) |
| `POSITIONLIE_ROCKET_DIST` | 1000 | PositionLie Rocket threshold |
| `DDOS_CONNECT_PER_IP_PER_10S` | 10 | L1 connect rate |
| `DDOS_PACKETS_PER_SESSION_PER_S` | 256 | L2 packet rate |
| `DDOS_BYTES_PER_SESSION_PER_S` | 262144 | L3 bandwidth (256 KiB) |
| `SIGNAL_REPORT_THRESHOLD_SCORE` | 50 | 가중 점수 보고 임계 |
| `SERVER_DAMAGE_OVERRIDE` | false | Module 13 — false=Detection-Only, true=Mitigation |
| `DAMAGE_TOLERANCE_LOWER` | 0.7 | 보고 dmg < 서버 dmg × 0.7 → DamageReduce |
| `DAMAGE_TOLERANCE_UPPER` | 1.3 | 보고 dmg > 서버 dmg × 1.3 → DamageInflate |
| `DAMAGE_NULLIFICATION_TIMEOUT_MS` | 300 | PEER_SHOT 후 HIT_TICK 대기 시간 |
| `HP_CONSISTENCY_WINDOW_MS` | 5000 | HP 일관성 검증 윈도우 |
| `HP_CONSISTENCY_TOLERANCE` | 5.0 | HP 절대값 허용 오차 |

### 4.3 튜닝 절차
1. 거짓 양성 / 거짓 음성 사례 수집 (월별)
2. 임계값 후보 결정 (기존 ±10~30%)
3. **dev/staging 환경에서 1 주 적용**
4. 거짓 양성/음성 비율 비교
5. 적용 또는 롤백

### 4.4 튜닝 시 주의
- 단일 환경변수 변경 → 단일 시그널 영향
- 여러 변수 동시 변경 시 효과 분리 어려움
- 변경 이력은 git commit + 운영 로그 양쪽에 기록

---

## 5. 거짓 양성 / 음성 보고 흐름

### 5.1 거짓 양성 (정상 유저가 플래그)
```
유저 신고 → CS 접수 → 운영자 1차 검토 (ghost / 리플레이)
         → 정상 판정 → 케이스 종료 + 임계 튜닝 데이터 누적
         → 핵 판정 → 정식 처리 (3 단계 재진입)
```

### 5.2 거짓 음성 (핵 유저가 미플래그)
```
플레이어 신고 → CS 접수 → 운영자 매치 ID 추적 (match_uuid)
            → 시그널 미발생 확인 → 신규 핵 패턴 가능성
            → 보안팀에 시그널 추가 작업 요청
```

---

## 6. 신규 핵 출현 시 대응

```
1. 사례 수집 (CS 접수 + 직접 관찰)
2. 패턴 정의
   - 어떤 패킷이 어떻게 변조되는가
   - 어떤 게임 상태가 비정상인가
   - 서버측에서 관찰 가능한 신호는 무엇인가
3. 시그널 추가 작업 (보안팀)
   - CheatSignalType enum 신규 ID 할당
   - Validator 또는 inline 핸들러 추가
   - 임계값 초기 추정
4. dev/staging 검증
5. production 단계 도입
   - 첫 1 주: Low severity 만 발행 (관찰)
   - 거짓 양성 확인 후 적정 severity 적용
6. 운영자 가이드 업데이트
```

---

## 7. 비상 시나리오

### 7.1 anticheat 모듈이 false-positive 폭증
- 즉시 환경변수로 해당 시그널 disable: `SIGNAL_<TYPE>_ENABLED=false` (구현 시)
- 또는 시그널 보고 임계 상향 (`SIGNAL_REPORT_THRESHOLD_SCORE` 상향)
- 운영자 큐 임시 정리
- **Module 13 (Damage) 만의 케이스**: Mitigation Mode 에서 false-positive 발생 시 즉시 `SERVER_DAMAGE_OVERRIDE=false` 토글 → Detection-Only 로 복귀 (핵 효과는 다시 작동, 다만 데미지 흐름 정상화). 매치서버 재시작 후 적용.

### 7.2 community-api 다운 → 시그널 보고 실패
- `MAPIClient` 의 retry 큐 (MAX 100) 가 임시 보존
- 100 초과 시 FIFO drop — 시그널 손실 발생
- 복구 후 자동 재전송
- **동작에는 영향 없음** (안티치트 fail-open)

### 7.3 DDoS 게이트 false-positive (정상 유저 차단)
- IP 화이트리스트 도입 (구현 시)
- 또는 임계 임시 상향 (`DDOS_CONNECT_PER_IP_PER_10S` 등)

### 7.4 매치서버 ↔ community-api HMAC secret 유출
- secret 즉시 회전 (32B 신규 생성)
- `MATCH_RESULT_WEBHOOK_SECRET` 양측 동시 갱신
- 매치서버 + community-api 재시작
- 사고 보고서 작성 (회전 사실 + 시점 + 영향 범위)

---

## 8. 메트릭 / 모니터링 권장 항목

| 메트릭 | 빈도 | 알림 임계 |
|--------|------|----------|
| 시간당 시그널 수 (전체) | 1 분 | 평소 평균 × 3 |
| 시간당 시그널 수 (타입별) | 1 분 | 타입별 평균 × 5 |
| `MAPIClient` 재시도 큐 깊이 | 1 분 | 50 이상 |
| `MAPIClient` 5xx 응답률 | 1 분 | 5% 이상 |
| DDoS L1 차단 IP 수 | 5 분 | 평소 평균 × 5 |
| DDoS L2/L3 위반 세션 수 | 5 분 | 평소 평균 × 5 |
| 매치서버 메모리 사용량 | 1 분 | 80% 이상 |
| `anticheat_signals` 테이블 인서트 율 | 5 분 | (기준치 추후 결정) |

---

## 9. 운영자 권한 분리 (RBAC 권장)

| 역할 | 권한 |
|------|------|
| **CS** | 시그널 조회 / 케이스 노트 작성 |
| **GM (Junior)** | + ghost-mode (한도 제한) / 경고 발송 |
| **GM (Senior)** | + 일시 정지 / 임계 튜닝 제안 |
| **Security** | + 영구 정지 / 임계 튜닝 적용 / Validator 비활성화 |
| **Engineering** | + 환경변수 변경 / 코드 수정 |

자동 밴 권한은 어느 역할에도 부여하지 않음.

---

## 10. 운영자 교육 체크리스트

신규 운영자 온보딩 시:

- [ ] 본 Runbook 통독
- [ ] [`04-anticheat-catalog.md`](./04-anticheat-catalog.md) 통독
- [ ] 금지 시그널 (§3, 카탈로그) 이해 — 헤드샷 비율 등
- [ ] 자동 밴 금지 정책 이해
- [ ] ghost-mode 사용법 실습
- [ ] match_uuid 로 라운드 추적 실습
- [ ] 거짓 양성 사례 5건 시뮬레이션
- [ ] 신규 핵 보고 양식 작성 실습
