# Jenkins + ArgoCD CI/CD 구축 가이드 (Jenkins ArgoCD CI/CD)

## 1. 개요

Jenkins와 ArgoCD를 함께 사용할 때는 Jenkins를 CI(Continuous Integration) 도구로, ArgoCD를 CD(Continuous Delivery) 도구로 분리하는 구조가 운영 환경에 적합함. Jenkins가 소스 코드 테스트, 이미지 빌드, 보안 검사, 이미지 push, GitOps 리포지토리 변경까지 담당하고, ArgoCD는 GitOps 리포지토리의 선언적 매니페스트를 클러스터에 동기화함.

**핵심 원칙**

| 원칙 | 설명 |
|------|------|
| 책임 분리 | Jenkins는 빌드와 검증, ArgoCD는 배포와 동기화 담당 |
| GitOps 우선 | Jenkins가 Kubernetes API에 직접 배포하지 않고 GitOps 리포지토리만 변경 |
| 이미지 불변성 | `latest` 태그 금지, Git SHA 또는 SemVer 태그 사용 |
| 환경 분리 | dev/staging/prod overlay를 분리하고 승인 정책도 환경별 분리 |
| 최소 권한 | Jenkins는 ECR push와 GitOps repo 쓰기 권한만 보유 |

**Jenkins 단독 배포와 Jenkins + ArgoCD 비교**

| 항목 | Jenkins 단독 배포 | Jenkins + ArgoCD GitOps |
|------|------------------|--------------------------|
| 배포 방식 | Jenkins가 `kubectl apply` 또는 Helm 실행 | ArgoCD가 Git 변경 감지 후 sync |
| 배포 이력 | Jenkins job 이력 중심 | Git commit + ArgoCD sync history |
| 롤백 | Jenkins job 재실행 또는 수동 Helm rollback | Git revert 또는 ArgoCD rollback |
| 클러스터 권한 | Jenkins에 Kubernetes 배포 권한 필요 | ArgoCD에만 Kubernetes 배포 권한 부여 |
| 운영 안정성 | Jenkins 장애 시 배포 상태 추적 어려움 | Git desired state 기준으로 지속 조정 |

**전체 플로우**

```text
[개발자]
    │
    │ 1. git push 또는 PR merge
    ▼
[Git Repository - Source]
    │
    │ 2. webhook
    ▼
[Jenkins]
    │
    ├─ 3. checkout
    ├─ 4. unit test / lint / SAST
    ├─ 5. docker build
    ├─ 6. image scan
    ├─ 7. ECR push
    └─ 8. GitOps repo image tag 업데이트
            │
            ▼
[Git Repository - GitOps]
    │
    │ 9. ArgoCD polling 또는 webhook
    ▼
[ArgoCD]
    │
    ├─ 10. desired state와 live state 비교
    ├─ 11. sync 실행
    └─ 12. health 상태 확인
            │
            ▼
[EKS Cluster]
    │
    └─ 13. Deployment rolling update
```

---

## 2. 설명

### 2-1. 권장 아키텍처

Jenkins가 애플리케이션 소스 리포지토리를 기준으로 이미지를 만들고, GitOps 리포지토리의 환경별 overlay에 이미지 태그를 반영함. ArgoCD Application은 각 환경 overlay를 바라보도록 구성함.

```text
source-repo/
├── Jenkinsfile
├── Dockerfile
├── src/
└── tests/

gitops-repo/
├── apps/
│   └── my-app/
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── kustomization.yaml
│       └── overlays/
│           ├── dev/
│           │   └── kustomization.yaml
│           ├── staging/
│           │   └── kustomization.yaml
│           └── prod/
│               └── kustomization.yaml
└── argocd/
    └── applications/
        ├── my-app-dev.yaml
        ├── my-app-staging.yaml
        └── my-app-prod.yaml
```

**도구별 책임**

| 도구 | 담당 | 보유 권한 |
|------|------|-----------|
| Jenkins | 테스트, 빌드, 이미지 스캔, ECR push, GitOps repo 변경 | ECR push, GitOps repo write |
| ArgoCD | GitOps repo 감시, Kubernetes 리소스 sync, drift 복구 | Kubernetes apply, get, list, watch |
| ECR | 컨테이너 이미지 저장, 이미지 스캔 | Jenkins push, EKS pull |
| GitOps repo | 배포 상태의 Single Source of Truth | PR 리뷰, branch protection |

### 2-2. Jenkins 인증 방식

Jenkins가 어디에서 실행되는지에 따라 AWS 인증 방식을 다르게 선택함.

| 실행 위치 | 권장 인증 방식 | 설명 |
|-----------|----------------|------|
| EKS 내부 Jenkins | IRSA(IAM Roles for Service Accounts) | ServiceAccount에 IAM Role 연결, 장기 Access Key 불필요 |
| EC2 Jenkins | Instance Profile | EC2 Role로 ECR push 권한 부여 |
| 외부 VM Jenkins | OIDC 또는 단기 STS | 장기 Access Key 대신 federation 구성 권장 |
| 레거시 Jenkins | Jenkins Credentials | Access Key 사용 시 로테이션과 최소 권한 필수 |

**EKS 내부 Jenkins IRSA 예시**

```yaml
# jenkins-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: cicd
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<CICD_ACCOUNT_ID>:role/jenkins-irsa-ecr-gitops
```

```hcl
# terraform/cicd-account/jenkins-irsa-policy.tf
resource "aws_iam_policy" "jenkins_ecr_push" {
  name        = "jenkins-ecr-push-policy"
  description = "Jenkins ECR push 최소 권한"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ECRGetAuthToken"
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken"
        ]
        Resource = "*"
      },
      {
        Sid    = "ECRPush"
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
          "arn:aws:ecr:ap-northeast-2:<PROD_ACCOUNT_ID>:repository/<APP_NAME>",
          "arn:aws:ecr:ap-northeast-2:<STAGING_ACCOUNT_ID>:repository/<APP_NAME>"
        ]
      }
    ]
  })
}
```

### 2-3. Jenkinsfile 예시

Jenkins Pipeline은 PR에서는 테스트와 이미지 빌드까지만 수행하고, `main` merge 이후에만 이미지 push와 GitOps repo 변경을 수행함. 운영 배포는 GitOps repo PR 승인 후 ArgoCD가 처리함.

예시는 AWS CLI, Docker CLI, Git, Kustomize, Trivy가 포함된 전용 Jenkins agent 이미지를 사용함. 실제 운영에서는 해당 이미지를 내부 ECR에 저장하고 취약점 스캔과 태그 고정을 적용함.

**이미지 빌드 방식 선택**

| 방식 | 장점 | 주의점 |
|------|------|--------|
| Docker socket mount | 기존 Docker build와 호환성 높음 | Jenkins agent가 노드 Docker 권한을 공유하므로 격리 약함 |
| Kaniko | Docker daemon 불필요 | 일부 Dockerfile 동작 차이 확인 필요 |
| BuildKit rootless | 캐시와 보안 균형 우수 | Jenkins agent 이미지와 캐시 저장소 구성 필요 |

```groovy
// Jenkinsfile
pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
    - name: cicd
      image: <CICD_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/jenkins-cicd-tools:2026.05.31
      command: ["cat"]
      tty: true
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock
  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
"""
    }
  }

  environment {
    AWS_REGION = 'ap-northeast-2'
    APP_NAME = '<APP_NAME>'
    ECR_REGISTRY = '<PROD_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com'
    ECR_REPOSITORY = '<APP_NAME>'
    GITOPS_REPO_URL = 'git@github.com:<YOUR_ORG>/<GITOPS_REPO>.git'
    TARGET_ENV = 'staging'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.IMAGE_TAG = env.BRANCH_NAME == 'main' ? env.GIT_SHA : "pr-${env.CHANGE_ID}-${env.GIT_SHA}"
          env.IMAGE_URI = "${env.ECR_REGISTRY}/${env.ECR_REPOSITORY}:${env.IMAGE_TAG}"
        }
      }
    }

    stage('Test') {
      steps {
        sh '''
          set -e
          # 프로젝트에 맞는 테스트 명령어로 교체
          ./gradlew test
        '''
      }
    }

    stage('Build Image') {
      steps {
        container('cicd') {
          sh '''
            set -e
            docker build \
              --build-arg GIT_SHA=${GIT_SHA} \
              --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
              -t ${IMAGE_URI} .
          '''
        }
      }
    }

    stage('Image Scan') {
      steps {
        container('cicd') {
          sh '''
            set -e
            trivy image \
              --severity HIGH,CRITICAL \
              --exit-code 1 \
              ${IMAGE_URI}
          '''
        }
      }
    }

    stage('Push Image') {
      when {
        branch 'main'
      }
      steps {
        container('cicd') {
          sh '''
            set -e
            aws ecr get-login-password --region ${AWS_REGION} \
              | docker login --username AWS --password-stdin ${ECR_REGISTRY}
            docker push ${IMAGE_URI}
          '''
        }
      }
    }

    stage('Update GitOps') {
      when {
        branch 'main'
      }
      steps {
        container('cicd') {
          sshagent(credentials: ['gitops-repo-deploy-key']) {
            sh '''
              set -e
              rm -rf gitops
              git clone ${GITOPS_REPO_URL} gitops
              cd gitops/apps/${APP_NAME}/overlays/${TARGET_ENV}

              kustomize edit set image ${APP_NAME}=${IMAGE_URI}

              cd ../../../../
              git config user.name "jenkins"
              git config user.email "jenkins@<YOUR_DOMAIN>"
              git checkout -b deploy/${APP_NAME}-${TARGET_ENV}-${IMAGE_TAG}
              git add apps/${APP_NAME}/overlays/${TARGET_ENV}/kustomization.yaml
              git commit -m "chore: deploy ${APP_NAME} ${TARGET_ENV} ${IMAGE_TAG}"
              git push origin deploy/${APP_NAME}-${TARGET_ENV}-${IMAGE_TAG}
            '''
          }
        }
      }
    }
  }

  post {
    always {
      cleanWs()
    }
  }
}
```

### 2-4. ArgoCD Application 예시

환경별 Application을 분리하면 sync 정책, 승인 방식, 알림 정책을 독립적으로 운영하기 쉬움.

```yaml
# argocd/applications/my-app-staging.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-staging
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:<YOUR_ORG>/<GITOPS_REPO>.git
    targetRevision: main
    path: apps/my-app/overlays/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
```

**환경별 sync 정책**

| 환경 | Jenkins 동작 | ArgoCD sync 정책 | 승인 방식 |
|------|--------------|------------------|-----------|
| dev | image push 후 GitOps direct push | auto-sync | 자동 |
| staging | image push 후 GitOps PR 생성 | auto-sync 또는 manual-sync | PR 리뷰 |
| prod | 검증된 image tag를 prod overlay로 승격 | manual-sync 권장 | PR 리뷰 + 수동 sync |

### 2-5. 운영에서 지켜야 할 기준

| 항목 | 기준 |
|------|------|
| 이미지 태그 | `<git-sha>`, `staging-<git-sha>`, `v<semver>` 사용 |
| Jenkins 권한 | ECR push와 GitOps repo write만 허용 |
| ArgoCD 권한 | 배포 namespace 중심 RBAC 적용 |
| Secret | Jenkins Credentials 또는 External Secrets 사용, Git 저장 금지 |
| 배포 승인 | prod는 GitOps PR 승인과 ArgoCD RBAC로 이중 통제 |
| 롤백 | Git revert를 기본 절차로 사용 |

---

## 3. 트러블슈팅

### 증상 1: Jenkins에서 ECR 로그인 실패

```text
An error occurred (AccessDeniedException) when calling the GetAuthorizationToken operation
```

**원인**: Jenkins IAM Role에 `ecr:GetAuthorizationToken` 권한이 없거나 IRSA ServiceAccount annotation이 잘못됨.

**해결 방법**:

```bash
# Jenkins pod의 ServiceAccount 확인
kubectl get pod -n cicd <JENKINS_AGENT_POD> -o jsonpath='{.spec.serviceAccountName}'

# ServiceAccount annotation 확인
kubectl get sa -n cicd jenkins -o yaml

# EKS 내부에서 AWS identity 확인
kubectl exec -n cicd <JENKINS_AGENT_POD> -c aws -- aws sts get-caller-identity
```

### 증상 2: GitOps repo push 실패

```text
ERROR: Permission to <YOUR_ORG>/<GITOPS_REPO>.git denied to jenkins.
fatal: Could not read from remote repository.
```

**원인**: Jenkins에 등록된 deploy key 또는 GitHub App 권한이 GitOps 리포지토리 쓰기 권한을 갖지 않음.

**해결 방법**:

```bash
# Jenkins credential ID 확인 후 GitOps repo deploy key 재등록
# GitHub Repository > Settings > Deploy keys > Allow write access 체크
ssh -T git@github.com
```

### 증상 3: ArgoCD가 변경된 이미지 태그를 반영하지 않음

```text
Application status: Synced
Pod image is still old tag
```

**원인**: Jenkins가 GitOps repo의 다른 path를 수정했거나 ArgoCD Application의 `spec.source.path`가 실제 overlay path와 다름.

**해결 방법**:

```bash
# ArgoCD Application path 확인
argocd app get my-app-staging

# GitOps repo에서 실제 변경 파일 확인
git log --oneline -- apps/my-app/overlays/staging/kustomization.yaml

# 수동 diff 확인
argocd app diff my-app-staging
```

### 증상 4: ArgoCD sync는 성공했지만 Pod가 새 이미지로 뜨지 않음

```text
Failed to pull image "<ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/<APP_NAME>:<IMAGE_TAG>"
```

**원인**: EKS 노드 Role 또는 Pod Identity/IRSA에 ECR pull 권한이 없거나 이미지 태그가 ECR에 존재하지 않음.

**해결 방법**:

```bash
# ECR 이미지 태그 존재 확인
aws ecr describe-images \
  --region ap-northeast-2 \
  --repository-name <APP_NAME> \
  --image-ids imageTag=<IMAGE_TAG>

# Pod 이벤트 확인
kubectl describe pod -n my-app <POD_NAME>
```

---

## 4. 모니터링 및 알람

Jenkins와 ArgoCD는 각각 다른 실패 지점을 담당하므로 모니터링도 분리함. Jenkins는 빌드 실패, 테스트 실패, 이미지 push 실패를 감시하고, ArgoCD는 sync 실패, health degraded, drift 발생을 감시함.

**모니터링 대상**

| 영역 | 지표/이벤트 | 알람 기준 |
|------|-------------|-----------|
| Jenkins job | build result, duration, queue time | main 브랜치 build failure 발생 |
| Jenkins agent | pod pending, container restart | agent pod pending 5분 이상 |
| ECR push | image push 실패, scan finding | CRITICAL 취약점 발견 |
| GitOps repo | PR 생성 실패, push 실패 | GitOps update stage 실패 |
| ArgoCD sync | `argocd_app_sync_status` | `OutOfSync` 지속 |
| ArgoCD health | `argocd_app_health_status` | `Degraded` 발생 |
| Kubernetes rollout | Deployment unavailable replicas | 배포 후 unavailable 5분 이상 |

**PrometheusRule 예시**

```yaml
# monitoring/argocd-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: argocd-alerts
  namespace: monitoring
spec:
  groups:
    - name: argocd.rules
      rules:
        - alert: ArgoCDApplicationOutOfSync
          expr: argocd_app_info{sync_status!="Synced"} == 1
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "ArgoCD Application OutOfSync"
            description: "{{ $labels.name }} application is not synced"

        - alert: ArgoCDApplicationDegraded
          expr: argocd_app_info{health_status="Degraded"} == 1
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "ArgoCD Application Degraded"
            description: "{{ $labels.name }} application health is degraded"
```

**Jenkins에서 남겨야 하는 빌드 메타데이터**

| 메타데이터 | 목적 |
|------------|------|
| source commit SHA | 장애 발생 시 소스 변경 추적 |
| image URI | Kubernetes에 배포된 이미지와 CI 산출물 연결 |
| build number | Jenkins job 로그 추적 |
| GitOps commit SHA | 배포 변경 이력 추적 |
| ArgoCD application name | sync 상태와 배포 결과 연결 |

---

## 5. TIP

### 현장 팁

| 상황 | 권장 방식 |
|------|-----------|
| Jenkins에서 바로 배포하고 싶은 경우 | dev 환경만 허용, staging/prod는 GitOps repo 변경으로 제한 |
| prod 자동 배포가 부담되는 경우 | prod Application은 auto-sync 비활성화, PR 승인 후 수동 sync |
| 장애 롤백이 필요한 경우 | GitOps repo에서 이전 image tag로 revert 후 ArgoCD sync |
| 여러 서비스가 있는 경우 | ApplicationSet 또는 App of Apps로 Application 관리 |
| Secret 배포가 필요한 경우 | Sealed Secrets, External Secrets Operator, SOPS 중 하나로 표준화 |

### 체크리스트

| 항목 | 확인 |
|------|------|
| Jenkins가 Kubernetes 배포 권한을 직접 갖지 않음 | 예 |
| Jenkins 이미지 태그가 Git SHA 기반임 | 예 |
| GitOps repo에 branch protection이 적용됨 | 예 |
| ArgoCD prod sync 권한이 제한됨 | 예 |
| 롤백 절차가 Git revert 기준으로 문서화됨 | 예 |
| Jenkins/ArgoCD 실패 알람이 Slack 또는 PagerDuty로 전달됨 | 예 |

**관련 문서**
- GitOps 워크플로우: `../02-gitops/gitops-workflow.md`
- ArgoCD EKS 배포 및 IRSA 인증: `../02-gitops/argocd-eks-deployment.md`
- ArgoCD App of Apps: `../02-gitops/argocd-app-of-apps.md`
- 이미지 태깅 전략: `cicd-image-tagging-strategy.md`
- 멀티 계정 CI/CD 아키텍처: `cicd-multi-account-architecture.md`
- DevSecOps 이미지 스캔: `../05-security/devsecops-image-scan.md`
