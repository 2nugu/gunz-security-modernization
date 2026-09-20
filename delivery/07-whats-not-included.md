**English** | [한국어](./ko/07-whats-not-included.md)

# What's NOT Included

> **Readers**: all stakeholders
> **Purpose**: state explicitly the areas this package does not cover. As an honesty signal, the company must be able to know clearly "if we take this, what gets solved and what remains".

---

## 1. Client Anti-Cheat

| Item | This package | Separate work recommended |
|------|----------|---------------|
| Memory protection (PAGE_GUARD, anti-tamper) | ✗ | ✅ |
| DLL injection defense | ✗ | ✅ |
| Debugger detection (countering IsDebuggerPresent bypass) | ✗ | ✅ |
| Code integrity check (CRC / hash) | ✗ | ✅ |
| Hook detection (IAT/EAT/inline) | ✗ | ✅ |
| Virtual machine detection | ✗ | △ (generally not recommended) |
| Blocking client packet tampering itself | ✗ | ✅ |
| External anti-cheat solution integration (XIGNCODE, etc.) | ✗ | Company decision |

This package is centered on **server-side validation**. Client memory / code region protection is a different domain, and the standard is for the company to operate a separate solution internally (its own or an external anti-cheat vendor).

---

## 2. Precise Noclip / Terrain Validation

| Item | This package | Reason |
|------|----------|------|
| BSP terrain sampling | ✗ | Requires linking the RealSpace2 engine first |
| Wall pass-through detection | ✗ | Same |
| Terrain absolute Z range check | △ | FlyHack heuristic only (false-positive risk on jump maps) |
| Map bounding-box absolute coordinate check | ✗ | Future work item |

Precise noclip detection requires access to RealSpace2's BSP tree to judge whether a position sample is inside or outside the terrain. This package does not proceed to the engine integration stage.

---

## 3. In-Game Resource Validation

| Item | This package | Reason |
|------|----------|------|
| Server-authoritative magazine count | ✗ | Needed to block infinite-ammo hacks. Separate work |
| Per-weapon max damage table (structure) | △ | Structure exists in Module 13. The exact balance table of the company's live GunZ must be replaced with company-side data |
| Precise hit zone (head/body/legs) judgment | ✗ | Module 13 defaults to Body. Exact hit zones require RealSpace2 collision integration |
| Shotgun multi-pellet policy | ✗ | Decision needed: per-pellet vs aggregated reporting |
| Splash damage (Rocket/Grenade) automatic radius calculation | ✗ | Automatic detection of victims within radius not implemented |
| Item/skill cooldown authority | ✗ | This package validates fire interval (RapidFire) only |
| Character stat authority | ✗ | This package validates position/attack/HP only |
| Legitimate recovery (medkits, skills) registration API | △ | Only the `RecordHPRestore(amount, source)` interface of Module 13 is specified. Actual game logic integration is separate |

---

## 4. Advanced Anti-Cheat Signals

| Item | This package | Reason |
|------|----------|------|
| Aim persistence (micro-jitter) | ✗ | Requires sending the input stream to the server + client anti-cheat integration |
| Second-derivative analysis of aim trajectory | ✗ | Same |
| Conditional accuracy (selective statistics) | ✗ | Requires building per-match statistics infrastructure |
| On/off aimbot detection | ✗ | Unresolved ([`04-anticheat-catalog.md`](./04-anticheat-catalog.md) §5) |
| Pre-aim through walls | ✗ | Requires line-of-sight computation infrastructure |
| ESP / box hack detection | ✗ | Client memory protection domain |
| Direct map hack detection | ✗ | Client memory protection domain |
| Machine-learning-based behavior analysis | ✗ | Separate project |

---

## 5. Network / Protocol

| Item | This package | Reason |
|------|----------|------|
| TLS 1.3 login channel wrapper | ✗ | Planning stage (Phase 7) |
| ASIO asynchronous sockets | ✗ | Performance optimization, low priority |
| QUIC / UDP transition | ✗ | Game protocol redesign domain |
| v1 / v2 negotiation | ✗ | Currently assumes cutover. Gradual rollout is separate work |
| Cross-region matchmaking | ✗ | Unrelated to this package |

---

## 6. Operations Tools

| Item | This package | Reason |
|------|----------|------|
| Signal dashboard UI | ✗ | community-api provides the data; dashboard is separate |
| Ghost-mode client tool | ✗ | Design only, implementation separate |
| Automatic replay saving / bookmarks | △ | Automatic recording is enabled; signal → replay linkage is separate |
| Operator RBAC system | ✗ | Privilege separation recommended only |
| Notification system (Slack/Discord bots) | ✗ | Separate work on top of community-api |
| Statistics dashboard (Grafana, etc.) | ✗ | Metrics export only, dashboard separate |

---

## 7. Data / DB

| Item | This package | Reason |
|------|----------|------|
| MS-SQL → Postgres migration tool | ✗ | Alembic new schema only; data migration tool separate |
| ETL of existing operational data | ✗ | Same |
| Backup / recovery policy | ✗ | Company operating policy domain |
| Sharding / replication | ✗ | Company infrastructure domain |

---

## 8. Build / Platform

| Item | This package | Reason |
|------|----------|------|
| x64 build | ✗ | Win32 only verified. Low priority |
| Linux match server | ✗ | Win32 IOCP dependency |
| ARM build | ✗ | Same |
| Android / iOS | ✗ | Mobile GunZ is a separate project |
| Steam / external platform integration | ✗ | Company decision |

---

## 9. Game Logic / Content

| Item | This package | Reason |
|------|----------|------|
| Adding new game modes | ✗ | This package is security only |
| Balance changes | ✗ | Same |
| New weapons / items | ✗ | Same |
| UI / UX improvements | ✗ | Same |
| Graphics / shader improvements | ✗ | Same |
| Sound / VFX | ✗ | Same |

---

## 10. Live Operations Integration

| Item | This package | Reason |
|------|----------|------|
| Wire compatibility with live GunZ | ✗ | Separate work after reviewing the company's live tree |
| Integration with the company's in-house Auth / Account system | ✗ | This package is a standalone community-api |
| Integration with the company's in-house payment / cash system | ✗ | Separate work |
| Integration with the company's in-house logging / analytics | ✗ | Separate work |
| Operating policy / SLA | ✗ | Company decision |

---

## 11. Validation

| Item | This package | Reason |
|------|----------|------|
| Live-run match server ↔ client handshake logs | ✗ | Build verification only completed; live-run smoke test not performed |
| Threshold tuning based on live traffic distribution | ✗ | Estimates from a self-built public-tree environment. Re-tuning after measuring company traffic recommended |
| 10K+ concurrent connection load test | ✗ | Requires separate infrastructure |
| Security penetration test (external audit) | ✗ | Company security team or external audit recommended |
| Static analysis (Coverity, SonarQube) | ✗ | Registration in the company's in-house analysis system recommended |

---

## 12. One-Line Summary

This package focuses on **(a) the 5 known attack surfaces of the original protocol security**, **(b) a hack pattern catalog based on GunZ official-server play + public-tree build testing**, and **(c) 2 novel detection axes**. All other areas are recommended to be handled by the company's own work or by a separate package.
