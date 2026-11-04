# 10. Cloudflare 뒤에 둔 오리진에서 겪은 캐시·스트리밍·메서드 문제

- **프로젝트**: kitae1645/mafia42_test
- **원본**: 커밋 `eaa915f` (2026-06-19), `cdd7916` (2026-06-20), `407e669` (2026-06-25) + 후속 PR #143 (2026-07-20)
- **스택**: Python 표준 라이브러리 `http.server` 기반 커스텀 웹서버, SSE, PWA 서비스 워커, Cloudflare 프록시
- **분류**: 캐싱 / 리버스 프록시
- **CS 접점**: 캐시는 한 곳에 있지 않다. 브라우저, 서비스 워커, 엣지 프록시가 각자 다른 기본값으로 판단하고, 그 사이의 계약은 응답 헤더다.

## 한 줄 요약

배포가 반영되지 않는 문제는 **캐시 지시 헤더 부재**였고(no-store + `?v=` 캐시버스터), 실시간 피드가 멈춘 것은 **프록시 버퍼링**이었고(`X-Accel-Buffering: no`), 사이트 전체 501은 **프록시가 보내는 HEAD 요청**을 서버가 처리하지 못한 것이었습니다(`do_HEAD`).

## 1. 배포했는데 옛 JS/HTML이 계속 나옴

### 증상

- 서버에는 새 코드가 올라갔고 API 응답도 새 값인데, 화면은 이전 버전 그대로.
- 새로고침(F5)으로는 그대로, **강력 새로고침·시크릿 창에서는 정상**. 이 차이가 브라우저 캐시를 가리킵니다.
- 사용자마다 보이는 버전이 달라 문의가 산발적으로 들어옴.

### 조사

1. 응답 헤더를 확인 → `Content-Type`, `Content-Length`만 있고 `Cache-Control`도 `ETag`도 `Last-Modified`도 없음.
2. 정적 파일 서빙 함수가 `_serve_bytes`를 그대로 호출하고 있었음. 파일을 매 요청마다 디스크에서 새로 읽는 구조였지만, **헤더가 없으니 브라우저가 알아서 캐시**했습니다.
3. "서버가 옛 파일을 준다"는 가설은 폐기 — 디스크에서 매번 읽고 있었고, 시크릿 창에서 새 버전이 보이는 것이 반증.

### 원인

캐시 정책을 **명시하지 않은 것**이 정책이 되어버렸습니다. 휴리스틱 캐싱은 헤더가 없을 때 브라우저가 자체적으로 만료를 추정하는 동작이라, 배포 시점과 무관하게 옛 자산이 살아남습니다.

### 해결

```diff
-        self._serve_bytes(body, content_type)
+        self.send_response(200)
+        self.send_header("Content-Type", content_type)
+        self.send_header("Content-Length", str(len(body)))
+        self.send_header("Cache-Control", "no-cache, no-store, must-revalidate")
+        self.send_header("Pragma", "no-cache")
+        self.send_header("Expires", "0")
+        self.end_headers()
+        self.wfile.write(body)
```

여기에 스크립트 URL 캐시버스터를 함께 붙였습니다.

```diff
-<script src="megaphone-monitor.js"></script>
+<script src="megaphone-monitor.js?v=eaa915f"></script>
```

- 헤더는 "지금부터는 캐시하지 마라"이고, 캐시버스터는 "이미 캐시된 것을 무효화"합니다. 둘은 대체재가 아니라 세트입니다. 헤더만 넣으면 이미 캐시를 들고 있는 사용자는 그대로입니다.
- `?v=` 값은 커밋 SHA로 두면 배포마다 자동으로 바뀝니다(수동 증가는 반드시 언젠가 잊습니다).

## 2. PWA를 붙이자 캐시 계층이 하나 더 생김

서비스 워커 도입(PR #27) 후 캐시 주체가 셋이 됐습니다: 브라우저 HTTP 캐시 / 서비스 워커 캐시 / Cloudflare 엣지.

현재 `sw.js`의 정책:

| 대상 | 전략 | 이유 |
|------|------|------|
| `/api/*`, `/events` | network-only | 실시간 데이터·SSE를 캐시하면 오답이 된다 |
| `/signin`, `/signup` 문서 | network-only | 오프라인 폼은 의미 없고, 클라이언트마다 다른 화면이 된다 |
| 그 외 문서 | network-first + 오프라인 폴백 | 오프라인 셸은 유지하되 최신 우선 |
| 정적 자산 | cache-first + 백그라운드 갱신 | 속도 |

여기서 한 번 더 밟은 지점: **인증 스크립트가 캐시에 남아 고친 버그를 되살렸습니다.** `/signup.js`가 cache-first 대상이라, 서버에서 고친 검증 흐름이 기존 방문자에게는 옛 스크립트로 계속 돌았습니다. PR #143에서 네트워크 전용으로 분리했습니다.

```diff
 const NETWORK_ONLY_DOCUMENTS = new Set(['/signin', '/signup']);
+const NETWORK_ONLY_AUTH_SCRIPTS = new Set(['/signin.js', '/signup.js']);
```

```js
// Auth scripts must track their network-only HTML. A stale cached script can
// otherwise revive a fixed credential or verification flow on first visit.
if (url.origin === self.location.origin &&
    NETWORK_ONLY_AUTH_SCRIPTS.has(url.pathname)) {
  event.respondWith(fetch(event.request));
  return;
}
```

`CACHE_NAME`(`megaphone-v2`)을 올리고 `activate`에서 이전 캐시를 지우는 것도 같은 이유입니다. 캐시 이름이 그대로면 정책을 바꿔도 옛 엔트리가 살아남습니다.

## 3. 실시간 피드가 Cloudflare를 통과하면 흐르지 않음

### 증상

- `text/event-stream` 응답이 오리진 직접 접속에서는 즉시 흐르는데, 도메인(Cloudflare 경유)으로는 이벤트가 뭉쳐서 늦게 도착하거나 스트림이 끊김.

### 원인

Cloudflare 앞단의 nginx가 응답을 **버퍼링**합니다. `Cache-Control: no-cache`는 "저장하지 마라"는 뜻이고, 버퍼링은 캐시가 아니라 **전송 방식**의 문제라 막지 못합니다.

### 해결

```diff
 self.send_header("Content-Type", "text/event-stream")
 self.send_header("Cache-Control", "no-cache")
 self.send_header("Connection", "keep-alive")
+self.send_header("X-Accel-Buffering", "no")  # Tell Cloudflare nginx not to buffer SSE
```

`X-Accel-Buffering`은 원래 nginx 계열이 해석하는 헤더입니다. Cloudflare도 같은 규약을 따르기 때문에 통했습니다. 이후 SSE 연결 수 상한과 핸들러 입력 가드도 별도로 추가했습니다(과부하 방지).

## 4. 사이트 전체가 501 — 범인은 프록시의 HEAD 요청

### 증상

- 배포 직후 사이트가 통째로 501. 오리진에 직접 붙으면 정상, 도메인으로만 실패.

### 원인

Python의 `BaseHTTPRequestHandler`는 `do_<METHOD>`가 없는 메서드에 **501 Not Implemented**를 돌려줍니다. 우리 서버에는 `do_GET`/`do_POST`만 있었고, Cloudflare가 오리진 헬스체크로 보내는 `HEAD` 요청이 501을 받았습니다. 그 501이 사용자 응답으로까지 전파됐습니다.

### 해결

```diff
+def do_HEAD(self) -> None:  # noqa: N802
+    self.do_GET()
+
 def do_GET(self) -> None:  # noqa: N802
```

라우팅을 복제하지 않고 위임하되, 본문 쓰기만 건너뜁니다.

```diff
 self.end_headers()
-self.wfile.write(body)
+if self.command != "HEAD":
+    self.wfile.write(body)
```

`_serve_bytes`, `_serve_file`, `_serve_logs_export`, `_serve_export` 전부 같은 가드를 넣었습니다. 한 곳이라도 빠지면 그 경로만 HEAD에서 응답 본문을 쓰게 되고, 커넥션이 어긋납니다.

## 검증 — 지금 실제로 어떤 헤더가 나가는가

운영 도메인에 직접 요청한 결과(최근 실측):

```
$ curl -I https://<도메인>/
HTTP/2 200
cache-control: no-cache, no-store, must-revalidate
expires: 0
pragma: no-cache
server: cloudflare
cf-cache-status: DYNAMIC        ← 엣지에 캐시되지 않음(원본 그대로 전달)

$ curl -I https://<도메인>/megaphone-monitor.js
cache-control: no-cache, no-store, must-revalidate
cf-cache-status: DYNAMIC
```

스크립트 태그도 `megaphone-monitor.js?v=<배포 식별자>` 형태를 유지하고 있습니다.

## 재발 방지

- 정적 응답에 캐시 헤더를 **항상 명시**하고, 캐시버스터를 배포 식별자로 자동 갱신
- 캐시 정책이 바뀌면 서비스 워커 `CACHE_NAME`을 올리고 이전 캐시 삭제
- 인증·검증처럼 "고친 것이 즉시 적용돼야 하는" 스크립트는 네트워크 전용으로 분리
- 오리진이 받는 메서드를 GET/POST로 가정하지 않기 (헬스체크·모니터링은 HEAD를 쓴다)
- 스트리밍 경로는 프록시를 통과시켜 실제로 확인

## 일반화

- **캐시는 계층마다 따로 존재하고, 각 계층은 헤더가 없으면 자기 기준으로 판단한다.** "코드는 배포했다"와 "사용자가 새 코드를 받았다" 사이에는 최소 세 개의 캐시가 있다.
- **캐시는 버그를 되살린다.** 수정한 코드가 캐시된 사본에 가려지면, 사용자 입장에서는 고치지 않은 것과 같다. 그래서 인증·검증·결제 같은 흐름은 캐시 대상에서 빼는 쪽이 안전하다.
- **캐시와 버퍼링은 다른 문제다.** 스트리밍이 멈추면 캐시 헤더가 아니라 전송 계층(프록시 버퍼)을 의심한다.
- **프록시는 우리가 쓰지 않는 메서드도 보낸다.** `HEAD` 하나로 사이트 전체가 501이 될 수 있다.
