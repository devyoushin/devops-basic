# CLAUDE.md — devops-basic 지식 베이스

DevOps 운영 경험 기반의 개인 지식 베이스. CI/CD, GitOps, 배포 전략, IaC, DevSecOps 문서를 관리.

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
├── docs/90-standards/                         # Claude 작성 규칙
│   ├── doc-writing.md             # 문서 스타일 가이드
│   ├── devops-conventions.md      # GitHub Actions/Helm/Terraform 코드 규칙
│   └── security-checklist.md     # DevOps 보안 체크리스트
│
├── docs/91-templates/                     # 재사용 문서 템플릿
│   ├── service-doc.md             # 서비스 문서 스캐폴딩
│   └── runbook.md                 # 운영 Runbook
│
├── docs/99-agents/                        # Claude 전문 에이전트
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

## 커스텀 슬래시 커맨드

| 커맨드 | 사용법 | 설명 |
|--------|--------|------|
| `/new-doc` | `/new-doc cicd github-oidc` | 신규 서비스 문서 스캐폴딩 생성 |
| `/review-doc` | `/review-doc docs/01-cicd/cicd-multi-account-architecture.md` | 문서 품질 검토 및 개선안 제시 |
| `/search-kb` | `/search-kb ArgoCD IRSA` | 지식 베이스 키워드 검색 및 관련 문서 목록 반환 |

---

## 파일 네이밍 규칙

```
docs/{카테고리}/{서비스}-{주제}.md
```

- 카테고리: `cicd`, `gitops`, `deployment`, `iac`, `security`
- 서비스 약어: `cicd`, `argocd`, `helm`, `terraform`, `github-actions`
- 주제: 소문자 영어, 하이픈 구분
- 예시: `docs/01-cicd/cicd-github-actions-ecr.md`, `docs/02-gitops/argocd-eks-deployment.md`

---

## 문서 작성 원칙

1. **실제 경험 기반** — 운영 중 실제로 겪은 이슈와 해결 방법 위주
2. **재현 가능한 코드** — GitHub Actions YAML, Terraform, Helm 복붙 즉시 실행 가능 수준
3. **원인 중심 트러블슈팅** — 증상만 나열하지 말고 근본 원인 설명
4. **한국어 기술 문서** — 주요 개념은 영어 원문 병기
5. **보안 우선** — 최소 권한, Secrets 노출 방지, OIDC 방식 권장
6. **모니터링 포함** — 파이프라인 실패 알림, 배포 상태 모니터링 포함

세부 규칙은 `docs/90-standards/` 디렉토리를 참조.

---

## 핵심 아키텍처 전제

이 지식 베이스의 모든 문서는 아래 아키텍처를 기준으로 작성됨:

- **AWS 멀티 계정**: CICD Account / Prod Account / Staging Account 분리
- **cicd-user**: CICD Account의 IAM User, ECR push 전용 최소 권한
- **GitOps**: GitHub/GitLab을 Single Source of Truth로 활용
- **ArgoCD**: EKS 내부 실행, IRSA(IAM Roles for Service Accounts)로 클러스터 접근
- **이미지 태그**: `{git-sha}` 또는 `{env}-{git-sha}` 형식, `latest` 금지

---

## 코드 예시 기준

| 항목 | 기준값 |
|------|--------|
| AWS 리전 | `ap-northeast-2` |
| Terraform provider | `~> 5.0` |
| GitHub Actions runner | `ubuntu-latest` |
| ArgoCD 버전 | `v2.x` |
| EKS 버전 | `1.29+` |
| Helm 버전 | `v3.x` |

---

## 카테고리별 문서 목록

### docs/01-cicd/
| 파일 | 주제 |
|------|------|
| `cicd-multi-account-architecture.md` | 멀티 계정 CI/CD 아키텍처 전체 (★ 핵심 문서) |
| `cicd-github-actions-ecr.md` | GitHub Actions + ECR 파이프라인 구성 |
| `cicd-image-tagging-strategy.md` | Docker 이미지 태깅 전략 및 ECR Lifecycle |

### docs/02-gitops/
| 파일 | 주제 |
|------|------|
| `argocd-eks-deployment.md` | ArgoCD EKS 배포 및 IRSA 인증 (★ 핵심 문서) |
| `argocd-app-of-apps.md` | App of Apps 패턴, ApplicationSet 멀티 환경 관리 |
| `gitops-workflow.md` | GitOps 원칙, Branch 전략, 롤백 방법 |

### docs/03-deployment/
| 파일 | 주제 |
|------|------|
| `deployment-strategies.md` | Rolling/Blue-Green/Canary 비교 및 구현 |
| `helm-release-management.md` | Helm chart 구조, 환경별 values.yaml, ArgoCD 연동 |

### docs/04-iac/
| 파일 | 주제 |
|------|------|
| `terraform-cicd-workflow.md` | Terraform 자동화, Atlantis, Remote State |

### docs/05-security/
| 파일 | 주제 |
|------|------|
| `devsecops-image-scan.md` | 컨테이너 이미지 스캔 (ECR Inspector, Trivy) |
| `secrets-in-pipeline.md` | 파이프라인 Secrets 관리, OIDC, External Secrets Operator |

---

## 추가 예정 주제 (백로그)

- `docs/01-cicd/cicd-gitlab-cicd.md` — GitLab CI/CD 파이프라인 구성
- `docs/02-gitops/argocd-notifications.md` — ArgoCD 알림 (Slack, PagerDuty)
- `docs/03-deployment/deployment-verification.md` — 배포 검증 (Smoke Test, Canary 분석)
- `docs/04-iac/crossplane-vs-terraform.md` — Crossplane vs Terraform 비교
- `docs/05-security/sbom-supply-chain.md` — SBOM 생성, Supply Chain 보안
