**English** | [한국어](../ko/modules/02-ecdhe-key-exchange.md)

# Module 02 — X25519 ECDHE Key Exchange

## Purpose
Symmetric key agreement with an ephemeral key pair per session. Secures **Forward Secrecy**.

## Interface

```cpp
namespace Security {

class KeyExchangeState {
public:
    KeyExchangeState();
    ~KeyExchangeState();   // wipes private key

    // At server start
    bool ServerGenerateKeyPair();
    const uint8_t* GetServerPublic() const;  // 32B

    // Client — after receiving server public key
    bool ClientHandleChallenge(const uint8_t* serverPub);
    const uint8_t* GetClientPublic() const;  // 32B

    // Server — after receiving client public key
    bool ServerDeriveSessionKeys(const uint8_t* clientPub);

    // Both sides — extract KDF input
    const uint8_t* GetSharedRx() const;  // 32B
    const uint8_t* GetSharedTx() const;  // 32B

    void WipePrivateKey();
};

// canonical min‖max IKM → BLAKE2b → 32B symmetric key
bool DeriveSymmetricKey(const KeyExchangeState* kxState, uint8_t outKey32[32]);

}
```

## KDF — Canonical min‖max Order

```cpp
bool DeriveSymmetricKey(...) {
    const uint8_t* rx = kxState->GetSharedRx();
    const uint8_t* tx = kxState->GetSharedTx();

    // Key point: sort by memcmp(rx, tx, 32) <= 0
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

**Why `min‖max`**:
- libsodium `crypto_kx_*_session_keys` is mirror-split (server rx == client tx)
- Hashing in `rx‖tx` order makes server/client IKM diverge → symmetric key mismatch → first LOGIN packet fails to decrypt
- Sorting by `memcmp` is identical on both sides → guarantees matching IKM

**No `#ifdef BUILD_MATCH_SERVER` branching** — a single copy of this code must exist identically in all three places: CSCommon, match-server, netprobe.

## Integration Points
- `CSCommon/Include/KeyExchange.h` new
- `CSCommon/Source/KeyExchange.cpp` new
- Server handshake: [`../08-integration-guide.md`](../08-integration-guide.md) §2.3, §2.4
- Client handshake: [`../08-integration-guide.md`](../08-integration-guide.md) §3.2

## Wire

| Packet | Direction | Payload |
|------|------|---------|
| `MC_MATCH_ECDHE_CHALLENGE` (2901) | S → C | BLOB 32 B (server public key) |
| `MC_MATCH_ECDHE_RESPONSE` (2902) | C → S | BLOB 32 B (client public key) |

Both packets are sent and received as v1. `SetV2Active(true)` after the handshake completes.

## Failure Modes
| Condition | Result |
|------|------|
| `ServerGenerateKeyPair` fails | Server fails to start (libsodium not initialized, etc.) |
| Server/client KDF divergence | First v2 packet fails to decrypt — suspect canonical order violation |
| Private key not wiped | Exposed on memory dump. `WipePrivateKey` + `sodium_memzero` mandatory |
| Client enables v2 *before* sending RESPONSE | RESPONSE itself is v2 → server decrypt failure |

## Test Vectors
```
Server private key (test only; ephemeral in practice):
  77076d0a7318a57d3c16c17251b26645df4c2f87ebc0992ab177fba51db92c2a
Server public key:
  8520f0098930a754748b7ddcb43ef75a0dbf3a0d26381af4eba4a98eaa9b4e6a

Client private key:
  5dab087e624a8a4b79e17f8b83800ee66f3bb1292618b6fd1c2f8b27ff88e0eb
Client public key:
  de9edb7d7b7dc1b4d35b61c2ece435373f8343c85b78674dadfc7e146f882b4f

→ DeriveSymmetricKey result: <to be attached after recording — must be identical on server/client>
```

## Limitations
- X25519 only (NIST curves such as P-256 not supported)
- post-quantum KEM (Kyber, etc.) not integrated

## Cross-References
- AEAD: [`01-aes-gcm-crypter.md`](./01-aes-gcm-crypter.md)
- Integration pitfalls: [`../08-integration-guide.md`](../08-integration-guide.md) §7.1, §7.2
