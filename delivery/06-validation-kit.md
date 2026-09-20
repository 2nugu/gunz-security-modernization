**English** | [한국어](./ko/06-validation-kit.md)

# Validation Kit

> **Audience**: QA / security validation team
> **Purpose**: Reproduce this package's behavior on your own within 30 minutes. The answer to "does this actually work?"

---

## 1. Prerequisites

| Item | Version | Notes |
|------|------|------|
| OS | Windows 10/11 or Linux | Linux/macOS is fine for backend only |
| Docker Desktop | 4.x+ | Required |
| Python | 3.11+ | For running pytest |
| Visual Studio | 2022 (v143) or compatible | Client/server build (optional) |
| `curl` | any | For HMAC E2E verification |

---

## 2. Four validation scenarios

### 2.1 Backend full stack (10 min)

Goal: bring up community-api + Postgres + Redis.

```bash
# From the project root
docker compose up -d

# Verify
curl http://127.0.0.1:8100/docs
# → FastAPI Swagger UI should come up

curl http://127.0.0.1:8100/health
# → responds {"status":"ok"}
```

Expected results:
- 3 containers running (`gunz_api`, `gunz_postgres`, `gunz_redis`)
- Postgres alembic head = `003_add_match_uuid_to_signals`
- Redis responds PONG

### 2.2 HMAC E2E — 3 scenarios (5 min)

Goal: confirm that match server ↔ API HMAC verification works.

```bash
# Scenario A — valid signature (200)
SECRET="test-secret-32-bytes-min"
BODY='{"reports":[{"player_uid_high":1,"player_uid_low":2,"signal_type":"WeaponSpoof","severity":"High","value":1.0,"threshold":0,"detail":"test"}]}'
SIG=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$SECRET" -hex | awk '{print $2}')

curl -X POST http://127.0.0.1:8100/api/anticheat/report \
  -H "Content-Type: application/json" \
  -H "X-Signature: $SIG" \
  -d "$BODY"
# Expected: HTTP 200

# Scenario B — duplicate (409)
curl -X POST http://127.0.0.1:8100/api/anticheat/report \
  -H "Content-Type: application/json" \
  -H "X-Signature: $SIG" \
  -H "Idempotency-Key: same-key" \
  -d "$BODY"
curl -X POST http://127.0.0.1:8100/api/anticheat/report \
  -H "Content-Type: application/json" \
  -H "X-Signature: $SIG" \
  -H "Idempotency-Key: same-key" \
  -d "$BODY"
# Expected: second request HTTP 409

# Scenario C — invalid signature (401)
curl -X POST http://127.0.0.1:8100/api/anticheat/report \
  -H "Content-Type: application/json" \
  -H "X-Signature: deadbeefdeadbeef" \
  -d "$BODY"
# Expected: HTTP 401
```

### 2.3 pytest regression (10 min)

Goal: regression-check the backend logic.

```bash
docker compose exec backend pytest -v
# or on the host:
cd community-api
pytest -v
```

Expected results:
```
============== 57 passed, 12 skipped, 3 xfailed in N.NNs ==============
```

Details:
- `test_phase2_integration.py` — OpenAPI router registration + E2E journey
- `test_shop_race.py` — Postgres concurrency (auto-skipped on SQLite)
- `test_anticheat_*.py` — HMAC + duplicates + signal reporting

### 2.4 Match server build (optional, 30 min)

Goal: produce a match server binary that includes the new security modules.

```powershell
# Windows / Visual Studio 2022
cd match-server\build
msbuild MatchServer.sln /p:Configuration=Release /p:Platform=Win32

# Output
ls bin\Release\MatchServer.exe
```

Expected size: ~3.7 MB (Release|Win32)

Build verification:
```powershell
# Check the new symbols with the strings command
strings bin\Release\MatchServer.exe | findstr "MAPIClient"
strings bin\Release\MatchServer.exe | findstr "X-Signature"
strings bin\Release\MatchServer.exe | findstr "/api/anticheat/report"
```

---

## 3. Signal behavior demonstration (live)

### 3.1 Environment setup
```bash
export COMMUNITY_API_URL=http://127.0.0.1:8100
export MATCH_RESULT_WEBHOOK_SECRET="test-secret-32-bytes-min"
match-server\build\bin\Release\MatchServer.exe
```

### 3.2 Reproducing WeaponSpoof (deliberate tampering)

When the test client sends an ATTACK_TICK packet to the server, forge the reported weapon type as a weapon class that is not equipped:

```cpp
// Tampered client code (test only)
ZPostAttackTick(/*WeaponType=*/(BYTE)Rocket);  // actually equipped: Melee
```

Expected results:
1. Match server log: `WeaponSpoof: uid=... reportedClass=Rocket equipped=Melee severity=Critical`
2. community-api log: `POST /api/anticheat/report 200`
3. DB query:
```sql
SELECT signal_type, severity, match_uuid, detail
FROM anticheat_signals
WHERE signal_type = 'WeaponSpoof'
ORDER BY created_at DESC LIMIT 1;
```
→ `match_uuid` is populated in the `stage-<H>-<L>-<epoch>` format

### 3.3 Reproducing SpeedHack

```cpp
// Tampered client — 32 Hz position sends with coordinates accelerated 5x
ZPOSTCMD1(MC_MATCH_POSITION_TICK, x + 1000, y + 1000, z, ...);
```

Expected results:
- First violation: Low severity
- 3 accumulated: escalates to Medium
- 5 accumulated: escalates to High

### 3.4 Reproducing DamageReduce / HPHack (Module 13)

```cpp
// Tampered client — shrink dmg to 1/5 when reporting HIT_TICK
ZPostHitTick(atkUID, weaponType, /*dmg=*/realDmg * 0.2f, srcPos);
```

Expected results:
1. Match server log: `DamageVal: Reduce <uid> wc=<W> dist=<D> reported=<X> server=<Y> ratio=0.2 streak=N`
2. ratio 0.2 < tolerance 0.5 → **High** severity issued immediately
3. HPConsistency check accumulating on a 1-second cycle: after 5 seconds, HP delta vs. sum of hits mismatch → additional `HPHack` signal possible
4. Arrival at community-api (when a callback is registered):
```sql
SELECT signal_type, severity, value, threshold, detail
FROM anticheat_signals
WHERE signal_type IN ('DamageManipulation', 'HPHack')
ORDER BY created_at DESC LIMIT 5;
```

### 3.5 Reproducing PositionLie

```cpp
// Tampered client — forge the HIT_TICK srcPos to a coordinate 1500u away from the actual position
ZPostHitTick(targetUID, weaponType, dmg, /*srcPos=*/{X+1500, Y+1500, Z});
```

Expected results:
- If the weapon class is SMG, threshold 800u; 1500u is 1.875x → Medium
- If the weapon class is Melee, threshold 400u; 1500u is 3.75x → High

---

## 4. DDoS gate demonstration

### 4.1 L1 — Connect Flood
```bash
# 11 consecutive connection attempts from the same IP
for i in {1..11}; do
  nc -z 127.0.0.1 7777 &
done
```

Expected:
- First 10 are normal (TCP accept followed by ECDHE handshake)
- 11th: immediate `Disconnect` — match server log shows `MConnectRateLimit: blocked IP=127.0.0.1`

### 4.2 L2 — Packet Flood
```cpp
// Tampered client — send more than 256 packets within 1 second
for (int i = 0; i < 300; ++i) ZPOSTCMD0(MC_MATCH_HEARTBEAT);
```
Expected: session disconnect starting from the 257th packet.

### 4.3 L3 — Bandwidth
```cpp
// Tampered client — send more than 256 KiB within 1 second
for (int i = 0; i < 100; ++i) ZPOSTCMD1(LARGE_BLOB, 4096B);
```
Expected: disconnect at the point the cumulative total exceeds 256 KiB.

---

## 5. Demo videos (if provided)

| Video | Length | Content |
|------|------|------|
| 01-handshake.mp4 | 2~3 min | ECDHE handshake + Wireshark packet capture. v1 plaintext vs v2 ciphertext comparison |
| 02-ddos-gate.mp4 | 1~2 min | L1/L2/L3 behavior. Deliberate flood → gate logs |
| 03-anticheat.mp4 | 3~5 min | WeaponSpoof / PositionLie / SpeedHack reproduction. Signal → arrival on the community-api dashboard |

Videos are **subtitles only, no audio** — avoids the audio burden when circulated internally within the company.

---

## 6. Validation report template

```
[Project]     GunZ Security Modernization Validation Report
[Validator]   ___
[Date]        ___
[Environment] OS / Docker / Python versions

[Scenario results — implemented modules (#01~#12)]
- Backend full stack up:             [ Pass / Fail ]
- HMAC E2E 3 scenarios:              [ Pass / Fail ]   200/409/401 all responded ___
- pytest regression:                 [ Pass / Fail ]   passed/skipped/xfailed = ___
- Match server build:                [ Pass / Fail / N/A ]   binary size ___
- WeaponSpoof signal arrived:        [ Pass / Fail / N/A ]
- SpeedHack escalation:              [ Pass / Fail / N/A ]
- PositionLie per-weapon threshold:  [ Pass / Fail / N/A ]
- DDoS L1/L2/L3 blocking:            [ Pass / Fail / N/A ]

[Scenario results — Module 13 (Detection-Only mode)]
- MDamageValidator build verification:   [ Pass / Fail ]   new symbols confirmed in binary strings
- Weapon profile registration (RegisterDefaultProfiles): [ Pass / Fail ]
- DamageReduce signal (reported dmg < server × 0.7): [ Pass / Fail / N/A ]
- DamageInflate signal (reported dmg > server × 1.3): [ Pass / Fail / N/A ]
- HPConsistency window check:            [ Pass / Fail / N/A ]
- HPHack signal (HP delta mismatch):     [ Pass / Fail / N/A ]
- OnPlayerLeave memory cleanup:          [ Pass / Fail / N/A ]
- Mitigation toggle (SetMitigationMode): [ N/A — enable after validating the live damage flow ]
  → Enable after adding a Mitigation branch to the OnPeerDamage path in ZRule* and deciding on regen-mechanic registration

[Observations]
- ___

[Questions]
- ___
```

---

## 7. Common problems

### 7.1 Docker container fails to start
- Host port 8000 occupied (Manager.exe etc.) → check the 8100 mapping in docker-compose.yml
- Postgres data directory permissions → `docker compose down -v` then restart

### 7.2 pytest failures
- `@pytest.mark.postgres` guard → auto-skipped in SQLite environments (normal)
- Missing dependencies → `pip install -r requirements-dev.txt`

### 7.3 Match server build failure
- v143 PlatformToolset → specify `/p:PlatformToolset=v143` explicitly on the command line
- Missing libsodium headers → check the include paths in `Directory.Build.props`
- WindowsTargetPlatformVersion → 10.0.26100.0

### 7.4 ECDHE handshake failure
- Host without AES-NI support → `MPacketCrypterV2::InitKey` returns false, with an explicit message in the log
- Server/client KDF divergence → check the canonical min‖max ordering (no `#ifdef BUILD_MATCH_SERVER` branching)

---

## 8. Next steps after validation

If validation passes:
1. Sign the NDA (access to module source / integration guide)
2. Review [`08-integration-guide.md`](./08-integration-guide.md)
3. Decide on phased adoption starting with Phase A (DDoS)
4. Discuss the license mode ([`03-license-inventory.md`](./03-license-inventory.md))

If validation fails:
1. Document the failing scenario
2. Identify environment differences (OS / build tools / dependency versions)
3. Send environment information + logs to the author of this package
