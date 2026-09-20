**English** | [한국어](./README.ko.md)

# GunZ Security & Anti-Cheat Modernization

A delivery package for research work that replaces the server security, stability and anti-cheat layer of the original GunZ (2007, MAIET) with modern standards.
The security layer was written fresh on top of public GitHub source trees (GunZ-The-Duel, FGunZ, RefinedGunz, etc.) and verified to build and run against the public-tree baseline.

- AES-256-GCM AEAD packet encryption, X25519 ECDHE key exchange
- Three-tier DDoS gate / rate limiting
- Server-authoritative position ring buffer driving movement, combat and damage validation
- GunZ-specific anti-cheat signal catalog
- Separate document tracks for decision makers, engineers and operators, plus a validation kit

**Documentation entry point: [`delivery/README.md`](./delivery/README.md)** (Korean original: [`delivery/ko/README.md`](./delivery/ko/README.md))

## Layout

| Path | Contents |
|------|----------|
| `delivery/` | Delivery documents in English (briefs, 13 module specs, integration guide, operations runbook, validation kit, license inventory) |
| `delivery/ko/` | Korean originals, same file structure |
| `BGM/` | New BGM tracks (mp3) |
| `Wallpapers/` | Wallpaper images |
| `rankmark/` | Rank mark images |
| `newbie_mark/` | Newbie mark images |
| `Crosshair/` | Crosshair assets |

## License

[MIT](./LICENSE). Adopt, modify and redistribute freely; no NDA, license negotiation or compensation process is required.
For third-party dependency licensing see [`delivery/03-license-inventory.md`](./delivery/03-license-inventory.md).

― Hong-gu Lee <2nugu@naver.com>
