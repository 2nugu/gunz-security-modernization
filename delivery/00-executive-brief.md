**English** | [한국어](./ko/00-executive-brief.md)

# Executive Brief — GunZ Security Modernization

> **Readers**: business / planning directors. Compressed to a level where a decision can be made in 5 minutes.

---

## One-Line Summary

A body of work that replaces the packet security, DDoS defense, and server-side anti-cheat layers of the original GunZ (2007 MAIET) with modern standards — **12 new modules + integration-point specification + validation kit**. It assumes live GunZ likely already has equivalent solutions; the differentiating value of this package lies in the **anti-cheat signal catalog** built from GunZ play + development experience and **two new detection axes** (Weapon Spoof / Position Lie).

---

## Key Figures

| Area | Original | This work | Effect |
|------|---------|--------|------|
| Packet cipher | XOR + bit shift + 0xF0 (32B key) | AES-256-GCM AEAD + X25519 ECDHE | Authenticated integrity, PFS, keystream-recovery attacks blocked |
| Peer IP exposure | Plaintext broadcast | `0.0.0.0:0` masking + server relay | DDoS targeting blocked, IP harvesting blocked |
| Connect-flood | Unlimited | per-IP 10 conn / 10 s | accept-flood blocked (cut before alloc) |
| Packet-flood | Unlimited | per-session 256 packets / 1 s | Small-packet flood blocked |
| Bandwidth-flood | Unlimited | per-session 256 KiB / 1 s | Large-packet saturation blocked |
| Server-authoritative position | None (P2P trust) | 32-slot ring buffer + 32 Hz piggyback | Rewind verification, server-side detection possible |
| Speed/Teleport/Fly validation | None | `MMovementValidator` | 4-level severity escalation |
| RapidFire/InfiniteSlash validation | None | `MCombatValidator` per-weapon-class fire interval | GunZ's signature hacks blocked |
| AutoAim/ImpossibleHit validation | None | `MCombatValidator` range × 1.3 + hit-rate 95%+ | Server-side detection |
| Weapon spoof | Undetectable | Server-authoritative 3-slot equipment cross-check | **Axis unique to this work** |
| Position lie | Undetectable | POSITION × HIT cross-check | **Axis unique to this work** |
| **Damage-reduction hack / invincibility hack / HPHack** | Undetectable, unblockable | `MDamageValidator` Detection-Only implementation complete — weapon table + tolerance + HP consistency | **Closes the victim-auth blind spot. Mitigation toggle neutralizes the hack's effect itself** |
| External reporting | None | HMAC-SHA256 + retry queue | Operator dashboard integration |
| match_uuid tracking | None | Lifecycle propagation | Per-round analysis |
| DB | MS-SQL | PostgreSQL + Alembic | Migration-capable infrastructure |
| Deployment | Manual | Multi-stage Docker (210 MB, non-root) | Standard container deployment |

---

## Expected Effects of Adoption

1. **Modern security baseline secured**. The non-standard XOR variant from 2007 cannot cope with modern attack models. Introducing AEAD + ECDHE protects against packet tampering, replay, and key-leak scenarios.
2. **Server stability**. Scenarios where accept-flood / packet-flood / bandwidth-flood take down a single machine are blocked by the three-tier gate. Gating immediately after IOCP worker entry minimizes alloc pressure.
3. **Operable anti-cheat**. Signal → severity classification → operator review flow, with no auto-ban. Avoids the false-positive disputes that auto-bans create.
4. **Formalization of the GunZ hack catalog**. Hack patterns observed on the official server and in a self-built public-tree environment, organized as signals and tuning values.

---

## Adoption Risks

| Risk | Mitigation |
|--------|----------|
| Protocol v1 → v2 transition — legacy client compatibility | Only the 4-packet handshake window is v1; v2 enforced afterward. Gradual rollout requires additional v1/v2 negotiation work (currently not implemented) |
| New libsodium dependency | ISC license, static linking possible, de facto industry standard |
| Hosts without AES-NI | Explicit failure at InitKey — diagnosable at the handshake stage |
| Anti-cheat threshold tuning burden | Package defaults are estimates measured in a self-built public-tree environment. Adjustment after measuring live traffic distribution recommended |
| Win32 IOCP dependency | Current code assumes Win32. Cross-platform port is separate work |
| DB change (MS-SQL → Postgres) | Affects backend infrastructure only, no client impact. Adoption can be decided separately depending on the company's operating environment |

---

## Adoption Units (Standalone Value per Phase)

| Phase | Unit | Implementation status | Standalone adoption | Expected effect |
|-------|------|----------|----------|----------|
| **Phase A** | DDoS three-tier gate + IP masking | Implemented | ✅ | Immediate server availability improvement. No protocol change |
| **Phase B** | AES-256-GCM + ECDHE | Implemented | ✅ | Packet security. Requires builds on both client and server |
| **Phase C** | Position History + Movement Validator | Implemented | ✅ (depends on B) | Server-side detection of speed hacks / teleport hacks |
| **Phase D** | Combat Validator + 2 novel axes (WeaponSpoof, PositionLie) | Implemented | ✅ (depends on C) | Detection of infinite slash / RapidFire / AutoAim |
| **Phase E** | Signal Collector + API reporting | Implemented | ✅ (depends on D) | Operator dashboard integration |
| **Phase F** | community-api + Postgres + match_uuid | Implemented | △ (infrastructure) | Per-round analysis, HMAC internal API |
| **Phase G** | **Damage Validator** (Detection-Only → Mitigation) | **Implemented** (Detection-Only by default) | Staged adoption (depends on D, E) | Counters damage-reduction hack / invincibility hack / HPHack |

Each Phase can be adopted standalone without the preceding Phases (F infrastructure stage separate). Adopting Phase A alone already realizes the server stability value on its own.

**Staged adoption of Phase G**:
- Step G1 — Enable Detection-Only (1–2 weeks, measure false-positive distribution)
- Step G2 — Accumulate operator reviews (1–2 weeks, verify weapon table + recovery mechanics)
- Step G3 — Enable Mitigation (environment variable toggle, instant rollback possible)

Details → [`modules/13-damage-validator.md`](./modules/13-damage-validator.md) §11

---

## License Summary

| Dependency | License | Client impact | Server impact |
|--------|----------|----------------|----------|
| libsodium | ISC | Static linking possible | Static linking possible |
| WinHTTP | Windows SDK | None | None |
| FastAPI / SQLAlchemy / Alembic | MIT | None | Backend dependency |
| PostgreSQL | PostgreSQL License | None | Database |
| Redis | RSAL/SSPL (7.x) or BSD (6.x/Valkey) | None | Operating policy review needed |
| Docker | Apache 2.0 | None | Deployment infrastructure |

Details → [`03-license-inventory.md`](./03-license-inventory.md)

---

## Package Contents

- 21 documents (Tier 1–3 + appendices)
- 12 new code modules (`Security::*` namespace, `~2,400 LoC`)
- Integration-point specification (Layer B — pseudocode + call signatures)
- Validation kit (`docker compose up` + pytest 57/0 + HMAC E2E 3 scenarios)

---

## Recommended Next Steps

1. Review this Brief (5 min)
2. Review [`01-technical-brief.md`](./01-technical-brief.md) + [`02-before-after-comparison.md`](./02-before-after-comparison.md) (30 min)
3. Reproduce the validation kit yourself ([`06-validation-kit.md`](./06-validation-kit.md), 30 min)
4. Decide which of Phases A–G to adopt

---

## Author / Terms of Use

- Author: Hong-gu Lee <2nugu@naver.com>
- This package is freely released and can be adopted without NDA / license negotiation / compensation procedures
- Author's rights: continued follow-up analysis based on public materials + non-commercial OSS release (see README for details)
