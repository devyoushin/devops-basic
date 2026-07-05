# GitHub Actions + ECR 파이프라인 (GitHub Actions ECR Pipeline)

## 1. 개요

GitHub Actions에서 Docker 이미지를 빌드하고 Amazon ECR에 push하는 파이프라인 구성. OIDC(OpenID Connect) 방식으로 AWS 인증하여 Access Key 없이 임시 자격 증명 사용 권장.

**두 가지 인증 방식 비교**

| 항목 | Access Key 방식 | OIDC 방식 (권장) |
|------|----------------|-----------------|
| 자격 증명 보관 | GitHub Secrets에 영구 저장 | 임시 토큰 (1시간) |
| 유출 위험 | 높음 (GitHub Secrets 노출 시) | 낮음 (임시 토큰) |
| 로테이션 | 수동 90일 주기 | 자동 (매 실행마다 갱신) |
| 설정 복잡도 | 낮음 | 중간 (OIDC Provider 사전 설정) |
| AWS 감사 | IAM User로 기록 | IAM Role + session으로 기록 |

---

## 2. OIDC 방식 (권장)

### 2-1. AWS 사전 설정

```hcl
# terraform/cicd-account/github-oidc.tf

# GitHub OIDC Identity Provider
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  # GitHub의 OIDC 인증서 thumbprint (2024년 기준)
  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1",
    "1c58a3a8518e8759bf075b76b750d4f2df264fcd"
  ]

  tags = {
    Purpose   = "GitHub Actions OIDC"
    ManagedBy = "terraform"
  }
}

# IAM Role - GitHub Actions가 assume
resource "aws_iam_role" "github_actions" {
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
            # main 브랜치 + 모든 PR에서 assume 가능
            "token.actions.githubusercontent.com:sub" = [
              "repo:<YOUR_ORG>/<YOUR_REPO>:ref:refs/heads/main",
              "repo:<YOUR_ORG>/<YOUR_REPO>:pull_request"
            ]
          }
        }
      }
    ]
  })

  tags = {
    Purpose   = "GitHub Actions ECR push via OIDC"
    ManagedBy = "terraform"
  }
}

# ECR push 권한 정책 연결
resource "aws_iam_role_policy_attachment" "github_actions_ecr" {
  role       = aws_iam_role.github_actions.name
  policy_arn = aws_iam_policy.cicd_ecr_push.arn  # 멀티계정 아키텍처 문서 참고
}

output "github_actions_role_arn" {
  value = aws_iam_role.github_actions.arn
  # 이 값을 GitHub Secrets의 AWS_ROLE_ARN으로 등록
}
```

### 2-2. GitHub Actions Workflow (OIDC 방식, 전체 예시)

```yaml
# .github/workflows/build-push.yaml

name: Build and Push to ECR

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  AWS_REGION: ap-northeast-2
  PROD_ACCOUNT_ID: ${{ secrets.PROD_ACCOUNT_ID }}
  ECR_REPOSITORY: <APP_NAME>
  GITOPS_REPO: <YOUR_ORG>/<YOUR_GITOPS_REPO>

# OIDC 토큰 발급에 필요한 최소 권한
permissions:
  id-token: write   # OIDC JWT 토큰 발급
  contents: read    # 리포지토리 체크아웃
  pull-requests: write  # PR 코멘트 (선택)

jobs:
  build-and-push:
    name: Build Docker Image and Push to ECR
    runs-on: ubuntu-latest

    outputs:
      image_tag: ${{ steps.meta.outputs.image_tag }}
      image_digest: ${{ steps.push.outputs.digest }}

    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          role-session-name: GitHubActions-${{ github.run_id }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Verify AWS identity
        run: aws sts get-caller-identity

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
        with:
          registries: ${{ env.PROD_ACCOUNT_ID }}
          # 크로스 계정 ECR에 로그인

      - name: Generate image metadata
        id: meta
        run: |
          GIT_SHA=$(git rev-parse --short HEAD)
          BRANCH_NAME="${GITHUB_REF_NAME//\//-}"  # 슬래시를 하이픈으로 치환

          if [ "${{ github.event_name }}" == "pull_request" ]; then
            # PR 빌드: pr-{번호}-{sha}
            IMAGE_TAG="pr-${{ github.event.pull_request.number }}-${GIT_SHA}"
          else
            # main 빌드: {sha}
            IMAGE_TAG="${GIT_SHA}"
          fi

          echo "image_tag=${IMAGE_TAG}" >> $GITHUB_OUTPUT
          echo "git_sha=${GIT_SHA}" >> $GITHUB_OUTPUT
          echo "IMAGE_TAG=${IMAGE_TAG}" >> $GITHUB_ENV

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        id: push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64
          push: ${{ github.event_name != 'pull_request' }}  # PR은 빌드만, main은 push
          tags: |
            ${{ env.PROD_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}
          # 캐시 활용으로 빌드 시간 단축
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            GIT_SHA=${{ steps.meta.outputs.git_sha }}
            VERSION=${{ steps.meta.outputs.image_tag }}

      - name: Extract image digest
        if: github.event_name != 'pull_request'
        run: |
          DIGEST="${{ steps.push.outputs.digest }}"
          echo "Image digest: ${DIGEST}"
          echo "IMAGE_DIGEST=${DIGEST}" >> $GITHUB_ENV

  # main 브랜치 push 시에만 GitOps repo 업데이트
  update-gitops:
    name: Update GitOps Repository
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.event_name != 'pull_request'

    steps:
      - name: Checkout GitOps repository
        uses: actions/checkout@v4
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}  # GitOps repo 쓰기 권한 PAT
          path: gitops

      - name: Update image tag in manifests
        run: |
          cd gitops
          IMAGE_TAG="${{ needs.build-and-push.outputs.image_tag }}"

          # kustomize 방식
          cd apps/<APP_NAME>/overlays/prod
          kustomize edit set image \
            <APP_NAME>=${{ env.PROD_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${IMAGE_TAG}

          # 또는 sed 방식 (kustomize 미사용 시)
          # sed -i "s|image: .*/<APP_NAME>:.*|image: ${{ env.PROD_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${IMAGE_TAG}|" deployment.yaml

      - name: Commit and push image tag update
        run: |
          cd gitops
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          IMAGE_TAG="${{ needs.build-and-push.outputs.image_tag }}"
          git add .
          git diff --staged --quiet || git commit -m "chore: update <APP_NAME> image to ${IMAGE_TAG}

          Source: ${{ github.repository }}@${{ github.sha }}
          Workflow: ${{ github.workflow }} #${{ github.run_number }}"
          git push
```

---

## 3. Access Key 방식 (레거시 참고)

### GitHub Secrets 설정

```
GitHub Repository > Settings > Secrets and variables > Actions

AWS_ACCESS_KEY_ID     = AKIA...
AWS_SECRET_ACCESS_KEY = xxxxxxxx
PROD_ACCOUNT_ID       = 222222222222
```

### Workflow (Access Key 방식)

```yaml
# .github/workflows/build-push-legacy.yaml (Access Key 방식 - 비권장)

name: Build and Push (Access Key)

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (Access Key)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2
        # ⚠ Access Key 방식: OIDC 방식으로 마이그레이션 권장

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
        with:
          registries: ${{ secrets.PROD_ACCOUNT_ID }}

      - name: Build and push
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/<APP_NAME>:$IMAGE_TAG .
          docker push $ECR_REGISTRY/<APP_NAME>:$IMAGE_TAG
```

---

## 4. 멀티 환경 매트릭스 빌드

```yaml
# .github/workflows/build-matrix.yaml
# staging, prod 각각 다른 ECR에 push

name: Multi-Environment Build

on:
  push:
    branches: [main, staging]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - branch: main
            env: prod
            account_id_secret: PROD_ACCOUNT_ID
            role_arn_secret: PROD_AWS_ROLE_ARN
          - branch: staging
            env: staging
            account_id_secret: STAGING_ACCOUNT_ID
            role_arn_secret: STAGING_AWS_ROLE_ARN

    # 현재 브랜치에 해당하는 환경만 빌드
    if: github.ref_name == matrix.branch

    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[matrix.role_arn_secret] }}
          aws-region: ap-northeast-2

      - name: Build and push to ${{ matrix.env }} ECR
        run: |
          GIT_SHA=$(git rev-parse --short HEAD)
          IMAGE_TAG="${{ matrix.env }}-${GIT_SHA}"
          ECR_URI="${{ secrets[matrix.account_id_secret] }}.dkr.ecr.ap-northeast-2.amazonaws.com/<APP_NAME>"

          aws ecr get-login-password --region ap-northeast-2 | \
            docker login --username AWS --password-stdin "${ECR_URI}"

          docker build -t "${ECR_URI}:${IMAGE_TAG}" .
          docker push "${ECR_URI}:${IMAGE_TAG}"

          echo "Pushed: ${ECR_URI}:${IMAGE_TAG}"
```

---

## 5. 이미지 다이제스트 추출 및 검증

```yaml
# 빌드 후 이미지 다이제스트를 GitOps repo에 기록하는 패턴
- name: Extract and verify image digest
  run: |
    IMAGE_URI="${{ env.PROD_ACCOUNT_ID }}.dkr.ecr.ap-northeast-2.amazonaws.com/<APP_NAME>:${{ env.IMAGE_TAG }}"

    # ECR에서 다이제스트 조회
    DIGEST=$(aws ecr describe-images \
      --repository-name <APP_NAME> \
      --image-ids imageTag=${{ env.IMAGE_TAG }} \
      --query 'imageDetails[0].imageDigest' \
      --output text \
      --region ap-northeast-2)

    echo "Image: ${IMAGE_URI}"
    echo "Digest: ${DIGEST}"
    echo "IMAGE_DIGEST=${DIGEST}" >> $GITHUB_ENV

    # ArgoCD에서 다이제스트로 이미지 고정 가능 (재현성 보장)
    # image: <ECR_URI>@sha256:xxxx
```

---

## 6. 트러블슈팅

### 증상 1: OIDC "Not authorized to perform sts:AssumeRoleWithWebIdentity"

```
Error: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

**원인**: IAM Role의 Trust Policy에서 `sub` 조건이 현재 워크플로우 컨텍스트와 불일치.

**해결 방법**:
```bash
# 실제 sub 값 확인 (워크플로우 로그에서)
# 또는 Trust Policy 조건을 StringLike + 와일드카드로 완화
# "token.actions.githubusercontent.com:sub": "repo:<ORG>/<REPO>:*"
```

---

### 증상 2: ECR login "no basic auth credentials"

```
Error response from daemon: no basic auth credentials
```

**원인**: `amazon-ecr-login` 액션의 `registries` 파라미터가 없거나 계정 ID 오타.

**해결 방법**:
```yaml
- name: Login to Amazon ECR
  uses: aws-actions/amazon-ecr-login@v2
  with:
    registries: "222222222222"  # 크로스 계정 ECR의 계정 ID (문자열)
```

---

### 증상 3: GitOps repo push 실패 "refusing to allow a GitHub App to create or update workflow files"

**원인**: `GITOPS_TOKEN`으로 사용한 PAT에 `workflow` 스코프 미포함.

**해결 방법**:
```
GitHub > Settings > Developer settings > Personal access tokens
- repo: Full control of private repositories
- workflow: Update GitHub Action workflows
```

---

## 7. 모니터링 및 알람

### 워크플로우 실패 알림 (Slack)

```yaml
# 워크플로우 마지막에 실패 시 Slack 알림
- name: Notify Slack on failure
  if: failure()
  uses: slackapi/slack-github-action@v1.26.0
  with:
    payload: |
      {
        "text": "❌ CI 빌드 실패",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "*빌드 실패*: `${{ github.repository }}`\n브랜치: `${{ github.ref_name }}`\n워크플로우: <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|#${{ github.run_number }}>"
            }
          }
        ]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

---

## 8. TIP

**TIP 1**: `actions/checkout@v4`의 `fetch-depth: 0` 옵션으로 전체 git 히스토리를 가져오면 `git log`, `git describe` 등 히스토리 기반 버전 생성 가능. 기본값(1)은 shallow clone.

**TIP 2**: `docker/build-push-action`의 캐시(`cache-from/cache-to: type=gha`) 활용 시 동일 레이어 재빌드 방지. 대규모 이미지에서 빌드 시간 50% 이상 단축 사례 있음.

**TIP 3**: PR 빌드는 push 없이 빌드만 수행(`push: false`)하여 ECR 과금 및 불필요한 이미지 누적 방지. `pr-{번호}-{sha}` 태그로 PR별 이미지 식별.

**관련 문서**
- [멀티 계정 CI/CD 아키텍처](./cicd-multi-account-architecture.md)
- [이미지 태깅 전략](./cicd-image-tagging-strategy.md)
- [Secrets 관리](../05-security/secrets-in-pipeline.md)
