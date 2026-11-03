# 04. macOS에서만 TLS 검증 테스트가 실패 — `openssl`이라는 이름의 함정

- **프로젝트**: kitae1645/nara-bid-predictor
- **원본**: PR #118 (머지 2026-08-07, 이슈 #105)
- **스택**: Python 3.12, Docker Compose, `infra/postgres/validate-tls.sh`, OpenSSL/LibreSSL, pytest
- **분류**: 크로스플랫폼 호환성
- **CS 접점**: 실행 파일의 이름이 같다고 인터페이스가 같은 것은 아니다. CLI 인자도 API 표면의 일부다.

## 한 줄 요약

PostgreSQL TLS 검증 스크립트가 OpenSSL 전용 플래그(`-checkhost`, `x509 -ext`)를 써서 macOS 기본 `openssl`(= LibreSSL 3.3.6)에서 실패했다. 두 구현이 모두 지원하는 `x509 -in ... -noout -text` + SAN grep으로 바꿔 양쪽에서 동일하게 동작시켰다.

## 증상

CI(Linux, OpenSSL)는 그린, 개발자 macOS에서만:

```
test_tls_validator_accepts_valid_server_material  FAILED
```

- 실패 원인이 인증서 내용이 아니라 **플래그를 모른다**는 형태였습니다. 즉 검증 로직이 돌지도 못했습니다.
- macOS의 `/usr/bin/openssl`은 OpenSSL이 아니라 LibreSSL입니다(`openssl version` → `LibreSSL 3.3.6`).

## 조사

1. 실패 메시지에서 `unknown option` 확인 → 스크립트 로직 이전 단계.
2. `openssl x509 -help`를 두 환경에서 비교. `-checkhost`는 OpenSSL에만 존재. 추가로 `x509 -ext subjectAltName`도 LibreSSL에 없음(이건 처음엔 예상하지 못했고 고치는 중에 발견).
3. TLS 검증이 실제로 강제해야 하는 불변식이 무엇인지 재확인: ① CA로 체인 검증 ② 유효기간 ③ **SAN에 `DNS:db` 포함** ④ 개인키-인증서 쌍 일치. ③만 구현 방법을 바꾸면 됩니다.

## 원인

스크립트가 "openssl이 있으면 같은 openssl이다"라고 가정했습니다. macOS 기본 번들은 LibreSSL이고, OpenSSL 1.1.1에서 추가된 편의 플래그가 없습니다. 이름만 보고 호환성을 가정한 것이 원인입니다.

## 해결

호스트네임 바인딩 검사를 최소공통분모로 교체:

```diff
-openssl x509 -checkhost db -noout -in "$cert_source" >/dev/null
-openssl x509 -noout -ext subjectAltName -in "$cert_source" \
+# Hostname binding: the `-checkhost` flag and `x509 -ext` are OpenSSL-only;
+# LibreSSL (macOS) lacks both. Read the SAN via `-text`, which both support.
+openssl x509 -in "$cert_source" -noout -text \
   | grep -Eq 'DNS:[[:space:]]*db([,[:space:]]|$)'
```

- `-text`는 두 구현 모두 지원하고, SAN 섹션을 포함한 텍스트 덤프를 냅니다.
- grep 정규식에 `[,[:space:]]|$` 경계를 둬서 `DNS:db.example.com` 같은 오탐을 막습니다. `db`만 매칭하면 안 됩니다.
- **동작은 바뀌지 않습니다.** Linux/OpenSSL에서도 이전에도 같은 grep이 있었고, 이제 그것이 유일한 호스트네임 검사입니다. 만료·미래 유효기간·잘못된 체인·잘못된 purpose·잘못된 SAN·키 불일치 케이스 매트릭스가 양쪽에서 동일하게 통과합니다.

계약 테스트도 함께 수정:

```diff
-assert "openssl x509 -checkhost db" in validator_script
-assert "subjectAltName" in validator_script
+assert "openssl x509 -checkhost" not in validator_script   # OpenSSL-only
+assert "openssl x509 -in" in validator_script
+assert "DNS:[[:space:]]*db" in validator_script
```

그리고 `_fake_openssl_environment` shim에 `-noout -text` 케이스를 추가해, 권한/소유권 테스트가 shim 아래에서도 같은 경로를 타도록 했습니다.

## 검증

| 환경 | 명령 | 결과 |
|------|------|------|
| macOS / LibreSSL 3.3.6 | `pytest test_compose_contract.py` | 44 passed (이전 1 failed) |
| macOS | `test_ci_contract.py` + `test_compose_contract.py` | 117 passed |
| 공통 | `ruff format --check`, `git diff --check` | 통과 |
| CI / OpenSSL | 동일 스크립트 | 매트릭스 전부 통과 (동작 불변) |

## 재발 방지

- 계약 테스트가 **금지 패턴**(`-checkhost` 부재)까지 검사. 다시 누가 편한 플래그를 쓰면 CI에서 잡힙니다.
- 테스트용 shim이 실제 사용 플래그를 반영하도록 유지. shim이 낡으면 테스트가 거짓 통과합니다.

## 일반화

- **`openssl`, `sed`, `awk`, `tar`, `python3`… 이름이 같은데 구현이 다른 도구가 넘친다.** 특히 macOS 기본 번들은 BSD/LibreSSL 계열이고, Docker 안은 GNU/OpenSSL 계열인 경우가 많다.
- **호환성 문제를 만나면 "불변식"으로 돌아가라.** 플래그가 아니라 "SAN에 DNS:db가 있어야 한다"가 요구사항이다. 그러면 구현을 자유롭게 바꿀 수 있다.
- **테스트가 통과하는 플랫폼이 하나뿐이면 그 테스트는 플랫폼을 검사하고 있는 것이다.** 개발자 머신에서 한 번은 돌려봐야 한다.
