# Module 03 — DDoS 3계층 게이트

## Purpose
오리지널은 connect/packet/bandwidth 모두 무제한. 단일 IP 가 매치서버를 다운시키는 시나리오를 3계층으로 차단.

## 구성

| 레이어 | 위치 | 단위 | 기본값 |
|-------|------|------|--------|
| L1 | `RCP_IO_ACCEPT` | per-IP | 10 connections / 10 s |
| L2 | `RCP_IO_READ` | per-session | 256 packets / 1 s |
| L3 | `RCP_IO_READ` | per-session | 256 KiB / 1 s |

L1 은 **alloc 전 컷** — MCommObject 할당 전에 차단. L2/L3 는 세션 진입 후, IOCP 가 세션별 read 를 직렬화하므로 트래커 자체는 락 불필요.

## Interface — L1 (`MConnectRateLimit`)

```cpp
namespace Security {

class MConnectRateLimit {
public:
    static MConnectRateLimit& Instance();

    // true = 통과, false = 차단
    bool CheckAndRecord(const char* szIP);

    void SetParams(uint32_t windowMs, uint32_t maxConnects, uint32_t retentionMs);
    // 기본: window=10000, max=10, retention=60000

private:
    MConnectRateLimit();  // singleton
    // pimpl: unordered_map<string, deque<ms>> + lastSeen + GC
};

}
```

## Interface — L2 (`MPacketRateLimit`)

```cpp
namespace Security {

class MPacketRateLimit {
public:
    MPacketRateLimit();  // per-instance, 싱글톤 아님
    bool RecordAndCheck();  // 패킷 1 개 카운트
    void SetParams(uint32_t windowMs, uint32_t maxPackets);
    // 기본: window=1000, max=256
    // windowMs=0 또는 maxPackets=0 → 비활성
};

}
```

`MCommObject` 가 멤버로 보유: `Security::MPacketRateLimit m_PacketRate`. `GetPacketRate()` 접근자.

## Interface — L3 (`MBandwidthThrottle`)

```cpp
namespace Security {

class MBandwidthThrottle {
public:
    MBandwidthThrottle();  // per-instance
    bool RecordAndCheck(size_t bytes);
    void SetParams(uint32_t windowMs, uint64_t maxBytes);
    // 기본: window=1000, max=262144 (256 KiB)
    // windowMs=0 또는 maxBytes=0 → 비활성
};

}
```

## Integration Points
- `CSCommon/Security/MConnectRateLimit.{h,cpp}` 신규
- `CSCommon/Security/MPacketRateLimit.{h,cpp}` 신규
- `CSCommon/Security/MBandwidthThrottle.{h,cpp}` 신규
- `MServer::RCPCallback` 의 `RCP_IO_ACCEPT` / `RCP_IO_READ` 핸들러
- `MCommObject` 에 L2/L3 멤버 + 접근자

## 시간 소스
- `std::chrono::steady_clock` (64-bit 단조)
- 사유: `stdafx.h` 가 `_WIN32_WINNT=0x0501` 고정 → `GetTickCount64` 미노출

## Configuration

| 환경변수 | 기본 | 의미 |
|---------|------|------|
| `DDOS_CONNECT_PER_IP_PER_10S` | 10 | L1 max |
| `DDOS_CONNECT_WINDOW_MS` | 10000 | L1 window |
| `DDOS_CONNECT_RETENTION_MS` | 60000 | L1 GC retention |
| `DDOS_PACKETS_PER_SESSION_PER_S` | 256 | L2 max |
| `DDOS_PACKETS_WINDOW_MS` | 1000 | L2 window |
| `DDOS_BYTES_PER_SESSION_PER_S` | 262144 | L3 max |
| `DDOS_BYTES_WINDOW_MS` | 1000 | L3 window |

각 게이트는 `windowMs=0` 또는 `max=0` 으로 환경변수 통해 비활성 가능.

## Failure Modes
| 조건 | 결과 |
|------|------|
| 정상 트래픽이 임계 근접 → 간헐적 차단 | 임계 상향 (튜닝) |
| `unordered_map` 메모리 압박 (대규모 IP 분산 공격) | retention GC 단축 또는 LRU 도입 (향후) |
| GC 사이클 동안 lock 컨텐션 | (현재) 단일 mutex. 16-shard 도입은 향후 |

## Test Vectors

### L1 — Connect Flood
```
입력: 동일 IP 11 회 연속 connect
기대:
  1~10 회: CheckAndRecord → true
  11 회:   CheckAndRecord → false (차단)
  60 초 후: deque GC, 새 윈도우
```

### L2 — Packet Flood
```
입력: 1 초 내 300 패킷
기대:
  1~256: RecordAndCheck → true
  257~300: false (Disconnect)
```

### L3 — Bandwidth
```
입력: 1 초 내 100 × 4 KiB 패킷 = 400 KiB
기대:
  ~64 패킷째 (256 KiB 누적): false (Disconnect)
```

## Limitations
- L1 의 `unordered_map` 자체가 대규모 IP 분산 공격에서 메모리 압박 가능. 향후 LRU 또는 16-shard 분산
- IPv6 미지원 (현 GunZ 가 IPv4 전용)
- IP 화이트리스트 (정상 NAT 게이트웨이 등) 미지원

## Cross-References
- 통합: [`../08-integration-guide.md`](../08-integration-guide.md) §2.1, §2.2
- 운영: [`../05-operations-runbook.md`](../05-operations-runbook.md) §7.3
