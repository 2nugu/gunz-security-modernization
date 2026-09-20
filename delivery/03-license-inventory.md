**English** | [한국어](./ko/03-license-inventory.md)

# License Inventory

> **Readers**: legal / compliance / security governance. Go/no-go judgment on dependency adoption.
> **Principle**: evaluate the license impact on the client (commercial game binary) and the server (backend infrastructure) separately.

---

## 1. Dependency List

### 1.1 Client (Gunz.exe) Side

| Library | License | SPDX | Static/dynamic | Impact |
|----------|---------|------|---------|------|
| libsodium 1.0.20 | ISC | `ISC` | Static possible | **Commercial distribution allowed, no license propagation** |
| WinHTTP | Windows SDK EULA | (proprietary) | Dynamic (system) | Windows platform dependency, no additional impact |
| Microsoft Detours | MIT | `MIT` | Static (if used) | Not used by this package |
| Existing GunZ dependencies | (unchanged) | — | — | Not added by this work |

**New client-side license obligations**:
- Include the libsodium ISC license text in the game EULA / Credits / About
- No other obligations (zero copyleft obligations such as GPL, LGPL)

### 1.2 Server (MatchServer.exe) Side

| Library | License | SPDX | Static/dynamic | Impact |
|----------|---------|------|---------|------|
| libsodium 1.0.20 | ISC | `ISC` | Static | Same |
| WinHTTP | Windows SDK EULA | (proprietary) | Dynamic | Same |

The server is in-house infrastructure, so there are no external distribution license obligations.

### 1.3 Backend (community-api) Side

| Library | License | SPDX | Impact |
|----------|---------|------|------|
| Python | PSF License | `PSF-2.0` | None |
| FastAPI | MIT | `MIT` | None |
| Starlette | BSD-3-Clause | `BSD-3-Clause` | None |
| SQLAlchemy | MIT | `MIT` | None |
| Alembic | MIT | `MIT` | None |
| pydantic | MIT | `MIT` | None |
| psycopg2 | LGPL with exception | `LGPL-3.0-or-later WITH PSPCOPG-EXCEPTION` | LGPL, but dynamic linking + exception clause → no impact on in-house operation |
| pytest | MIT | `MIT` | Test only |
| uvicorn | BSD-3-Clause | `BSD-3-Clause` | None |

### 1.4 Infrastructure

| Component | License | Impact |
|---------|---------|------|
| PostgreSQL 16 | PostgreSQL License | None (BSD-like) |
| Redis 6.x | BSD-3-Clause | None |
| Redis 7.x+ | RSAL/SSPL (dual) | **Operating policy review needed** — affects SaaS resale. No impact on in-house operation |
| Valkey (Redis 7 fork) | BSD-3-Clause | Alternative to Redis 7.x, no impact |
| Docker Engine | Apache 2.0 | None |
| Python (3.11+) | PSF License | None |

**Redis recommendation**:
- For in-house operation only, Redis 7.x can be used as-is
- If there is any possibility of resale in SaaS form, **Redis 6.x or Valkey** is recommended
- This package uses the Redis dependency only as a cache layer (not required, can be disabled)

---

## 2. License Compatibility Matrix

```
Adopted library  →  This work's code  →  GunZ client (commercial)
   (ISC)              (company policy)      (company IP)
   compatible ✓         compatible ✓          no impact ✓

Adopted library  →  This work's code  →  GunZ server (in-house)
   (ISC/MIT)            (company policy)      (company IP)
   compatible ✓         compatible ✓          no impact ✓
```

The **only obligation** arising from adopting this package: include the libsodium ISC license text.

---

## 3. Full Text of the libsodium ISC License

```
ISC License

Copyright (c) 2013-2024
Frank Denis <j at pureftpd dot org>

Permission to use, copy, modify, and/or distribute this software for any
purpose with or without fee is hereby granted, provided that the above
copyright notice and this permission notice appear in all copies.

THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
```

---

## 4. License of This Package's Code

The new code included in this package (`Security::*` namespace, FastAPI routers, etc.) is **free for the company to use**. It can be adopted without NDA, license negotiation, assignment agreements, or similar procedures.

The rights retained by the author are the 2 items stated at the top of the README (continued follow-up analysis based on public materials + non-commercial OSS release), and they remain in effect regardless of whether the company adopts the package.

The only additional external dependency license obligation is inclusion of the libsodium ISC text in §3.

---

## 5. Source / Base Tree Licenses

This work referenced the following public trees:

| Tree | Public location | License notice |
|------|----------|--------------|
| GunZ-The-Duel | GitHub | (see the repository LICENSE file) |
| RefinedGunz | GitHub | (see the repository LICENSE file) |
| Gunz1.5-main | GitHub | (see the repository LICENSE file) |

The new code included in this package (`Security::*`) does not exist in the trees above. The files at the patch sites referenced by the integration guide ([`08-integration-guide.md`](./08-integration-guide.md)) (`MServer.cpp`, `MMatchServer.cpp`, etc.) are integrated at the equivalent locations in the company's own copy — **this package does not redistribute company-side source**.

---

## 6. Compliance Checklist

Items for the company to carry out when adopting this package:

- [ ] Include the libsodium ISC license text in the game EULA / Credits / startup-screen license notice
- [ ] If using Redis 7.x, review the possibility of SaaS resale → downgrade to Redis 6.x or Valkey if needed
- [ ] Register the new dependencies in the company's OSS review system
- [ ] (Optional) Static analysis / security audit — review by the in-house security team

---

## 7. Frequently Asked Questions from the Company (Pre-emptive Answers)

**Q. Can OpenSSL be used instead of libsodium?**
A. Yes. AES-256-GCM can be implemented equivalently with OpenSSL `EVP_aead_aes_256_gcm`, X25519 with `EVP_PKEY_X25519`. However, OpenSSL is Apache 2.0 (3.0+) or dual SSLeay+OpenSSL (1.1.1), a larger dependency footprint. libsodium being ISC-only keeps compliance simple.

**Q. What about hosts without AES-NI?**
A. `MPacketCrypterV2::InitKey` returns false immediately → explicit failure at the handshake stage. Prevents the earlier silent-drop regression. When adopting for the live service, confirming AES-NI on every match server host is mandatory.

**Q. Alternatives to WinHTTP?**
A. cURL (MIT/X derivative, no impact) or the company's in-house HTTP client can be used. WinHTTP was chosen because it has zero external dependencies (part of the Windows SDK).

**Q. If this package's code is merged into the company's own tree, can the author claim rights later?**
A. This package is released for free use and the company can adopt it without a separate agreement. The author retains only two rights: (a) continued follow-up analysis based on public materials, (b) GitHub release under a non-commercial OSS license, and these remain in effect regardless of the company's adoption. Whether or not the company adopts this package, there is no intent to assert any other rights.
