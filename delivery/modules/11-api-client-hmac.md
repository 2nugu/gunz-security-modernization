**English** | [한국어](../ko/modules/11-api-client-hmac.md)

# Module 11 — API Client (WinHTTP + HMAC-SHA256)

## Purpose
Sends signals asynchronously to community-api. WinHTTP (standard in the Windows SDK) + libsodium HMAC. No external dependencies (avoids extra libraries such as libcurl).

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

    // synchronous
    APIResponse SendCheatReports(const std::vector<CheatReport>& reports);

    // asynchronous (detached std::thread, fire-and-forget)
    void SendCheatReportsAsync(const std::vector<CheatReport>& reports);

    // retry queue (MAX 100, FIFO drop)
    void EnqueueRetry(const std::vector<CheatReport>& reports);
    void RetryPending();
};

}
```

## HMAC convention

```
body = JSON(reports)
sig  = hex(HMAC-SHA256(secret, body))

POST /api/anticheat/report HTTP/1.1
Host: <community-api>
Content-Type: application/json
X-Signature: <sig hex, 64 chars>
Content-Length: <body length>

<body>
```

Verification on the community-api side:
```python
def verify_signature(body: bytes, secret: str, header: str) -> bool:
    expected = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, header)
```

## JSON payload

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

## Response codes

| HTTP | Meaning |
|------|------|
| 200 | Success — DB insert complete |
| 401 | Signature verification failed — suspect secret mismatch |
| 409 | Duplicate (when the `Idempotency-Key` header is used) |
| 4xx | Client error — check the payload |
| 5xx | Server error — to the retry queue |

## Retry policy

```
Async call → synchronous call on a separate thread → branch on result
  200 / 4xx (other than 5xx): done (4xx is not retried either; a payload error is a permanent failure)
  5xx or network error: EnqueueRetry()

Retry queue:
  - MAX 100 (FIFO drop when exceeded)
  - RetryPending() called periodically (e.g. every 5 minutes)
  - no exponential backoff (simplification — can be added later)
```

## Integration Points
- New `CSCommon/Security/MAPIClient.{h,cpp}`
- In `MMatchServer::OnCreate`, read env and call `Init()` + register `SetReportCallback`
- Call `RetryPending()` from a periodic worker or the main loop

## Configuration

| Env var | Default | Meaning |
|---------|------|------|
| `COMMUNITY_API_URL` | (none) | Callback not registered when unset |
| `MATCH_RESULT_WEBHOOK_SECRET` | (none) | 32 B recommended |
| `API_RETRY_QUEUE_MAX` | 100 | Retry queue size |
| `API_RETRY_INTERVAL_MS` | 300000 | 5 minutes |
| `API_TIMEOUT_MS` | 5000 | Single-request timeout |

## Failure Modes
| Condition | Result |
|------|------|
| env not set | Callback not registered, memory-only operation (local development possible) |
| community-api down | Retry queue accumulates, 5xx retried |
| secret rotation | Update both sides simultaneously, then restart |
| Network disconnect | Retry queue accumulates, FIFO drop beyond 100 |
| HMAC mismatch | 401, operations alert |

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
→ DB query: SELECT * FROM anticheat_signals WHERE match_uuid=...
```

## Security considerations

### Secret rotation procedure
For the operational procedure see [`../05-operations-runbook.md`](../05-operations-runbook.md) §7.4 (managed together with the incident response flow).

Summary from this module's side: update the `MATCH_RESULT_WEBHOOK_SECRET` env var on both community-api and the match server simultaneously, then restart. During the short rotation window the retry queue may accumulate 401 responses, but it recovers automatically as long as it stays within the queue MAX of 100.

### Replay attack
- HMAC by itself does not detect replay
- An `X-Timestamp` header + server-side window check can be added later
- Currently deduplicated via `Idempotency-Key` on the community-api side

### Payload integrity
- Tampering detected via HMAC-SHA256
- Length limit: body MAX 1 MiB (community-api side policy)

## Limitations
- WinHTTP synchronous call + std::thread async wrapper — true asynchrony (`WinHttpQueryDataAvailable` callback) not implemented
- Simple retry backoff (fixed interval)
- HTTPS certificate verification (currently uses WinHTTP default behavior — separate configuration needed if the company uses an internal CA)

## Cross-References
- Signal emission: [`08-signal-collector.md`](./08-signal-collector.md)
- match_uuid: [`12-match-uuid-pipeline.md`](./12-match-uuid-pipeline.md)
- Validation: [`../06-validation-kit.md`](../06-validation-kit.md) §2.2
- Integration: [`../08-integration-guide.md`](../08-integration-guide.md) §2.11
