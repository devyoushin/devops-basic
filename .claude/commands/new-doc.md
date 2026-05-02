# /new-doc — 신규 문서 스캐폴딩 생성

DevOps 지식 베이스에 새 마크다운 문서를 생성하는 커맨드.

## 사용법

```
/new-doc {카테고리} {주제}
```

## 예시

```
/new-doc cicd gitlab-ci-ecr
/new-doc gitops argocd-notifications
/new-doc deployment argo-rollouts-canary
/new-doc security sbom-supply-chain
/new-doc iac crossplane-vs-terraform
```

## 카테고리 목록

| 카테고리 | 디렉토리 | 대상 주제 |
|----------|----------|-----------|
| `cicd` | docs/cicd/ | GitHub Actions, GitLab CI, 파이프라인, 이미지 빌드 |
| `gitops` | docs/gitops/ | ArgoCD, Flux, GitOps 워크플로우 |
| `deployment` | docs/deployment/ | 배포 전략, Helm, Argo Rollouts |
| `iac` | docs/iac/ | Terraform, Atlantis, Crossplane |
| `security` | docs/security/ | DevSecOps, 이미지 스캔, Secrets |

## 실행 지침

이 커맨드를 받으면 아래 절차를 따릅니다:

1. `templates/service-doc.md`를 기반으로 새 파일 생성
2. 파일 경로: `docs/{카테고리}/{카테고리}-{주제}.md`
3. 제목과 개요 섹션을 주제에 맞게 채움
4. `rules/doc-writing.md` 규칙 준수
5. `rules/devops-conventions.md` 코드 컨벤션 준수
6. `rules/security-checklist.md` 보안 항목 포함
7. `CLAUDE.md`의 카테고리별 문서 목록 업데이트
8. 생성된 파일 경로와 주요 섹션 구조 안내

## 기본값

- AWS 리전: `ap-northeast-2`
- Terraform provider: `~> 5.0`
- GitHub Actions runner: `ubuntu-latest`
- 플레이스홀더: `<ACCOUNT_ID>`, `<CLUSTER_NAME>`, `<APP_NAME>`
