**English** | [한국어](../ko/modules/03-ddos-rate-limit.md)

# Module 03 — DDoS Three-Tier Gate

## Purpose
The original has no limits on connect/packet/bandwidth. Blocks the scenario where a single IP takes down the match server, in three tiers.

## Structure

| Layer | Location | Unit | Default |
|-------|------|------|--------|
| L1 | `RCP_IO_ACCEPT` | per-IP | 10 connections / 10 s |
| L2 | `RCP_IO_READ` | per-session | 256 packets / 1 s |
| L3 | `RCP_IO_READ` | per-session | 256 KiB / 1 s |

L1 is a **pre-alloc cut** — blocks before MCommObject allocation. L2/L3 run after session entry; since IOCP serializes reads per session, the trackers themselves need no lock.

## Interface — L1 (`MConnectRateLimit`)

```cpp
namespace Security {

class MConnectRateLimit {
public:
    static MConnectRateLimit& Instance();

    // true = pass, false = block
    bool CheckAndRecord(const char* szIP);

    void SetParams(uint32_t windowMs, uint32_t maxConnects, uint32_t retentionMs);
    // defaults: window=10000, max=10, retention=60000

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
    MPacketRateLimit();  // per-instance, not a singleton
    bool RecordAndCheck();  // counts 1 packet
    void SetParams(uint32_t windowMs, uint32_t maxPackets);
    // defaults: window=1000, max=256
    // windowMs=0 or maxPackets=0 → disabled
};

}
```

Held as a member by `MCommObject`: `Security::MPacketRateLimit m_PacketRate`. Accessor `GetPacketRate()`.

## Interface — L3 (`MBandwidthThrottle`)

```cpp
namespace Security {

class MBandwidthThrottle {
public:
    MBandwidthThrottle();  // per-instance
    bool RecordAndCheck(size_t bytes);
    void SetParams(uint32_t windowMs, uint64_t maxBytes);
    // defaults: window=1000, max=262144 (256 KiB)
    // windowMs=0 or maxBytes=0 → disabled
};

}
```

## Integration Points
- `CSCommon/Security/MConnectRateLimit.{h,cpp}` new
- `CSCommon/Security/MPacketRateLimit.{h,cpp}` new
- `CSCommon/Security/MBandwidthThrottle.{h,cpp}` new
- `RCP_IO_ACCEPT` / `RCP_IO_READ` handlers in `MServer::RCPCallback`
- L2/L3 members + accessors in `MCommObject`

## Time Source
- `std::chrono::steady_clock` (64-bit monotonic)
- Reason: `stdafx.h` pins `_WIN32_WINNT=0x0501` → `GetTickCount64` not exposed

## Configuration

| Environment variable | Default | Meaning |
|---------|------|------|
| `DDOS_CONNECT_PER_IP_PER_10S` | 10 | L1 max |
| `DDOS_CONNECT_WINDOW_MS` | 10000 | L1 window |
| `DDOS_CONNECT_RETENTION_MS` | 60000 | L1 GC retention |
| `DDOS_PACKETS_PER_SESSION_PER_S` | 256 | L2 max |
| `DDOS_PACKETS_WINDOW_MS` | 1000 | L2 window |
| `DDOS_BYTES_PER_SESSION_PER_S` | 262144 | L3 max |
| `DDOS_BYTES_WINDOW_MS` | 1000 | L3 window |

Each gate can be disabled via environment variable with `windowMs=0` or `max=0`.

## Failure Modes
| Condition | Result |
|------|------|
| Normal traffic near threshold → intermittent blocking | Raise threshold (tuning) |
| `unordered_map` memory pressure (large distributed-IP attack) | Shorten retention GC or introduce LRU (future) |
| Lock contention during GC cycle | (Currently) single mutex. 16-shard split is future work |

## Test Vectors

### L1 — Connect Flood
```
Input: 11 consecutive connects from the same IP
Expected:
  Connects 1~10: CheckAndRecord → true
  Connect 11:    CheckAndRecord → false (blocked)
  After 60 s:    deque GC, new window
```

### L2 — Packet Flood
```
Input: 300 packets within 1 s
Expected:
  1~256: RecordAndCheck → true
  257~300: false (Disconnect)
```

### L3 — Bandwidth
```
Input: 100 × 4 KiB packets within 1 s = 400 KiB
Expected:
  Around packet ~64 (256 KiB cumulative): false (Disconnect)
```

## Limitations
- L1's `unordered_map` itself can come under memory pressure in a large distributed-IP attack. Future: LRU or 16-shard split
- No IPv6 support (current GunZ is IPv4-only)
- No IP whitelist (legitimate NAT gateways, etc.)

## Cross-References
- Integration: [`../08-integration-guide.md`](../08-integration-guide.md) §2.1, §2.2
- Operations: [`../05-operations-runbook.md`](../05-operations-runbook.md) §7.3
