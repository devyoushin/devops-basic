# Terraform CI/CD 워크플로우 (Terraform CI/CD Workflow)

## 1. 개요

Terraform plan/apply 자동화. GitHub Actions 기반 파이프라인과 Atlantis 기반 파이프라인 비교. Remote State는 S3 + DynamoDB Lock으로 팀 협업 및 동시 실행 방지.

**두 가지 Terraform CI/CD 방식**

| 항목 | GitHub Actions | Atlantis |
|------|----------------|----------|
| 동작 방식 | PR/Push 이벤트 → Workflow 실행 | PR comment → plan/apply |
| 설치 복잡도 | 낮음 (GitHub 네이티브) | 중간 (Atlantis 서버 필요) |
| plan 결과 확인 | Actions 로그 | PR comment (인라인) |
| 자동 apply | PR merge 시 가능 | PR merge 또는 comment |
| 멀티 workspace | 수동 설정 | 자동 (repo 구조 기반) |
| 권장 환경 | 소규모 팀, 단순 구조 | 중대규모 팀, 복잡한 구조 |

---

## 2. Remote State 구성 (S3 + DynamoDB)

### 2-1. Terraform Bootstrap (State 인프라)

```hcl
# terraform/bootstrap/main.tf
# Terraform State 저장소를 처음 한 번만 수동으로 생성

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-2"
}

# S3 버킷 (Terraform State 저장)
resource "aws_s3_bucket" "terraform_state" {
  bucket = "<YOUR_ORG>-terraform-state"

  # 실수로 삭제 방지
  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Purpose   = "Terraform Remote State"
    ManagedBy = "manual"  # bootstrap 리소스는 Terraform으로 관리하지 않음
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"  # 이전 State 복구 가능
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"  # KMS 암호화 (Access Key 등 민감 정보 보호)
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB (State Lock - 동시 apply 방지)
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Purpose = "Terraform State Lock"
  }
}
```

### 2-2. 환경별 State 분리

```hcl
# terraform/prod/backend.tf

terraform {
  backend "s3" {
    bucket         = "<YOUR_ORG>-terraform-state"
    key            = "prod/terraform.tfstate"  # 환경별 prefix로 분리
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"

    # 크로스 계정 State 접근 (CICD Account에서 Prod Account State 관리 시)
    # role_arn = "arn:aws:iam::<PROD_ACCOUNT_ID>:role/terraform-state-access"
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

```
State 분리 구조:
terraform-state S3 버킷
├── prod/terraform.tfstate        # Prod 계정 인프라
├── staging/terraform.tfstate     # Staging 계정 인프라
├── dev/terraform.tfstate         # Dev 계정 인프라
└── cicd/terraform.tfstate        # CICD 계정 인프라 (cicd-user, OIDC 등)
```

---

## 3. GitHub Actions 기반 Terraform CI/CD

### 3-1. 워크플로우 전체 예시

```yaml
# .github/workflows/terraform.yaml

name: Terraform CI/CD

on:
  push:
    branches:
      - main
    paths:
      - 'terraform/**'
  pull_request:
    branches:
      - main
    paths:
      - 'terraform/**'

env:
  TF_VERSION: "1.7.x"
  AWS_REGION: ap-northeast-2

permissions:
  id-token: write   # OIDC
  contents: read
  pull-requests: write  # PR comment 작성

jobs:
  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest

    # 변경된 환경만 plan (matrix 사용)
    strategy:
      matrix:
        environment: [prod, staging]
        include:
          - environment: prod
            role_arn_secret: PROD_TF_ROLE_ARN
            working_dir: terraform/prod
          - environment: staging
            role_arn_secret: STAGING_TF_ROLE_ARN
            working_dir: terraform/staging

    defaults:
      run:
        working-directory: ${{ matrix.working_dir }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[matrix.role_arn_secret] }}
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: TerraformPlan-${{ matrix.environment }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        id: fmt
        run: terraform fmt -check -recursive
        continue-on-error: true

      - name: Terraform Init
        id: init
        run: terraform init

      - name: Terraform Validate
        id: validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: |
          terraform plan \
            -no-color \
            -out=tfplan-${{ matrix.environment }} \
            2>&1 | tee plan-output.txt
        continue-on-error: true

      - name: Upload plan artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ matrix.environment }}
          path: ${{ matrix.working_dir }}/tfplan-${{ matrix.environment }}
          retention-days: 1

      - name: Comment PR with plan result
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            const planOutput = fs.readFileSync('${{ matrix.working_dir }}/plan-output.txt', 'utf8');
            const maxLength = 60000;
            const truncatedOutput = planOutput.length > maxLength
              ? planOutput.substring(0, maxLength) + '\n... (truncated)'
              : planOutput;

            const body = `## Terraform Plan - ${{ matrix.environment }}

            #### Format: \`${{ steps.fmt.outcome }}\`
            #### Init: \`${{ steps.init.outcome }}\`
            #### Validate: \`${{ steps.validate.outcome }}\`
            #### Plan: \`${{ steps.plan.outcome }}\`

            <details>
            <summary>Show Plan</summary>

            \`\`\`terraform
            ${truncatedOutput}
            \`\`\`

            </details>`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });

      - name: Terraform Plan Status
        if: steps.plan.outcome == 'failure'
        run: exit 1

  terraform-apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    needs: terraform-plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    # 환경별 순차 apply (동시 apply 방지)
    strategy:
      max-parallel: 1
      matrix:
        environment: [staging, prod]  # staging 먼저
        include:
          - environment: prod
            role_arn_secret: PROD_TF_ROLE_ARN
            working_dir: terraform/prod
          - environment: staging
            role_arn_secret: STAGING_TF_ROLE_ARN
            working_dir: terraform/staging

    environment: ${{ matrix.environment }}  # GitHub Environment (수동 승인 설정 가능)

    defaults:
      run:
        working-directory: ${{ matrix.working_dir }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[matrix.role_arn_secret] }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init

      - name: Download plan artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan-${{ matrix.environment }}
          path: ${{ matrix.working_dir }}

      - name: Terraform Apply
        run: |
          terraform apply \
            -auto-approve \
            -no-color \
            tfplan-${{ matrix.environment }}
```

### 3-2. Terraform IAM Role (OIDC)

```hcl
# terraform/cicd-account/terraform-oidc.tf

resource "aws_iam_role" "terraform_ci" {
  name = "terraform-ci-role"

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
            "token.actions.githubusercontent.com:sub" = [
              "repo:<YOUR_ORG>/<YOUR_IaC_REPO>:ref:refs/heads/main",
              "repo:<YOUR_ORG>/<YOUR_IaC_REPO>:pull_request"
            ]
          }
        }
      }
    ]
  })
}

# Terraform이 인프라를 관리하는 데 필요한 권한 (환경별로 제한)
resource "aws_iam_role_policy_attachment" "terraform_admin" {
  role       = aws_iam_role.terraform_ci.name
  # 운영 환경에서는 필요한 서비스만 허용하는 커스텀 Policy 권장
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}
```

---

## 4. Atlantis 기반 Terraform CI/CD

### 4-1. Atlantis 서버 설치 (EKS)

```yaml
# atlantis-helm-values.yaml

orgAllowlist: "github.com/<YOUR_ORG>/*"
defaultTFVersion: "1.7.x"

github:
  user: atlantis-bot
  token: <GITHUB_TOKEN>
  secret: <WEBHOOK_SECRET>

repoConfig: |
  repos:
  - id: /.*/
    allowed_overrides: [apply_requirements, workflow]
    allow_custom_workflows: true

  workflows:
    default:
      plan:
        steps:
          - init
          - plan:
              extra_args: ["-no-color"]
      apply:
        steps:
          - apply:
              extra_args: ["-no-color"]

serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/atlantis-irsa

# EKS 내부 접근만 허용 (GitHub webhook은 ALB를 통해 수신)
service:
  type: ClusterIP
```

### 4-2. atlantis.yaml (리포지토리 설정)

```yaml
# atlantis.yaml (IaC 리포지토리 루트)

version: 3

# 각 환경 디렉토리별 프로젝트 정의
projects:
  - name: prod
    dir: terraform/prod
    workspace: default
    autoplan:
      enabled: true
      when_modified: ["*.tf", "*.tfvars", "../modules/**/*.tf"]
    apply_requirements:
      - approved  # PR 승인 후 apply 가능
      - mergeable  # merge 가능 상태에서만 apply

  - name: staging
    dir: terraform/staging
    workspace: default
    autoplan:
      enabled: true
      when_modified: ["*.tf", "*.tfvars", "../modules/**/*.tf"]
    apply_requirements:
      - approved

  - name: cicd
    dir: terraform/cicd-account
    workspace: default
    autoplan:
      enabled: true
      when_modified: ["*.tf"]
    apply_requirements:
      - approved
```

---

## 5. Terraform 모듈 구조

```
terraform/
├── modules/                       # 공유 모듈
│   ├── eks/                       # EKS 클러스터 모듈
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── ecr/                       # ECR 리포지토리 모듈
│   │   └── main.tf
│   └── argocd-irsa/               # ArgoCD IRSA 모듈
│       └── main.tf
│
├── prod/                          # Prod 계정 인프라
│   ├── backend.tf
│   ├── provider.tf
│   ├── main.tf                    # 모듈 호출
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars           # Prod 환경 변수 (민감하지 않은 값만)
│
├── staging/                       # Staging 계정 인프라
│   └── ...
│
└── cicd-account/                  # CICD 계정 인프라
    ├── backend.tf
    ├── main.tf
    └── ...
```

---

## 6. 트러블슈팅

### 증상 1: "Error acquiring the state lock"

```
Error: Error acquiring the state lock
Lock Info:
  ID:        xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  Path:      prod/terraform.tfstate
  Operation: OperationTypePlan
  Who:       github-actions@runner
  Created:   2024-01-01 00:00:00 UTC
```

**원인**: 이전 plan/apply가 비정상 종료되어 DynamoDB Lock이 해제되지 않음.

**해결 방법**:
```bash
# Lock 강제 해제 (원인 확인 후 신중하게 실행)
terraform force-unlock <LOCK_ID>

# 또는 DynamoDB에서 직접 삭제
aws dynamodb delete-item \
  --table-name terraform-state-lock \
  --key '{"LockID": {"S": "prod/terraform.tfstate"}}' \
  --region ap-northeast-2
```

---

### 증상 2: plan과 apply 사이에 상태 변경으로 충돌

**원인**: plan 후 다른 사람이 apply → plan 결과와 실제 State 불일치.

**해결 방법**:
```bash
# 최신 State를 기반으로 re-plan 후 apply
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

---

## 7. 모니터링 및 알람

```bash
# Terraform State S3 버킷 변경 감지 (CloudTrail 활용)
# CloudTrail → EventBridge → SNS → Slack

# 의심스러운 State 수정 이벤트 탐지
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=<YOUR_ORG>-terraform-state \
  --region ap-northeast-2 \
  --query 'Events[?contains(EventName, `Put`) || contains(EventName, `Delete`)]'
```

---

## 8. TIP

**TIP 1**: `terraform plan -detailed-exitcode` 활용. exit code 0=성공(변경없음), 1=오류, 2=성공(변경있음). CI에서 변경이 없을 때 apply 건너뜀.

```bash
terraform plan -detailed-exitcode -out=tfplan
EXIT_CODE=$?
if [ $EXIT_CODE -eq 0 ]; then
  echo "No changes, skipping apply"
elif [ $EXIT_CODE -eq 1 ]; then
  echo "Plan failed"
  exit 1
elif [ $EXIT_CODE -eq 2 ]; then
  echo "Changes detected, applying..."
  terraform apply tfplan
fi
```

**TIP 2**: `terraform state mv`, `terraform import` 작업 전 반드시 State 백업.

```bash
# State 백업
terraform state pull > terraform.tfstate.backup.$(date +%Y%m%d%H%M%S)
```

**TIP 3**: PR마다 plan 결과가 PR comment에 자동으로 달리면 인프라 변경 리뷰가 코드 리뷰와 동일 맥락에서 가능. Atlantis 방식이 이 UX를 기본 제공.

**관련 문서**
- [멀티 계정 CI/CD 아키텍처](../cicd/cicd-multi-account-architecture.md)
- [DevSecOps 보안 체크리스트](../security/secrets-in-pipeline.md)
