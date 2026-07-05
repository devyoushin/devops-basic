# Helm 릴리스 관리 (Helm Release Management)

## 1. 개요

Helm은 Kubernetes 패키지 매니저. Chart로 애플리케이션을 패키징하고 values.yaml로 환경별 설정을 분리. ArgoCD와 연동 시 GitOps 방식으로 Helm 릴리스를 선언적으로 관리.

**Helm vs Kustomize 선택 기준**

| 항목 | Helm | Kustomize |
|------|------|-----------|
| 복잡한 조건부 로직 | 지원 (Go template) | 미지원 |
| 외부 차트 재사용 | 지원 (Chart dependency) | 불편 |
| 단순 환경별 오버라이드 | 가능하나 복잡 | 간단 |
| 학습 곡선 | 높음 | 낮음 |
| ArgoCD 지원 | 완전 지원 | 완전 지원 |

---

## 2. Helm Chart 기본 구조

```
my-app/                          # Chart 루트
├── Chart.yaml                   # Chart 메타데이터
├── values.yaml                  # 기본값 (공통)
├── values-dev.yaml              # Dev 환경 오버라이드
├── values-staging.yaml          # Staging 환경 오버라이드
├── values-prod.yaml             # Prod 환경 오버라이드
└── templates/
    ├── _helpers.tpl             # 재사용 헬퍼 함수
    ├── deployment.yaml          # Deployment
    ├── service.yaml             # Service
    ├── hpa.yaml                 # HorizontalPodAutoscaler
    ├── pdb.yaml                 # PodDisruptionBudget
    ├── ingress.yaml             # Ingress
    ├── serviceaccount.yaml      # ServiceAccount
    ├── configmap.yaml           # ConfigMap
    └── NOTES.txt                # 설치 후 출력 메시지
```

### Chart.yaml

```yaml
# Chart.yaml

apiVersion: v2
name: my-app
description: "My Application Helm Chart"
type: application
version: 1.2.3         # Chart 버전 (SemVer)
appVersion: "abc1234"  # 앱 이미지 버전 (CI가 업데이트)

maintainers:
  - name: Platform Team
    email: platform@example.com

dependencies:
  # 외부 차트 의존성 (예: Redis, PostgreSQL)
  - name: redis
    version: "19.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled  # values.yaml에서 활성화 여부 제어
```

### values.yaml (공통 기본값)

```yaml
# values.yaml

replicaCount: 1

image:
  repository: <ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/my-app
  tag: "latest"         # CI 파이프라인에서 git-sha로 오버라이드
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

podDisruptionBudget:
  enabled: false
  minAvailable: 1

ingress:
  enabled: false
  className: alb
  annotations: {}
  hosts: []
  tls: []

serviceAccount:
  create: true
  annotations: {}
  name: ""

nodeSelector: {}
tolerations: []
affinity: {}

# 환경별 설정
env:
  LOG_LEVEL: info
  ENV: local

# ConfigMap 데이터
configMap:
  enabled: false
  data: {}

# External Secrets
externalSecret:
  enabled: false
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore

redis:
  enabled: false
```

### values-prod.yaml (Prod 오버라이드)

```yaml
# values-prod.yaml

replicaCount: 3

image:
  tag: "abc1234"  # CI가 git-sha로 업데이트

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 2Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 2

ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: <ACM_CERT_ARN>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - api.example.com

serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<PROD_ACCOUNT_ID>:role/my-app-irsa

env:
  LOG_LEVEL: warn
  ENV: prod

externalSecret:
  enabled: true
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: /prod/my-app/database-url
    - secretKey: API_KEY
      remoteRef:
        key: /prod/my-app/api-key
```

### templates/deployment.yaml

```yaml
# templates/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
    app.kubernetes.io/version: {{ .Values.image.tag | quote }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
      annotations:
        # ConfigMap/Secret 변경 시 Pod 재시작 트리거
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      serviceAccountName: {{ include "my-app.serviceAccountName" . }}

      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}

          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}

          {{- if .Values.externalSecret.enabled }}
          envFrom:
            - secretRef:
                name: {{ include "my-app.fullname" . }}-secret
          {{- end }}

          resources:
            {{- toYaml .Values.resources | nindent 12 }}

          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3

          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]

      terminationGracePeriodSeconds: 60

      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

---

## 3. ArgoCD + Helm 연동

### ArgoCD Application (Helm 방식)

```yaml
# argocd-application-helm.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-prod
  namespace: argocd
spec:
  project: prod
  source:
    repoURL: https://github.com/<YOUR_ORG>/<YOUR_GITOPS_REPO>
    targetRevision: main
    path: charts/my-app  # Helm chart 경로

    helm:
      releaseName: my-app
      valueFiles:
        - values.yaml
        - values-prod.yaml
      # 추가 값 인라인 오버라이드 (선택)
      values: |
        image:
          tag: abc1234  # CI가 values-prod.yaml에서 업데이트
      # 또는 set으로 개별 값 오버라이드
      # parameters:
      #   - name: image.tag
      #     value: abc1234

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

### Chart 버전 고정 전략

```yaml
# chart 버전과 앱 버전을 분리 관리

# ArgoCD에서 특정 Chart 버전 고정
source:
  targetRevision: "1.2.3"  # Git 태그로 Chart 버전 고정
  # 또는 특정 commit SHA로 고정
  # targetRevision: abc1234

# 외부 Helm 리포지토리 사용 시
source:
  repoURL: https://charts.example.com
  chart: my-app
  targetRevision: "1.2.3"  # Chart 버전 고정 (SemVer 범위 X, 정확한 버전)
  helm:
    valueFiles:
      - $values/my-app/values-prod.yaml  # 별도 values 리포지토리
```

---

## 4. Helm 명령어 참조

```bash
# Chart 생성 (스캐폴딩)
helm create my-app

# 렌더링 검증 (실제 배포 없이)
helm template my-app ./charts/my-app \
  -f values.yaml \
  -f values-prod.yaml \
  --namespace my-app

# 설치
helm install my-app ./charts/my-app \
  -f values.yaml \
  -f values-prod.yaml \
  --namespace my-app \
  --create-namespace

# 업그레이드
helm upgrade my-app ./charts/my-app \
  -f values.yaml \
  -f values-prod.yaml \
  --namespace my-app \
  --atomic           # 실패 시 자동 롤백
  --timeout 5m

# 히스토리 확인
helm history my-app -n my-app

# 롤백
helm rollback my-app 3 -n my-app  # 리비전 3으로 롤백

# 현재 values 확인
helm get values my-app -n my-app

# 릴리스 삭제
helm uninstall my-app -n my-app

# 의존성 업데이트
helm dependency update ./charts/my-app
```

---

## 5. Helm Diff로 변경 미리보기

```bash
# helm-diff 플러그인 설치
helm plugin install https://github.com/databus23/helm-diff

# 업그레이드 전 변경 사항 확인
helm diff upgrade my-app ./charts/my-app \
  -f values.yaml \
  -f values-prod.yaml \
  --namespace my-app
```

---

## 6. 트러블슈팅

### 증상 1: "rendered manifests contain a resource that already exists"

**원인**: `helm install` 시 이미 동일 이름의 리소스가 존재 (이전 수동 배포 잔재).

**해결 방법**:
```bash
# 기존 리소스를 Helm 관리로 편입
kubectl label <RESOURCE_TYPE> <RESOURCE_NAME> \
  app.kubernetes.io/managed-by=Helm \
  -n my-app

kubectl annotate <RESOURCE_TYPE> <RESOURCE_NAME> \
  meta.helm.sh/release-name=my-app \
  meta.helm.sh/release-namespace=my-app \
  -n my-app

# 이후 helm upgrade로 재시도
```

---

### 증상 2: Helm 업그레이드 중 "--atomic" 롤백 후 이전 버전도 비정상

**원인**: DB 마이그레이션이 이미 실행되어 이전 버전 스키마와 호환 불가.

**해결 방법**:
```bash
# Helm 히스토리 확인
helm history my-app -n my-app

# 정상 작동하던 마지막 리비전 확인 후 수동 롤백
helm rollback my-app <LAST_GOOD_REVISION> -n my-app

# DB 마이그레이션은 backward compatible하게 작성 필수
# (새 컬럼 추가는 nullable로, 컬럼 삭제는 앱 배포 완료 후)
```

---

## 7. 모니터링 및 알람

```bash
# 배포된 Helm 릴리스 상태 확인
helm list -n my-app -A

# FAILED 상태 릴리스 확인
helm list -A --filter "status=failed"

# ArgoCD에서 Helm 릴리스 동기화 상태
argocd app get my-app-prod
```

---

## 8. TIP

**TIP 1**: `_helpers.tpl`에 공통 레이블/어노테이션을 정의하면 일관성 유지. `app.kubernetes.io/version`에 이미지 태그를 기록하면 배포 버전 추적 용이.

**TIP 2**: CI 파이프라인에서 `helm lint`와 `helm template | kubectl apply --dry-run=client -f -` 조합으로 Helm chart 문법 오류 사전 감지.

```yaml
# .github/workflows/helm-lint.yaml
- name: Helm lint
  run: helm lint ./charts/my-app -f values-prod.yaml

- name: Helm template validate
  run: |
    helm template my-app ./charts/my-app \
      -f values-prod.yaml \
      --namespace my-app | \
    kubectl apply --dry-run=client -f -
```

**TIP 3**: Chart 버전(`Chart.yaml`의 `version`)과 앱 버전(`appVersion`)을 분리 관리. Chart 구조 변경 시 Chart 버전을 올리고, 앱 코드 변경 시 appVersion(image tag)만 변경. CI에서 appVersion만 자동 업데이트하도록 설정.

**관련 문서**
- [배포 전략](./deployment-strategies.md)
- [ArgoCD EKS 배포](../02-gitops/argocd-eks-deployment.md)
- [App of Apps 패턴](../02-gitops/argocd-app-of-apps.md)
