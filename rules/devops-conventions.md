# DevOps 코드 컨벤션 (DevOps Code Conventions)

GitHub Actions YAML, Helm, Terraform 코드 작성 시 따라야 할 규칙.

---

## 1. GitHub Actions YAML 컨벤션

### 워크플로우 파일 네이밍

```
.github/workflows/
├── build-push.yaml          # 빌드 및 ECR push
├── terraform-plan.yaml      # Terraform plan (PR)
├── terraform-apply.yaml     # Terraform apply (merge)
├── secret-scan.yaml         # 비밀 스캔
└── image-scan.yaml          # 취약점 스캔
```

### 필수 항목

```yaml
# 모든 워크플로우에 포함
name: "<명확한 워크플로우 이름>"  # 한국어 또는 영어

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
    paths:           # 변경된 경로에만 트리거 (불필요한 실행 방지)
      - 'src/**'
      - 'Dockerfile'

env:
  AWS_REGION: ap-northeast-2  # 리전 상수화

permissions:         # 최소 권한 명시 (기본값 read-all 대신)
  id-token: write
  contents: read
```

### 명명 규칙

```yaml
jobs:
  build-and-push:        # 소문자 + 하이픈
    name: "Build and Push Docker Image"  # 사람이 읽는 이름

    steps:
      - name: "Checkout source code"    # 각 step에 name 필수
      - name: "Configure AWS credentials (OIDC)"
      - name: "Build and push Docker image"
```

### secrets 참조 방식

```yaml
# ✅ 권장: 환경 변수로 받기
env:
  MY_SECRET: ${{ secrets.MY_SECRET }}

# ❌ 비권장: 인라인 직접 사용 (디버그 시 로그 노출 위험)
run: echo ${{ secrets.MY_SECRET }}
```

---

## 2. Terraform 컨벤션

### 파일 구조

```
terraform/{환경}/
├── backend.tf       # Remote State 설정
├── provider.tf      # AWS Provider 버전 고정
├── main.tf          # 주요 리소스 (또는 모듈 호출)
├── variables.tf     # 변수 선언
├── outputs.tf       # 출력값
└── terraform.tfvars # 변수 값 (민감 정보 제외)
```

### Provider 버전 고정

```hcl
terraform {
  required_version = ">= 1.6.0"  # 최소 버전 명시

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # 마이너 버전 허용, 메이저 고정
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}
```

### 리소스 네이밍

```hcl
# {서비스}-{목적}-{환경} 패턴
resource "aws_iam_role" "argocd_irsa" {}          # 소문자 + 언더스코어
resource "aws_iam_policy" "cicd_ecr_push" {}
resource "aws_ecr_repository" "app" {}

# 이름 속성: {prefix}-{서비스}-{목적} 패턴
name = "prod-argocd-irsa"
name = "cicd-ecr-push-policy"
```

### Tags 필수 항목

```hcl
# 모든 리소스에 필수 태그
tags = {
  Environment = "prod"           # dev / staging / prod
  Application = "my-app"         # 애플리케이션 이름
  ManagedBy   = "terraform"      # terraform / manual
  Team        = "platform"       # 담당 팀
}
```

### 변수 선언 형식

```hcl
variable "cluster_name" {
  description = "EKS 클러스터 이름"
  type        = string
  # default 없음 = 필수 입력값

  validation {
    condition     = length(var.cluster_name) > 0
    error_message = "cluster_name은 빈 문자열일 수 없음."
  }
}

variable "enable_irsa" {
  description = "ArgoCD IRSA 활성화 여부"
  type        = bool
  default     = true
}
```

### 민감 정보 처리

```hcl
# output에 sensitive = true 설정
output "cicd_user_secret_key" {
  value     = aws_iam_access_key.cicd_user.secret
  sensitive = true  # terraform output 시 마스킹
}

# 민감 변수
variable "db_password" {
  description = "데이터베이스 비밀번호"
  type        = string
  sensitive   = true  # plan 출력에서 마스킹
}
```

---

## 3. Helm Chart 컨벤션

### Chart.yaml 형식

```yaml
apiVersion: v2
name: my-app                       # 소문자 + 하이픈
description: "My Application"      # 한국어 또는 영어
type: application                  # application 또는 library
version: 1.0.0                     # Chart SemVer (Chart 구조 변경 시 업데이트)
appVersion: "abc1234"              # 앱 이미지 태그 (CI가 업데이트)
```

### values.yaml 네이밍

```yaml
# camelCase 사용 (Kubernetes 컨벤션)
replicaCount: 1
image:
  repository: ""
  tag: ""
  pullPolicy: IfNotPresent

# 복수형: 여러 항목
resources: {}
nodeSelector: {}
tolerations: []
affinity: {}

# 활성화 플래그: enabled
autoscaling:
  enabled: false
  minReplicas: 1

ingress:
  enabled: false
```

### Template 헬퍼 (\_helpers.tpl)

```
# 필수 헬퍼 함수
{{- define "my-app.name" }}
{{- define "my-app.fullname" }}
{{- define "my-app.labels" }}           # 공통 레이블
{{- define "my-app.selectorLabels" }}   # 셀렉터 레이블
{{- define "my-app.serviceAccountName" }}
```

### Helm 버전 관리 전략

```
Chart 버전 올리는 기준:
- MAJOR: breaking change (values 구조 변경, 리소스 삭제)
- MINOR: 신규 기능 추가 (새 values 항목, 옵션 추가)
- PATCH: 버그 수정, 보안 패치

appVersion:
- CI 파이프라인에서 git-sha로 자동 업데이트
- 수동 변경 금지
```

---

## 4. Kubernetes YAML 컨벤션

### 필수 레이블

```yaml
metadata:
  labels:
    app.kubernetes.io/name: my-app
    app.kubernetes.io/instance: my-app-prod
    app.kubernetes.io/version: abc1234    # 이미지 태그
    app.kubernetes.io/component: backend  # frontend / backend / worker
    app.kubernetes.io/managed-by: argocd  # argocd / helm / kubectl
```

### Resource Requests/Limits 필수

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
# requests 없으면 BestEffort QoS → 노드 압박 시 가장 먼저 종료됨
```

### Security Context 권장

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1001
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

---

## 5. 코드 리뷰 체크리스트

### GitHub Actions PR

- [ ] 최소 permissions 설정 (write 권한 최소화)
- [ ] OIDC 방식 AWS 인증 (Access Key 없음)
- [ ] 비밀값 env 변수로 분리 (인라인 사용 금지)
- [ ] 실패 시 알림 step 포함
- [ ] PR과 main 브랜치 동작 구분

### Terraform PR

- [ ] 모든 리소스에 태그 포함
- [ ] 민감 출력값 sensitive = true
- [ ] 리소스 삭제는 `prevent_destroy` 검토
- [ ] State 변경 최소화 (import/mv 우선)
- [ ] plan 결과 PR comment 확인

### Helm PR

- [ ] Chart 버전 업데이트
- [ ] values.yaml default 값 타당성 검토
- [ ] `helm lint` 통과
- [ ] `helm template` 렌더링 확인
