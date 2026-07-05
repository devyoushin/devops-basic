# 배포 전략 (Deployment Strategies)

## 1. 개요

Kubernetes에서 사용 가능한 배포 전략 비교 및 Argo Rollouts를 활용한 고급 배포 패턴. 서비스 SLA, 리소스 비용, 롤백 속도에 따라 전략 선택.

**전략별 비교**

| 전략 | 다운타임 | 리소스 비용 | 롤백 속도 | 복잡도 | 권장 용도 |
|------|----------|-------------|-----------|--------|-----------|
| Recreate | 있음 | 낮음 | 빠름 | 낮음 | 개발/스테이징 환경 |
| Rolling Update | 없음 | 낮음 | 중간 | 낮음 | 일반 서비스 |
| Blue/Green | 없음 | 높음 (2배) | 즉시 | 중간 | 중요 서비스, DB 마이그레이션 |
| Canary | 없음 | 낮음 | 중간 | 높음 | 신기능 점진적 출시 |

---

## 2. Rolling Update (기본 전략)

```yaml
# deployment.yaml - Rolling Update

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: my-app

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # 동시에 추가로 생성 가능한 Pod 수 (또는 %)
      maxUnavailable: 1  # 동시에 Unavailable 허용 Pod 수

  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: <ECR_URI>/my-app:abc1234
          ports:
            - containerPort: 8080

          # 롤링 업데이트 안전성: readinessProbe 필수
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3

          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3

          # Graceful shutdown: preStop hook
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          terminationGracePeriodSeconds: 60

      # 가용성 보장: 여러 노드에 분산
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-app
```

### PodDisruptionBudget (롤링 업데이트 안전 보장)

```yaml
# pdb.yaml

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  minAvailable: 4  # 최소 4개 Pod는 항상 Running 상태 유지
  # 또는
  # maxUnavailable: 2
```

---

## 3. Blue/Green 배포

Kubernetes 기본 기능으로 Blue/Green 구현 (Service selector 전환 방식).

```yaml
# blue-deployment.yaml (현재 버전)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
  namespace: my-app
  labels:
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
        - name: my-app
          image: <ECR_URI>/my-app:v1.0.0
```

```yaml
# green-deployment.yaml (새 버전)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
  namespace: my-app
  labels:
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
        - name: my-app
          image: <ECR_URI>/my-app:v2.0.0
```

```yaml
# service.yaml - Blue/Green 전환 포인트

apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: my-app
spec:
  selector:
    app: my-app
    version: blue  # ← 이 값을 'green'으로 변경하면 트래픽 전환
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# Blue → Green 전환 (순간 전환, 다운타임 없음)
kubectl patch service my-app -n my-app \
  -p '{"spec":{"selector":{"version":"green"}}}'

# Green 검증 실패 시 즉시 Blue로 롤백
kubectl patch service my-app -n my-app \
  -p '{"spec":{"selector":{"version":"blue"}}}'

# 전환 성공 후 Blue Deployment 삭제
kubectl delete deployment my-app-blue -n my-app
```

---

## 4. Argo Rollouts (Canary/Blue-Green 고급)

### 4-1. Argo Rollouts 설치

```bash
# kubectl plugin 설치
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# Argo Rollouts Controller 설치
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### 4-2. Canary 배포 (Argo Rollouts)

```yaml
# rollout-canary.yaml

apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: my-app

  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: <ECR_URI>/my-app:abc1234
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5

  strategy:
    canary:
      # Canary Service (새 버전 트래픽 수신)
      canaryService: my-app-canary
      # Stable Service (기존 버전 트래픽 수신)
      stableService: my-app-stable

      # 트래픽 분할 단계
      steps:
        - setWeight: 10    # 10% 트래픽 → Canary
        - pause:           # 5분 대기 (자동 진행 또는 수동 승인)
            duration: 5m
        - setWeight: 30    # 30% → Canary
        - pause:
            duration: 10m
        - analysis:        # 메트릭 분석 (성공 시 계속, 실패 시 자동 롤백)
            templates:
              - templateName: success-rate
        - setWeight: 50    # 50% → Canary
        - pause: {}        # 무한 대기 (수동 승인 필요)
        - setWeight: 100   # 100% → Canary (완전 전환)

      # 분석 실패 시 자동 롤백
      autoPromotionEnabled: false

      # Anti-affinity: Canary Pod를 Stable Pod와 다른 노드에 배치
      antiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution: {}
```

### 4-3. AnalysisTemplate (자동 검증)

```yaml
# analysis-template.yaml

apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: my-app
spec:
  metrics:
    - name: success-rate
      # Prometheus 기반 성공률 측정
      provider:
        prometheus:
          address: http://prometheus-server.monitoring:80
          query: |
            sum(rate(
              http_requests_total{
                app="my-app",
                status!~"5.."
              }[5m]
            )) /
            sum(rate(
              http_requests_total{app="my-app"}[5m]
            ))
      successCondition: result[0] >= 0.99  # 99% 이상 성공 시 진행
      failureCondition: result[0] < 0.95   # 95% 미만 시 자동 롤백
      interval: 60s
      count: 5  # 5회 측정

    - name: latency-p99
      provider:
        prometheus:
          address: http://prometheus-server.monitoring:80
          query: |
            histogram_quantile(0.99,
              rate(http_request_duration_seconds_bucket{app="my-app"}[5m])
            )
      successCondition: result[0] <= 0.5  # p99 < 500ms
      failureCondition: result[0] > 1.0   # p99 > 1s 시 롤백
      interval: 60s
      count: 3
```

### 4-4. Blue/Green (Argo Rollouts)

```yaml
# rollout-bluegreen.yaml

apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app

  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: <ECR_URI>/my-app:abc1234

  strategy:
    blueGreen:
      activeService: my-app          # 현재 트래픽 수신 Service
      previewService: my-app-preview # 새 버전 미리보기 Service
      autoPromotionEnabled: false    # 수동 승인 후 전환
      scaleDownDelaySeconds: 300     # 전환 후 5분 뒤 Blue 삭제
      prePromotionAnalysis:          # 전환 전 자동 검증
        templates:
          - templateName: success-rate
      postPromotionAnalysis:         # 전환 후 검증
        templates:
          - templateName: success-rate
```

```bash
# Argo Rollouts 배포 관리 명령어

# 배포 상태 확인
kubectl argo rollouts get rollout my-app -n my-app --watch

# Canary 다음 단계로 진행 (수동 승인)
kubectl argo rollouts promote my-app -n my-app

# 즉시 완전 배포 (모든 단계 건너뜀)
kubectl argo rollouts promote my-app -n my-app --full

# 롤백
kubectl argo rollouts undo my-app -n my-app

# 특정 리비전으로 롤백
kubectl argo rollouts undo my-app -n my-app --to-revision=3

# 배포 일시 중지
kubectl argo rollouts pause my-app -n my-app

# 배포 재개
kubectl argo rollouts resume my-app -n my-app
```

---

## 5. ArgoCD + Argo Rollouts 연동

```yaml
# ArgoCD Application에서 Rollout 사용 시
# ignoreDifferences로 Rollout이 관리하는 필드 ArgoCD가 무시하도록 설정

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-prod
  namespace: argocd
spec:
  # ... 기존 설정

  ignoreDifferences:
    - group: argoproj.io
      kind: Rollout
      jsonPointers:
        - /spec/replicas  # Argo Rollouts가 replicas 관리
    - group: apps
      kind: Deployment    # Rollout이 생성한 Deployment
      jsonPointers:
        - /spec/replicas
        - /spec/template/spec/containers/0/image
```

---

## 6. 트러블슈팅

### 증상 1: Rolling Update 중 트래픽 손실 (5xx 오류)

**원인**: readinessProbe 없거나 initialDelaySeconds 너무 짧음. 또는 preStop hook 없어 연결 강제 종료.

**해결 방법**:
```yaml
containers:
  - name: my-app
    readinessProbe:
      initialDelaySeconds: 30   # 앱 기동 시간 여유 확보
      failureThreshold: 3
      periodSeconds: 10
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 15"]  # 로드밸런서 deregister 대기
terminationGracePeriodSeconds: 90  # preStop + 처리 시간 여유 확보
```

---

### 증상 2: Canary 분석이 항상 실패 ("NoData")

**원인**: Prometheus에 Canary 관련 메트릭이 아직 수집되지 않음.

**해결 방법**:
```yaml
metrics:
  - name: success-rate
    # count를 늘리거나 initialDelay 추가
    initialDelay: 2m  # 첫 측정 전 2분 대기 (메트릭 안정화)
    interval: 60s
    count: 5
```

---

## 7. 모니터링 및 알람

```bash
# Argo Rollouts 상태 Prometheus 메트릭
# rollout_info{name, namespace, phase}
# phase: Progressing, Paused, Degraded, Healthy

# AlertManager 규칙
- alert: RolloutDegraded
  expr: rollout_info{phase="Degraded"} == 1
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Rollout {{ $labels.name }} is Degraded"
```

---

## 8. TIP

**TIP 1**: Canary 배포 초기에는 `pause: duration: 5m` 대신 `pause: {}` (수동 승인)로 시작하여 팀이 메트릭을 직접 확인하는 문화를 먼저 정착시킨 후 자동화 권장.

**TIP 2**: Blue/Green에서 DB 마이그레이션이 있을 경우, 새 버전(Green)이 이전 스키마와 호환되도록 backward compatible 마이그레이션 우선 적용 → Green 배포 → 이후 이전 버전에서만 필요한 컬럼 삭제.

**관련 문서**
- [Helm 릴리스 관리](./helm-release-management.md)
- [ArgoCD EKS 배포](../02-gitops/argocd-eks-deployment.md)
- [GitOps 워크플로우](../02-gitops/gitops-workflow.md)
