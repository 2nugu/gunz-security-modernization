**English** | [한국어](./ko/04-anticheat-catalog.md)

# GunZ Anti-Cheat Catalog

> **Audience**: operators / GM team / security analysts
> **Purpose**: catalog of hack patterns observed while playing on the official GunZ server and in self-built tests of the public source tree, with the detection mapping of this package. States explicitly why generic FPS anti-cheat literature does not apply to GunZ as-is.
> **Principle**: no automatic bans — signal → severity classification → operator review. The cost of a false positive exceeds that of a false negative.

---

## 1. How GunZ differs from a generic FPS (premises of the detection design)

| Trait | Generic FPS (CS/Valorant) | GunZ |
|------|----------------------|------|
| Camera sensitivity | Normal to low | Very high (action camera) |
| Aim location | Head first | **Legs first** (the leg trajectory of an airborne character is easy to predict) |
| Primary weapon | Rifle/pistol | **Shotgun-heavy** (wide spread makes precise aim meaningless) |
| Movement | Strafing | **Air dash / butterfly / slash shot** — camera rotates on sub-second scales |
| Line of sight | Corner holding | Wall-run/jump produces sudden appearances on top of or behind walls |
| Time scale | Seconds/minutes | Slash combo in a 30-50 ms window |

**What this difference means**: many signals designed for generic FPS degrade into **pure play-style signals** in GunZ.

---

## 2. Hack catalog (by observed frequency)

### 2.1 Movement class

| Hack | Behavior | Detected by this package | Module | severity |
|----|------|---------------|------|----------|
| **Speed hack** | Applies a movement speed multiplier | ✅ | `MMovementValidator` | Speed > 1800 u/s cutoff |
| **Teleport hack** | Instantaneous position jump | ✅ | `MMovementValidator` | TeleportSuspect, based on Distance/dt |
| **Fly hack** | Levitation ignoring gravity | △ (heuristic) | `MMovementValidator` | FlyHack airtime, capped at Medium |
| **Noclip** | Passing through walls | ✗ (BSP not integrated) | — | Requires linking the RealSpace2 engine first |
| **Jump hack / infinite jump** | Bypasses the airborne jump count limit | △ | Future `MMovementValidator` extension | jumpCount validation not implemented |

### 2.2 Combat class — firing / hits

The **most common hack category** observed while playing on the official GunZ server. Infinite slash and RapidFire in particular are frequent.

| Hack | Behavior | Detected by this package | Module | severity |
|----|------|---------------|------|----------|
| **Infinite slash / InfiniteSlash** ⭐ | Spams slash ignoring the cooldown. GunZ's signature hack | ✅ | `MCombatValidator::ValidateAttack` | Melee 450±100 ms threshold, escalation on consecutive violations |
| **RapidFire (firearms)** ⭐ | Ignores the weapon fire interval | ✅ | `MCombatValidator::ValidateAttack` | Per weapon (Shotgun 1100±150, Revolver 300±80, SMG 70±30, Rifle 100±40, Rocket 1800±200) |
| **Weapon cooldown bypass (general)** | Common to all weapon classes | ✅ | Handled together with the two items above | — |
| **AutoAim / aimbot** | Automatic target tracking | △ (statistical) | `MCombatValidator::ValidateHit` | hit-rate > 95% after 50 shots (never a standalone verdict) |
| **On/off aimbot** | Activated only at decisive moments | ✗ | — | Unsolved (§5) |
| **ImpossibleHit** | Hit beyond weapon range | ✅ | `MCombatValidator::ValidateHit` | Weapon range × 1.3 |
| **WeaponSpoof** | Attacks with a weapon that is not equipped | ✅ | inline ATTACK_TICK | **Novel axis**, cross-checked against server-authoritative 3-slot state |
| **PositionLie / PacketManipulation** | Attacker tampers with srcPos | ✅ | inline HIT_TICK | **Novel axis**, POSITION × HIT cross-check |
| **Infinite ammo hack** | Ignores an empty magazine | ✗ | — | Requires introducing a server-authoritative magazine count |

### 2.2b Combat class — damage / hits taken (victim-auth blind spot)

The blind spot created by the fact that this package's HIT_TICK is **victim-authoritative**. **Module 13 (Damage Validator)** closes it.

| Hack | Behavior | Detected by this package | Module | severity |
|----|------|---------------|------|----------|
| **Damage-reduction hack (DamageReduce)** ⭐ | Victim reports received damage as 1/N or omits it | ✅ | `MDamageValidator` ServerDamageOverride + tolerance validation | Reported < server-computed × 0.7 → High |
| **Invincibility hack (Invincibility)** | Victim blocks all HIT_TICKs | ✅ | `MDamageValidator` DamageNullification (PEER_SHOT × HIT_TICK matching) | High |
| **HP hack (HPHack)** | Victim manipulates its own HP directly in memory | ✅ | `MDamageValidator` HPConsistency (HP delta vs sum of hits) | Critical |
| **Auto-heal / regen hack** | HP recovers for no known reason | ✅ | `MDamageValidator` HPInconsistency | High |
| **Damage-inflation hack (DamageInflate)** | Boss-level damage from a single hit (1-shot) | ✅ | `MDamageValidator` reported > server-computed × 1.3 | High |

#### Core design — Server-Side Damage Override
- Ignore client-reported damage. The server deducts from the victim's HP the value it computes from the weapon table + distance falloff
- The victim's HIT_TICK is trusted only for **the fact that a hit occurred**; **the damage amount is server-authoritative**
- Dual-report cross-check: time gap between arrival of the attacker's PEER_SHOT and arrival of the victim's HIT_TICK ≤ 300 ms

#### HPConsistency — the most reliable check
```
Over the time window (5 s):
    sumHits     = Σ HIT_TICK.dmg
    HP_expected = HP_start - sumHits + legitimate healing
    if abs(HP_now - HP_expected) > 5: HPHack
```

Details → [`modules/13-damage-validator.md`](./modules/13-damage-validator.md)

### 2.3 Information / vision class

| Hack | Behavior | Detected by this package | Notes |
|----|------|---------------|------|
| **ESP / box hack** | Visualizes enemy positions | ✗ | Client memory protection domain; cannot be detected directly on the server |
| **Map hack** | Minimap exposure + pre-aim | △ (indirect) | Statistical — signal when tracking behavior through occluders accumulates |
| **Pre-aim through walls** | Aim already completed before the enemy enters line of sight | △ | Worth investigating, not implemented |

### 2.4 Meta / operations class

| Hack | Behavior | Detected by this package | Notes |
|----|------|---------------|------|
| **DLL injection** | Injects external code into the client process | ✗ | Client anti-cheat domain (not included in this package) |
| **Memory patching** | Direct modification of Gunz.exe memory | ✗ | Client anti-cheat domain |
| **Packet tampering** | Modifies packets bound for the server | ✅ | AES-256-GCM AEAD detects tampering via the GCM tag |
| **Replay Attack** | Retransmits past packets | ✅ | Sequence counter in the nonce |
| **Account Sharing / Boost** | Rank boosting via account sharing | ✗ | Policy domain, unrelated to this package |

---

## 3. Forbidden signals (high false-positive risk — do not implement)

Given the nature of GunZ play, the signals below **occur readily in pure human play**. Weight them 0 or do not include them in code:

| Signal | Why it is invalid in GunZ |
|--------|-------------------|
| Crosshair snap speed (e.g. > 60° rotation in < 20ms) | High sensitivity + action play + slash shot patterns are normal |
| Headshot ratio | GunZ players aim at the legs. Typical 5-10%, pros 15% or less. A high ratio is what is anomalous |
| Shotgun accuracy | Wide spread makes "was the aim precise" meaningless |
| Single-frame angle change | The camera itself rotates during a dash |
| Reaction time (< 50 ms) | Achievable through slash-combo muscle memory. Pros genuinely react in < 50 ms |

Flagging on these signals means **pro players get caught over and over** — the false-positive cost is the operations team's trust.

---

## 4. Signals worth investigating (not implemented)

### 4.1 Persistence of abnormal aiming

Core idea — during high-speed combat, humans show micro-jitter, drift, over-aiming and correction. An aimbot, once it acquires a target, holds it.

| Metric | Description | Implementation difficulty |
|------|------|-----------|
| Target-lock persistence frames | Consecutive frames in which the crosshair stays within ±N px of the enemy while both are moving | Medium |
| Aim stability during slash motion | Suspicious if aim does not shake during shaking motion frames | Medium |
| Reacquisition time after dash | < 50 ms + straight, constant-velocity convergence trajectory | High |
| Absence of mouse micro-jitter | Absence of fine oscillation at 10 Hz or above | Very high (requires sending the input stream to the server) |
| Second derivative of the aim trajectory | Humans accelerate, decelerate and overshoot; bots smoothstep | Very high |

This package does **not implement** this area. Sending the input stream to the server carries a heavy bandwidth burden and requires integration with a client anti-cheat.

### 4.2 Conditional accuracy (selective statistics)

Overall hit rate is meaningless, but **selectively high hit rates only under hard conditions** are a strong signal.

| Condition | Why it is hard | Bot vs human |
|------|-----------|-----------|
| First-shot hit (immediately after entering line of sight) | Humans need perception and reaction time | Bot locks instantly |
| Firing right after a 180° turn | Camera inertia + re-aiming required | Bot fires exactly on the frame the rotation completes |
| Long-range shot during an airborne crossing | Hard to predict | Bot computes the prediction line itself |
| Counterattack immediately after being hit | Hit-stagger motion + divided attention | Bot is unaffected |

This package does **not implement** this area. Per-match distribution analysis combined with operator review is recommended.

---

## 5. The hardest case — the on/off aimbot

### Why it is hard
- Normally pure human play → aggregate averages look normal
- Activated for only a few shots at decisive moments (1:1 situations, flag fights, end of round)
- One or two "miraculous shots" per match are enough to flip the outcome of a fight

### Candidate detection approaches (all unsolved)

| Approach | Principle | Risk |
|------|------|------|
| **Aggregate only "miraculous shots"** | Narrow the §4.1/§4.2 signals to the firing-frame window and score individual shots | Definition is vague; a pro's "shot of a lifetime" shows the same pattern |
| **Post-hoc replay review** | Signal flag → administrator jumps to the suspicious shot frame and reviews by eye | Requires operations staff. Consistent with the no-automatic-ban principle |
| **Per-match bimodal analysis** | Whether one player's shot-quality distribution is a normal cluster + an outlier cluster | Noisy with insufficient samples; needs aggregation over hundreds of matches |
| **Session deviation from long-term trend** | Outlier ratio spikes in a particular session relative to the personal baseline | Skill improvement / being in good form gives the same signal |
| **Weighted community reports** | Mix player reports into the signal | Threat of malicious mass reporting |

None of these approaches is implemented in this package. **Additional work after an operations policy decision** is required.

---

## 6. Severity classification policy

```
CheatSeverity:
  Low      = 1 point  (single occurrence, near threshold)
  Medium   = 3 points
  High     = 7 points
  Critical = 10 points

Weighted score = (Low count) + 3×(Med) + 7×(High) + 10×(Crit)
```

### Classification guide
- **Low**: single violation, possible measurement noise. False-positive rate may be high
- **Medium**: 1.5-2× the threshold. Intent suspected
- **High**: 2× the threshold or more, or accumulated consecutive violations. Operator priority queue
- **Critical**: immediate automatic enrollment in ghost-mode monitoring. E.g. WeaponSpoof immediately.

### Escalation rules (e.g. `MMovementValidator`)
```
Consecutive violations    severity
1                 Low
3                 Medium
5                 High
10                Critical
```

Reset condition: N minutes of normal behavior or end of match.

---

## 7. Operator decision flow

```
Signal raised
    │
    ▼
Weighted score accumulated (16-shard Collector)
    │
    ├── score < threshold → log only, not shown on dashboard
    │
    ├── threshold ≤ score < high threshold → operator queue
    │       │
    │       ▼
    │   Operator review (ghost-mode monitoring or replay)
    │       │
    │       ├── ruled normal → label as noise, threshold tuning data
    │       └── ruled hack → manual sanction (warning / temporary suspension / permanent)
    │
    └── score ≥ high threshold → automatic ghost-mode enrollment + operator alert
```

No automatic ban stage. Every sanction passes through an operator's hands.

---

## 8. Future work (in priority order)

1. **Live tuning of weapon damage profiles** — replace Module 13's table with the company's live GunZ weapon balance table
2. **Precise hit zone (head/body/legs) determination** — apply per-zone multipliers (currently simplified to the Body default)
3. **Shotgun multi-pellet handling policy** — decide between per-pellet vs aggregated reporting
4. **Splash damage (Rocket/Grenade)** — automatic computation of victims within the radius
5. **BSP terrain sampling** — precise noclip detection (requires linking the RealSpace2 engine first)
6. **Server-authoritative magazine count** — blocks the infinite ammo hack
7. **Direction/heading spoofing validation** — consistency of the attacker's view vector
8. **Absolute map bounding-box check** — blocks movement outside the map
9. **Time-rewind cross-check** — hit validation integrated with lag compensation
10. **§4.1 aiming persistence signals** — after deciding the input-stream transmission architecture
11. **§4.2 conditional accuracy** — after building per-match statistics infrastructure
12. **On/off aimbot countermeasures** — after the operations policy decision
13. **Client anti-cheat** (separate package)

---

## 9. References

- Validator modules:
  - Movement → [`modules/06-movement-validator.md`](./modules/06-movement-validator.md)
  - Combat (firing/hits) → [`modules/07-combat-validator.md`](./modules/07-combat-validator.md)
  - **Damage / HP** → [`modules/13-damage-validator.md`](./modules/13-damage-validator.md)
- Novel axes:
  - WeaponSpoof → [`modules/09-weapon-spoof-novel.md`](./modules/09-weapon-spoof-novel.md)
  - PositionLie → [`modules/10-position-lie-novel.md`](./modules/10-position-lie-novel.md)
- Operations flow → [`05-operations-runbook.md`](./05-operations-runbook.md)
