[English](../../modules/01-aes-gcm-crypter.md) | **한국어**

# Module 01 — AES-256-GCM Crypter V2

## Purpose
오리지널 XOR + rot8 + 0xF0 변형을 표준 AEAD 로 교체. 패킷 기밀성 + 무결성 + replay 보호.

## Interface

```cpp
namespace Security {

class MPacketCrypterV2 {
public:
    MPacketCrypterV2();
    ~MPacketCrypterV2();

    // AES-NI 미지원 시 false (호스트에서 핸드셰이크 단계 실패).
    bool InitKey(const uint8_t* key, size_t keyLen);  // keyLen == 32

    // 평문 src(nSrcLen) → [nonce 12B + cipher + tag 16B] dst.
    // dst 버퍼 크기 ≥ nSrcLen + GZ_V2_OVERHEAD(36).
    int Encrypt(const uint8_t* src, int nSrcLen,
                uint8_t* dst, int nDstLen,
                uint32_t sessionId, uint32_t direction, uint32_t sequence);

    // [nonce + cipher + tag] src(nSrcLen) → 평문 dst.
    // 길이 검증은 size_t 비부호 비교 (32-bit wrap 공격 봉쇄).
    int Decrypt(const uint8_t* src, int nSrcLen,
                uint8_t* dst, int nDstLen);
};

constexpr int GZ_V2_OVERHEAD = 36;  // header 20 + tag 16

}
```

## Integration Points
- `CSCommon/Include/MPacketCrypterV2.h` 신규
- `CSCommon/Source/MPacketCrypterV2.cpp` 신규
- `MCommandBuilder::InitCryptV2(MPacketCrypterV2*)` 추가
- `MCommandBuilder::MakeCommand` 에 `MSGID_COMMAND_V2` case 추가 (Decrypt 후 SetData)
- `MClient::SendCommand` v2 분기
- `MServer::SendCommand` 의 v2 분기 — `LockCommList` 상태로 v2 활성 여부 확인 후 `SendMsgCommandV2` 호출 (send counter 일관성)

## Wire Format

```
[ Header 20B ][ Ciphertext NB ][ GCM Tag 16B ]

Header:
  uint16_t  nMagic
  uint16_t  nMsgID = MSGID_COMMAND_V2 (102)
  uint16_t  nSize    (전체 길이, 평문 — AEAD 가 nSize 변조 검출 안 함)
  uint16_t  nReserved
  uint32_t  nSessionID
  uint32_t  nDirection (0=C→S, 1=S→C)
  uint32_t  nSequence  (replay 방지)

Nonce (12B): nSessionID ‖ nDirection ‖ nSequence
AAD       : NULL (현재). 향후 헤더 바인딩 여지.
```

## Configuration
- 알고리즘 고정 (AES-256-GCM)
- 키 크기 고정 (32 B)
- AAD 현재 NULL
- HW 가속 강제 (AES-NI)

## Failure Modes
| 조건 | 결과 |
|------|------|
| AES-NI 미지원 호스트 | `InitKey` false, 핸드셰이크 단계 실패 |
| 길이 검증 실패 (`(size_t)plainLen + 36 != (size_t)nSrcLen`) | `Decrypt` false |
| GCM tag 검증 실패 | `Decrypt` false (변조 탐지) |
| nonce sequence wrap (2^32) | sequence 카운터 wrap 시 재핸드셰이크 권장 |

## Test Vectors
```
key       = "0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20" (hex)
plaintext = "Hello, GunZ"
nonce     = sessionId=0x12345678, direction=0, sequence=1

→ Encrypt 결과: <기록 후 첨부>
→ Decrypt 시 동일 평문 복원
```

## Limitations / Future Work
- AAD NULL — 헤더 변조는 GCM tag 가 검출하지만 헤더-페이로드 바인딩은 향후 강화 가능
- nonce 2^32 wrap 자동 재핸드셰이크 미구현 (현재는 세션 종료까지 sequence 단조 증가 가정)
- AES-NI 폴백 (소프트웨어 AES) 미제공 — 미지원 호스트 사용 불가가 정책

## Cross-References
- 키 합의: [`02-ecdhe-key-exchange.md`](./02-ecdhe-key-exchange.md)
- 통합: [`../08-integration-guide.md`](../08-integration-guide.md) §2.3, §3.1
