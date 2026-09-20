# GunZ Security & Anti-Cheat Modernization

오리지널 GunZ (2007, MAIET) 의 서버 보안·안정성·안티치트 레이어를 현대 표준으로 교체한 연구 작업물의 전달 패키지입니다.
공개 GitHub 트리 (GunZ-The-Duel, FGunZ, RefinedGunz 등) 위에서 신규 작성한 보안 레이어이며, 공개 버전 기준 빌드 동작까지 확인했습니다.

- AES-256-GCM AEAD 패킷 암호화, X25519 ECDHE 키 교환
- DDoS 3계층 게이트 / 레이트 리밋
- 서버 권위 좌표 링버퍼 기반 이동·전투·데미지 검증
- GunZ 특화 안티치트 시그널 카탈로그
- 의사결정자 / 엔지니어 / 운영자 별 문서 트리 + 검증 키트

**문서 진입점: [`delivery/README.md`](./delivery/README.md)**

## 구성

| 경로 | 내용 |
|------|------|
| `delivery/` | 전달 문서 패키지 (브리프, 모듈 명세 13종, 통합 가이드, 운영 런북, 검증 키트, 라이선스 인벤토리) |
| `BGM/` | 신규 BGM 트랙 (mp3) |
| `Wallpapers/` | 배경화면 이미지 |
| `rankmark/` | 랭크 마크 이미지 |
| `newbie_mark/` | 뉴비 마크 이미지 |
| `Crosshair/` | 크로스헤어 |

## 라이선스

[MIT](./LICENSE). NDA, 라이선스 협상, 보상 절차 없이 자유롭게 채택·수정·재배포할 수 있습니다.
의존성 라이선스 검토는 [`delivery/03-license-inventory.md`](./delivery/03-license-inventory.md) 를 참고하세요.

― 이홍구 <2nugu@naver.com>
