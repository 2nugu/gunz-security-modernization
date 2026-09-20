**English** | [한국어](./ko/02-before-after-comparison.md)

# Before / After — One-Page Comparison

> A side-by-side comparison of the original GunZ (2007 MAIET) protocol behavior and the deliverables of this work.

---

## 1. Security / encryption

| Axis | Before | After | What the difference means |
|----|--------|-------|-----------|
| Symmetric cipher algorithm | XOR + rot8 + 0xF0 (32B key, non-standard) | AES-256-GCM (libsodium) | Standard AEAD, AES-NI accelerated |
| Integrity | nCheckSum 6B (simple sum) | 16B GCM tag | Authenticated integrity |
| Nonce / IV | None (same plaintext → same ciphertext) | 12B (`session ‖ direction ‖ seq`) | Blocks replay and known-plaintext attacks |
| Key agreement | Static seed transmission | X25519 ECDHE ephemeral key pair | Forward secrecy |
| On key compromise | All past sessions decryptable | Past sessions protected | PFS secured |
| AAD binding | None | NULL (room for header binding later) | Header tampering detection is covered by the GCM tag |
| HW acceleration fallback | — | Explicit failure at the handshake stage when AES-NI is unsupported | Prevents silent-drop incidents |

---

## 2. Network topology

| Axis | Before | After | What the difference means |
|----|--------|-------|-----------|
| Peer IP/port exposure | Plaintext broadcast (`ResponsePeerList`, `StageEnterBattle`) | `0.0.0.0:0` masking | Blocks DDoS targeting, prevents IP harvesting |
| P2P connection | Direct connection attempt | Server relay fallback | Blocks NAT punch-through bypass |
| Admin/Event exception branch | Present | Removed (everything masked) | Blocks privilege-escalation scenario |

---

## 3. Server stability (DDoS)

| Axis | Before | After | What the difference means |
|----|--------|-------|-----------|
| Connect-flood | Unlimited | per-IP 10 conn / 10 s | Cut at the accept stage, zero alloc burden |
| Packet-flood | Unlimited | per-session 256 packets / 1 s | Blocks small-packet floods |
| Bandwidth-flood | Unlimited | per-session 256 KiB / 1 s | Blocks large-packet saturation |
| Gating location | — | Immediately on IOCP worker entry | Minimum-cost cut |
| Gate disable | — | window=0 or max=0 (operational tuning) | Adjustable per environment |
| Memory GC | — | 60 s window, deque + lastSeen | Prevents leaks in long-running operation |

---

## 4. Anti-cheat (server-side validation)

| Axis | Before | After | What the difference means |
|----|--------|-------|-----------|
| Server-authoritative position | None (P2P trust) | 32-slot ring buffer | Rewind / cross-check possible |
| Position transmission | Peer broadcast only | `MC_MATCH_POSITION_TICK` 32 Hz piggyback | Sent to the server simultaneously |
| Speed/Teleport/Fly validation | None | `MMovementValidator` 4-stage severity escalation | Server-side detection |
| RapidFire/InfiniteSlash (infinite heavy slash) | None | Per-weapon-class fire interval validation | Server-side detection |
| AutoAim/ImpossibleHit | None | range×1.3 + hit-rate 95%+ after 50 shots | Server-side detection |
| Weapon Spoof | Undetectable | Server-authoritative 3 equipped slots vs. reported weapon type | **Unique axis of this work** |
| Position Lie | Undetectable | POSITION × HIT cross-check (Euclidean) | **Unique axis of this work** |
| **Damage-reduction hack (DamageReduce)** | Undetectable, unblockable | Server-Side Damage Override | **Closes the victim-auth blind spot** |
| **Invincibility hack (Invincibility)** | Undetectable | DamageNullification (PEER_SHOT × HIT_TICK) | This work |
| **HP memory manipulation hack (HPHack)** | Undetectable | HPConsistency (HP delta vs. sum of hits) | This work |
| **Damage amplification hack (DamageInflate)** | Undetectable | Reported dmg > server dmg × 1.3 | This work |
| Per-weapon threshold | — | Melee 400u ~ Rocket 1000u | Differentiated by class |
| Hit report authority | Attacker | Victim (`ZMyCharacter::OnDamaged`) | Blocks self-manipulation |

---

## 5. Signal processing / operations

| Axis | Before | After | What the difference means |
|----|--------|-------|-----------|
| Signal collection | None | `MSignalCollector` (16-shard) | Distributes lock contention |
| Weighted score | None | L×1 + M×3 + H×7 + C×10 | Cumulative evaluation |
| Automatic sanctions | (None) | **Prohibited** (policy) | Avoids false-positive disputes |
| Operator reporting | None | HMAC-SHA256 + retry queue (MAX 100) | External system integration |
| Round identification | None | `match_uuid = stage-<H>-<L>-<epochSec>` lifecycle propagation | Per-round analysis |
| ghost-mode | None | Designed (transparent admin connection) | Real-time monitoring |

---

## 6. Backend infrastructure

| Axis | Before | After | What the difference means |
|----|--------|-------|-----------|
| DB | MS-SQL only | PostgreSQL + Alembic | Migratable |
| Cache | None | Redis (optional) | Hot-data separation |
| Internal API | None | FastAPI + HMAC-SHA256 | Signed internal communication |
| Routers | — | game_accounts / leaderboard / shop / admin_game | Modularized |
| Tests | — | pytest 57 passed / 0 failed | Regression verification |
| Deployment | Manual | Multi-stage Docker (210 MB, non-root uid 1000) | Standard container |
| Asynchronous processing | — | MatchServer → API HMAC + local retry queue | Recovery from transient failures |

---

## 7. Protocol / compatibility

| Axis | Before | After |
|----|--------|-------|
| Packet ID uniqueness | `MSGID_COMMAND` | `MSGID_COMMAND` (v1, handshake only) + `MSGID_COMMAND_V2 = 102` |
| Header size | 6 B (`MPacketHeader`) | 20 B (`MPacketHeaderV2`) |
| New MSGIDs | — | 2901, 2902, 2903, 2904, 2905 |
| Negotiation | — | None currently (cutover assumed). Separate work for gradual rollout |

---

## 8. Code / build environment

| Axis | Before | After |
|----|--------|-------|
| Compiler | VC6 / VS9 | VS18 v143/v145 |
| Standard | C++03 | C++14 (limited) |
| External dependencies | None (in-tree) | libsodium (ISC) |
| Encoding | EUC-KR/cp949 | UTF-8 BOM (Korean comments in .h/.cpp) |
| Windows SDK | 6.0 | 10.0.26100 |
| `#ifdef BUILD_MATCH_SERVER` branching | Used | **Prohibited** (unified in canonical order) |

---

## 9. Summary at a glance

```
Before                                     After
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Packet security]
  XOR + rot8                          →    AES-256-GCM AEAD              [implemented]
  Static seed key                     →    X25519 ECDHE PFS              [implemented]
  No integrity                        →    16B GCM tag                   [implemented]
  No nonce                            →    12B per-direction nonce       [implemented]

[Topology]
  Peer IP in plaintext                →    0.0.0.0:0 + server relay      [implemented]

[Stability]
  Unlimited connect/packet/bandwidth  →    Three-tier gate               [implemented]

[Anti-cheat — movement/combat]
  P2P trust                           →    Server-authoritative position + Validator   [implemented]
  No detection infrastructure         →    16-shard Collector + HMAC API [implemented]
  WeaponSpoof / PositionLie impossible →   2 novel axes                  [implemented]
  No infinite heavy slash / RapidFire detection → Per-weapon-class fire interval   [implemented]

[Anti-cheat — damage/HP] (Module 13, Detection-Only by default)
  Damage-reduction hack unblockable   →    Weapon table + tolerance      [implemented]
  Damage amplification hack (1-shot)  →    Detect serverDmg × 1.3 exceeded   [implemented]
  HP memory manipulation hack undetectable → HPConsistency window check [implemented]
  Cheat effect neutralized via Mitigation toggle → SetMitigationMode(true)   [implemented]

[Operations]
  Automatic sanctions / manual        →    Signal → severity triage → operator   [implemented]
  No round identification             →    match_uuid lifecycle          [implemented]
```

`[implemented]` = code written in this package + build verification passed
`[designed]` = design specification only in this package; code written after adoption decision
