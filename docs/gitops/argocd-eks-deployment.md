# ArgoCD EKS 배포 및 IRSA 인증 (ArgoCD EKS Deployment with IRSA)

## 1. 개요

ArgoCD가 EKS 클러스터에 접근하는 3가지 방식 중 IRSA(IAM Roles for Service Accounts) 방식이 운영 환경 권장. kubeconfig Secret 방식은 자격 증명 관리 부담이 크고, in-cluster SA 방식은 단일 클러스터만 지원.

**전체 배포 플로우**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ArgoCD EKS 배포 전체 플로우                        │
└─────────────────────────────────────────────────────────────────────┘

[개발자]
    │ 1. 코드 커밋 (git push)
    ▼
[GitHub - 소스 리포지토리]
    │ 2. webhook → CI 트리거
    ▼
[GitHub Actions - CICD Account]
    │ 3. docker build
    │ 4. ECR push (CICD Account → Prod Account, cross-account)
    │ 5. GitOps repo image tag 업데이트 (PR merge 또는 direct push)
    ▼
[GitHub - GitOps 리포지토리]
    │  k8s-manifests/
    │  └── apps/my-app/overlays/prod/
    │      └── kustomization.yaml (image tag 변경됨)
    │
    │ 6. ArgoCD polling (기본 3분) 또는 webhook으로 즉시 감지
    ▼
[ArgoCD - Prod Account EKS 내부]
    │ 7. Git diff 계산: desired state vs live state
    │ 8. IRSA Role → sts:AssumeRoleWithWebIdentity
    │ 9. EKS API Server에 인증 (aws-auth 또는 Access Entry)
    │ 10. kubectl apply (Deployment, Service 등)
    ▼
[EKS Cluster - Prod Account]
    │ 11. Pod 롤링 업데이트
    │ 12. 새 이미지로 컨테이너 실행
    ▼
[배포 완료]
    │ 13. ArgoCD 동기화 상태: Synced
    │ 14. Slack 알림 (선택)
```

---

## 2. EKS 인증 방식 3가지 비교

| 방식 | 설명 | 장점 | 단점 | 권장 |
|------|------|------|------|------|
| In-cluster Service Account | ArgoCD가 설치된 클러스터만 관리 | 설정 간단 | 단일 클러스터만 지원 | 단일 클러스터 환경 |
| kubeconfig Secret | 각 클러스터 kubeconfig를 K8s Secret으로 저장 | 모든 클러스터 지원 | 자격 증명 만료/로테이션 부담 | 지양 |
| IRSA (권장) | IAM Role → EKS API 인증 | 자격 증명 자동 갱신, 최소 권한 | IRSA 설정 필요 | **멀티 클러스터 운영 시** |

### IRSA 방식 동작 원리

```
ArgoCD Pod
├── ServiceAccount: argocd-server
│   └── annotation: eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/argocd-irsa
│
│ EKS OIDC Webhook이 SA annotation 감지
│ → AWS STS에 AssumeRoleWithWebIdentity 요청
│ → 임시 자격 증명 (1시간, 자동 갱신)
▼
IAM Role: argocd-irsa
├── Trust Policy: EKS OIDC Provider ←→ argocd ServiceAccount
└── IAM Policy: eks:DescribeCluster, eks:ListClusters
    (EKS 클러스터 정보 조회 → kubeconfig 동적 생성)

ArgoCD → EKS API Server 인증
├── aws eks get-token --cluster-name <CLUSTER_NAME> (내부 동작)
├── aws-auth ConfigMap 또는 EKS Access Entry에 argocd-irsa Role 등록
└── kubectl apply (Deployment 등)
```

---

## 3. ArgoCD 설치 (EKS Helm)

### 3-1. Helm 설치

```bash
# ArgoCD Helm 리포지토리 추가
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# ArgoCD 설치 (argocd 네임스페이스)
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 6.x.x \
  --values argocd-values.yaml
```

### 3-2. argocd-values.yaml

```yaml
# argocd-values.yaml

global:
  # 멀티 계정 환경에서 리전 명시
  env:
    - name: AWS_DEFAULT_REGION
      value: ap-northeast-2

server:
  service:
    type: ClusterIP  # 내부 접근만 (ALB Ingress 또는 kubectl port-forward)

  ingress:
    enabled: true
    ingressClassName: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internal
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/certificate-arn: <ACM_CERT_ARN>
    hosts:
      - argocd.<INTERNAL_DOMAIN>
    tls:
      - hosts:
          - argocd.<INTERNAL_DOMAIN>

  # IRSA 설정: argocd-server ServiceAccount에 IAM Role 연결
  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::<PROD_ACCOUNT_ID>:role/argocd-irsa

configs:
  params:
    # HTTPS 강제 (ALB에서 TLS 종료 시)
    server.insecure: "true"

  # 클러스터 등록 (Secret 방식은 ArgoCD Secret으로 관리)
  # IRSA 방식은 argocd CLI 또는 Secret으로 추가 (섹션 5 참조)

  rbac:
    policy.default: role:readonly
    policy.csv: |
      p, role:devops, applications, *, */*, allow
      p, role:devops, clusters, get, *, allow
      p, role:devops, repositories, *, *, allow
      g, devops-team, role:devops

  # GitHub SSO (선택)
  oidc.config: |
    name: GitHub
    issuer: https://token.actions.githubusercontent.com
    clientID: <GITHUB_CLIENT_ID>
    clientSecret: $oidc.github.clientSecret

applicationSet:
  enabled: true

notifications:
  enabled: true
  # Slack 알림 설정은 별도 Secret
```

---

## 4. IRSA 설정 (Terraform)

### 4-1. ArgoCD IRSA IAM Role

```hcl
# terraform/prod-account/argocd-irsa.tf

# EKS 클러스터 OIDC Provider 조회
data "aws_eks_cluster" "prod" {
  name = var.cluster_name
}

data "aws_iam_openid_connect_provider" "eks" {
  url = data.aws_eks_cluster.prod.identity[0].oidc[0].issuer
}

# ArgoCD IRSA Role
resource "aws_iam_role" "argocd_irsa" {
  name = "argocd-irsa"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = data.aws_iam_openid_connect_provider.eks.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            # EKS OIDC Provider URL의 aud 조건
            "${replace(data.aws_iam_openid_connect_provider.eks.url, "https://", "")}:aud" = "sts.amazonaws.com"
            # argocd 네임스페이스의 argocd-server ServiceAccount만 assume 가능
            "${replace(data.aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:argocd:argocd-server"
          }
        }
      }
    ]
  })

  tags = {
    Purpose     = "ArgoCD EKS 접근 (IRSA)"
    ManagedBy   = "terraform"
    Environment = "prod"
  }
}

# ArgoCD가 EKS 클러스터 정보를 조회하고 토큰을 생성하는 권한
resource "aws_iam_policy" "argocd_eks" {
  name        = "argocd-eks-policy"
  description = "ArgoCD가 EKS 클러스터에 접근하기 위한 최소 권한"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "EKSDescribeCluster"
        Effect = "Allow"
        Action = [
          "eks:DescribeCluster",
          "eks:ListClusters"
        ]
        Resource = [
          # Prod 클러스터
          "arn:aws:eks:ap-northeast-2:<PROD_ACCOUNT_ID>:cluster/<PROD_CLUSTER_NAME>",
          # Staging 클러스터 (크로스 계정)
          "arn:aws:eks:ap-northeast-2:<STAGING_ACCOUNT_ID>:cluster/<STAGING_CLUSTER_NAME>"
        ]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "argocd_eks" {
  role       = aws_iam_role.argocd_irsa.name
  policy_arn = aws_iam_policy.argocd_eks.arn
}

output "argocd_irsa_role_arn" {
  value = aws_iam_role.argocd_irsa.arn
  # Helm values의 eks.amazonaws.com/role-arn에 사용
}
```

### 4-2. EKS aws-auth ConfigMap에 ArgoCD Role 추가

```hcl
# terraform/prod-account/eks-auth.tf
# aws-auth ConfigMap 방식 (EKS 1.28 이하 또는 혼용)

resource "kubernetes_config_map_v1_data" "aws_auth" {
  metadata {
    name      = "aws-auth"
    namespace = "kube-system"
  }

  data = {
    mapRoles = yamlencode([
      # EKS Node Group Role (기존)
      {
        rolearn  = aws_iam_role.eks_node.arn
        username = "system:node:{{EC2PrivateDNSName}}"
        groups   = ["system:bootstrappers", "system:nodes"]
      },
      # ArgoCD IRSA Role 추가
      {
        rolearn  = aws_iam_role.argocd_irsa.arn
        username = "argocd"
        groups   = ["system:masters"]  # 배포 권한 필요 시 masters
        # 최소 권한 적용 시 별도 ClusterRoleBinding 사용
      }
    ])
  }

  force = true
}
```

### 4-3. EKS Access Entry 방식 (EKS 1.29+, 권장)

```hcl
# terraform/prod-account/eks-access-entry.tf
# aws-auth ConfigMap 대신 EKS Access Entry 사용 (EKS 1.29+)

resource "aws_eks_access_entry" "argocd" {
  cluster_name  = var.cluster_name
  principal_arn = aws_iam_role.argocd_irsa.arn
  type          = "STANDARD"

  tags = {
    Purpose   = "ArgoCD IRSA Access"
    ManagedBy = "terraform"
  }
}

resource "aws_eks_access_policy_association" "argocd_admin" {
  cluster_name  = var.cluster_name
  principal_arn = aws_iam_role.argocd_irsa.arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"

  access_scope {
    type = "cluster"
  }
}
```

---

## 5. 멀티 클러스터 등록

### 5-1. ArgoCD CLI로 클러스터 등록 (IRSA 방식)

```bash
# ArgoCD 서버 접근
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 로그인
argocd login localhost:8080 \
  --username admin \
  --password $(kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d)

# Staging 클러스터 컨텍스트 추가
aws eks update-kubeconfig \
  --region ap-northeast-2 \
  --name <STAGING_CLUSTER_NAME> \
  --profile staging-account

# Staging 클러스터 등록
argocd cluster add <STAGING_CONTEXT_NAME> \
  --name staging \
  --namespace argocd
  # ArgoCD가 타겟 클러스터에 argocd-manager ServiceAccount 생성

# 등록된 클러스터 확인
argocd cluster list
```

### 5-2. Kubernetes Secret으로 클러스터 등록 (Declarative)

```yaml
# staging-cluster-secret.yaml
# ArgoCD가 관리하는 클러스터 Secret

apiVersion: v1
kind: Secret
metadata:
  name: staging-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: staging
  server: https://<STAGING_EKS_API_ENDPOINT>
  config: |
    {
      "awsAuthConfig": {
        "clusterName": "<STAGING_CLUSTER_NAME>",
        "roleARN": "arn:aws:iam::<STAGING_ACCOUNT_ID>:role/argocd-irsa-staging"
      },
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<BASE64_CA_DATA>"
      }
    }
```

---

## 6. ArgoCD Application 정의

### 6-1. 단일 애플리케이션

```yaml
# apps/my-app-prod.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # 앱 삭제 시 K8s 리소스도 삭제
spec:
  project: prod

  source:
    repoURL: https://github.com/<YOUR_ORG>/<YOUR_GITOPS_REPO>
    targetRevision: main
    path: apps/my-app/overlays/prod

    # Kustomize 사용 시
    kustomize:
      version: v5.x.x

    # Helm 사용 시 (kustomize와 동시 사용 불가)
    # helm:
    #   releaseName: my-app
    #   valueFiles:
    #     - values-prod.yaml

  destination:
    server: https://kubernetes.default.svc  # in-cluster
    # 또는 크로스 클러스터
    # server: https://<EKS_API_ENDPOINT>
    namespace: my-app

  syncPolicy:
    automated:
      prune: true      # Git에서 제거된 리소스 K8s에서도 삭제
      selfHeal: true   # 수동 변경을 Git 상태로 자동 복구
      allowEmpty: false  # 빈 sync 방지
    syncOptions:
      - CreateNamespace=true  # 네임스페이스 자동 생성
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # Sync 상태 조건
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # HPA가 관리하는 replicas는 무시
```

### 6-2. ArgoCD Project 정의

```yaml
# projects/prod-project.yaml

apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: prod
  namespace: argocd
spec:
  description: "Production environment"

  # 허용된 소스 리포지토리
  sourceRepos:
    - https://github.com/<YOUR_ORG>/<YOUR_GITOPS_REPO>
    - https://charts.helm.sh/stable

  # 허용된 배포 대상
  destinations:
    - server: https://kubernetes.default.svc
      namespace: "*"
    - server: https://<PROD_EKS_API_ENDPOINT>
      namespace: "*"

  # 클러스터 수준 리소스 허용 (CRD, ClusterRole 등)
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"

  # 네임스페이스 수준 리소스
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"

  roles:
    - name: prod-deployer
      description: "Prod 배포 권한"
      policies:
        - p, proj:prod:prod-deployer, applications, sync, prod/*, allow
        - p, proj:prod:prod-deployer, applications, get, prod/*, allow
```

---

## 7. Sync Policy 상세

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `automated.prune` | false | Git 삭제 → K8s 리소스 삭제 |
| `automated.selfHeal` | false | 수동 변경 → Git 상태로 복구 |
| `automated.allowEmpty` | false | 리소스 0개인 sync 허용 |
| `syncOptions.CreateNamespace` | false | 네임스페이스 자동 생성 |
| `syncOptions.PruneLast` | false | 삭제를 sync 마지막에 실행 |
| `retry.limit` | 0 (무한) | sync 재시도 횟수 |

### Sync 상태 강제 실행

```bash
# 애플리케이션 강제 sync
argocd app sync my-app-prod --force

# 특정 리소스만 sync
argocd app sync my-app-prod \
  --resource apps:Deployment:my-app

# Dry-run (실제 변경 없이 확인)
argocd app diff my-app-prod
```

---

## 8. 트러블슈팅

### 증상 1: ArgoCD가 "cluster 'xxx' has been invalidated" 오류

**원인**: EKS 클러스터 엔드포인트 변경 또는 IRSA 토큰 만료.

**해결 방법**:
```bash
# 클러스터 재등록
argocd cluster rm <CLUSTER_URL>
argocd cluster add <CONTEXT_NAME> --name <CLUSTER_NAME>

# IRSA Role 연결 확인
kubectl describe sa argocd-server -n argocd
# annotation: eks.amazonaws.com/role-arn 확인
```

---

### 증상 2: ArgoCD sync 시 "Unauthorized" 오류

```
FATA[0001] rpc error: code = PermissionDenied
desc = permission denied: applications, sync, prod/my-app
```

**원인**: ArgoCD RBAC Policy 미설정 또는 사용자가 Project에 권한 없음.

**해결 방법**:
```bash
# RBAC 정책 확인
kubectl get configmap argocd-rbac-cm -n argocd -o yaml

# 임시로 admin으로 sync
argocd app sync my-app-prod --auth-token $(argocd account generate-token --account admin)
```

---

### 증상 3: selfHeal로 인한 의도치 않은 롤백

**원인**: K8s 리소스를 직접 수정(`kubectl edit`) 후 ArgoCD가 Git 상태로 복구.

**해결 방법**:
```bash
# 1. ArgoCD App sync 일시 중지
argocd app set my-app-prod --sync-policy none

# 2. 긴급 수정 적용
kubectl edit deployment my-app -n my-app

# 3. GitOps repo에 동일 변경 반영 후 sync 재활성화
argocd app set my-app-prod --sync-policy automated --self-heal
```

---

### 증상 4: IRSA "not authorized to perform: sts:AssumeRoleWithWebIdentity"

**원인**: aws-auth ConfigMap 또는 EKS Access Entry에 argocd-irsa Role 미등록.

**해결 방법**:
```bash
# aws-auth 확인
kubectl get configmap aws-auth -n kube-system -o yaml

# argocd-irsa Role이 mapRoles에 있는지 확인
# 없으면 Terraform으로 추가

# EKS Access Entry 확인 (1.29+)
aws eks list-access-entries --cluster-name <CLUSTER_NAME> --region ap-northeast-2
```

---

## 9. 모니터링 및 알람

### ArgoCD 애플리케이션 상태 모니터링

```bash
# 전체 앱 상태 확인
argocd app list

# 특정 앱 상세 상태
argocd app get my-app-prod

# 비정상 앱만 필터
argocd app list --sync-status OutOfSync
argocd app list --health-status Degraded
```

### Prometheus + Grafana 연동

```yaml
# ArgoCD가 Prometheus metrics 노출 (기본 활성화)
# :8082/metrics 엔드포인트

# 주요 메트릭
# argocd_app_info{name, namespace, project, sync_status, health_status}
# argocd_app_sync_total{name, phase}  (sync 횟수)
# argocd_cluster_info{name, server}  (클러스터 상태)
```

### Slack 알림 설정

```yaml
# argocd-notifications-cm (ConfigMap)
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  template.app-sync-failed: |
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name }}",
          "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#E96D76",
          "fields": [
            {"title": "Sync Status", "value": "{{.app.status.sync.status}}", "short": true},
            {"title": "Repository", "value": "{{.app.spec.source.repoURL}}", "short": true}
          ]
        }]
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
```

---

## 10. TIP

**TIP 1**: ArgoCD `ignoreDifferences`로 HPA가 관리하는 `spec.replicas`를 무시하면 ArgoCD와 HPA 충돌 방지. Git에 replicas를 고정값으로 두면 HPA 스케일 아웃 후 ArgoCD sync로 원복되는 문제 발생.

**TIP 2**: `App of Apps` 패턴으로 ArgoCD Application 자체를 Git으로 관리. 새 서비스 추가 시 GitOps repo에 Application YAML 추가만으로 배포 자동화.

**TIP 3**: `argocd app wait --health` 명령어로 CI 파이프라인에서 배포 완료를 동기적으로 대기 가능. 배포 후 헬스 체크 자동화에 활용.

```bash
# CI 파이프라인에서 배포 완료 대기
argocd app sync my-app-prod
argocd app wait my-app-prod \
  --health \
  --timeout 300  # 5분 타임아웃
```

**TIP 4**: 멀티 클러스터 환경에서 `argocd cluster rotate-auth <CLUSTER_URL>`로 클러스터 인증 정보 주기적 갱신. IRSA 방식은 자동 갱신되므로 이 작업 불필요.

**관련 문서**
- [App of Apps 패턴](./argocd-app-of-apps.md)
- [GitOps 워크플로우](./gitops-workflow.md)
- [멀티 계정 CI/CD 아키텍처](../cicd/cicd-multi-account-architecture.md)
