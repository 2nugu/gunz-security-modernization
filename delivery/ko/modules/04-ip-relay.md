[English](../../modules/04-ip-relay.md) | **한국어**

# Module 04 — IP Relay (Peer Address Masking)

## Purpose
오리지널은 `MC_MATCH_RESPONSE_PEER_LIST` 와 `StageEnterBattle` 의 peer blob 에 실 IP/Port 평문 송신. **DDoS 표적화 / IP 수집 / NAT 우회** 의 직접 원인.

## Approach
두 호출 지점에서 `dwIP=0`, `nPort=0` 로 무조건 마스킹. 클라 P2P 직결 실패 → `MC_MATCH_REQUEST_PEER_RELAY` 폴백 → 매치서버가 P2P 트래픽 중계.

## Integration Points

### 위치 1 — ResponsePeerList
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

### 위치 2 — StageEnterBattle
```cpp
// MMatchServer_Stage.cpp::StageEnterBattle (~L467)
PeerBlob blob;
blob.dwIP  = 0;     // [WAS] pObj->GetIP();
blob.nPort = 0;     // [WAS] pObj->GetPort();
```

### 제거 대상 분기
오리지널 일부 빌드에 존재했던 다음 분기는 **모두 제거**:

```cpp
// 제거
if (admin || eventTeam || forcedNAT) {
    blob.dwIP = obj->GetIP();
    blob.nPort = obj->GetPort();
} else {
    blob.dwIP = 0;
    blob.nPort = 0;
}
```

→ 권한 상승 시나리오 차단을 위해 무조건 마스킹.

## Verification
`CSCommon/Source` 전체에서 `dwIP = pObj->GetIP()` / `nPort = pObj->GetPort()` 패턴이 두 호출 지점 외에 없음을 확인. 남은 `GetIP()/GetPort()` 호출은 서버 내부 로깅/프리미엄IP캐시/피어주소저장 용도로 클라이언트로 송출되지 않음.

## Configuration
없음. 마스킹은 무조건 활성.

## Failure Modes
| 조건 | 결과 |
|------|------|
| 클라 P2P punch-through 시도 (0.0.0.0:0) | 실패 → `MC_MATCH_REQUEST_PEER_RELAY` 폴백 |
| 매치서버 부하 증가 (P2P 트래픽 중계 부담) | 인프라 측면에서 매치서버 NIC 대역폭 + CPU 증가 |
| 일부 P2P 전용 기능 (탄도 동기화 등) 의 latency 증가 | 측정 후 보정 필요 |

## 운영 영향

### 매치서버 부하 추정
- P2P 트래픽: per-player ~5 KB/s peer broadcast (BasicInfo 32 Hz)
- 8 명 매치 → 8 × 7 = 56 P2P 흐름 → 클라 1 명당 ~35 KB/s 수신
- 매치서버 중계 시: 8 × 5 KB/s = 40 KB/s 입력, 56 × 5 KB/s = 280 KB/s 출력 per match

10K 동접 (1250 매치) → 매치서버 중계 트래픽: ~50 MB/s in, ~350 MB/s out. 1 Gbps NIC 기준 28 % 사용.

## Test Vectors
```
[변경 전]
PeerList 패킷 덤프 → blob.dwIP = 0xC0A80101 (192.168.1.1)
                     blob.nPort = 7777

[변경 후]
PeerList 패킷 덤프 → blob.dwIP = 0
                     blob.nPort = 0

→ 클라 P2P 시도 → connect() 0.0.0.0:0 실패
→ MC_MATCH_REQUEST_PEER_RELAY 자동 송신
→ 매치서버 중계 시작
```

## Limitations
- 매치서버 자체 IP 는 노출됨 (필연). 매치서버 보호는 외부 레이어 (CDN / 스크러버) 에 의존
- `MServer::ResponsePeerList` 외에 IP 가 새는 경로가 신규로 추가되지 않도록 코드 리뷰 정책 필요

## Cross-References
- 통합: [`../08-integration-guide.md`](../08-integration-guide.md) §2.5
- 위협 모델: [`../01-technical-brief.md`](../01-technical-brief.md) §1 T3
