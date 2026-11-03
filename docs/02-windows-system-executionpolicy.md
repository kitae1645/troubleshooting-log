# 02. SYSTEM 계정 배포가 스크립트 실행 전에 거부됨

- **프로젝트**: kitae1645/maple_megaphone
- **원본**: PR #2 (머지 2026-10-31)
- **스택**: GitHub Actions self-hosted runner (`runs-on: [self-hosted, Windows, X64, ...]`), PowerShell, `tools\deploy\deploy.ps1`
- **분류**: OS 권한·정책
- **CS 접점**: 같은 프로그램이라도 "누가 실행하는가"에 따라 적용되는 정책이 다르다. 실행 정책은 사용자·머신·프로세스 범위를 가진다.

## 한 줄 요약

첫 라이브 배포가 컨트롤러 스크립트 실행 전에 거부됐고, 원인은 SYSTEM 계정의 Restricted PowerShell 실행 정책이었다. 머신 전역 정책은 건드리지 않고 **스텝 단위 셸 정의**에 `-ExecutionPolicy Bypass -File`을 넣어 해결했다.

## 증상

- 배포 워크플로가 `Activate exact tested commit` 스텝에서 실패.
- 실패 지점이 `deploy.ps1` 내부 로직이 아니라 **스크립트를 로드하는 순간**. 즉 우리 코드는 한 줄도 실행되지 않았습니다.
- 같은 스크립트를 관리자 세션에서 손으로 돌리면 정상 동작. 경계가 "대화형 관리자" vs "SYSTEM 서비스"로 갈렸습니다.

## 조사

1. 워크플로가 self-hosted 러너에서 도는데, 러너 서비스가 SYSTEM으로 설치되어 있었습니다.
2. `Get-ExecutionPolicy -List`로 범위별 정책 확인 → MachinePolicy/UserPolicy가 Restricted, 프로세스 범위만 우리가 제어 가능.
3. `deploy.ps1` 자체는 문제가 없었고, 서명도 정상. 배제하고 나니 남은 변수는 **정책 상속 경로**뿐이었습니다.

## 원인

PowerShell 실행 정책은 범위가 있습니다: `MachinePolicy` > `UserPolicy` > `Process` > `CurrentUser` > `LocalMachine`. 관리자 대화형 세션에서는 CurrentUser 정책이 RemoteSigned여서 스크립트가 돌지만, SYSTEM 계정에는 그 CurrentUser 설정이 없고 MachinePolicy가 Restricted로 남아 있습니다.

GitHub Actions의 `shell: powershell`은 그 시점의 정책을 그대로 상속합니다. 그래서 **같은 파일이 사용자에 따라 되고 안 되는** 형태로 나타났습니다.

## 해결

머신 전역 정책을 바꾸는 대신(보안 정책 변경 + 전체 영향), 그 스텝에만 유효한 셸 정의를 씁니다.

```diff
-        shell: powershell
+        shell: powershell -NoLogo -NoProfile -NonInteractive -ExecutionPolicy Bypass -File {0}
```

- `-ExecutionPolicy Bypass`는 **그 프로세스에만** 적용됩니다. 머신 정책·레지스트리는 그대로.
- `-NoProfile`은 러너 호스트의 프로필 스크립트가 끼어들지 않게 해 재현성을 확보합니다.
- `-NonInteractive`는 인증서 저장소 프롬프트 같은 것이 떠서 워크플로가 매달리는 상황을 막습니다.
- `-NoLogo`는 로그 노이즈 제거.

거부된 배포는 그동안 기존 서비스를 되돌려 놓은 상태로 두고, 이 변경을 검증한 뒤 다시 배포했습니다.

## 검증

| 확인 | 방법 | 결과 |
|------|------|------|
| 계약 테스트(RED→GREEN) | `tools/deploy/tests/test_workflow_contract.py`가 셸 정의 문자열을 검사 | 수정 전 실패, 수정 후 통과 |
| 거부 경로 | SYSTEM 컨텍스트에서 스텝 실행 | 스크립트 로드 후 정상 진행 |
| 정책 불변 | `Get-ExecutionPolicy -List` 머신/사용자 범위 재확인 | 변경 없음 |

계약 테스트를 먼저 RED로 만든 게 중요했습니다. 이 실패 모드는 "테스트는 다 통과하는데 배포만 안 되는" 형태라서, 워크플로 파일 자체를 검사 대상으로 삼지 않으면 회귀를 못 잡습니다.

## 재발 방지

- 계약 테스트가 `shell:` 라인을 고정. 누군가 `shell: powershell`로 되돌리면 CI가 잡습니다.
- 배포는 항상 실제 실행 주체(SYSTEM)에서 검증. 관리자 세션 성공은 증거로 치지 않습니다.

## 일반화

- **권한 문제를 만나면 "누가 실행했는가"를 먼저 확정하라.** 사용자·서비스 계정·스케줄러·컨테이너는 각각 다른 프로필과 정책을 상속한다.
- **정책은 좁은 범위에서 푼다.** 머신 전역 예외는 문제를 한 번에 없애지만, 그 호스트에서 도는 모든 스크립트의 안전망을 같이 없앤다.
- **워크플로 파일도 코드다.** 문자열 수준의 계약 테스트가 없으면 이런 실패는 배포 때만 드러난다.
