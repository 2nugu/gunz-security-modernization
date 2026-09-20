**English** | [한국어](../ko/modules/01-aes-gcm-crypter.md)

# Module 01 — AES-256-GCM Crypter V2

## Purpose
Replaces the original XOR + rot8 + 0xF0 transform with a standard AEAD. Packet confidentiality + integrity + replay protection.

## Interface

```cpp
namespace Security {

class MPacketCrypterV2 {
public:
    MPacketCrypterV2();
    ~MPacketCrypterV2();

    // false when AES-NI is unsupported (handshake stage fails on the host).
    bool InitKey(const uint8_t* key, size_t keyLen);  // keyLen == 32

    // Plaintext src(nSrcLen) → [nonce 12B + cipher + tag 16B] dst.
    // dst buffer size ≥ nSrcLen + GZ_V2_OVERHEAD(36).
    int Encrypt(const uint8_t* src, int nSrcLen,
                uint8_t* dst, int nDstLen,
                uint32_t sessionId, uint32_t direction, uint32_t sequence);

    // [nonce + cipher + tag] src(nSrcLen) → plaintext dst.
    // Length validation uses size_t unsigned comparison (blocks 32-bit wrap attacks).
    int Decrypt(const uint8_t* src, int nSrcLen,
                uint8_t* dst, int nDstLen);
};

constexpr int GZ_V2_OVERHEAD = 36;  // header 20 + tag 16

}
```

## Integration Points
- `CSCommon/Include/MPacketCrypterV2.h` new
- `CSCommon/Source/MPacketCrypterV2.cpp` new
- Add `MCommandBuilder::InitCryptV2(MPacketCrypterV2*)`
- Add `MSGID_COMMAND_V2` case to `MCommandBuilder::MakeCommand` (SetData after Decrypt)
- `MClient::SendCommand` v2 branch
- v2 branch in `MServer::SendCommand` — check whether v2 is active while holding `LockCommList`, then call `SendMsgCommandV2` (send counter consistency)

## Wire Format

```
[ Header 20B ][ Ciphertext NB ][ GCM Tag 16B ]

Header:
  uint16_t  nMagic
  uint16_t  nMsgID = MSGID_COMMAND_V2 (102)
  uint16_t  nSize    (total length, plaintext — AEAD does not detect nSize tampering)
  uint16_t  nReserved
  uint32_t  nSessionID
  uint32_t  nDirection (0=C→S, 1=S→C)
  uint32_t  nSequence  (replay protection)

Nonce (12B): nSessionID ‖ nDirection ‖ nSequence
AAD       : NULL (currently). Room for header binding later.
```

## Configuration
- Algorithm fixed (AES-256-GCM)
- Key size fixed (32 B)
- AAD currently NULL
- HW acceleration mandatory (AES-NI)

## Failure Modes
| Condition | Result |
|------|------|
| Host without AES-NI | `InitKey` false, handshake stage fails |
| Length validation failure (`(size_t)plainLen + 36 != (size_t)nSrcLen`) | `Decrypt` false |
| GCM tag verification failure | `Decrypt` false (tampering detected) |
| nonce sequence wrap (2^32) | Re-handshake recommended when the sequence counter wraps |

## Test Vectors
```
key       = "0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20" (hex)
plaintext = "Hello, GunZ"
nonce     = sessionId=0x12345678, direction=0, sequence=1

→ Encrypt result: <to be attached after recording>
→ Decrypt restores the identical plaintext
```

## Limitations / Future Work
- AAD NULL — header tampering is detected by the GCM tag, but header-payload binding can be hardened later
- Automatic re-handshake on nonce 2^32 wrap not implemented (currently assumes monotonically increasing sequence until session end)
- No AES-NI fallback (software AES) provided — unsupported hosts are unusable by policy

## Cross-References
- Key agreement: [`02-ecdhe-key-exchange.md`](./02-ecdhe-key-exchange.md)
- Integration: [`../08-integration-guide.md`](../08-integration-guide.md) §2.3, §3.1
