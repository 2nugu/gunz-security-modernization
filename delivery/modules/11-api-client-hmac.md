# Module 11 — API Client (WinHTTP + HMAC-SHA256)

## Purpose
시그널을 community-api 로 비동기 전송. WinHTTP (Windows SDK 기본) + libsodium HMAC. 외부 의존성 없이 (libcurl 등 추가 라이브러리 회피).

## Interface

```cpp
namespace Security {

struct CheatReport {
    uint32_t player_uid_high;
    uint32_t player_uid_low;
    int      character_id;
    char     signal_type[32];
    char     severity[16];
    float    value;
    float    threshold;
    char     match_uuid[65];
    char     detail[256];
    long long timestamp;
};

struct APIResponse {
    int  http_status;
    char body[1024];
};

class MAPIClient {
public:
    static MAPIClient& Instance();

    bool Init(const char* baseURL, const char* hmacSecret);

    // 동기
    APIResponse SendCheatReports(const std::vector<CheatReport>& reports);

    // 비동기 (detached std::thread, fire-and-forget)
    void SendCheatReportsAsync(const std::vector<CheatReport>& reports);

    // 재시도 큐 (MAX 100, FIFO drop)
    void EnqueueRetry(const std::vector<CheatReport>& reports);
    void RetryPending();
};

}
```

## HMAC 규약

```
body = JSON(reports)
sig  = hex(HMAC-SHA256(secret, body))

POST /api/anticheat/report HTTP/1.1
Host: <community-api>
Content-Type: application/json
X-Signature: <sig hex 64자>
Content-Length: <body length>

<body>
```

community-api 측 검증:
```python
def verify_signature(body: bytes, secret: str, header: str) -> bool:
    expected = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, header)
```

## JSON 페이로드

```json
{
    "reports": [
        {
            "player_uid_high": 1,
            "player_uid_low": 2,
            "character_id": 100,
            "signal_type": "WeaponSpoof",
            "severity": "High",
            "value": 1.0,
            "threshold": 0.0,
            "match_uuid": "stage-1-2-1714867200",
            "detail": "reported=Rocket eq=[Katana, AK47, Beretta]",
            "timestamp": 1714867220123
        }
    ]
}
```

## 응답 코드

| HTTP | 의미 |
|------|------|
| 200 | 성공 — DB 인서트 완료 |
| 401 | 서명 검증 실패 — secret 불일치 의심 |
| 409 | 중복 (`Idempotency-Key` 헤더 사용 시) |
| 4xx | 클라 오류 — 페이로드 검증 |
| 5xx | 서버 오류 — 재시도 큐로 |

## 재시도 정책

```
Async 호출 → 동기 호출 thread 분리 → 결과 분기
  200 / 4xx (5xx 외): 종료 (4xx 도 재시도 안 함, 페이로드 오류면 영구 실패)
  5xx 또는 네트워크 오류: EnqueueRetry()

재시도 큐:
  - MAX 100 (초과 시 FIFO drop)
  - RetryPending() 가 주기적 호출 (5 분 등)
  - exponential backoff 미적용 (단순화 — 향후 추가 가능)
```

## Integration Points
- `CSCommon/Security/MAPIClient.{h,cpp}` 신규
- `MMatchServer::OnCreate` 에서 env 읽어 `Init()` + `SetReportCallback` 등록
- 주기 worker 또는 메인 루프에서 `RetryPending()` 호출

## Configuration

| 환경변수 | 기본 | 의미 |
|---------|------|------|
| `COMMUNITY_API_URL` | (없음) | 미설정 시 callback 미등록 |
| `MATCH_RESULT_WEBHOOK_SECRET` | (없음) | 32 B 권장 |
| `API_RETRY_QUEUE_MAX` | 100 | 재시도 큐 크기 |
| `API_RETRY_INTERVAL_MS` | 300000 | 5 분 |
| `API_TIMEOUT_MS` | 5000 | 단일 요청 타임아웃 |

## Failure Modes
| 조건 | 결과 |
|------|------|
| env 미설정 | callback 미등록, 메모리-only 동작 (로컬 개발 가능) |
| community-api 다운 | 재시도 큐 누적, 5xx 재시도 |
| secret 회전 | 양측 동시 갱신 후 재시작 |
| 네트워크 단절 | 재시도 큐 누적, 100 초과 시 FIFO drop |
| HMAC mismatch | 401, 운영 알림 |

## Test Vectors

```
secret = "test-secret-32-bytes-min"
body   = '{"reports":[{"player_uid_high":1,...}]}'
sig    = hex(HMAC-SHA256(secret, body))
       = "e8f4a8..."

POST /api/anticheat/report
  X-Signature: e8f4a8...
  body

→ HTTP 200
→ DB 조회: SELECT * FROM anticheat_signals WHERE match_uuid=...
```

## 보안 고려사항

### secret 회전 절차
운영 절차는 [`../05-operations-runbook.md`](../05-operations-runbook.md) §7.4 참조 (사고 대응 흐름과 통합 관리).

본 모듈 측면 요약: `MATCH_RESULT_WEBHOOK_SECRET` 환경변수를 community-api 와 매치서버 양측 동시 갱신 후 재시작. 회전 중 짧은 윈도우에서 재시도 큐가 401 응답으로 누적될 수 있으나 큐 MAX 100 내라면 자동 복구.

### 재생 공격 (replay attack)
- HMAC 자체는 replay 검출 안 함
- 향후 `X-Timestamp` 헤더 + 서버측 윈도우 검증 추가 가능
- 현재는 community-api 측 `Idempotency-Key` 로 dedup

### 페이로드 무결성
- HMAC-SHA256 으로 변조 검출
- 길이 한도: body MAX 1 MiB (community-api 측 정책)

## Limitations
- WinHTTP 동기 호출 + std::thread async wrapper — 본격 비동기 (`WinHttpQueryDataAvailable` 콜백) 미구현
- 재시도 backoff 단순 (고정 간격)
- HTTPS 인증서 검증 (현재 WinHTTP 기본 동작 사용 — 회사 내부 CA 사용 시 별도 설정 필요)

## Cross-References
- 시그널 발행: [`08-signal-collector.md`](./08-signal-collector.md)
- match_uuid: [`12-match-uuid-pipeline.md`](./12-match-uuid-pipeline.md)
- 검증: [`../06-validation-kit.md`](../06-validation-kit.md) §2.2
- 통합: [`../08-integration-guide.md`](../08-integration-guide.md) §2.11
