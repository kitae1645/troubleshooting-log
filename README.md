# Troubleshooting Log

kitae1645 조직에서 실제로 돌리는 프로젝트를 운영하면서 만난 문제와 해결 과정을 정리한 기록입니다.

"이거 왜 이렇게 됐지"로 시작해서 "다음에 같은 상황이면 이렇게 본다"로 끝나는 문서만 넣습니다. 재현 명령, 로그 근거, PR 링크가 없는 글은 넣지 않습니다.

- **범위**: 인코딩, OS 권한·정책, 동시성, 크로스플랫폼 CLI 호환성, CI 버전 드리프트, 외부 데이터 신뢰 경계, 공유 상태 집계, 브라우저·제3자 스크립트 제약
- **형식**: 케이스당 1문서(`docs/`), 템플릿은 [`templates/case-study.md`](templates/case-study.md)
- **근거**: 각 문서 상단의 PR 링크가 원본입니다. 본문은 그 PR과 머지 커밋에서 확인한 내용만 씁니다.

## 케이스 인덱스

| # | 증상 | CS 접점 | 프로젝트 | 원본 |
|---|------|---------|---------|------|
| 01 | [Actions 로그에서 Windows 명령 출력이 `?????`로 깨짐](docs/01-windows-cp949-mojibake.md) | 문자 인코딩 — 바이트와 텍스트의 경계 | mafia42_test | [#121](https://github.com/kitae1645/mafia42_test/pull/121) |
| 02 | [SYSTEM 계정 배포가 스크립트 실행 전에 거부됨](docs/02-windows-system-executionpolicy.md) | 권한 주체와 실행 정책의 범위 | maple_megaphone | [#2](https://github.com/kitae1645/maple_megaphone/pull/2) |
| 03 | [동시 실행한 런처가 업데이터 임시 파일에서 충돌](docs/03-windows-updater-race.md) | 경쟁 상태, check-then-act, 상호배제 | mafia42_test | [#153](https://github.com/kitae1645/mafia42_test/pull/153) |
| 04 | [macOS에서만 TLS 검증 테스트가 실패](docs/04-libressl-openssl-portability.md) | "같은 이름, 다른 구현" — CLI 호환성 | nara-bid-predictor | [#118](https://github.com/kitae1645/nara-bid-predictor/pull/118) |
| 05 | [CI가 구 Alembic 리비전을 검사해 마이그레이션마다 깨짐](docs/05-ci-postgres-alembic-drift.md) | 스키마 이력의 선형성, 계약 테스트 | nara-bid-predictor | [#158](https://github.com/kitae1645/nara-bid-predictor/pull/158) |
| 06 | [공공 API의 값 하나가 수집 배치 전체를 죽임](docs/06-dirty-api-payload-batch.md) | 장애 격리 — 배치 원자성 vs 부분 성공 | nara-bid-predictor | [#141](https://github.com/kitae1645/nara-bid-predictor/pull/141) |
| 07 | [실제로 연결된 채널이 "연결 끊김"으로 표시됨](docs/07-websocket-channel-refcount.md) | 공유 상태, 참조 카운팅, 낡은 식별자 | mafia42_test | [#130](https://github.com/kitae1645/mafia42_test/pull/130) |
| 08 | [iPhone Safari에서만 가입 캡차가 멈춤](docs/08-turnstile-ios-safari.md) | 제3자 스크립트 계약 불신, 서버측 검증 | mafia42_test | [#148](https://github.com/kitae1645/mafia42_test/pull/148) |
| 09 | [브라우저에서만 버튼이 조용히 실패 (`ERR_BLOCKED_BY_CLIENT`)](docs/09-brave-shields-path.md) | 경로 문자열이 필터 시그니처가 되는 경우 | tagVault | [#1](https://github.com/kitae1645/tagVault/pull/1) |

## 여러 케이스에서 반복된 것

1. **요청이 "없다"는 것도 증거다.** 서버 로그에 아무것도 안 남는 장애(09)와, 원격 명령의 출력만 깨지는 장애(01)는 둘 다 애플리케이션 로직 밖에서 발생했습니다. 로그가 비어 있으면 애플리케이션 레이어를 더 보지 말고 그 아래를 봅니다.
2. **외부에서 준 식별자는 언젠가 바뀐다.** 채널 서버 IP(07), 제3자 스크립트 콜백(08), 공공 API 응답 형식(06). 바뀌었을 때 조용히 오답이 되지 않도록 실패를 드러내는 쪽으로 설계합니다.
3. **버전은 테스트가 아니라 계약으로 고정한다.** 특정 리비전을 테스트에 박으면(05) 다음 변경에서 무조건 깨집니다. "지금 그래프의 head는 하나여야 한다" 같은 불변식을 검사합니다.
4. **권한·정책은 실행 주체별로 다르다.** 같은 스크립트가 관리자 세션에서는 되고 SYSTEM에서는 거부될 수 있습니다(02). 배포가 실제로 도는 주체에서 검증해야 합니다.
5. **보안 우회 경로를 만들면 서버가 다시 검증한다.** 클라이언트가 "이 브라우저는 캡차 못 함"이라고 주장하는 구조(08)는 서버측 UA 재검증 + 별도 한도 없이는 그냥 구멍입니다.

## 작성 규칙

- 제목은 증상 기준(`무엇이 어떻게 잘못 보였나`), 원인은 본문에서 밝힙니다.
- 사실은 전부 PR·로그·측정값에 연결합니다. 추정이면 "추정"이라고 씁니다.
- 검증 섹션에는 실제로 돌린 명령과 결과 수치를 남깁니다.
- 마지막 섹션은 그 케이스를 떠나 일반화한 문장 1~3개로 끝냅니다.

## 공개 범위

이 저장소는 비공개입니다. 원본 프로젝트가 비공개이고, 배포 경로·호스트 구성·보안 완화 로직이 포함되어 있습니다. 공개용으로 낼 때는 해당 부분을 지우고 "환경: Windows Server, 한국어 로캘" 수준으로 일반화해야 합니다.
