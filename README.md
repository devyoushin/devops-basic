# devops-basic

CI/CD, GitOps, 배포 전략, IaC, DevSecOps 운영 경험을 정리한 개인 지식 베이스입니다.

## 어디서 시작할까

- 문서 지도: `docs/README.md`
- 운영/실습 자산: `ops/README.md`
- AI 작업 지침: `CLAUDE.md`

## 구조

| 경로 | 내용 |
|------|------|
| `docs/` | CI/CD, GitOps, 배포, IaC, 보안 문서와 에이전트/규칙/템플릿 |
| `ops/` | 향후 파이프라인 예제, Helm/Terraform/GitOps 실습 자산 |
| `.claude/` | Claude Code 커맨드와 설정 |
| `CLAUDE.md` | Claude 작업 지침 |

## 학습 흐름

1. `docs/cicd/`에서 GitHub Actions, ECR, 멀티 계정 파이프라인 이해
2. `docs/gitops/`에서 ArgoCD와 GitOps 운영 방식 학습
3. `docs/deployment/`에서 배포 전략과 Helm 릴리스 관리 학습
4. `docs/iac/`, `docs/security/`에서 Terraform 자동화와 DevSecOps 확장
