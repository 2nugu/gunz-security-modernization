**English** | [한국어](../ko/modules/05-position-history.md)

# Module 05 — Position History Ring Buffer

## Purpose
Server-authoritative coordinate infrastructure. Stores the coordinate time series consumed by the anti-cheat Validators. Foundation for lag-comp / rewind validation.

## Interface

```cpp
namespace Security {

struct PositionSample {  // POD
    float x, y, z;
    long long tMs;
};

class MPositionHistory {
public:
    MPositionHistory();  // 32-slot ring

    void Record(float x, float y, float z, long long tMs);

    // most recent sample (false if none)
    bool GetLatest(float outXYZ[3], long long* outTMs) const;

    // interpolated coordinates at tMs (linear interpolation, false if out of range)
    bool QueryAt(long long tMs, float outXYZ[3]) const;

private:
    PositionSample m_Buf[32];
    uint32_t m_Head;
    uint32_t m_Count;
};

}
```

**STL-free header** — exposes only the `Sample` POD. Avoids ABI compatibility hazards (for environments where CSCommon is built under both v143/v145).

## Storage
- 32-slot ring buffer
- About 1 second of history at a 32 Hz send rate
- per-player memory: 32 × 16 B = 512 B

## Integration Points
- `CSCommon/Security/MPositionHistory.{h,cpp}` new
- `Security::MPositionHistory m_PositionHistory` member on `MMatchObject` + `GetPositionHistory()` accessor
- `Record()` in the POSITION_TICK handler
- `GetLatest()` in the HIT_TICK handler (victim-authoritative coordinates)
- (future) `QueryAt(shotClientTMs - RTT/2)` in the lag-comp hit validator

## Coordinate Transmission
- `MC_MATCH_POSITION_TICK = 2903` (MACHINE2MACHINE)
- 32 Hz, piggybacked next to the existing `MC_PEER_BASICINFO`
- Reuses the ZPACKEDBASICINFO blob (first 4B fTime + next 6B short XYZ)
- Not included in the replay-mode whitelist → automatically not sent during replay (safe)

## Configuration
- `RING_SIZE = 32` (compile-time constant)
- Time source: server-side `std::chrono::steady_clock` ms

## Failure Modes
| Condition | Result |
|------|------|
| History 0 right after connecting | `GetLatest` false → Validator skips (false-positive avoidance) |
| dt regression (monotonicity violation) | dt > 0 guard on the Validator side |
| Wrap after the ring fills | Overwrites the sample before head (intended behavior) |

## Test Vectors

```cpp
MPositionHistory h;
h.Record(0, 0, 0, 1000);
h.Record(100, 0, 0, 1100);  // 100 units / 100 ms = 1000 u/s

float xyz[3]; long long t;
h.GetLatest(xyz, &t);  // → (100, 0, 0, 1100)

float interp[3];
h.QueryAt(1050, interp);  // → (50, 0, 0) (linear interpolation)
h.QueryAt(900, interp);   // → false (out of range)
```

## Limitations
- 32 slots is 1 second of history at 32 Hz. If a longer lag-comp window is needed, the slot count must be increased
- Stores position only — anim bone / weapon orientation / view angle not included (needed for precise hit validation)
- Interpolation is linear — error possible on movements with a large acceleration curve (dash start points, etc.)

## Cross-References
- Consumer (Movement): [`06-movement-validator.md`](./06-movement-validator.md)
- Consumer (Combat): [`07-combat-validator.md`](./07-combat-validator.md)
- Consumer (PositionLie novel): [`10-position-lie-novel.md`](./10-position-lie-novel.md)
- Integration: [`../08-integration-guide.md`](../08-integration-guide.md) §2.6
