**English** | [한국어](./ko/05-operations-runbook.md)

# Operations Runbook

> **Audience**: operators / GM team / live operations engineers
> **Purpose**: the day-to-day operational flow of signal intake → judgment → sanction.

---

## 1. Daily workflow (Daily Loop)

```
┌─────────────────────────────────────────────────┐
│ 1. Check the operator queue (at start / every 4 h) │
│    └─ community-api dashboard / Slack alerts    │
└─────────────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────┐
│ 2. Sort by priority                             │
│    Critical > High > Medium                     │
│    Items with weighted score ≥ threshold (default 50) first │
└─────────────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────┐
│ 3. Handle each case                             │
│    a. Review signal details (type/value/threshold) │
│    b. Check per-round aggregation by match_uuid │
│    c. Monitor via ghost-mode if needed          │
│    d. Review replay if needed (once Phase 6 is introduced) │
└─────────────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────┐
│ 4. Verdict                                      │
│    ├─ Normal (false positive) → label as noise  │
│    ├─ Suspected hack (single) → send warning    │
│    ├─ Confirmed hack (repeated) → temporary suspension │
│    └─ Confirmed hack (malicious) → permanent ban │
└─────────────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────┐
│ 5. Post-hoc record                              │
│    - Basis for the verdict (signal_type, severity, value) │
│    - Outcome of handling                        │
│    - Threshold adjustment data (handed to the tuning team monthly) │
└─────────────────────────────────────────────────┘
```

---

## 2. Response guide by signal type

### 2.1 Movement class (100-199)

| Signal | First response | Notes |
|--------|--------|------|
| `SpeedHack` (101) | High or above → ghost-mode for 30 min | Legitimate dash ~1500 u/s passes, 1800+ cut off |
| `TeleportSuspect` (102) | Medium or above → check accumulation per match_uuid | Respawn/warp are exempt. Review if exemption bypass is suspected |
| `FlyHack` (103) | Capped at Medium → never a standalone verdict | airtime heuristic, false positives possible on jump maps |

### 2.2 Combat class (200-299)

| Signal | First response | Notes |
|--------|--------|------|
| `RapidFire` (201) | High → ghost-mode | Per-weapon-class threshold |
| `InfiniteSlash` (202) | High → ghost-mode | Melee 450±100 ms |
| `AutoAim` (203) | Medium → accumulate match statistics | hit-rate > 95% after 50 shots is never a standalone verdict; pattern accumulation required |
| `ImpossibleHit` (204) | High → standalone verdict possible | Exceeds range × 1.3 |
| `WeaponSpoof` (205) | **Critical → immediate ghost-mode + alert** | Based on server-authoritative data, almost no false positives |
| `DamageReduce` (220) | High → ghost-mode | Victim-reported dmg < server-computed × 0.7. False positives possible if a legitimate heal/skill is not registered — check the healing mechanic enum |
| `DamageInflate` (221) | High → standalone verdict possible | Victim-reported dmg > server-computed × 1.3. Damage-inflation hack (1-shot suspected) |
| `DamageNullification` (222) | High → ghost-mode | HIT_TICK not received within 300 ms after the attacker's PEER_SHOT. Invincibility hack / HIT_TICK blocking suspected |
| `HPHack` (223) | **Critical → immediate ghost-mode + alert** | HP delta vs reported hit sum mismatch by 5+. HP memory manipulation hack, almost no false positives |
| `HPInconsistency` (224) | Medium → accumulate per match | HP recovers for no known reason. Escalate to High if it recurs after legitimate healing mechanics are registered |

**Operational caution for the Damage class (220-224)**:
- Meaning differs depending on Module 13's deployment mode
  - **Detection-Only Mode**: signals only are emitted; the hack's effect is not neutralized. Respond with manual operator sanctions
  - **Mitigation Mode**: signals emitted + victim HP deduction forced to the server-computed value → the hack's effect itself is neutralized
- Run the first 1-2 weeks after introducing Module 13 in Detection-Only → measure the false-positive distribution → tune tolerance → enable Mitigation

### 2.3 Network class (500-599)

| Signal | First response | Notes |
|--------|--------|------|
| `PacketManipulation` (501) | High → ghost-mode | PositionLie. High when exceeding 2× the per-weapon threshold |
| `ReplayAttack` (502) | (automatically blocked, AEAD nonce) | Alert only, no operational action required |

---

## 3. Ghost Mode (transparent administrator connection) operation

### 3.1 How to enter
- Match server console command `/ghost <player_uid>` (to be implemented)
- Or call the community-api admin router

### 3.2 What is visible in ghost state
- The suspected player's first-person view (optional)
- Real-time signal stream (filtered to that player only)
- Positions of all players in the match (debug view)

### 3.3 What is *not* visible in ghost state
- The ghost itself is not exposed to the suspected player (excluded from peer broadcast)
- No effect on game results (death handling, score, etc.)

### 3.4 Operating principles
- The fact of entering ghost is itself preserved in logs (abuse prevention)
- Ghost time limit (default 60 min), renewable
- Attach ghost results to the case notes

---

## 4. Threshold tuning guide

### 4.1 Tuning cadence
- Regular review once a month
- Ad-hoc review when false-positive reports accumulate
- Event review when a new hack appears

### 4.2 Tunable environment variables

| Environment variable | Default | Meaning |
|---------|-------|------|
| `MOVEMENT_MAX_SPEED_UPS` | 1800 | Speed threshold (u/s) |
| `MOVEMENT_TELEPORT_DIST_PER_SEC` | 5000 | Teleport threshold (u/s) |
| `MOVEMENT_FLYHACK_AIRTIME_MS` | 5000 | FlyHack airtime threshold |
| `COMBAT_MELEE_INTERVAL_MS` | 450 | Melee fire interval |
| `COMBAT_AUTOAIM_MIN_SHOTS` | 50 | AutoAim minimum shot count |
| `COMBAT_AUTOAIM_HIT_RATE` | 0.95 | AutoAim hit-rate threshold |
| `POSITIONLIE_MELEE_DIST` | 400 | PositionLie Melee threshold (units) |
| `POSITIONLIE_ROCKET_DIST` | 1000 | PositionLie Rocket threshold |
| `DDOS_CONNECT_PER_IP_PER_10S` | 10 | L1 connect rate |
| `DDOS_PACKETS_PER_SESSION_PER_S` | 256 | L2 packet rate |
| `DDOS_BYTES_PER_SESSION_PER_S` | 262144 | L3 bandwidth (256 KiB) |
| `SIGNAL_REPORT_THRESHOLD_SCORE` | 50 | Weighted score reporting threshold |
| `SERVER_DAMAGE_OVERRIDE` | false | Module 13 — false=Detection-Only, true=Mitigation |
| `DAMAGE_TOLERANCE_LOWER` | 0.7 | Reported dmg < server dmg × 0.7 → DamageReduce |
| `DAMAGE_TOLERANCE_UPPER` | 1.3 | Reported dmg > server dmg × 1.3 → DamageInflate |
| `DAMAGE_NULLIFICATION_TIMEOUT_MS` | 300 | Wait time for HIT_TICK after PEER_SHOT |
| `HP_CONSISTENCY_WINDOW_MS` | 5000 | HP consistency validation window |
| `HP_CONSISTENCY_TOLERANCE` | 5.0 | HP absolute tolerance |

### 4.3 Tuning procedure
1. Collect false-positive / false-negative cases (monthly)
2. Decide candidate thresholds (existing ±10-30%)
3. **Apply in the dev/staging environment for 1 week**
4. Compare false-positive/negative rates
5. Apply or roll back

### 4.4 Cautions when tuning
- Change a single environment variable → affects a single signal
- Changing several variables at once makes effects hard to separate
- Record the change history in both git commits and the operations log

---

## 5. False positive / negative reporting flow

### 5.1 False positive (a legitimate user is flagged)
```
User report → CS intake → operator first review (ghost / replay)
         → ruled normal → close case + accumulate threshold tuning data
         → ruled hack → formal handling (re-enter step 3)
```

### 5.2 False negative (a hacking user is not flagged)
```
Player report → CS intake → operator traces the match ID (match_uuid)
            → confirm no signal was raised → possible new hack pattern
            → request signal addition work from the security team
```

---

## 6. Response when a new hack appears

```
1. Collect cases (CS intake + direct observation)
2. Define the pattern
   - Which packets are tampered with, and how
   - Which game state is abnormal
   - What signals are observable server-side
3. Signal addition work (security team)
   - Assign a new ID in the CheatSignalType enum
   - Add a Validator or inline handler
   - Initial threshold estimate
4. dev/staging validation
5. Staged production rollout
   - First 1 week: emit Low severity only (observe)
   - Apply the appropriate severity after confirming false positives
6. Update the operator guide
```

---

## 7. Emergency scenarios

### 7.1 Anticheat module produces a surge of false positives
- Immediately disable the signal via environment variable: `SIGNAL_<TYPE>_ENABLED=false` (once implemented)
- Or raise the signal reporting threshold (raise `SIGNAL_REPORT_THRESHOLD_SCORE`)
- Temporarily clear the operator queue
- **Case specific to Module 13 (Damage)**: if false positives occur in Mitigation Mode, immediately toggle `SERVER_DAMAGE_OVERRIDE=false` → revert to Detection-Only (hack effects work again, but the damage flow is normalized). Takes effect after a match server restart.

### 7.2 community-api down → signal reporting fails
- The retry queue of `MAPIClient` (MAX 100) holds them temporarily
- Beyond 100, FIFO drop — signal loss occurs
- Automatic retransmission after recovery
- **No effect on gameplay** (anti-cheat is fail-open)

### 7.3 DDoS gate false positive (legitimate users blocked)
- Introduce an IP whitelist (once implemented)
- Or temporarily raise thresholds (`DDOS_CONNECT_PER_IP_PER_10S`, etc.)

### 7.4 Match server ↔ community-api HMAC secret leaked
- Rotate the secret immediately (generate a new 32B value)
- Update `MATCH_RESULT_WEBHOOK_SECRET` on both sides simultaneously
- Restart match server + community-api
- Write an incident report (fact of rotation + time + scope of impact)

---

## 8. Recommended metrics / monitoring items

| Metric | Frequency | Alert threshold |
|--------|------|----------|
| Signals per hour (total) | 1 min | Usual average × 3 |
| Signals per hour (per type) | 1 min | Per-type average × 5 |
| `MAPIClient` retry queue depth | 1 min | 50 or more |
| `MAPIClient` 5xx response rate | 1 min | 5% or more |
| DDoS L1 blocked IP count | 5 min | Usual average × 5 |
| DDoS L2/L3 violating session count | 5 min | Usual average × 5 |
| Match server memory usage | 1 min | 80% or more |
| `anticheat_signals` table insert rate | 5 min | (baseline to be decided) |

---

## 9. Operator privilege separation (RBAC recommended)

| Role | Privileges |
|------|------|
| **CS** | View signals / write case notes |
| **GM (Junior)** | + ghost-mode (with limits) / send warnings |
| **GM (Senior)** | + temporary suspension / propose threshold tuning |
| **Security** | + permanent ban / apply threshold tuning / disable Validators |
| **Engineering** | + change environment variables / modify code |

Automatic ban privileges are granted to no role.

---

## 10. Operator training checklist

When onboarding a new operator:

- [ ] Read this Runbook in full
- [ ] Read [`04-anticheat-catalog.md`](./04-anticheat-catalog.md) in full
- [ ] Understand the forbidden signals (§3, catalog) — headshot ratio, etc.
- [ ] Understand the no-automatic-ban policy
- [ ] Hands-on practice with ghost-mode
- [ ] Hands-on practice tracing rounds by match_uuid
- [ ] Simulate 5 false-positive cases
- [ ] Practice filling out the new-hack report form
