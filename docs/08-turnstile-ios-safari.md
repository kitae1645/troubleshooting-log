# 08. iPhone Safari에서만 가입 캡차가 멈춤

- **프로젝트**: kitae1645/mafia42_test
- **원본**: PR #148 (머지 2026-07-22)
- **스택**: Python 3.12 웹서버, Cloudflare Turnstile, 바닐라 JS, iPhone Safari(WebKit)
- **분류**: 제3자 스크립트 / 브라우저 제약
- **CS 접점**: 외부 스크립트의 콜백 계약은 우리가 보장할 수 없다. 콜백을 신뢰하지 않고 상태를 관찰하는 설계가 필요하고, 우회 경로를 만들면 서버가 다시 검증해야 한다.

## 한 줄 요약

iPhone Safari에서 Turnstile 스크립트는 로드되는데 `onload` 콜백이 오지 않아 8초 타임아웃 후 도움말 패널만 떴다. 500ms 폴링 백업 + iOS Safari 감지 + 실패 1회 이상일 때 서버측 폴백(UA 재검증 + 별도 레이트리밋)으로 복구했다.

## 증상

- Chrome, Edge, macOS Safari에서는 정상. **iPhone Safari에서만** 실패.
- 네트워크 탭에서 `challenges.cloudflare.com` 스크립트는 200으로 로드됨.
- 그런데 위젯이 그려지지 않고 `window._turnstileReady` 계열 초기화 콜백이 호출되지 않음 → 8초 뒤 타임아웃 패널.

## 조사

1. 플랫폼 경계가 명확했음(iOS Safari만). 공통 원인 후보를 빠르게 배제.
2. Cloudflare 커뮤니티의 동일 증상들 확인: iOS 트래픽에서 invalid token 비율이 비정상적으로 높고, iOS 콘솔에 "Blocked a frame with origin" 류 메시지. ITP(Intelligent Tracking Prevention)와 iframe 차단, Private Relay가 얽힘.
3. 1차 원인은 `?onload=` 콜백 미호출 — API는 초기화되지만 우리 콜백이 안 불림.
4. 2차 원인: Turnstile이 성공해도 Safari ITP가 세션 쿠키를 막아 "가입은 됐는데 로그인 안 됨" 상태가 될 수 있음. 이건 별도 문제라 범위를 나눠 기록.

## 원인

제3자 스크립트의 콜백 기반 초기화 계약이 특정 WebKit 환경에서 신뢰할 수 없음. 우리 코드는 "콜백이 온다"를 전제로 상태 머신을 만들었고, 전제가 깨지자 복구 경로가 타임아웃뿐이었습니다.

## 해결

클라이언트:

- **500ms 폴링 백업**: `window.turnstile`이 준비됐는지 주기적으로 확인. 콜백이 드롭돼도 복구.
- **iOS Safari 판별**: iOS의 Chrome/Firefox/Edge(모두 WebKit 래퍼)를 제외하고 **네이티브 Safari만** 식별.
- **실패 카운터**: 타임아웃 / error-callback / expired / unsupported / 빈 토큰 등 모든 실패원에서 증가.
- **`unsupported-callback` 처리**: 실행 자체가 불가능하다고 보고되는 브라우저 대응.
- 재시도 시 `turnstile_fallback: true, browser: 'safari-ios', turnstile_failures: N` 전송.
- 캐시 버스터(`signup.js?v=...`)로 구버전 스크립트 고정 방지.

서버 (중요):

- 폴백 조건을 **모두** 만족할 때만 Turnstile 검증 생략: `turnstile_fallback=true` + `browser=safari-ios` + `turnstile_failures>=1` + **User-Agent가 실제 iOS Safari**.
- 폴백 전용 레이트리미터 분리: IP당 **시간당 2회**(일반 가입은 5회/60초). 클라이언트 주장만으로 무제한 우회가 되지 않게.
- `_AuthRateLimiter`를 `max_attempts`/`window_seconds` 파라미터화해 재사용.

## 검증

| 테스트 | 검증 내용 |
|--------|-----------|
| `test_signup_polls_when_turnstile_onload_callback_is_missing` | 콜백이 드롭된 상황에서 폴링이 복구하는가 |
| `test_ios_safari_reports_failed_turnstile_for_server_fallback` | iOS Safari UA + 실패 카운터로 폴백 경로가 동작하는가 |
| 기존 가입/인증 테스트 | 폴백 조건이 아닌 요청은 여전히 차단되는가 |

## 후속 변경

이 폴백은 처음엔 iOS Safari로 한정했지만, 이후 **모든 브라우저**로 넓혔습니다. 캡차가 아예 동작하지 않는 환경(확장 프로그램 간섭, 사내망 프록시 등)에서 가입 자체가 막히는 쪽이 더 큰 손실이라고 판단했기 때문입니다. 남용 방지는 폴백 전용 실패 카운터, IP별 시간당 한도, 서버 전체 시간당 한도의 3중 제한으로 옮겼습니다.

## 재발 방지

- 실패원별 카운터를 하나로 모아, "왜 실패했는지"를 클라이언트가 서버에 말할 수 있게 함.
- 폴백은 항상 **좁은 조건 + 별도 한도**. 브라우저 판별 문자열은 서버가 다시 검사.

## 일반화

- **콜백에 의존하는 초기화는 관찰 가능한 상태로 보강하라.** `if (window.X)` 폴링은 조잡해 보이지만, 제3자 스크립트에서는 정당한 방어다.
- **우회로를 만들 때 위협 모델을 같이 적어라.** "iOS Safari는 캡차 못 함"을 그대로 믿으면 스푸핑 가능한 구멍이 된다. UA 재검증 + 별도 레이트리밋이 그 구멍을 좁힌다.
- **1차 원인과 2차 원인을 분리하라.** 콜백 미호출(해결)과 ITP 쿠키 차단(별개)을 한 이슈로 뭉치면 둘 다 미해결로 남는다.
