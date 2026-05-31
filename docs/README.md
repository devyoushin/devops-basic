# DevOps Docs

DevOps 학습 문서는 운영 주제별로 나눠 관리합니다.

| 폴더 | 내용 |
|------|------|
| `cicd/` | CI/CD 파이프라인, Jenkins, GitHub Actions, 이미지 태깅, 멀티 계정 배포 |
| `gitops/` | ArgoCD, GitOps 워크플로우, App of Apps |
| `deployment/` | 배포 전략, Helm 릴리스 관리 |
| `iac/` | Terraform CI/CD 워크플로우 |
| `security/` | DevSecOps, 이미지 스캔, 파이프라인 Secret 관리 |
| `agents/` | AI 전문 에이전트 프롬프트 |
| `rules/` | 문서 작성과 DevOps 운영 규칙 |
| `templates/` | 재사용 문서 템플릿 |

실행 가능한 예제와 자동화 자산은 `../ops/README.md`를 참고합니다.

## 추천 학습 순서

| 순서 | 문서 | 목적 |
|------|------|------|
| 1 | `cicd/cicd-image-tagging-strategy.md` | 불변 이미지 태그 기준 이해 |
| 2 | `gitops/gitops-workflow.md` | GitOps 배포 흐름 이해 |
| 3 | `cicd/jenkins-argocd-cicd.md` | Jenkins와 ArgoCD 기반 CI/CD 구축 |
| 4 | `gitops/argocd-eks-deployment.md` | ArgoCD EKS 운영과 IRSA 인증 이해 |
| 5 | `security/devsecops-image-scan.md` | 이미지 스캔과 보안 게이트 적용 |
