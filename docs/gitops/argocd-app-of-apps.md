# ArgoCD App of Apps 패턴 (App of Apps Pattern)

## 1. 개요

ArgoCD Application 리소스 자체를 Git으로 관리하는 패턴. 루트 Application이 다른 Application들을 생성/관리. ApplicationSet을 활용하면 dev/staging/prod 멀티 환경을 단일 정의로 관리 가능.

**App of Apps vs ApplicationSet 비교**

| 항목 | App of Apps | ApplicationSet |
|------|-------------|----------------|
| 방식 | Application이 Application 배포 | Generator로 Application 자동 생성 |
| 유연성 | 높음 (개별 설정 가능) | 중간 (템플릿 기반) |
| 관리 복잡도 | 중간 | 낮음 |
| 멀티 환경 | 수동 정의 필요 | Generator로 자동화 |
| 권장 용도 | 복잡한 의존성 관리 | 반복 패턴 환경 자동화 |

---

## 2. App of Apps 패턴

### 2-1. GitOps 리포지토리 구조

```
gitops-repo/
├── apps/                          # 루트 App (모든 App을 관리)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── my-app.yaml           # Application 정의
│       ├── payment-service.yaml  # Application 정의
│       └── notification.yaml    # Application 정의
│
├── my-app/                        # 실제 앱 매니페스트
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       │   └── kustomization.yaml
│       ├── staging/
│       │   └── kustomization.yaml
│       └── prod/
│           └── kustomization.yaml
│
└── payment-service/
    └── ...
```

### 2-2. 루트 Application (Bootstrap)

```yaml
# 클러스터에 처음 한 번만 적용하는 Bootstrap Application
# kubectl apply -f bootstrap/root-app.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR_ORG>/<YOUR_GITOPS_REPO>
    targetRevision: main
    path: apps  # 이 디렉토리의 Application들을 모두 배포

  destination:
    server: https://kubernetes.default.svc
    namespace: argocd

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 2-3. 개별 Application 템플릿 (apps/templates/my-app.yaml)

```yaml
# apps/templates/my-app.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: prod
  source:
    repoURL: https://github.com/<YOUR_ORG>/<YOUR_GITOPS_REPO>
    targetRevision: main
    path: my-app/overlays/prod
    kustomize:
      version: v5.x.x
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 3. ApplicationSet으로 멀티 환경 관리

### 3-1. Git Directory Generator

GitOps 리포지토리의 디렉토리 구조를 기반으로 Application 자동 생성.

```yaml
# applicationsets/my-app-all-envs.yaml

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-environments
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/<YOUR_ORG>/<YOUR_GITOPS_REPO>
        revision: main
        directories:
          - path: my-app/overlays/*  # dev, staging, prod 자동 감지
          - path: my-app/overlays/experimental
            exclude: true  # experimental 환경은 제외

  template:
    metadata:
      # 디렉토리 이름으로 Application 이름 생성
      # my-app/overlays/prod → my-app-prod
      name: "my-app-{{path.basename}}"
      namespace: argocd
    spec:
      project: "{{path.basename}}"  # dev, staging, prod 프로젝트
      source:
        repoURL: https://github.com/<YOUR_ORG>/<YOUR_GITOPS_REPO>
        targetRevision: main
        path: "{{path}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "my-app-{{path.basename}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### 3-2. List Generator (환경별 다른 클러스터)

```yaml
# applicationsets/my-app-multi-cluster.yaml

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-multi-cluster
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            cluster: https://kubernetes.default.svc  # in-cluster (dev)
            namespace: my-app-dev
            replicas: "1"
            cpu: "100m"
            memory: "128Mi"
          - env: staging
            cluster: https://<STAGING_EKS_API_ENDPOINT>
            namespace: my-app-staging
            replicas: "2"
            cpu: "200m"
            memory: "256Mi"
          - env: prod
            cluster: https://<PROD_EKS_API_ENDPOINT>
            namespace: my-app
            replicas: "3"
            cpu: "500m"
            memory: "512Mi"

  template:
    metadata:
      name: "my-app-{{env}}"
      namespace: argocd
    spec:
      project: "{{env}}"
      source:
        repoURL: https://github.com/<YOUR_ORG>/<YOUR_GITOPS_REPO>
        targetRevision: main
        path: my-app/overlays/{{env}}
        kustomize:
          # Kustomize patch로 환경별 값 오버라이드
          patches:
            - target:
                kind: Deployment
                name: my-app
              patch: |
                - op: replace
                  path: /spec/replicas
                  value: {{replicas}}
      destination:
        server: "{{cluster}}"
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: "{{env}}" != "prod"  # prod는 수동 sync
```

### 3-3. Matrix Generator (환경 × 앱 조합)

```yaml
# applicationsets/all-apps-all-envs.yaml
# 여러 앱 × 여러 환경의 모든 조합 자동 생성

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: all-apps-all-envs
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          # 앱 목록
          - list:
              elements:
                - app: my-app
                  team: backend
                - app: payment-service
                  team: payment
                - app: notification
                  team: platform
          # 환경 목록
          - list:
              elements:
                - env: staging
                  cluster: https://<STAGING_EKS_API_ENDPOINT>
                - env: prod
                  cluster: https://<PROD_EKS_API_ENDPOINT>

  template:
    metadata:
      name: "{{app}}-{{env}}"
      namespace: argocd
      labels:
        app: "{{app}}"
        env: "{{env}}"
        team: "{{team}}"
    spec:
      project: "{{env}}"
      source:
        repoURL: https://github.com/<YOUR_ORG>/<YOUR_GITOPS_REPO>
        targetRevision: main
        path: "{{app}}/overlays/{{env}}"
      destination:
        server: "{{cluster}}"
        namespace: "{{app}}-{{env}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

## 4. Kustomize 멀티 환경 구조

```yaml
# my-app/base/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - hpa.yaml

commonLabels:
  app: my-app
  managed-by: argocd
```

```yaml
# my-app/overlays/prod/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: my-app

namePrefix: ""

# 이미지 태그 업데이트 (CI에서 자동 변경)
images:
  - name: my-app
    newName: <PROD_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/my-app
    newTag: abc1234  # CI가 업데이트하는 태그

# 환경별 replicas 오버라이드
patches:
  - target:
      kind: Deployment
      name: my-app
    patch: |
      - op: replace
        path: /spec/replicas
        value: 3
  - target:
      kind: HorizontalPodAutoscaler
      name: my-app
    patch: |
      - op: replace
        path: /spec/minReplicas
        value: 3
      - op: replace
        path: /spec/maxReplicas
        value: 10

# prod 전용 리소스
resources:
  - pod-disruption-budget.yaml
```

---

## 5. 트러블슈팅

### 증상 1: ApplicationSet이 Application을 생성하지 않음

**원인**: Generator 조건이 현재 Git 구조와 불일치 (path 패턴 오류).

**해결 방법**:
```bash
# ApplicationSet 상태 확인
kubectl get applicationset -n argocd
kubectl describe applicationset my-app-environments -n argocd

# Generator가 감지한 path 목록 확인 (Events 섹션)
kubectl get events -n argocd --field-selector reason=ApplicationGenerated
```

---

### 증상 2: 루트 App이 하위 Application을 삭제함

**원인**: `prune: true` + apps/templates에서 Application YAML 제거 시 해당 앱과 K8s 리소스 모두 삭제.

**해결 방법**:
```yaml
# 삭제 방지: finalizer 제거 후 Application 직접 삭제
argocd app set <APP_NAME> --finalizer none
argocd app delete <APP_NAME>

# 또는 cascade=false로 K8s 리소스는 유지하고 ArgoCD Application만 삭제
argocd app delete <APP_NAME> --cascade=false
```

---

## 6. 모니터링 및 알람

```bash
# ApplicationSet 관련 이벤트 모니터링
kubectl get events -n argocd --watch

# 생성된 Application 목록 확인
kubectl get applications -n argocd -l argocd.argoproj.io/app-set-name=my-app-environments
```

---

## 7. TIP

**TIP 1**: ApplicationSet의 `syncPolicy.preserveResourcesOnDeletion: true`를 설정하면 ApplicationSet 삭제 시 생성된 Application과 K8s 리소스를 보존. 실수로 인한 데이터 손실 방지.

**TIP 2**: 신규 환경 추가는 GitOps repo에 `my-app/overlays/<NEW_ENV>/kustomization.yaml` 파일 추가만으로 자동 배포. 운영팀 승인 없이 개발팀이 셀프 서비스로 환경 추가 가능.

**TIP 3**: `argocd-image-updater`와 ApplicationSet 조합 시 각 환경별 이미지 태그를 자동 관리. staging에 배포된 이미지를 검증 후 prod에 promote하는 워크플로우 구현 가능.

**관련 문서**
- [ArgoCD EKS 배포 및 IRSA](./argocd-eks-deployment.md)
- [GitOps 워크플로우](./gitops-workflow.md)
- [Helm 릴리스 관리](../deployment/helm-release-management.md)
