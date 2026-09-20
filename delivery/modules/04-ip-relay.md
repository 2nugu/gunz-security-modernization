**English** | [한국어](../ko/modules/04-ip-relay.md)

# Module 04 — IP Relay (Peer Address Masking)

## Purpose
The original sends real IP/Port in plaintext in `MC_MATCH_RESPONSE_PEER_LIST` and the peer blob of `StageEnterBattle`. Direct cause of **DDoS targeting / IP harvesting / NAT bypass**.

## Approach
Unconditionally mask with `dwIP=0`, `nPort=0` at both call sites. Client P2P direct connection fails → `MC_MATCH_REQUEST_PEER_RELAY` fallback → the match server relays P2P traffic.

## Integration Points

### Site 1 — ResponsePeerList
```cpp
// MMatchServer.cpp::ResponsePeerList (~L2147)
for (auto& obj : peers) {
    PeerBlob blob;
    blob.uid    = obj->GetUID();
    blob.dwIP   = 0;       // [WAS] obj->GetIP();
    blob.nPort  = 0;       // [WAS] obj->GetPort();
    // ...
}
```

### Site 2 — StageEnterBattle
```cpp
// MMatchServer_Stage.cpp::StageEnterBattle (~L467)
PeerBlob blob;
blob.dwIP  = 0;     // [WAS] pObj->GetIP();
blob.nPort = 0;     // [WAS] pObj->GetPort();
```

### Branches to remove
The following branch, present in some original builds, is **removed entirely**:

```cpp
// removed
if (admin || eventTeam || forcedNAT) {
    blob.dwIP = obj->GetIP();
    blob.nPort = obj->GetPort();
} else {
    blob.dwIP = 0;
    blob.nPort = 0;
}
```

→ Unconditional masking to block privilege-escalation scenarios.

## Verification
Confirmed that across all of `CSCommon/Source` the `dwIP = pObj->GetIP()` / `nPort = pObj->GetPort()` pattern does not appear outside the two call sites. The remaining `GetIP()/GetPort()` calls are for server-internal logging / premium-IP cache / peer-address storage and are not sent to clients.

## Configuration
None. Masking is unconditionally active.

## Failure Modes
| Condition | Result |
|------|------|
| Client P2P punch-through attempt (0.0.0.0:0) | Fails → `MC_MATCH_REQUEST_PEER_RELAY` fallback |
| Increased match server load (P2P traffic relay burden) | Infrastructure side: increased match server NIC bandwidth + CPU |
| Increased latency for some P2P-only features (ballistics sync, etc.) | Measure, then compensate |

## Operational Impact

### Match server load estimate
- P2P traffic: per-player ~5 KB/s peer broadcast (BasicInfo 32 Hz)
- 8-player match → 8 × 7 = 56 P2P flows → ~35 KB/s received per client
- With match server relay: 8 × 5 KB/s = 40 KB/s in, 56 × 5 KB/s = 280 KB/s out per match

10K concurrent users (1250 matches) → match server relay traffic: ~50 MB/s in, ~350 MB/s out. 28 % utilization on a 1 Gbps NIC.

## Test Vectors
```
[Before]
PeerList packet dump → blob.dwIP = 0xC0A80101 (192.168.1.1)
                     blob.nPort = 7777

[After]
PeerList packet dump → blob.dwIP = 0
                     blob.nPort = 0

→ client P2P attempt → connect() 0.0.0.0:0 fails
→ MC_MATCH_REQUEST_PEER_RELAY sent automatically
→ match server relay begins
```

## Limitations
- The match server's own IP is exposed (unavoidable). Match server protection depends on an external layer (CDN / scrubber)
- A code review policy is needed so that no new IP-leaking path is added beyond `MServer::ResponsePeerList`

## Cross-References
- Integration: [`../08-integration-guide.md`](../08-integration-guide.md) §2.5
- Threat model: [`../01-technical-brief.md`](../01-technical-brief.md) §1 T3
