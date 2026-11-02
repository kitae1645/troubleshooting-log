# 06. 공공 API의 값 하나가 수집 배치 전체를 죽임

- **프로젝트**: kitae1645/nara-bid-predictor
- **원본**: PR [#141](https://github.com/kitae1645/nara-bid-predictor/pull/141) (머지 2026-08-28)
- **스택**: Python 3.12, Pydantic v2, 나라장터(G2B) OpenAPI 수집 크론
- **분류**: 외부 데이터 신뢰 경계
- **CS 접점**: 배치 처리를 하나의 원자적 단위로 다루면 데이터 품질 문제가 가용성 문제로 승격된다. 부분 성공과 실패 격리가 필요하다.

## 한 줄 요약

G2B API가 간헐적으로 콤마 포함 금액(`106,000,000`)이나 소수 4자리 초과 낙찰률을 보냈고, 항목 하나의 `NormalizationError`가 배치 전체를 중단시켜 수집 크론이 매일 실패했다. 항목 단위로 skip + 경고 로그를 남기고, 파서를 실제 응답 형식에 맞게 넓혔다.

## 증상

- 7/27~8/2 services 결과 수집 크론이 **매일** 실패.
- 실패 로그에 특정 공고 번호 하나가 아니라 배치 전체를 감싸는 정규화 예외.
- 같은 날 재실행해도 결과가 달라짐 → 데이터 자체가 가끔 더럽다는 신호.

## 조사

1. 실패한 응답 원문을 보관해 확인. 두 종류의 편차:
   - `sucsfbidAmt`: 천 단위 콤마가 포함된 문자열 (`106,000,000`)
   - `sucsfbidRate`: 소수점 4자리 초과
2. 정규화 코드가 리스트 컴프리헨션 안에서 예외를 던지고 있었고, 그 예외가 상위로 전파되면 남은 항목 처리 없이 함수가 종료됩니다. 즉 **격리 지점이 없었음**.
3. "API 스펙을 위반했으니 우리가 맞춰줄 필요 없다"는 선택지도 있었지만, 이 API는 우리 통제 밖이고 크론이 매일 돌아야 합니다. 스펙 준수 여부를 우리가 강제할 수단이 없으므로 격리가 맞다고 판단.

## 원인

두 가지가 겹쳤습니다.

1. **파서가 스펙의 이상적 형태만 가정**: 콤마 없는 정수, 소수 4자리 이하.
2. **실패 전파 범위가 너무 큼**: 항목 단위 실패가 배치 단위 실패가 됨.

## 해결

```diff
-    normalized = [
-        _normalize_awarded_item(item, ...)
-        for item in envelope.response.body.items.item
-    ]
+    normalized: list[NormalizedAward] = []
+    for item in envelope.response.body.items.item:
+        try:
+            normalized.append(_normalize_awarded_item(item, ...))
+        except NormalizationError as exc:
+            # A single dirty item (e.g. comma-separated sucsfbidAmt, or a rate
+            # with excessive decimals) must not kill the whole award batch.
+            # Skip it, record the reason, and keep the rest.
+            logger.warning(
+                "Skipping award item due to normalization failure: %s (notice=%s)",
+                exc,
+                item.bid_notice_number,
+            )
```

파서도 실제 응답에 맞게 확장:

- `_parse_positive_krw`: 천 단위 콤마 허용
- `_parse_awarded_rate`: 소수점 4자리까지 허용
- 기존 "reject" 기대 테스트를 "skip" 기대로 갱신, 더러운 값 회귀 테스트 추가

주의: 스키마 검증(`payload failed schema validation`)과 result code 검사는 **그대로 전체를 중단**시킵니다. 항목 하나의 값 오염과, 응답 자체가 다른 API 응답인 상황은 다르게 다뤄야 하기 때문입니다.

## 검증

| 확인 | 명령/측정 | 결과 |
|------|-----------|------|
| G2B 테스트 | `pytest apps/api/tests/g2b/` | 470 passed (환경 의존 1 제외) |
| 정적 검사 | `ruff check`, `mypy` | 통과 |
| 운영 실측 | 배포 후 429 응답이 섞인 구간 수집 | 배치 중단 없이 전진, skip 로그 기록 |

## 재발 방지

- skip 사유를 `bid_notice_number`와 함께 경고 로그로 남김. 다음에 같은 값이 늘어나면 빈도로 감지 가능.
- 회귀 테스트에 더러운 값 케이스를 고정.

## 일반화

- **외부 입력 경계에서는 "부분 성공"을 1급 상태로 만들어라.** 배치 전체를 하나의 트랜잭션처럼 다루면, 가장 약한 입력 하나가 전체 가용성을 결정한다.
- **격리 수준은 신뢰 수준에 맞춘다.** 항목 값 오염은 skip, 응답 정체 불일치는 전체 중단. 둘을 같은 예외로 처리하면 관측도 안 되고 복구도 안 된다.
- **외부 API 스펙 위반을 우리 코드의 정당성으로 방어하지 마라.** 상대가 바뀌면 우리 크론이 죽는다. 파서를 넓히고 skip을 계측하는 쪽이 운영 가능한 선택이다.
