# Module 02 — X25519 ECDHE Key Exchange

## Purpose
세션마다 임시 키쌍으로 대칭 키 합의. **Forward Secrecy** 확보.

## Interface

```cpp
namespace Security {

class KeyExchangeState {
public:
    KeyExchangeState();
    ~KeyExchangeState();   // 비밀키 wipe

    // 서버 시작 시
    bool ServerGenerateKeyPair();
    const uint8_t* GetServerPublic() const;  // 32B

    // 클라이언트 — 서버 공개키 수신 후
    bool ClientHandleChallenge(const uint8_t* serverPub);
    const uint8_t* GetClientPublic() const;  // 32B

    // 서버 — 클라 공개키 수신 후
    bool ServerDeriveSessionKeys(const uint8_t* clientPub);

    // 양측 — KDF 입력 추출
    const uint8_t* GetSharedRx() const;  // 32B
    const uint8_t* GetSharedTx() const;  // 32B

    void WipePrivateKey();
};

// canonical min‖max IKM → BLAKE2b → 32B 대칭 키
bool DeriveSymmetricKey(const KeyExchangeState* kxState, uint8_t outKey32[32]);

}
```

## KDF — Canonical min‖max 순서

```cpp
bool DeriveSymmetricKey(...) {
    const uint8_t* rx = kxState->GetSharedRx();
    const uint8_t* tx = kxState->GetSharedTx();

    // 핵심: memcmp(rx, tx, 32) <= 0 기준 정렬
    const uint8_t *lo, *hi;
    if (memcmp(rx, tx, 32) <= 0) { lo = rx; hi = tx; }
    else                          { lo = tx; hi = rx; }

    uint8_t ikm[64];
    memcpy(ikm,      lo, 32);
    memcpy(ikm + 32, hi, 32);

    crypto_generichash_blake2b(outKey32, 32, ikm, 64, NULL, 0);
    sodium_memzero(ikm, 64);
    return true;
}
```

**왜 `min‖max` 인가**:
- libsodium `crypto_kx_*_session_keys` 가 미러-분할 (서버 rx == 클라 tx)
- `rx‖tx` 순서로 해시하면 서버/클라 IKM 갈라짐 → 대칭 키 불일치 → 첫 LOGIN 패킷 디크립트 실패
- `memcmp` 기준 정렬은 양측 동일 → IKM 일치 보장

**`#ifdef BUILD_MATCH_SERVER` 분기 금지** — 이 코드 단일 사본이 CSCommon, match-server, netprobe 세 곳 모두에 동일하게 존재해야 함.

## Integration Points
- `CSCommon/Include/KeyExchange.h` 신규
- `CSCommon/Source/KeyExchange.cpp` 신규
- 서버 핸드셰이크: [`../08-integration-guide.md`](../08-integration-guide.md) §2.3, §2.4
- 클라 핸드셰이크: [`../08-integration-guide.md`](../08-integration-guide.md) §3.2

## Wire

| 패킷 | 방향 | 페이로드 |
|------|------|---------|
| `MC_MATCH_ECDHE_CHALLENGE` (2901) | S → C | BLOB 32 B (서버 공개키) |
| `MC_MATCH_ECDHE_RESPONSE` (2902) | C → S | BLOB 32 B (클라 공개키) |

두 패킷 모두 v1 으로 송수신. 핸드셰이크 완료 후 `SetV2Active(true)`.

## Failure Modes
| 조건 | 결과 |
|------|------|
| `ServerGenerateKeyPair` 실패 | 서버 기동 실패 (libsodium 미초기화 등) |
| 서버/클라 KDF 갈라짐 | 첫 v2 패킷 디크립트 실패 — canonical 순서 위반 의심 |
| 비밀키 미소거 | 메모리 덤프 시 노출. `WipePrivateKey` + `sodium_memzero` 필수 |
| 클라가 v2 활성을 RESPONSE 송신 *전* 에 함 | RESPONSE 자체가 v2 → 서버 디크립트 실패 |

## Test Vectors
```
서버 비밀키 (테스트 전용, 실제는 임시):
  77076d0a7318a57d3c16c17251b26645df4c2f87ebc0992ab177fba51db92c2a
서버 공개키:
  8520f0098930a754748b7ddcb43ef75a0dbf3a0d26381af4eba4a98eaa9b4e6a

클라 비밀키:
  5dab087e624a8a4b79e17f8b83800ee66f3bb1292618b6fd1c2f8b27ff88e0eb
클라 공개키:
  de9edb7d7b7dc1b4d35b61c2ece435373f8343c85b78674dadfc7e146f882b4f

→ DeriveSymmetricKey 결과: <기록 후 첨부 — 서버/클라 동일해야 함>
```

## Limitations
- X25519 만 지원 (P-256 등 NIST 곡선 미지원)
- post-quantum KEM (Kyber 등) 미통합

## Cross-References
- AEAD: [`01-aes-gcm-crypter.md`](./01-aes-gcm-crypter.md)
- 통합 함정: [`../08-integration-guide.md`](../08-integration-guide.md) §7.1, §7.2
