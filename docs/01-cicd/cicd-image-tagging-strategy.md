# 이미지 태그 전략 (Container Image Tagging Strategy)

## 1. 개요

Docker 이미지 태그는 배포 추적성, 롤백 능력, 보안 감사의 핵심. `latest` 태그는 재현성과 추적성을 모두 해치므로 운영 환경에서 사용 금지. git-sha 기반 태깅이 현장 표준.

**`latest` 태그 사용 금지 이유**
- 어떤 코드가 배포되었는지 추적 불가
- ArgoCD 등 GitOps 도구가 변경 감지 불가 (항상 최신이라고 인식)
- 롤백 시 이전 이미지를 특정할 수 없음
- 동일 태그로 다른 이미지가 덮어쓰일 수 있음

---

## 2. 태그 전략 비교

| 전략 | 예시 | 장점 | 단점 |
|------|------|------|------|
| `latest` | `app:latest` | 간단 | 추적 불가, 재현성 없음, **운영 금지** |
| git-sha (short) | `app:abc1234` | 커밋 추적 가능, 간결 | 7자리 충돌 가능성 (매우 낮음) |
| git-sha (full) | `app:abc1234abc1234abc1234` | 충돌 없음 | 태그가 길어 가독성 저하 |
| env-sha | `app:prod-abc1234` | 환경 구분 명확 | 태그 관리 복잡도 증가 |
| semver | `app:v1.2.3` | 릴리스 주기 명확 | 수동 버전 관리 필요 |
| semver+sha | `app:v1.2.3-abc1234` | 릴리스+커밋 모두 추적 | 태그 생성 로직 복잡 |
| timestamp | `app:20240101-abc1234` | 빌드 시점 파악 | 시간대 혼선 가능 |

**권장**: git-sha 또는 semver+sha 조합.

---

## 3. git-sha 태깅 구현

### CI 파이프라인에서 태그 생성

```bash
#!/bin/bash
# scripts/generate-image-tag.sh

set -e

# Short SHA (7자리) - 대부분의 케이스에서 충분
GIT_SHA=$(git rev-parse --short HEAD)

# Full SHA - 충돌 가능성 완전 제거
GIT_SHA_FULL=$(git rev-parse HEAD)

# 환경 포함 태그
DEPLOY_ENV=${DEPLOY_ENV:-prod}  # 환경 변수로 주입
IMAGE_TAG_WITH_ENV="${DEPLOY_ENV}-${GIT_SHA}"

# Semantic version + SHA (태그가 있을 경우)
if git describe --tags --exact-match 2>/dev/null; then
  GIT_TAG=$(git describe --tags --exact-match)
  IMAGE_TAG_SEMVER="${GIT_TAG}-${GIT_SHA}"
else
  IMAGE_TAG_SEMVER="${GIT_SHA}"
fi

echo "GIT_SHA=${GIT_SHA}"
echo "IMAGE_TAG=${GIT_SHA}"
echo "IMAGE_TAG_ENV=${IMAGE_TAG_WITH_ENV}"
echo "IMAGE_TAG_SEMVER=${IMAGE_TAG_SEMVER}"

# GitHub Actions 환경 변수로 export
if [ -n "$GITHUB_ENV" ]; then
  echo "GIT_SHA=${GIT_SHA}" >> $GITHUB_ENV
  echo "IMAGE_TAG=${GIT_SHA}" >> $GITHUB_ENV
fi
```

### GitHub Actions에서 태그 생성

```yaml
- name: Generate image tag
  id: tag
  run: |
    GIT_SHA=$(git rev-parse --short HEAD)
    GIT_SHA_FULL=$(git rev-parse HEAD)

    # PR 빌드와 main 빌드 구분
    if [ "${{ github.event_name }}" == "pull_request" ]; then
      IMAGE_TAG="pr-${{ github.event.pull_request.number }}-${GIT_SHA}"
    elif [ "${{ github.ref_name }}" == "main" ]; then
      IMAGE_TAG="${GIT_SHA}"
    else
      BRANCH="${{ github.ref_name }}"
      IMAGE_TAG="${BRANCH//\//-}-${GIT_SHA}"  # branch/name → branch-name
    fi

    echo "image_tag=${IMAGE_TAG}" >> $GITHUB_OUTPUT
    echo "git_sha=${GIT_SHA}" >> $GITHUB_OUTPUT
    echo "git_sha_full=${GIT_SHA_FULL}" >> $GITHUB_OUTPUT
```

---

## 4. ECR Lifecycle Policy 연계

이미지 태그 전략과 Lifecycle Policy를 연계하여 ECR 스토리지 비용 관리.

### 기본 Lifecycle Policy

```hcl
# terraform/prod-account/ecr-lifecycle.tf

resource "aws_ecr_lifecycle_policy" "app" {
  repository = aws_ecr_repository.app.name

  policy = jsonencode({
    rules = [
      {
        # PR 빌드 이미지는 7일 후 삭제
        rulePriority = 1
        description  = "PR 빌드 이미지 7일 보관"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["pr-"]
          countType     = "sinceImagePushed"
          countUnit     = "days"
          countNumber   = 7
        }
        action = {
          type = "expire"
        }
      },
      {
        # staging 태그 이미지는 30일 보관
        rulePriority = 2
        description  = "Staging 이미지 30일 보관"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["staging-"]
          countType     = "sinceImagePushed"
          countUnit     = "days"
          countNumber   = 30
        }
        action = {
          type = "expire"
        }
      },
      {
        # prod 이미지는 최근 50개 보관 (롤백을 위해 충분한 수량 유지)
        rulePriority = 3
        description  = "Prod 이미지 최근 50개 보관"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["prod-"]
          countType     = "imageCountMoreThan"
          countNumber   = 50
        }
        action = {
          type = "expire"
        }
      },
      {
        # 태그 없는 이미지 (빌드 실패 중간 레이어 등)는 1일 후 삭제
        rulePriority = 4
        description  = "Untagged 이미지 1일 보관"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 1
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}
```

### git-sha 태깅과 Lifecycle Policy 연계

```bash
# 태그 전략별 Lifecycle Policy 대응

# 태그 없이 sha만 사용하는 경우 → tagPrefixList 없이 countType으로 관리
# 예: app:abc1234 (prefix 없음)
{
  "rulePriority": 1,
  "description": "최근 100개 이미지만 보관",
  "selection": {
    "tagStatus": "any",
    "countType": "imageCountMoreThan",
    "countNumber": 100
  },
  "action": {"type": "expire"}
}
```

---

## 5. ArgoCD Image Updater 연동

ArgoCD Image Updater는 ECR의 새 이미지를 자동으로 감지하고 GitOps repo를 업데이트.

### Image Updater 설치

```bash
# Helm으로 설치
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd-image-updater argo/argocd-image-updater \
  --namespace argocd \
  --set config.registries[0].name=ECR \
  --set config.registries[0].prefix=<ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com \
  --set config.registries[0].api_url=https://<ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com \
  --set config.registries[0].credentials="ext:/scripts/ecr-login.sh" \
  --set config.registries[0].default=true
```

### ArgoCD Application에 Image Updater 어노테이션 추가

```yaml
# argocd-application.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  annotations:
    # Image Updater 활성화
    argocd-image-updater.argoproj.io/image-list: |
      app=<ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/<APP_NAME>

    # 업데이트 전략: semver (또는 latest, digest, name)
    argocd-image-updater.argoproj.io/app.update-strategy: semver

    # 특정 태그 패턴만 추적 (git-sha 방식 예시)
    # argocd-image-updater.argoproj.io/app.update-strategy: name
    # argocd-image-updater.argoproj.io/app.allow-tags: regexp:^prod-[a-f0-9]{7}$

    # GitOps repo에 커밋하도록 설정
    argocd-image-updater.argoproj.io/git-branch: main
    argocd-image-updater.argoproj.io/write-back-method: git

spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR_ORG>/<YOUR_GITOPS_REPO>
    targetRevision: main
    path: apps/my-app/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
```

### ECR 인증 스크립트 (IRSA 기반)

```bash
#!/bin/bash
# /scripts/ecr-login.sh
# ArgoCD Image Updater가 ECR 인증을 위해 호출

set -e

AWS_ACCOUNT_ID="<ACCOUNT_ID>"
AWS_REGION="ap-northeast-2"

# IRSA 환경에서 aws-cli가 자동으로 IRSA 토큰을 사용
TOKEN=$(aws ecr get-login-password --region ${AWS_REGION})
echo "AWS:${TOKEN}"
```

---

## 6. 멀티 아키텍처 이미지 태깅

ARM64(Graviton)/AMD64 혼용 환경에서의 멀티 아키텍처 이미지 관리.

```yaml
# GitHub Actions: 멀티 아키텍처 빌드
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push multi-arch image
  uses: docker/build-push-action@v5
  with:
    platforms: linux/amd64,linux/arm64
    push: true
    tags: |
      ${{ env.ECR_URI }}:${{ env.IMAGE_TAG }}
    # Docker manifest list로 자동 처리 (플랫폼별 이미지를 하나의 태그로)
```

---

## 7. 트러블슈팅

### 증상 1: 동일 git-sha인데 이미지 내용이 다름

**원인**: `--short` SHA 충돌 (7자리 기준 확률 매우 낮으나 대규모 모노레포에서 발생 가능) 또는 빌드 환경 차이 (non-deterministic build).

**해결 방법**:
```bash
# Full SHA 사용 (40자리, 충돌 없음)
IMAGE_TAG=$(git rev-parse HEAD)

# 또는 빌드 캐시 완전 비활성화 후 재빌드
docker build --no-cache -t app:$(git rev-parse --short HEAD) .
```

---

### 증상 2: ArgoCD Image Updater가 ECR 이미지를 감지 못함

**원인**: ECR 인증 실패 또는 IRSA 권한 부족.

**해결 방법**:
```bash
# Image Updater 로그 확인
kubectl logs -n argocd deployment/argocd-image-updater --tail=50

# ECR 접근 권한 확인
aws ecr describe-images \
  --repository-name <APP_NAME> \
  --region ap-northeast-2 \
  --query 'imageDetails[*].{Tag:imageTags,Digest:imageDigest,PushedAt:imagePushedAt}' \
  --output table
```

---

### 증상 3: ECR Lifecycle Policy 적용 후 현재 배포 중인 이미지가 삭제됨

**원인**: Lifecycle Policy가 태그를 기준으로 삭제하여 현재 배포 이미지도 제거.

**해결 방법**:
```bash
# 보호해야 할 이미지에 별도 태그 추가 (protected 등)
aws ecr put-image \
  --repository-name <APP_NAME> \
  --image-tag protected-$(git rev-parse --short HEAD) \
  --image-manifest "$(aws ecr batch-get-image \
    --repository-name <APP_NAME> \
    --image-ids imageTag=$(git rev-parse --short HEAD) \
    --query 'images[0].imageManifest' --output text)" \
  --region ap-northeast-2

# Lifecycle Policy에서 protected 태그 제외
# tagPrefixList에서 "protected" prefix는 만료 대상에서 제외
```

---

## 8. 모니터링 및 알람

### ECR 이미지 수 모니터링

```bash
# 리포지토리별 이미지 수 확인
aws ecr describe-images \
  --repository-name <APP_NAME> \
  --query 'length(imageDetails)' \
  --output text \
  --region ap-northeast-2

# 태그별 분류 확인
aws ecr describe-images \
  --repository-name <APP_NAME> \
  --query 'imageDetails[*].{Tags:imageTags,Size:imageSizeInBytes,PushedAt:imagePushedAt}' \
  --output table \
  --region ap-northeast-2
```

### ECR 스토리지 비용 알람

```hcl
# ECR 스토리지 비용 알람 (CloudWatch Billing)
resource "aws_cloudwatch_metric_alarm" "ecr_storage_cost" {
  alarm_name          = "ecr-storage-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "EstimatedCharges"
  namespace           = "AWS/Billing"
  period              = 86400  # 1일
  statistic           = "Maximum"
  threshold           = 50  # $50 초과 시 알람

  dimensions = {
    ServiceName = "AmazonECR"
    Currency    = "USD"
  }

  alarm_actions = [aws_sns_topic.billing_alerts.arn]
}
```

---

## 9. TIP

**TIP 1**: ECR에서 이미지 다이제스트(sha256)로 배포하면 태그 변경 없이도 이미지 내용 변경 감지 가능. ArgoCD에서 `image: <ECR_URI>@sha256:xxxx` 형식 지원.

**TIP 2**: 이미지 빌드 시 `org.opencontainers.image.revision` 레이블에 git SHA를 기록하면 런닝 컨테이너에서 소스 커밋 역추적 가능.

```dockerfile
LABEL org.opencontainers.image.revision="${GIT_SHA}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.version="${VERSION}"
```

**TIP 3**: 프로덕션 배포된 이미지는 최소 30일 보관 권장 (장애 발생 시 이전 버전 분석 필요). Lifecycle Policy의 `countNumber`를 일수가 아닌 개수 기준으로 설정하면 배포 빈도에 관계없이 일정 수량 유지.

**관련 문서**
- [멀티 계정 CI/CD 아키텍처](./cicd-multi-account-architecture.md)
- [GitHub Actions + ECR 파이프라인](./cicd-github-actions-ecr.md)
- [DevSecOps 이미지 스캔](../05-security/devsecops-image-scan.md)
