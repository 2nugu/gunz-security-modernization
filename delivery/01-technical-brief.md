**English** | [한국어](./ko/01-technical-brief.md)

# Technical Brief — GunZ Security Modernization

> **Readers**: server team lead, security owner. Go/no-go adoption decision within 30 minutes.

---

## 1. Threat Model

The five known attack surfaces of the original GunZ protocol and this work's countermeasures:

### T1 — XOR Keystream Recovery

- **Attack**: collecting many packets encrypted under the same key allows keystream recovery by XOR against known plaintext (header constants, etc.). No nonce → identical plaintext → identical ciphertext.
- **Countermeasure**: AES-256-GCM AEAD. 12B nonce (`session_id ‖ direction ‖ sequence`) yields a different keystream on every regeneration. 16B GCM tag detects tampering.
- **Residual risk**: AAD is currently NULL — `nSize` stays plaintext. Header tampering is covered by the GCM tag, but header-payload binding can be hardened later via AAD.

### T2 — Static Seed Key → Past Sessions Exposed on Key Leak

- **Attack**: during key agreement the server sends a static seed → if the seed leaks, every past session can be decrypted (no forward secrecy).
- **Countermeasure**: X25519 ECDHE ephemeral key pair. Fresh key pair per session; private key wiped immediately after agreement. **PFS secured**.
- **Key decision**: canonical KDF order — sort into `min ‖ max` by `memcmp(rx, tx, 32) ≤ 0`, then BLAKE2b over the IKM. Server/client branching (`#ifdef`) forbidden. Prevents the incident where server and client IKM diverge → first LOGIN packet fails to decrypt.

### T3 — Peer IP Plaintext Exposure

- **Attack**: real IP/Port in plaintext in the peer blob of `MC_MATCH_RESPONSE_PEER_LIST` / `StageEnterBattle`. IP harvesting → DDoS targeting → denial of service → NAT punch-through bypass.
- **Countermeasure**: unconditional `dwIP=0 / nPort=0` masking at both call sites. `ServerBased` forced. Client P2P direct connection fails → `MC_MATCH_REQUEST_PEER_RELAY` fallback → server relays P2P traffic.
- **Residual risk**: the match server's own IP is exposed (unavoidable). DDoS protection depends on an external layer (CDN/scrubber).

### T4 — No Server-Authoritative Position

- **Attack**: position/velocity/attack information is P2P trust-based. A client can claim arbitrary coordinates → teleport hack, speed hack, AutoAim, ImpossibleHit, WeaponSpoof, PositionLie.
- **Countermeasure**: 32-slot ring buffer (`MPositionHistory`) + 32 Hz piggyback (`MC_MATCH_POSITION_TICK = 2903`). Establishes server-side position authority. Validators consume this data.
- **Residual risk**: BSP terrain sampling not integrated — precise noclip detection requires linking the RealSpace2 engine first. Currently an absolute Z check (FlyHack heuristic, capped at Medium severity).

### T5 — Accept-Flood / Packet-Flood / Bandwidth-Flood

- **Attack**: unlimited TCP accepts, unlimited inbound packet count/bandwidth per session → a single IP can take down the match server.
- **Countermeasure**: three-tier gate — `MConnectRateLimit` (per-IP, accept stage) + `MPacketRateLimit` (per-session, packet count) + `MBandwidthThrottle` (per-session, bytes). Gating immediately on IOCP worker entry → on violation, cut before alloc.

---

## 2. Architecture

### 2.1 Data Flow (overall)

```
Client                     Match server                community-api          DB
   │                          │                              │                │
   │ ── ECDHE Challenge ───>  │                              │                │
   │ <── ECDHE Response ───   │                              │                │
   │ ── v2 Encrypted Cmd ─>   │                              │                │
   │  POSITION_TICK 32Hz   →  │ → MPositionHistory.Record    │                │
   │  ATTACK_TICK         →   │ → CombatValidator.Validate   │                │
   │  HIT_TICK            →   │ → CombatValidator.Validate   │                │
   │                          │   ↓                          │                │
   │                          │   MSignalCollector.Push      │                │
   │                          │   (16-shard)                 │                │
   │                          │   ↓                          │                │
   │                          │   MAPIClient.SendAsync ──>   │ HMAC verify    │
   │                          │   (WinHTTP+HMAC-SHA256)      │ ↓              │
   │                          │   retry queue MAX 100        │ INSERT ─────>  │ anticheat_signals
   │                          │                              │                │
   │ ←── DDoS L1: per-IP   ── │                              │                │
   │     (Accept stage)       │                              │                │
   │ ←── DDoS L2: per-sess ── │                              │                │
   │ ←── DDoS L3: bandwidth ─ │                              │                │
```

### 2.2 ECDHE Handshake Sequence

```
1. TCP Accept
2. Server: InitCryptCommObject() — generate v1 seed key + ECDHE keypair
3. Server → Client: MC_MATCH_ECDHE_CHALLENGE (v1-encrypted, BLOB=server public key 32B)
4. Server → Client: MC_MATCH_REPLYCONNECT (v1)
5. Client: ClientHandleChallenge → DeriveSymmetricKey (canonical min‖max)
6. Client: GetCrypterV2()->InitKey
7. Client → Server: MC_MATCH_ECDHE_RESPONSE (v1-encrypted, BLOB=client public key 32B)
   ← Note: SetV2Active(true) flips *after* the RESPONSE is sent
8. Server: ServerDeriveSessionKeys → DeriveSymmetricKey → InitKey → SetV2Active(true)
9. All subsequent packets: MSGID_COMMAND_V2 (102), AES-256-GCM
```

Timing subtlety — if the client calls `SetV2Active(true)` before sending the RESPONSE, the RESPONSE itself goes out as a v2 frame while the server has not yet enabled v2 → decrypt failure. In the client handler, use a synchronous SendCommand instead of Post, then flip the v2 flag.

### 2.3 Anti-Cheat Signal Pipeline

```
[Client] ZGame::Tick (32 Hz)
   ├── ZPostBasicInfo (peer broadcast, existing)
   └── ZPOSTCMD1(MC_MATCH_POSITION_TICK, blob)  ← new, piggyback

[Client] ZPostShot / ZPostShotMelee
   └── ZPostAttackTick (CLOAK_CMD_ID factor=53817)

[Client] ZMyCharacter::OnDamaged  ← victim-authoritative
   └── ZPostHitTick (CLOAK_CMD_ID factor=49217)

[Server] MMatchServer_OnCommand
   ├── case MC_MATCH_POSITION_TICK → MMovementValidator.Validate
   ├── case MC_MATCH_ATTACK_TICK   → MCombatValidator.ValidateAttack
   └── case MC_MATCH_HIT_TICK      → MCombatValidator.ValidateHit
                                     + PositionLie cross-check

[Server] Validator → CheatSignal {type, severity, value, threshold, detail}
       → MSignalCollector.Push (16-shard, lock-distributed)
       → SetReportCallback lambda
       → MAPIClient.SendCheatReportsAsync (fire-and-forget + retry)

[community-api] POST /api/anticheat/report (X-Signature: hex(HMAC-SHA256(body)))
       → verify_signature
       → INSERT INTO anticheat_signals (... match_uuid ...)
```

---

## 3. Module Map

| # | Module | Location (proposed) | LoC (≈) | Dependencies | Standalone adoption | Implementation status |
|---|------|-----------|--------|--------|----------|----------|
| 01 | `MPacketCrypterV2` | `CSCommon/{Include,Source}/MPacketCrypterV2.{h,cpp}` | 280 | libsodium | Together with 02 | Implemented |
| 02 | `KeyExchange` | `CSCommon/{Include,Source}/KeyExchange.{h,cpp}` | 220 | libsodium | Together with 01 | Implemented |
| 03a | `MConnectRateLimit` | `CSCommon/Security/MConnectRateLimit.{h,cpp}` | 130 | std::chrono | ✅ | Implemented |
| 03b | `MPacketRateLimit` | `CSCommon/Security/MPacketRateLimit.{h,cpp}` | 110 | std::chrono | ✅ | Implemented |
| 03c | `MBandwidthThrottle` | `CSCommon/Security/MBandwidthThrottle.{h,cpp}` | 110 | std::chrono | ✅ | Implemented |
| 04 | IP Relay (peer mask) | `MMatchServer.cpp:2147` + `MMatchServer_Stage.cpp:467` | 30 | None | ✅ | Implemented |
| 05 | `MPositionHistory` | `CSCommon/Security/MPositionHistory.{h,cpp}` | 110 | None (STL-free) | Dependency of 06/07 | Implemented |
| 06 | `MMovementValidator` | `CSCommon/Security/MMovementValidator.{h,cpp}` | 240 | 05, 08 | ✅ (after 05+08) | Implemented |
| 07 | `MCombatValidator` | `CSCommon/Security/MCombatValidator.{h,cpp}` | 320 | 05, 08 | ✅ (after 05+08) | Implemented |
| 08 | `MSignalCollector` | `CSCommon/Security/MSignalCollector.{h,cpp}` | 180 | None | Infrastructure for 06/07/09/10 | Implemented |
| 09 | Weapon Spoof (inline) | `MMatchServer_OnCommand.cpp` ATTACK_TICK handler | 60 | 07, 08 | After 07 + 08 | Implemented |
| 10 | Position Lie (inline) | `MMatchServer_OnCommand.cpp` HIT_TICK handler | 80 | 05, 07, 08 | After 05 + 07 + 08 | Implemented |
| 11 | `MAPIClient` | `CSCommon/Security/MAPIClient.{h,cpp}` | 220 | WinHTTP, libsodium | After 08 | Implemented |
| 12 | match_uuid pipeline | `MMatchStage.{h,cpp}` + `MMatchServer_Stage.cpp` | 50 | 08 | After 08 | Implemented |
| 13 | `MDamageValidator` | `CSCommon/Include/MDamageValidator.{h,cpp}` | 350 | 05, 08, weapon table | After 07 + 08 | **Implemented** (Detection-Only) |

**Modules 01~13**: build verification passed (CSCommon.lib / MatchServer.exe Release|Win32 v143, binary 4.26 MB). New symbols (`DamageVal`, `HPHack`, `HPInconsistency`, `DamageManipulation`) confirmed via `strings`. LoC is the actual code line count.

**Default mode of Module 13**: Detection-Only (`SERVER_DAMAGE_OVERRIDE=false`). Signals are emitted only; victim HP deduction follows the existing flow unchanged. Mitigation mode is enabled by an environment-variable toggle (implemented; operator decision after validating the live damage flow).

---

## 4. Wire Compatibility

### 4.1 Packet Format v2

```
[ MPacketHeaderV2 (20B) ][ Ciphertext (N B) ][ GCM Tag (16B) ]

MPacketHeaderV2:
  uint16_t  nMagic;          // packet magic
  uint16_t  nMsgID;          // 102 = MSGID_COMMAND_V2
  uint16_t  nSize;           // total length (header+ct+tag), plaintext
  uint16_t  nReserved;
  uint32_t  nSessionID;      // first 4B of nonce
  uint32_t  nDirection;      // next 4B of nonce (0=C2S, 1=S2C)
  uint32_t  nSequence;       // last 4B of nonce (replay protection)

Overhead: GZ_V2_OVERHEAD = 36 (header 20 + tag 16)
```

### 4.2 New MSGIDs

| ID | Name | Direction | Purpose |
|----|------|------|------|
| 102 | `MSGID_COMMAND_V2` | C↔S | v2 packet framing |
| 2901 | `MC_MATCH_ECDHE_CHALLENGE` | S→C | Sends ECDHE server public key |
| 2902 | `MC_MATCH_ECDHE_RESPONSE` | C→S | Sends ECDHE client public key |
| 2903 | `MC_MATCH_POSITION_TICK` | C→S (MACHINE2MACHINE) | 32 Hz position piggyback |
| 2904 | `MC_MATCH_ATTACK_TICK` | C→S (MACHINE2MACHINE) | Fire event |
| 2905 | `MC_MATCH_HIT_TICK` | C→S (MACHINE2MACHINE) | Hit report (victim-auth) |

### 4.3 v1 vs v2 Negotiation

Current implementation: **no legacy client compatibility** — only the 4-packet handshake window is v1; all subsequent packets are forced to v2.

Recommended for live service adoption:
- (Option A) Gradual rollout — add a v1/v2 negotiation bit to the header, run both in parallel for a period after the client patch ships
- (Option B) Big-bang rollout — force client update, then cutover

This package assumes Option B — an explicit cutover at the sunset point simplifies the security analysis.

---

## 5. Operational Impact

### 5.1 Memory (per-session)

Figures are **estimates** based on static analysis (no runtime measurement performed). Actual memory may vary with compiler padding / allocator overhead.

| Module | Additional memory (≈) |
|------|--------------|
| `MPacketCrypterV2` | ≈ 700 B (AES context + nonce counter) |
| `KeyExchangeState` | ≈ 80 B (freed after handshake) |
| `MPacketRateLimit` | ≈ 100 B (deque<ms> + counter) |
| `MBandwidthThrottle` | ≈ 80 B |
| `MPositionHistory` | 32 × 16 B = 512 B (sample POD) |
| `PlayerMovementState` | ≈ 80 B |
| `PlayerCombatState` | 20 × ≈ 24 B + counters = ≈ 600 B |
| **Total (≈)** | **≈ 2.2 KB / session** |

At 10,000 concurrent connections ≈ 22 MB additional. Negligible (measurement recommended).

### 5.2 CPU

| Module | Cost |
|------|------|
| AES-256-GCM (AES-NI on) | ~0.3 cycles/byte. 1500 B packet ~450 cycles. 0.45 µs/packet at 1 GHz |
| ECDHE (per session, once) | X25519 point-scalar multiplication ~70 µs |
| Validator calls | per-tick O(1) ~ O(20) — combat history reverse scan of at most 20 entries |
| Signal Push | O(1), contention spread across 16-shard locks |

### 5.3 Bandwidth

Figures are **theoretical values** based on the packet format. No measurement performed.

| Additional packet | Frequency | Size (incl. header, ≈) | Per-player addition (≈) |
|----------|------|------------------|--------------------|
| `POSITION_TICK` | 32 Hz | ≈ 50 B (v2) | ≈ 1.6 KB/s |
| `ATTACK_TICK` | On fire (≈ 5 Hz peak) | ≈ 30 B | ≈ 150 B/s peak |
| `HIT_TICK` | On hit (≈ 2 Hz peak) | ≈ 50 B | ≈ 100 B/s peak |
| **Total (peak, ≈)** | | | **≈ 1.85 KB/s/player** |

At 10,000 concurrent users, peak ≈ 18.5 MB/s. Under 15 % of a 1 Gbps match-server NIC (measurement recommended).

### 5.4 New Environment Variables

| Environment variable | Required | Purpose | Behavior when unset |
|----------|------|------|--------------|
| `COMMUNITY_API_URL` | △ | community-api endpoint | callback not registered, memory-only |
| `MATCH_RESULT_WEBHOOK_SECRET` | △ | HMAC-SHA256 secret (32 B recommended) | callback not registered |
| (Postgres connection) | ✅ (when adopting Phase F) | DB connection | community-api fails to start |

---

## 6. Recommended Phasing

| Phase | Unit | Implementation status | Depends on | Effort (≈) |
|-------|------|----------|------|--------------|
| A | DDoS three-tier + IP masking | Implemented | — | 2~3 person-days (integration + tuning) |
| B | AES-256-GCM + ECDHE + V2 packet framing | Implemented | A | 2~3 person-weeks (integration + compatibility verification + client build) |
| C | Position History + Movement Validator | Implemented | B | 1~2 person-weeks |
| D | Combat Validator + Weapon Spoof + Position Lie | Implemented | C | 2~3 person-weeks |
| E | Signal Collector + API Client + community-api | Implemented | D | 2~3 person-weeks (incl. backend infrastructure) |
| F | match_uuid pipeline + Postgres migration | Implemented | E | 1~2 person-weeks |
| **G** | **Damage Validator** (Detection-Only) | **Implemented** | D, E | 1 person-week (integration + weapon table import) + 1~2 weeks of operation |
| **G2** | **Damage Validator** (Mitigation enabled) | **Implemented, toggleable** | G | 1 person-week (verification + `SetMitigationMode(true)`) + healing mechanic registration separately |

Total (Phase A~F) 10~15 person-weeks — integration of the implemented portion of this package. **Phase G** Detection-Only code is complete in this package — at integration time, decide the weapon table import + healing mechanic registration flow on the ZRule side (estimated 1 person-week). **G2** is enabled immediately via the `SetMitigationMode(true)` toggle once false-positive verification passes after Detection-Only operation. If the existing damage flow is disrupted, roll back immediately by toggling to false.

Each Phase can ship independently (G after D, E).

---

## 7. Verified Areas / Not Performed

### Verified
- Build: CSCommon.lib / MatchServer.exe / Gunz.exe (Release|Win32, v143)
- Handshake: ECDHE Challenge/Response flow code path complete
- HMAC E2E: community-api 3 scenarios (200 valid / 409 dedup / 401 badsig)
- pytest: 57 passed / 12 skipped / 3 xfailed / 0 failed
- Integration: docker compose full stack running, Alembic 003 applied

### Verification Not Performed
- Successful handshake log from a live-running MatchServer ↔ Gunz (next work item)
- Threshold tuning based on production traffic distribution
- Load testing at 10K+ concurrent connections
- Wire compatibility with live GunZ (separate work)

---

## 8. Key Technical Decisions

### 8.1 STL-free Header (`MPositionHistory`)
- A header that pulls in STL threatens ABI compatibility. Only POD structs are exposed. Safe in an environment where CSCommon is built on both v143/v145.

### 8.2 16-shard Lock (`MSignalCollector`)
- N concurrent connections → single-mutex contention → push latency. Spread across 16 shards, routed by `hash(uid) % 16`.
- Addresses the contention threat identified in AUDIT-05.

### 8.3 victim-authoritative HIT_TICK + Server-Side Damage Override
- Attacker-authoritative reporting → attacker can self-manipulate hit rate
- Victim-authoritative reporting → **victim can reduce/ignore damage received** (damage-reduction hack)
- **Hybrid adopted**: the fact that a hit occurred is cross-checked from both sides' reports, srcPos from the attacker, vicPos server-authoritative, **damage amount computed by the server**
- Victim HP deduction uses `serverDmg` from the server weapon table rather than client-reported dmg → neutralizes damage-reduction hack / invincibility hack
- HPConsistency: compares the HIT_TICK sum over a time window against the HP delta → detects HP memory-manipulation hacks (Module 13)

### 8.4 No Auto-Ban
- Every signal goes to the operator queue. Cost of a false positive (wrongful ban of a real user) > cost of a false negative (a hacker active for a few more days).
- However, when the cumulative signal score (L×1 + M×3 + H×7 + C×10) exceeds the threshold, the player is automatically enrolled in the ghost-mode monitoring queue.

### 8.5 No `#ifdef BUILD_MATCH_SERVER` Branching
- If the ECDHE KDF diverges by server/client branching, IKM mismatch → handshake failure. Unified via canonical order.
- All three copies — CSCommon, match-server, netprobe — carry identical code.

---

## 9. Reference Documents

- One-page comparison → [`02-before-after-comparison.md`](./02-before-after-comparison.md)
- Anti-cheat catalog → [`04-anticheat-catalog.md`](./04-anticheat-catalog.md)
- Module details → [`modules/`](./modules/)
- Integration pseudocode → [`08-integration-guide.md`](./08-integration-guide.md)
- Validation reproduction → [`06-validation-kit.md`](./06-validation-kit.md)
