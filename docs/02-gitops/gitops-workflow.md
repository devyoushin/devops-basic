# GitOps 워크플로우 (GitOps Workflow)

## 1. 개요

GitOps는 Git 리포지토리를 인프라와 애플리케이션의 Single Source of Truth(진실의 단일 출처)로 활용하는 운영 방식. 선언적(declarative) 설정을 Git에 저장하고 자동화 도구(ArgoCD 등)가 실제 상태와 동기화.

**GitOps 4원칙 (OpenGitOps)**
1. **선언적 (Declarative)**: 시스템 상태를 선언적으로 기술
2. **버전 관리 (Versioned & Immutable)**: Git에 저장, 불변 히스토리
3. **자동화 (Pulled Automatically)**: 변경 감지 및 자동 적용
4. **지속적 조정 (Continuously Reconciled)**: 실제 상태를 원하는 상태로 지속 유지

---

## 2. GitOps 리포지토리 구조

### 2-1. 분리 구조 (권장)

```
소스 리포지토리 (app-source)          GitOps 리포지토리 (app-gitops)
app/                                  environments/
├── src/                              ├── dev/
├── Dockerfile                        │   ├── deployment.yaml
└── .github/workflows/                │   ├── service.yaml
    └── build-push.yaml               │   └── kustomization.yaml
    # CI가 빌드 후                     ├── staging/
    # app-gitops의                    │   └── ...
    # image tag 업데이트               └── prod/
                                          ├── deployment.yaml (image: :abc1234)
                                          └── kustomization.yaml
```

**소스/GitOps 리포지토리 분리 이유**
- CI 워크플로우 변경과 배포 설정 변경의 권한 분리
- GitOps repo에 소스 코드가 없어 공격 표면 최소화
- PR 리뷰어를 배포 승인자로 명확히 구분

### 2-2. GitOps 리포지토리 상세 구조

```
gitops-repo/
├── apps/                          # ArgoCD App of Apps (루트 Application)
│   ├── Chart.yaml
│   └── templates/
│       ├── my-app.yaml
│       └── payment-service.yaml
│
├── my-app/
│   ├── base/                      # 공통 베이스 매니페스트
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   ├── pdb.yaml               # PodDisruptionBudget
│   │   └── kustomization.yaml
│   └── overlays/                  # 환경별 오버라이드
│       ├── dev/
│       │   ├── kustomization.yaml # image tag, replicas (dev 기준)
│       │   └── patch-resources.yaml
│       ├── staging/
│       │   └── kustomization.yaml
│       └── prod/
│           ├── kustomization.yaml  # ← CI가 image tag 업데이트
│           └── patch-resources.yaml
│
├── infrastructure/                # 인프라 관련 K8s 리소스
│   ├── cert-manager/
│   ├── external-secrets/
│   └── ingress-nginx/
│
└── .github/
    └── workflows/
        └── validate.yaml          # PR 시 kustomize build 검증
```

---

## 3. Branch 전략

### 3-1. Environment Branch 전략

```
main ──────────────────────────────────────────────────────────▶
  │
  ├── Merge → prod/kustomization.yaml image tag 업데이트
  │           ArgoCD가 prod EKS에 배포
  │
staging ──────────────────────────────────────────────────────▶
  │
  ├── Merge → staging/kustomization.yaml image tag 업데이트
  │           ArgoCD가 staging EKS에 배포
  │
dev ──────────────────────────────────────────────────────────▶
  │
  └── Merge → dev/kustomization.yaml image tag 업데이트
              ArgoCD가 dev EKS에 배포
```

**단점**: 브랜치 관리 복잡도 증가. `main` 브랜치로 통합 후 overlay로 환경 구분하는 방식 권장.

### 3-2. Main Branch + Overlay 전략 (권장)

```
main (GitOps repo)
  │
  ├── my-app/overlays/dev/kustomization.yaml   (image: :pr-123-abc1234)
  ├── my-app/overlays/staging/kustomization.yaml (image: :staging-abc1234)
  └── my-app/overlays/prod/kustomization.yaml    (image: :abc1234)

# 각 overlay를 CI가 독립적으로 업데이트
# ArgoCD가 각 overlay path를 각 클러스터에 매핑
```

---

## 4. PR 기반 배포 플로우

### 4-1. 개발자 배포 플로우

```
1. 개발자: feature 브랜치에서 PR 생성
   └── GitHub Actions: docker build (push 없음), 테스트 실행

2. PR 리뷰어 승인 + main 브랜치 merge
   └── GitHub Actions:
       a. docker build + ECR push (이미지 태그: abc1234)
       b. GitOps repo PR 자동 생성
          - staging/kustomization.yaml: image tag → staging-abc1234

3. GitOps repo PR 리뷰 (자동 또는 수동 승인)
   └── PR merge → ArgoCD가 감지 → staging 배포

4. Staging 검증 완료 후 prod 배포 PR 생성
   └── prod/kustomization.yaml: image tag → abc1234
   └── PR merge → ArgoCD가 감지 → prod 배포

5. ArgoCD sync 완료 → Slack 알림
```

### 4-2. GitOps PR 자동 생성 스크립트

```python
#!/usr/bin/env python3
# scripts/update-gitops-pr.py
# CI에서 GitOps repo에 이미지 태그 업데이트 PR 자동 생성

import subprocess
import os
import requests
import json

GITOPS_REPO = os.environ["GITOPS_REPO"]  # org/repo 형식
GITOPS_TOKEN = os.environ["GITOPS_TOKEN"]
IMAGE_TAG = os.environ["IMAGE_TAG"]
APP_NAME = os.environ["APP_NAME"]
TARGET_ENV = os.environ.get("TARGET_ENV", "staging")

def create_gitops_pr():
    headers = {
        "Authorization": f"token {GITOPS_TOKEN}",
        "Accept": "application/vnd.github.v3+json"
    }

    # 1. GitOps repo clone
    subprocess.run([
        "git", "clone",
        f"https://{GITOPS_TOKEN}@github.com/{GITOPS_REPO}.git",
        "gitops"
    ], check=True)

    os.chdir("gitops")

    # 2. 새 브랜치 생성
    branch_name = f"update-{APP_NAME}-{TARGET_ENV}-{IMAGE_TAG}"
    subprocess.run(["git", "checkout", "-b", branch_name], check=True)

    # 3. kustomize image tag 업데이트
    subprocess.run([
        "kustomize", "edit", "set", "image",
        f"{APP_NAME}=<ECR_URI>/{APP_NAME}:{IMAGE_TAG}"
    ], cwd=f"{APP_NAME}/overlays/{TARGET_ENV}", check=True)

    # 4. 커밋 및 푸시
    subprocess.run(["git", "config", "user.name", "github-actions[bot]"], check=True)
    subprocess.run(["git", "config", "user.email", "github-actions[bot]@users.noreply.github.com"], check=True)
    subprocess.run(["git", "add", "."], check=True)
    subprocess.run([
        "git", "commit", "-m",
        f"chore: update {APP_NAME} {TARGET_ENV} image to {IMAGE_TAG}"
    ], check=True)
    subprocess.run(["git", "push", "origin", branch_name], check=True)

    # 5. PR 생성
    pr_data = {
        "title": f"[{TARGET_ENV}] Deploy {APP_NAME}:{IMAGE_TAG}",
        "body": f"""## 배포 요청

**앱**: `{APP_NAME}`
**환경**: `{TARGET_ENV}`
**이미지 태그**: `{IMAGE_TAG}`
**소스 커밋**: {os.environ.get('GITHUB_SHA', 'unknown')}

## 체크리스트
- [ ] Staging 테스트 통과 확인
- [ ] 롤백 계획 확인
""",
        "head": branch_name,
        "base": "main"
    }

    response = requests.post(
        f"https://api.github.com/repos/{GITOPS_REPO}/pulls",
        headers=headers,
        json=pr_data
    )
    response.raise_for_status()
    pr_url = response.json()["html_url"]
    print(f"PR created: {pr_url}")
    return pr_url

if __name__ == "__main__":
    create_gitops_pr()
```

---

## 5. 롤백 방법

### 5-1. Git Revert를 통한 롤백 (권장)

```bash
# 방법 1: 이전 커밋으로 revert
git log --oneline -- my-app/overlays/prod/kustomization.yaml

# 특정 커밋 revert
git revert <COMMIT_SHA>
git push origin main
# → ArgoCD가 자동으로 이전 이미지로 롤백

# 방법 2: 특정 이전 상태로 직접 변경
git show <COMMIT_SHA>:my-app/overlays/prod/kustomization.yaml > /tmp/old-kustomization.yaml
cp /tmp/old-kustomization.yaml my-app/overlays/prod/kustomization.yaml
git commit -m "revert: rollback my-app prod to <OLD_IMAGE_TAG>"
git push origin main
```

### 5-2. ArgoCD를 통한 즉시 롤백

```bash
# ArgoCD UI 또는 CLI로 이전 sync 상태로 롤백
# ArgoCD가 이전 sync 히스토리를 보관

# 이전 sync 히스토리 확인
argocd app history my-app-prod

# 특정 revision으로 롤백
argocd app rollback my-app-prod <REVISION_ID>

# 이 방법은 ArgoCD의 auto-sync를 일시 중지시킴
# 롤백 후 Git도 동일하게 업데이트하여 auto-sync 재활성화 권장
```

### 5-3. 긴급 롤백 절차 (Runbook)

```
1. 장애 확인 (Slack 알림 또는 모니터링 대시보드)
   └── ArgoCD Application 상태: Degraded 또는 OutOfSync

2. ArgoCD CLI/UI에서 즉시 이전 revision으로 롤백
   argocd app rollback my-app-prod <PREV_REVISION>
   (소요 시간: ~1-2분)

3. 롤백 완료 확인
   argocd app wait my-app-prod --health --timeout 120

4. 장애 원인 분석 (병렬 진행 가능)

5. Git 수정 후 재배포 또는 이전 이미지 태그로 Git revert

6. auto-sync 재활성화
   argocd app set my-app-prod --sync-policy automated --self-heal
```

---

## 6. GitOps PR 검증 자동화

```yaml
# .github/workflows/validate-gitops.yaml
# GitOps repo PR 시 자동 검증

name: Validate GitOps Manifests

on:
  pull_request:
    paths:
      - '*/overlays/**'
      - '*/base/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup kustomize
        uses: imranismail/setup-kustomize@v2
        with:
          kustomize-version: "5.x.x"

      - name: Setup kubeval
        run: |
          wget https://github.com/instrumenta/kubeval/releases/latest/download/kubeval-linux-amd64.tar.gz
          tar xf kubeval-linux-amd64.tar.gz
          sudo mv kubeval /usr/local/bin/

      - name: Validate all overlays
        run: |
          # 변경된 overlay만 검증
          for overlay in $(git diff --name-only HEAD~1 | grep "overlays/" | xargs -I{} dirname {} | sort -u); do
            echo "Validating: $overlay"
            kustomize build "$overlay" | kubeval --strict --kubernetes-version 1.29.0
          done

      - name: Dry-run ArgoCD diff (optional)
        if: env.ARGOCD_TOKEN != ''
        run: |
          argocd app diff my-app-prod --local my-app/overlays/prod
        env:
          ARGOCD_TOKEN: ${{ secrets.ARGOCD_TOKEN }}
          ARGOCD_SERVER: ${{ secrets.ARGOCD_SERVER }}
```

---

## 7. 트러블슈팅

### 증상 1: ArgoCD sync 후 Pod가 이전 이미지를 계속 사용

**원인**: `imagePullPolicy: IfNotPresent`이고 노드에 이전 이미지 캐시 존재.

**해결 방법**:
```yaml
# deployment.yaml
spec:
  template:
    spec:
      containers:
        - name: my-app
          imagePullPolicy: Always  # 또는 태그를 고정 SHA로 사용
          # IfNotPresent는 latest 태그나 동일 태그 재빌드 시 문제 발생
```

---

### 증상 2: Git revert 후 ArgoCD가 Synced인데 이전 이미지가 뜸

**원인**: revert 커밋이 이전 커밋과 동일한 내용 → ArgoCD가 변경으로 감지 못함.

**해결 방법**:
```bash
# ArgoCD 강제 refresh 및 sync
argocd app get my-app-prod --refresh
argocd app sync my-app-prod --force
```

---

## 8. 모니터링 및 알람

```bash
# GitOps repo commit 빈도 모니터링 (배포 빈도 측정)
# GitHub Insights > Pulse에서 확인 가능

# ArgoCD sync 성공/실패 Prometheus 메트릭
# argocd_app_sync_total{name="my-app-prod", phase="Succeeded"}
# argocd_app_sync_total{name="my-app-prod", phase="Failed"}

# DORA 메트릭 측정 (배포 빈도, 변경 리드타임)
# git log --oneline --since="30 days ago" -- overlays/prod/ | wc -l
```

---

## 9. TIP

**TIP 1**: GitOps repo의 `CODEOWNERS` 파일로 overlay별 리뷰어 지정. prod overlay는 시니어 엔지니어 또는 SRE 팀만 승인 가능하도록 설정.

```
# CODEOWNERS
*/overlays/prod/ @org/sre-team
*/overlays/staging/ @org/devops-team
```

**TIP 2**: GitOps repo commit message 컨벤션으로 배포 추적성 향상. `chore: update <app> <env> image to <tag>` 형식 통일 시 `git log --grep="update my-app prod"` 로 prod 배포 이력 추적 가능.

**TIP 3**: `git bisect`를 GitOps repo에 활용하면 특정 장애를 유발한 이미지 태그를 이진 탐색으로 빠르게 찾기 가능. 각 커밋이 하나의 배포를 나타내므로 매우 효과적.

**관련 문서**
- [ArgoCD EKS 배포 및 IRSA](./argocd-eks-deployment.md)
- [App of Apps 패턴](./argocd-app-of-apps.md)
- [GitHub Actions + ECR 파이프라인](../01-cicd/cicd-github-actions-ecr.md)
