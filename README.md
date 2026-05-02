# devops-basic — DevOps 운영 경험 기반 지식 베이스

AWS 멀티 계정 환경에서의 CI/CD 파이프라인, GitOps, 배포 전략, IaC 운영 경험을 정리한 개인 지식 베이스.

---

## 프로젝트 구조

```
devops-basic/
├── docs/                          # 지식 문서 (카테고리별 분류)
│   ├── cicd/        (3개)         # CI/CD 파이프라인, GitHub Actions, 이미지 태깅
│   ├── gitops/      (3개)         # ArgoCD, GitOps 워크플로우, App of Apps
│   ├── deployment/  (2개)         # 배포 전략, Helm 릴리스 관리
│   ├── iac/         (1개)         # Terraform CI/CD 자동화
│   └── security/    (2개)         # DevSecOps, Secrets 관리
│
├── rules/                         # Claude 작성 규칙
│   ├── doc-writing.md             # 문서 스타일 가이드
│   ├── devops-conventions.md      # GitHub Actions/Helm/Terraform 코드 규칙
│   └── security-checklist.md     # DevOps 보안 체크리스트
│
├── templates/                     # 재사용 문서 템플릿
│   ├── service-doc.md             # 서비스 문서 스캐폴딩
│   └── runbook.md                 # 운영 Runbook
│
├── agents/                        # Claude 전문 에이전트
│   ├── doc-writer.md              # DevOps 문서 작성 에이전트
│   └── pipeline-advisor.md       # CI/CD 파이프라인 설계/검토 에이전트
│
└── .claude/
    ├── settings.json              # 프로젝트 공유 설정
    └── commands/                  # 커스텀 슬래시 커맨드
        ├── new-doc.md             # /new-doc
        ├── review-doc.md          # /review-doc
        └── search-kb.md           # /search-kb
```

---

## 핵심 아키텍처 개요

### 멀티 계정 CI/CD 구조

```
CICD Account  ──(ECR cross-account push)──▶  Prod Account ECR
     │                                              │
     │ cicd-user (ECR push 전용)                    │ ArgoCD (IRSA)
     │                                              ▼
     └──(GitOps repo 업데이트)──▶  ArgoCD ──▶  EKS Cluster
```

- **CICD Account**: GitHub Actions Runner, cicd-user (ECR push 전용 최소 권한)
- **Prod/Staging Account**: ECR, EKS, ArgoCD (IRSA 방식 클러스터 접근)
- **GitOps**: Git이 Single Source of Truth, PR merge = 배포 트리거

---

## 문서 목록

### docs/cicd/ — CI/CD 파이프라인

| 파일 | 주제 |
|------|------|
| `cicd-multi-account-architecture.md` | 멀티 계정 CI/CD 아키텍처 (cicd-user, ECR 크로스 계정, 전체 배포 플로우) |
| `cicd-github-actions-ecr.md` | GitHub Actions + ECR 워크플로우 (OIDC 방식, Access Key 비교) |
| `cicd-image-tagging-strategy.md` | 이미지 태그 전략 (git-sha, ECR Lifecycle Policy, ArgoCD Image Updater) |

### docs/gitops/ — GitOps & ArgoCD

| 파일 | 주제 |
|------|------|
| `argocd-eks-deployment.md` | ArgoCD EKS 배포 (IRSA 인증, 멀티 클러스터, Sync Policy, Terraform) |
| `argocd-app-of-apps.md` | App of Apps 패턴, ApplicationSet, 멀티 환경 관리 |
| `gitops-workflow.md` | GitOps 원칙, Branch 전략, PR 기반 배포, 롤백 방법 |

### docs/deployment/ — 배포 전략

| 파일 | 주제 |
|------|------|
| `deployment-strategies.md` | Rolling/Blue-Green/Canary 비교, Argo Rollouts 연동 |
| `helm-release-management.md` | Helm chart 구조, values.yaml 환경별 오버라이드, ArgoCD 연동 |

### docs/iac/ — Infrastructure as Code

| 파일 | 주제 |
|------|------|
| `terraform-cicd-workflow.md` | Terraform plan/apply 자동화, Atlantis, Remote State (S3 + DynamoDB) |

### docs/security/ — DevSecOps

| 파일 | 주제 |
|------|------|
| `devsecops-image-scan.md` | ECR Inspector/Trivy 이미지 스캔, CI 파이프라인 통합, 취약점 빌드 실패 처리 |
| `secrets-in-pipeline.md` | GitHub Secrets vs AWS Secrets Manager, OIDC 인증, External Secrets Operator |

---

## 커스텀 슬래시 커맨드

| 커맨드 | 사용법 | 설명 |
|--------|--------|------|
| `/new-doc` | `/new-doc cicd github-oidc` | 신규 문서 스캐폴딩 생성 |
| `/review-doc` | `/review-doc docs/cicd/cicd-multi-account-architecture.md` | 문서 품질 검토 |
| `/search-kb` | `/search-kb ArgoCD IRSA` | 지식 베이스 키워드 검색 |

---

## 빠른 참조

### 전체 배포 파이프라인 요약

```
개발자 git push
    → GitHub Actions (CICD Account)
        → docker build
        → ECR push (cross-account to Prod Account)
        → GitOps repo image tag 업데이트
    → ArgoCD (Prod Account, IRSA 인증)
        → Git 변경 감지
        → EKS kubectl apply
        → 배포 완료
```

### 핵심 보안 원칙

- cicd-user는 ECR push 전용 (EKS, EC2 등 접근 불가)
- GitHub Actions는 OIDC 방식 권장 (Access Key 불필요)
- ArgoCD는 IRSA로 EKS 접근 (kubeconfig Secret 방식 지양)
- 이미지 태그 `latest` 사용 금지 (git-sha 또는 semantic versioning)

---

## 관련 지식 베이스

- [aws-basic](../aws-basic/) — AWS 서비스별 운영 경험 (EC2, EKS, CloudWatch, Network 등)
