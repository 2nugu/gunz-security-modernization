**English** | [한국어](./ko/README.md)

# GunZ Security & Anti-Cheat Modernization Package

This package is a research work that grew out of a personal affection for GunZ.

Using the GunZ trees published on GitHub (GunZ-The-Duel, FGunZ, RefinedGunz, Gunz1.5-main, and others) as a base, I laid out a concept for improving the security and anti-cheat layer, implemented it, and confirmed that the build works against the public versions.

If the company judges this work to be useful, you are welcome to adopt it as-is. No separate NDA, license negotiation, or compensation process is required.

I did, however, drink a fair amount of coffee along the way ☕

― Rights retained by the author

1. Freedom to continue follow-up analysis based on public materials
2. Publication on GitHub under a non-commercial OSS license (MIT/ISC, etc.)

These rights remain regardless of whether the company adopts the work.
Here's to GunZ living a long life.

― Author: Hong-gu Lee <2nugu@naver.com>
― Date: 2026-05-06

---

> **Purpose**: Delivery package for work that replaces the server security, stability, and anti-cheat layers of the original GunZ (2007 MAIET) with modern standards.
> **Base**: A newly written security layer on top of the public GitHub trees (`source-reference1` GunZ-The-Duel and others).

---

## 1. What this package is and is not

**This package is**:
- A **modern security layer** that closes the five known attack surfaces of the original protocol
- DDoS three-tier gate, AES-256-GCM AEAD, X25519 ECDHE, server-authoritative position ring buffer
- A **specialized anti-cheat signal catalog** based on play on the official GunZ server + build testing of the public trees
- A documentation tree split by audience: decision makers / engineers / operators

**This package is not**:
- Client-side anti-cheat (memory protection, DLL injection defense, etc. are not covered)
- A patch that can be merged directly into the live GunZ service (compatibility verification + integration into the company's internal tree are separate work)
- Changes to game logic, UI, or graphics (original gameplay is preserved)
- BSP-precise noclip detection, x64 build, TLS 1.3 login channel (planning stage)

For the full list of items not included, see [`07-whats-not-included.md`](./07-whats-not-included.md).

---

## 2. Entry points by audience

| Role | Read first | Time |
|------|-------------|------|
| **Business / planning director** | [`00-executive-brief.md`](./00-executive-brief.md) | 5 min |
| **Server team lead, security owner** | [`01-technical-brief.md`](./01-technical-brief.md) → [`02-before-after-comparison.md`](./02-before-after-comparison.md) | 30 min |
| **Legal / compliance** | [`03-license-inventory.md`](./03-license-inventory.md) | 10 min |
| **Operators / GM team** | [`04-anticheat-catalog.md`](./04-anticheat-catalog.md) → [`05-operations-runbook.md`](./05-operations-runbook.md) | 20 min |
| **Integration engineers** | [`08-integration-guide.md`](./08-integration-guide.md) → [`modules/`](./modules/) | 10~15 min per module |
| **QA / security validation team** | [`06-validation-kit.md`](./06-validation-kit.md) | 30 min (including reproduction) |

---

## 3. Directory index

```
delivery/
├── README.md                          This document
├── 00-executive-brief.md              Tier 1 — for decision makers (1~2 pages)
├── 01-technical-brief.md              Tier 2 — for technical directors
├── 02-before-after-comparison.md      One-page comparison table
├── 03-license-inventory.md            Dependency license inventory
├── 04-anticheat-catalog.md            GunZ-specific hack taxonomy + detection mapping
├── 05-operations-runbook.md           Operator decision flow
├── 06-validation-kit.md               Validation reproduction guide
├── 07-whats-not-included.md           Areas not covered (honesty signal)
├── 08-integration-guide.md            Integration point pseudocode
└── modules/                           Tier 3 — per-module reference (13 modules)
    ├── 01-aes-gcm-crypter.md
    ├── 02-ecdhe-key-exchange.md
    ├── 03-ddos-rate-limit.md
    ├── 04-ip-relay.md
    ├── 05-position-history.md
    ├── 06-movement-validator.md
    ├── 07-combat-validator.md             (fire rate / hit rate)
    ├── 08-signal-collector.md
    ├── 09-weapon-spoof-novel.md           (Novel axis #1)
    ├── 10-position-lie-novel.md           (Novel axis #2)
    ├── 11-api-client-hmac.md
    ├── 12-match-uuid-pipeline.md
    └── 13-damage-validator.md             (damage-reduction hack / invincibility hack / HPHack)
```

---

## 4. The 13 modules at a glance

| # | Module | Category | Implementation status | Risk | LoC (≈) |
|---|------|---------|----------|--------|--------|
| 01 | AES-256-GCM Crypter V2 | Encryption | **Implemented, build verified** | Medium (protocol change) | 280 |
| 02 | X25519 ECDHE Key Exchange | Key exchange | **Implemented, build verified** | Medium | 220 |
| 03 | DDoS three-tier gate | Stability | **Implemented, build verified** | Low | 380 (3 modules combined) |
| 04 | IP Relay (Peer Masking) | Topology | **Implemented** | Low | 30 (modifications only) |
| 05 | Position History Ring Buffer | Anti-cheat infrastructure | **Implemented, build verified** | Low | 110 |
| 06 | Movement Validator | Anti-cheat (movement) | **Implemented, build verified** | Medium (tuning required) | 240 |
| 07 | Combat Validator | Anti-cheat (fire/hit) | **Implemented, build verified** | Medium | 320 |
| 08 | Signal Collector (16-shard) | Anti-cheat infrastructure | **Implemented, build verified** | Low | 180 |
| 09 | **Weapon Spoof Detection** | Anti-cheat (novel) | **Implemented, build verified** | Low | 60 |
| 10 | **Position Lie Detection** | Anti-cheat (novel) | **Implemented, build verified** | Low | 80 |
| 11 | API Client (WinHTTP + HMAC) | Reporting pipe | **Implemented, build verified** | Low | 220 |
| 12 | match_uuid Lifecycle | Operability | **Implemented, build verified** | Low | 50 |
| 13 | **Damage Validator** | Anti-cheat (damage/HP) | **Implemented, build verified (Detection-Only mode)** | Medium (game-mechanic review) | 617 (.h 190 + .cpp 427) |

The **Novel items** (#09, #10) are **unique detection axes** that exist in no public reference tree outside this project.

**Module 13** closes the blind spot of the victim-authoritative HIT_TICK (damage-reduction hack / invincibility hack / HPHack). Implemented in **Detection-Only mode** (build verification passed). **Mitigation mode** (`SERVER_DAMAGE_OVERRIDE=true`) can be enabled immediately via an environment-variable toggle once the production environment's weapon damage table has been imported and the regen/skill mechanics enumerated. The default weapon profiles in this package are initialized with values measured in a self-built public-tree environment; replacing them is recommended for live deployment.

---

## 5. Sources / base trees

This work is based on the following public materials:

| Tree | Public location | Use |
|------|----------|------|
| GunZ-The-Duel | Public on GitHub | Community mod / client base |
| RefinedGunz | Public on GitHub | Reference for CMake/C++14 structure |
| Gunz1.5-main (`source-reference8`) | Public on GitHub | Version 1.5 assets/source |
| `source-reference11` | Public on GitHub | UI base |

The **new code (`Security::*` namespace)** included in this package does not exist in the trees above. I acknowledge that the live GunZ service may already be operating an equivalent or more advanced in-house solution; in that case, the value of this package lies in (a) the hack pattern catalog based on play on the official GunZ server + build testing of the public trees, and (b) the tuning values measured in a self-built public-tree environment.

---

## 6. Contact / terms of use

All materials in this package (documents + code + integration guide + validation kit) are **freely available**. There is no NDA stage, license negotiation, or compensation process.

If the company decides to adopt it, use it as-is; if not, that is fine too. In either case, the two author's rights listed at the top of this README remain in effect.

Contact: Hong-gu Lee <2nugu@naver.com>

---

## 7. Change history

| Date | Change |
|------|------|
| 2026-05-06 | Initial version |
