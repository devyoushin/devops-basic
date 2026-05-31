# AGENTS.md — devops-basic Codex 작업 지침

이 저장소는 CI/CD, GitOps, 배포 전략, IaC, DevSecOps 운영 지식 베이스입니다. Codex 작업 시 `CLAUDE.md`와 `docs/rules/`의 규칙을 동일하게 따릅니다.

## 공통 원칙

- 문서는 `docs/` 아래에 둡니다.
- 실행 가능한 파이프라인, Helm, Terraform, GitOps 예제는 `ops/` 아래에 둡니다.
- 보안 예시는 OIDC, 최소 권한, secret 노출 방지를 우선합니다.
- 배포 관련 문서는 rollback, promotion, environment split을 포함합니다.

## Claude와의 싱크

- Claude용 상세 지침은 `CLAUDE.md`를 참고합니다.
- 로컬 개인 설정은 `CLAUDE.local.md`에 두며 Git에 올리지 않습니다.
- Codex도 공통 규칙은 `docs/rules/`를 따릅니다.

## 작업 체크리스트

- 기존 사용자 변경을 먼저 확인합니다.
- YAML, shell, Terraform, workflow 파일은 가능한 범위에서 문법 검사합니다.
- 링크 검사와 `git diff --check`를 수행합니다.
