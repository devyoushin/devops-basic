# 멀티 계정 CI/CD 아키텍처 (Multi-Account CI/CD Architecture)

## 1. 개요

AWS 멀티 계정 환경에서 CI/CD 파이프라인을 운영할 때 CICD 전용 계정을 분리하여 최소 권한 원칙을 적용하는 아키텍처. cicd-user는 ECR push 권한만 보유하며, EKS·EC2 등 다른 리소스에 접근 불가.

**핵심 원칙**
- CICD Account: 빌드/배포 도구만 위치 (빌드 파이프라인, cicd-user)
- Prod/Staging Account: 실제 워크로드 (ECR, EKS, ArgoCD)
- 계정 간 신뢰: ECR Resource-based Policy로 크로스 계정 push 허용
- GitOps: Git이 배포 상태의 Single Source of Truth

---

## 2. 전체 배포 파이프라인 다이어그램

```
┌─────────────────────────────────────────────────────────────────────┐
│                         전체 배포 파이프라인                          │
└─────────────────────────────────────────────────────────────────────┘

[개발자]
    │ git push / PR merge
    ▼
[GitHub / GitLab]
    │ webhook → CI 트리거
    ▼
┌─────────────────────────────────────────────────────┐
│   CICD Account  (111111111111)                       │
│                                                      │
│  GitHub Actions Runner (또는 GitLab Runner)           │
│                                                      │
│  cicd-user (IAM User)                                │
│  ┌─────────────────────────────────┐                 │
│  │ ECR 최소 권한 Policy              │                 │
│  │ - ecr:GetAuthorizationToken     │                 │
│  │ - ecr:BatchCheckLayerAvailability│                 │
│  │ - ecr:CompleteLayerUpload       │                 │
│  │ - ecr:InitiateLayerUpload       │                 │
│  │ - ecr:PutImage                  │                 │
│  │ - ecr:UploadLayerPart           │                 │
│  │ (EKS, EC2, S3 등 접근 불가)      │                 │
│  └─────────────────────────────────┘                 │
│                                                      │
│  ① docker build --platform linux/amd64               │
│  ② aws ecr get-login-password → docker login         │
│  ③ docker push → ECR (Prod Account, cross-account)   │
│  ④ GitOps repo image tag 업데이트 (PR or direct push) │
└─────────────────────────────────────────────────────┘
         │                              │
         │ ECR cross-account push       │ GitOps repo 변경
         ▼                              ▼
┌────────────────────────┐    ┌────────────────────────┐
│   Prod Account         │    │   GitHub (GitOps repo)  │
│   (222222222222)       │    │                         │
│                        │    │  k8s-manifests/         │
│  ECR Repository        │◀───│  └── app/               │
│  └── app:abc1234       │    │      └── deployment.yaml│
│      (git-sha 태그)     │    │          image: ...     │
│                        │    │          :abc1234 ←변경  │
│  ArgoCD                │    └────────────────────────┘
│  (EKS 내부 실행)         │              │
│  - IRSA Role 사용       │              │ ArgoCD polling
│  - GitOps repo 감시     │◀─────────────┘ 변경 감지
│  - 변경 감지 → 동기화    │
│                        │
│  EKS Cluster           │
│  └── Deployment        │
│      image: app:abc1234│
│      (새 이미지로 업데이트)│
└────────────────────────┘
```

### Staging 계정 병렬 구조

```
┌─────────────────────────────────────────────────────────────────────┐
│   Staging Account  (333333333333)                                    │
│                                                                      │
│   ECR Repository  ← CICD Account cicd-user가 staging 태그로 push     │
│   ArgoCD (IRSA)   ← staging GitOps branch 감시                       │
│   EKS Cluster     ← staging 배포 대상                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. IAM 구성 상세

### 3-1. cicd-user IAM Policy (최소 권한)

ECR push에 필요한 최소 권한만 부여. `ecr:GetAuthorizationToken`은 IAM 레벨 작업이므로 Resource `*` 필수.

```hcl
# terraform/cicd-account/iam.tf

resource "aws_iam_user" "cicd_user" {
  name = "cicd-user"
  path = "/01-cicd/"

  tags = {
    Purpose     = "CICD pipeline ECR push"
    Environment = "cicd"
    ManagedBy   = "terraform"
  }
}

resource "aws_iam_policy" "cicd_ecr_push" {
  name        = "cicd-ecr-push-policy"
  description = "ECR push 전용 최소 권한 (EKS/EC2 접근 불가)"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        # GetAuthorizationToken은 리소스 ARN 지정 불가 (IAM 레벨)
        Sid    = "ECRGetAuthToken"
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken"
        ]
        Resource = "*"
      },
      {
        # 특정 ECR 리포지토리에만 push 권한 부여
        Sid    = "ECRPushImage"
        Effect = "Allow"
        Action = [
          "ecr:BatchCheckLayerAvailability",
          "ecr:CompleteLayerUpload",
          "ecr:InitiateLayerUpload",
          "ecr:PutImage",
          "ecr:UploadLayerPart",
          "ecr:BatchGetImage",
          "ecr:GetDownloadUrlForLayer"
        ]
        Resource = [
          # Prod Account ECR 리포지토리
          "arn:aws:ecr:ap-northeast-2:<PROD_ACCOUNT_ID>:repository/<APP_NAME>",
          # Staging Account ECR 리포지토리
          "arn:aws:ecr:ap-northeast-2:<STAGING_ACCOUNT_ID>:repository/<APP_NAME>"
        ]
      }
    ]
  })
}

resource "aws_iam_user_policy_attachment" "cicd_user_ecr" {
  user       = aws_iam_user.cicd_user.name
  policy_arn = aws_iam_policy.cicd_ecr_push.arn
}

# Access Key 생성 (GitHub Secrets에 등록)
resource "aws_iam_access_key" "cicd_user" {
  user = aws_iam_user.cicd_user.name
}

# Access Key는 Terraform state에 저장됨 - state 암호화 필수
output "cicd_user_access_key_id" {
  value     = aws_iam_access_key.cicd_user.id
  sensitive = true
}

output "cicd_user_secret_access_key" {
  value     = aws_iam_access_key.cicd_user.secret
  sensitive = true
}
```

### 3-2. ECR Resource-based Policy (크로스 계정 push 허용)

Prod Account ECR에 CICD Account의 cicd-user가 push할 수 있도록 Resource-based Policy 설정.

```hcl
# terraform/prod-account/ecr.tf

resource "aws_ecr_repository" "app" {
  name                 = "<APP_NAME>"
  image_tag_mutability = "MUTABLE"  # sha 태그는 고정, latest 태그는 가변

  image_scanning_configuration {
    scan_on_push = true  # 푸시 시 자동 취약점 스캔
  }

  encryption_configuration {
    encryption_type = "AES256"
  }

  tags = {
    Environment = "prod"
    ManagedBy   = "terraform"
  }
}

resource "aws_ecr_repository_policy" "cross_account_push" {
  repository = aws_ecr_repository.app.name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCICDAccountPush"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::<CICD_ACCOUNT_ID>:user/01-cicd/cicd-user"
        }
        Action = [
          "ecr:BatchCheckLayerAvailability",
          "ecr:CompleteLayerUpload",
          "ecr:InitiateLayerUpload",
          "ecr:PutImage",
          "ecr:UploadLayerPart",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage"
        ]
      },
      {
        # EKS 노드 및 Fargate가 이미지를 pull할 수 있도록
        Sid    = "AllowEKSPull"
        Effect = "Allow"
        Principal = {
          AWS = [
            "arn:aws:iam::<PROD_ACCOUNT_ID>:role/<EKS_NODE_ROLE>",
            "arn:aws:iam::<PROD_ACCOUNT_ID>:role/<ARGOCD_IRSA_ROLE>"
          ]
        }
        Action = [
          "ecr:BatchGetImage",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchCheckLayerAvailability"
        ]
      }
    ]
  })
}
```

### 3-3. GitHub OIDC 방식 (Access Key 없이 인증, 권장)

cicd-user Access Key 대신 GitHub OIDC로 IAM Role을 assume하는 방식. **Access Key 유출 위험 없음.**

```hcl
# terraform/cicd-account/github-oidc.tf

# GitHub OIDC Provider (계정당 1개)
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1",
    "1c58a3a8518e8759bf075b76b750d4f2df264fcd"
  ]
}

# GitHub Actions용 IAM Role
resource "aws_iam_role" "github_actions_ecr" {
  name = "github-actions-ecr-push"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.github.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            # 특정 리포지토리의 main 브랜치에서만 assume 가능
            "token.actions.githubusercontent.com:sub" = "repo:<YOUR_ORG>/<YOUR_REPO>:ref:refs/heads/main"
          }
        }
      }
    ]
  })

  tags = {
    Purpose   = "GitHub Actions ECR push (OIDC)"
    ManagedBy = "terraform"
  }
}

resource "aws_iam_role_policy_attachment" "github_actions_ecr" {
  role       = aws_iam_role.github_actions_ecr.name
  policy_arn = aws_iam_policy.cicd_ecr_push.arn
}
```

---

## 4. ECR 이미지 태그 전략

| 태그 형식 | 예시 | 용도 |
|-----------|------|------|
| `{git-sha}` | `abc1234` | 기본 태그, 모든 빌드 |
| `{env}-{git-sha}` | `prod-abc1234` | 환경별 구분 필요 시 |
| `{version}-{git-sha}` | `v1.2.3-abc1234` | Semantic Versioning 병행 시 |
| `latest` | `latest` | **사용 금지** (배포 추적 불가) |

### 태그 생성 스크립트

```bash
# CI 파이프라인에서 태그 생성
GIT_SHA=$(git rev-parse --short HEAD)          # 7자리 sha
GIT_SHA_FULL=$(git rev-parse HEAD)             # 40자리 full sha
IMAGE_TAG="${GIT_SHA}"                          # 기본 태그
IMAGE_TAG_ENV="${DEPLOY_ENV}-${GIT_SHA}"        # 환경 포함 태그

echo "IMAGE_TAG=${IMAGE_TAG}" >> $GITHUB_ENV
```

---

## 5. 계정별 ECR 아키텍처 패턴 비교

### 패턴 A: Prod Account에 ECR (권장)

```
CICD Account                    Prod Account
cicd-user ──cross-account push──▶ ECR ──pull──▶ EKS
```

- ECR과 EKS가 동일 계정에 위치 → pull 시 추가 인증 불필요
- cicd-user는 다른 계정 리소스에 push (격리 유지)

### 패턴 B: CICD Account에 ECR

```
CICD Account                    Prod Account
cicd-user ──push──▶ ECR         ECR ──cross-account pull──▶ EKS
```

- EKS 노드가 다른 계정 ECR을 pull → EKS Node Role에 cross-account 권한 필요
- ECR Lifecycle Policy 관리가 CICD Account에 집중됨

**운영 편의성 고려 시 패턴 A 권장.**

---

## 6. 보안 고려사항

### cicd-user Access Key 관리

```bash
# Access Key 생성 후 즉시 GitHub Secrets에 등록
# Settings > Secrets and variables > Actions
AWS_ACCESS_KEY_ID     = <cicd-user Access Key ID>
AWS_SECRET_ACCESS_KEY = <cicd-user Secret Access Key>

# Access Key 로테이션 (90일 주기 권장)
aws iam create-access-key --user-name cicd-user --region ap-northeast-2
aws iam delete-access-key --user-name cicd-user --access-key-id <OLD_KEY_ID> --region ap-northeast-2
```

### Access Key 유출 시 대응

```bash
# 1. 즉시 비활성화
aws iam update-access-key \
  --user-name cicd-user \
  --access-key-id <LEAKED_KEY_ID> \
  --status Inactive \
  --region ap-northeast-2

# 2. CloudTrail로 유출 키 사용 이력 확인
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=<LEAKED_KEY_ID> \
  --region ap-northeast-2

# 3. 새 키 생성 후 GitHub Secrets 업데이트
aws iam create-access-key --user-name cicd-user --region ap-northeast-2

# 4. 구키 삭제
aws iam delete-access-key \
  --user-name cicd-user \
  --access-key-id <OLD_KEY_ID> \
  --region ap-northeast-2
```

### GitHub OIDC로 마이그레이션 (Access Key 완전 제거)

```
현재: cicd-user IAM User + Access Key → GitHub Secrets
목표: GitHub OIDC → IAM Role (시간 제한 임시 자격 증명)

마이그레이션 절차:
1. aws-oidc-provider terraform 배포
2. github-actions-ecr IAM Role 생성
3. GitHub Actions workflow에서 AWS 인증 방식 변경
   (aws-actions/configure-aws-credentials@v4, role-to-assume 사용)
4. GitHub Secrets에서 Access Key 제거
5. cicd-user IAM User 삭제
```

---

## 7. 트러블슈팅

### 증상 1: ECR push 시 "denied: Your authorization token has expired"

```
Error response from daemon: denied: Your authorization token has expired.
Reauthenticate and try again.
```

**원인**: ECR 인증 토큰은 12시간 유효. 장시간 빌드 후 만료.

**해결 방법**:
```bash
# CI 파이프라인에서 push 직전에 재인증
aws ecr get-login-password --region ap-northeast-2 | \
  docker login --username AWS --password-stdin \
  <PROD_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com
```

---

### 증상 2: ECR cross-account push 시 403 Access Denied

```
Error response from daemon: denied: User: arn:aws:iam::<CICD_ACCOUNT_ID>:user/01-cicd/cicd-user
is not authorized to perform: ecr:InitiateLayerUpload
```

**원인**: ECR Resource-based Policy 미설정 또는 Principal ARN 오타.

**해결 방법**:
```bash
# ECR Policy 확인
aws ecr get-repository-policy \
  --repository-name <APP_NAME> \
  --region ap-northeast-2 \
  --profile prod-account

# cicd-user ARN 확인
aws iam get-user --user-name cicd-user --profile cicd-account
```

---

### 증상 3: GitOps repo 업데이트 후 ArgoCD가 감지 못함

**원인**: ArgoCD polling 주기 기본 3분. 또는 webhook 미설정.

**해결 방법**:
```bash
# ArgoCD webhook 설정 확인
kubectl get secret argocd-secret -n argocd -o yaml

# 강제 refresh
argocd app get <APP_NAME> --refresh

# webhook 설정 (GitHub → ArgoCD)
# GitHub repo Settings > Webhooks > Add webhook
# URL: https://<ARGOCD_URL>/api/webhook
# Content type: application/json
# Events: Push events, Pull request events
```

---

## 8. 모니터링 및 알람

### ECR push 성공/실패 모니터링

```bash
# CloudWatch Log Insights - GitHub Actions 실패 감지
# (CloudWatch Logs에 GitHub Actions 로그를 수집하는 경우)
fields @timestamp, @message
| filter @message like /ERROR/ or @message like /FAILED/
| sort @timestamp desc
| limit 20
```

### ECR 이미지 스캔 결과 알람

```hcl
# ECR 취약점 스캔 EventBridge 규칙
resource "aws_cloudwatch_event_rule" "ecr_scan_finding" {
  name        = "ecr-critical-finding"
  description = "ECR 이미지에서 CRITICAL 취약점 발견 시 알람"

  event_pattern = jsonencode({
    source      = ["aws.ecr"]
    detail-type = ["ECR Image Scan"]
    detail = {
      scan-status = ["COMPLETE"]
      finding-severity-counts = {
        CRITICAL = [{ numeric = [">", 0] }]
      }
    }
  })
}

resource "aws_cloudwatch_event_target" "ecr_scan_sns" {
  rule      = aws_cloudwatch_event_rule.ecr_scan_finding.name
  target_id = "SendToSNS"
  arn       = aws_sns_topic.devops_alerts.arn
}
```

---

## 9. TIP

**TIP 1**: cicd-user에 `iam:SimulatePrincipalPolicy` 권한을 부여하면 파이프라인에서 사전 권한 검증 가능. 단, 이 권한 자체가 정보 노출 위험이 있으므로 별도 감사 계정에만 부여 권장.

**TIP 2**: Prod ECR에 이미지 푸시 전 Staging에 먼저 푸시하여 테스트하는 `promote` 패턴 적용. 동일 이미지 다이제스트를 Staging → Prod로 재태깅.

```bash
# 이미지 프로모션: Staging 검증 후 Prod 태그 추가
STAGING_DIGEST=$(aws ecr describe-images \
  --repository-name <APP_NAME> \
  --image-ids imageTag=staging-${GIT_SHA} \
  --query 'imageDetails[0].imageDigest' \
  --output text \
  --profile staging-account)

# 동일 다이제스트에 prod 태그 추가
aws ecr put-image \
  --repository-name <APP_NAME> \
  --image-tag prod-${GIT_SHA} \
  --image-manifest "$(aws ecr batch-get-image \
    --repository-name <APP_NAME> \
    --image-ids imageDigest=${STAGING_DIGEST} \
    --query 'images[0].imageManifest' \
    --output text \
    --profile staging-account)" \
  --profile prod-account
```

**TIP 3**: `--region ap-northeast-2` 명시 권장. 환경 변수 `AWS_DEFAULT_REGION`에 의존하면 파이프라인 환경에 따라 다른 리전에 push되는 실수 발생 가능.

**관련 문서**
- [GitHub Actions + ECR 파이프라인](./cicd-github-actions-ecr.md)
- [이미지 태깅 전략](./cicd-image-tagging-strategy.md)
- [ArgoCD EKS 배포 및 IRSA](../02-gitops/argocd-eks-deployment.md)
