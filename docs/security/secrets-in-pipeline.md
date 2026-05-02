# 파이프라인 Secrets 관리 (Secrets Management in Pipeline)

## 1. 개요

CI/CD 파이프라인에서 민감 정보(DB 비밀번호, API 키, 인증서 등)를 안전하게 관리하는 방법. GitHub Secrets는 단순하지만 로테이션이 수동. AWS Secrets Manager는 자동 로테이션과 감사 지원.

**Secrets 보관 위치별 비교**

| 방식 | 보관 위치 | 로테이션 | 감사 로그 | 접근 제어 | 권장 |
|------|-----------|----------|-----------|-----------|------|
| GitHub Secrets | GitHub | 수동 | 제한적 | 리포/환경 레벨 | CI 파이프라인 자격증명 |
| AWS Secrets Manager | AWS | 자동 | CloudTrail | IAM Policy | 애플리케이션 런타임 |
| HashiCorp Vault | Self-hosted | 자동 | Vault Audit | Policy | AWS 외 멀티클라우드 |
| K8s Secret | etcd (암호화) | 수동 | K8s 감사 | RBAC | 단순 환경 |
| External Secrets | AWS/Vault 연동 | 소스 따라감 | 소스 따라감 | IAM+RBAC | K8s + AWS 조합 |

---

## 2. GitHub Actions Secrets 관리

### 2-1. Secrets 종류

```
GitHub Secrets 계층:
├── Organization Secrets   - 조직 전체 공유 (팀 공통)
├── Repository Secrets     - 리포지토리별
└── Environment Secrets    - 환경별 (dev/staging/prod 분리)
    # Environment Secrets는 필수 리뷰어, 대기 시간 설정 가능
    # Prod 환경은 시니어 엔지니어 승인 후 접근
```

### 2-2. GitHub Secrets CLI 관리

```bash
# GitHub CLI로 Secrets 설정
gh secret set AWS_ACCESS_KEY_ID --body "AKIA..." \
  --repo <YOUR_ORG>/<YOUR_REPO>

# 환경별 Secret 설정
gh secret set DATABASE_URL \
  --body "postgresql://user:pass@host:5432/db" \
  --repo <YOUR_ORG>/<YOUR_REPO> \
  --env prod

# Secrets 목록 확인 (값은 표시 안 됨)
gh secret list --repo <YOUR_ORG>/<YOUR_REPO>

# Secret 삭제
gh secret delete OLD_SECRET --repo <YOUR_ORG>/<YOUR_REPO>
```

### 2-3. GitHub Secrets 보안 규칙

```yaml
# ❌ 금지: Secrets 로그 출력
- run: echo "${{ secrets.DATABASE_URL }}"  # 마스킹되지만 습관적으로 금지

# ✅ 권장: env 변수로 전달
- name: Deploy
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
  run: |
    # 환경 변수로 안전하게 사용
    ./deploy.sh

# ✅ 권장: OIDC로 AWS 자격 증명 (Access Key Secret 불필요)
- name: Configure AWS
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}  # ARN만 Secret, 자격증명 없음
    aws-region: ap-northeast-2
```

---

## 3. OIDC로 AWS 인증 (Access Key 완전 제거)

### 3-1. GitHub OIDC → AWS IAM Role

```hcl
# terraform/cicd-account/github-oidc.tf

resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1",
    "1c58a3a8518e8759bf075b76b750d4f2df264fcd"
  ]
}

# 리포지토리별 세분화된 Role
resource "aws_iam_role" "github_app_deploy" {
  name = "github-app-deploy"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
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
          # 특정 리포지토리의 특정 브랜치만 허용 (최소 권한)
          "token.actions.githubusercontent.com:sub" = [
            "repo:<YOUR_ORG>/my-app:ref:refs/heads/main",
            "repo:<YOUR_ORG>/my-app:pull_request"
          ]
        }
      }
    }]
  })
}
```

### 3-2. GitHub Actions에서 OIDC 사용

```yaml
# .github/workflows/deploy.yaml

permissions:
  id-token: write   # OIDC JWT 발급 필수
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials (OIDC - Access Key 없음)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/github-app-deploy
          role-session-name: GitHubActions-${{ github.run_id }}
          aws-region: ap-northeast-2
          # access-key-id, secret-access-key 필드 없음!

      - name: Verify (Access Key 없이 AWS 접근)
        run: aws sts get-caller-identity
```

---

## 4. AWS Secrets Manager 연동

### 4-1. GitHub Actions에서 Secrets Manager 조회

```yaml
# GitHub Actions에서 런타임에 Secrets Manager 조회

- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: ap-northeast-2

- name: Fetch secrets from Secrets Manager
  run: |
    # 단일 시크릿
    DB_PASSWORD=$(aws secretsmanager get-secret-value \
      --secret-id /prod/my-app/db-password \
      --query SecretString \
      --output text \
      --region ap-northeast-2)

    # JSON 형태의 시크릿 파싱
    SECRET_JSON=$(aws secretsmanager get-secret-value \
      --secret-id /prod/my-app/credentials \
      --query SecretString \
      --output text \
      --region ap-northeast-2)

    DB_HOST=$(echo $SECRET_JSON | python3 -c "import json,sys; print(json.load(sys.stdin)['host'])")
    DB_USER=$(echo $SECRET_JSON | python3 -c "import json,sys; print(json.load(sys.stdin)['username'])")

    # GitHub Actions 마스킹 (로그에서 값 숨김)
    echo "::add-mask::${DB_PASSWORD}"
    echo "DB_PASSWORD=${DB_PASSWORD}" >> $GITHUB_ENV
```

### 4-2. AWS Secrets Manager Terraform 설정

```hcl
# terraform/prod-account/secrets.tf

resource "aws_secretsmanager_secret" "app_credentials" {
  name                    = "/prod/my-app/credentials"
  description             = "My App 프로덕션 데이터베이스 자격 증명"
  recovery_window_in_days = 7  # 삭제 후 7일간 복구 가능

  # 자동 로테이션 (RDS Lambda 함수 연동)
  # rotation_rules {
  #   automatically_after_days = 30
  # }

  tags = {
    Environment = "prod"
    Application = "my-app"
    ManagedBy   = "terraform"
  }
}

resource "aws_secretsmanager_secret_version" "app_credentials" {
  secret_id = aws_secretsmanager_secret.app_credentials.id
  secret_string = jsonencode({
    host     = "<RDS_ENDPOINT>"
    port     = 5432
    username = "my_app_user"
    password = "<INITIAL_PASSWORD_ROTATE_IMMEDIATELY>"
    dbname   = "my_app_db"
  })

  lifecycle {
    ignore_changes = [secret_string]  # 초기 설정 후 Terraform이 덮어쓰지 않도록
  }
}

# IAM Policy: 앱 IRSA Role이 시크릿 읽기 가능
resource "aws_iam_policy" "app_secrets_read" {
  name = "my-app-secrets-read"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ]
      Resource = [
        aws_secretsmanager_secret.app_credentials.arn,
        "arn:aws:secretsmanager:ap-northeast-2:<ACCOUNT_ID>:secret:/prod/my-app/*"
      ]
    }]
  })
}
```

---

## 5. ArgoCD에서 Secrets 관리

### 5-1. External Secrets Operator (권장)

```yaml
# external-secrets/cluster-secret-store.yaml
# ClusterSecretStore: AWS Secrets Manager 연결 (IRSA 사용)

apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
            # external-secrets-sa에 IRSA annotation 필요
```

```yaml
# apps/my-app/base/external-secret.yaml
# ExternalSecret: Secrets Manager → K8s Secret 동기화

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-credentials
  namespace: my-app
spec:
  refreshInterval: 1h  # 1시간마다 갱신

  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager

  target:
    name: my-app-credentials  # 생성될 K8s Secret 이름
    creationPolicy: Owner
    template:
      engineVersion: v2
      data:
        # K8s Secret의 키 이름과 값 형식 지정
        DATABASE_URL: "postgresql://{{ .username }}:{{ .password }}@{{ .host }}:{{ .port }}/{{ .dbname }}"
        DB_PASSWORD: "{{ .password }}"

  data:
    - secretKey: username
      remoteRef:
        key: /prod/my-app/credentials
        property: username
    - secretKey: password
      remoteRef:
        key: /prod/my-app/credentials
        property: password
    - secretKey: host
      remoteRef:
        key: /prod/my-app/credentials
        property: host
    - secretKey: port
      remoteRef:
        key: /prod/my-app/credentials
        property: port
    - secretKey: dbname
      remoteRef:
        key: /prod/my-app/credentials
        property: dbname
```

```yaml
# Deployment에서 ExternalSecret으로 생성된 K8s Secret 참조
spec:
  template:
    spec:
      containers:
        - name: my-app
          envFrom:
            - secretRef:
                name: my-app-credentials  # ExternalSecret이 생성한 Secret
```

### 5-2. Sealed Secrets (오프라인 암호화)

```bash
# Sealed Secrets 설치
helm install sealed-secrets sealed-secrets/sealed-secrets \
  -n kube-system

# kubeseal로 Secret 암호화 (공개키 사용)
kubeseal --format yaml < my-secret.yaml > my-sealed-secret.yaml
# my-sealed-secret.yaml은 Git에 커밋 가능 (복호화는 클러스터만 가능)

# 복호화 (클러스터 내부에서만)
kubectl get secret my-secret -n my-app -o yaml
```

---

## 6. Secrets 노출 방지

### 6-1. Git 커밋 전 비밀 스캔

```bash
# git-secrets 설치 및 설정
git secrets --install
git secrets --register-aws

# pre-commit hook으로 자동 스캔
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
git secrets --pre_commit_hook -- "$@"
EOF
chmod +x .git/hooks/pre-commit

# .gitleaksrc - Gitleaks 설정
[allowlist]
  paths = [".trivyignore", "*.md"]
```

### 6-2. GitHub Actions Secret 스캔

```yaml
# .github/workflows/secret-scan.yaml

name: Secret Scan

on:
  push:
    branches: [main]
  pull_request:

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 전체 히스토리 스캔

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 7. 트러블슈팅

### 증상 1: GitHub Actions 로그에서 Secret 값이 노출

**원인**: `echo $SECRET` 또는 디버그 출력으로 Secret 값이 로그에 포함.

**해결 방법**:
```bash
# 즉시 조치: GitHub > Actions > 해당 실행 로그 삭제
# 장기: Secret 즉시 로테이션

# 안전한 Secret 사용 패턴
- name: Use secret safely
  env:
    MY_SECRET: ${{ secrets.MY_SECRET }}
  run: |
    # env 변수로 받아서 사용 (직접 ${{ secrets.xxx }} 인라인 사용 지양)
    ./script.sh  # 스크립트 내부에서 $MY_SECRET 사용
```

---

### 증상 2: External Secrets Operator가 Secret을 갱신하지 않음

**원인**: ExternalSecret의 `refreshInterval`이 지나지 않았거나 IRSA 권한 오류.

**해결 방법**:
```bash
# ExternalSecret 상태 확인
kubectl get externalsecret -n my-app
kubectl describe externalsecret my-app-credentials -n my-app

# 강제 갱신
kubectl annotate externalsecret my-app-credentials \
  force-sync=$(date +%s) \
  -n my-app

# IRSA 권한 확인
kubectl logs -n external-secrets deployment/external-secrets --tail=50
```

---

## 8. 모니터링 및 알람

```bash
# Secrets Manager 접근 이상 감지 (CloudTrail + EventBridge)

# 이상 패턴: 비정상 시간 또는 IP에서 Secret 조회
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetSecretValue \
  --start-time $(date -v -1H -u +%Y-%m-%dT%H:%M:%SZ) \
  --region ap-northeast-2

# IAM Access Analyzer: 외부 접근 가능 Secret 탐지
aws accessanalyzer list-findings \
  --analyzer-arn <ANALYZER_ARN> \
  --filter '{"resourceType": {"eq": ["AWS::SecretsManager::Secret"]}}' \
  --region ap-northeast-2
```

---

## 9. TIP

**TIP 1**: GitHub Environment Secrets + Required Reviewers 조합으로 Prod 배포 전 수동 승인 게이트 구현. Prod Secret은 승인자가 있을 때만 접근 가능.

**TIP 2**: Secrets Manager의 `SecretString`에 JSON 저장 시 키별 접근 제어가 가능. `property` 필드로 필요한 필드만 가져오도록 ExternalSecret 설정.

**TIP 3**: CI/CD 파이프라인에서 AWS 자격 증명이 필요한 모든 곳에 OIDC를 사용하면 Access Key 관리 부담이 0이 됨. Access Key가 GitHub Secrets에 없으면 유출 자체가 불가능.

**관련 문서**
- [DevSecOps 이미지 스캔](./devsecops-image-scan.md)
- [멀티 계정 CI/CD 아키텍처](../cicd/cicd-multi-account-architecture.md)
- [GitHub Actions + ECR 파이프라인](../cicd/cicd-github-actions-ecr.md)
