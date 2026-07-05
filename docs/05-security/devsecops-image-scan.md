# DevSecOps 컨테이너 이미지 스캔 (Container Image Scanning)

## 1. 개요

CI 파이프라인에서 컨테이너 이미지 취약점을 스캔하여 CRITICAL 취약점이 발견된 경우 빌드를 실패시키는 DevSecOps 실천. ECR Inspector(클라우드 관리형)와 Trivy(오픈소스)를 상황에 따라 선택.

**스캔 도구 비교**

| 항목 | ECR Inspector | Trivy |
|------|---------------|-------|
| 유형 | AWS 관리형 서비스 | 오픈소스 CLI |
| 스캔 시점 | ECR push 후 자동 | 빌드 중 또는 push 후 |
| CI 통합 | EventBridge 이벤트 | GitHub Actions action |
| DB 업데이트 | AWS 자동 관리 | --db-repository로 pull |
| 비용 | Inspector 요금 | 무료 |
| 오프라인 | 불가 | 가능 (DB 캐시) |
| 권장 환경 | AWS 환경, 자동화 | 빠른 피드백, CI 통합 |

---

## 2. Trivy CI 통합 (GitHub Actions)

### 2-1. 빌드 중 스캔 (Push 전 게이트)

```yaml
# .github/workflows/build-scan-push.yaml

name: Build, Scan, and Push

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write
  contents: read
  security-events: write  # GitHub Security tab에 SARIF 업로드

jobs:
  build-scan-push:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image (push 전 로컬 빌드)
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true  # 로컬에 이미지 로드 (push 없이)
          tags: my-app:scan-target

      - name: Run Trivy vulnerability scan
        id: trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: my-app:scan-target
          format: table         # 콘솔 출력
          exit-code: 1          # CRITICAL 발견 시 exit 1 (빌드 실패)
          severity: CRITICAL,HIGH  # 이 심각도 이상만 실패 처리
          ignore-unfixed: true  # 패치 없는 취약점은 무시 (선택)
          vuln-type: os,library # OS 패키지 + 라이브러리 취약점 모두 스캔
          timeout: 10m

      - name: Run Trivy scan (SARIF 출력 - GitHub Security tab)
        if: always()  # 스캔 실패해도 SARIF는 업로드
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: my-app:scan-target
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH,MEDIUM
          ignore-unfixed: true

      - name: Upload SARIF to GitHub Security tab
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
          category: container-image

      - name: Configure AWS credentials (OIDC)
        if: github.event_name != 'pull_request'
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-2

      - name: Login to ECR and push
        if: github.event_name != 'pull_request'
        run: |
          GIT_SHA=$(git rev-parse --short HEAD)
          ECR_URI="${{ secrets.PROD_ACCOUNT_ID }}.dkr.ecr.ap-northeast-2.amazonaws.com/my-app"

          aws ecr get-login-password --region ap-northeast-2 | \
            docker login --username AWS --password-stdin "${ECR_URI}"

          docker tag my-app:scan-target "${ECR_URI}:${GIT_SHA}"
          docker push "${ECR_URI}:${GIT_SHA}"
```

### 2-2. 취약점 리포트 Slack 알림

```yaml
      - name: Parse Trivy results and notify Slack
        if: failure() && steps.trivy.outcome == 'failure'
        run: |
          # JSON 출력 재스캔 후 파싱
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest image \
            --format json \
            --severity CRITICAL \
            --quiet \
            my-app:scan-target > trivy-report.json

          CRITICAL_COUNT=$(cat trivy-report.json | \
            python3 -c "import json,sys; data=json.load(sys.stdin); \
            print(sum(len([v for v in r.get('Vulnerabilities',[]) if v['Severity']=='CRITICAL']) \
            for r in data.get('Results',[])))")

          curl -X POST "${{ secrets.SLACK_WEBHOOK_URL }}" \
            -H 'Content-type: application/json' \
            --data "{
              \"text\": \"🔴 *보안 스캔 실패*: \`my-app\`에서 CRITICAL 취약점 ${CRITICAL_COUNT}개 발견\n빌드: <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|#${{ github.run_number }}>\"
            }"
```

---

## 3. ECR Inspector 설정

### 3-1. ECR Inspector 활성화

```hcl
# terraform/prod-account/inspector.tf

# Inspector v2 활성화 (계정 레벨)
resource "aws_inspector2_enabler" "all" {
  account_ids    = [data.aws_caller_identity.current.account_id]
  resource_types = ["ECR", "EC2", "LAMBDA"]
}

# ECR 리포지토리에 스캔 활성화 (push 시 자동 스캔)
resource "aws_ecr_repository" "app" {
  name = "my-app"

  image_scanning_configuration {
    scan_on_push = true  # push 시 자동 스캔 (Enhanced Scanning with Inspector)
  }
}
```

### 3-2. CRITICAL 취약점 발견 시 EventBridge → SNS 알람

```hcl
# terraform/prod-account/ecr-scan-alert.tf

# ECR Inspector 스캔 결과 이벤트 규칙
resource "aws_cloudwatch_event_rule" "ecr_critical_finding" {
  name        = "ecr-critical-vulnerability"
  description = "ECR 이미지에서 CRITICAL 취약점 발견 시 알람"
  event_bus_name = "default"

  event_pattern = jsonencode({
    source      = ["aws.inspector2"]
    detail-type = ["Inspector2 Finding"]
    detail = {
      severity = ["CRITICAL"]
      status   = ["ACTIVE"]
      type     = ["PACKAGE_VULNERABILITY"]
      resources = {
        type = ["AWS_ECR_CONTAINER_IMAGE"]
      }
    }
  })
}

resource "aws_cloudwatch_event_target" "ecr_finding_lambda" {
  rule      = aws_cloudwatch_event_rule.ecr_critical_finding.name
  target_id = "NotifySlack"
  arn       = aws_lambda_function.security_notifier.arn
}
```

### 3-3. CRITICAL 발견 시 자동 이미지 격리 (Lambda)

```python
# lambda/ecr-scan-responder/handler.py
# Inspector 취약점 발견 → ECR 이미지 태그 제거 (격리)

import boto3
import json
import os

ecr_client = boto3.client('ecr', region_name='ap-northeast-2')

def handler(event, context):
    """
    Inspector2 CRITICAL Finding 발견 시 해당 이미지를 격리
    """
    print(f"Event: {json.dumps(event)}")

    detail = event.get('detail', {})
    severity = detail.get('severity', '')

    # CRITICAL만 처리
    if severity != 'CRITICAL':
        return {'statusCode': 200, 'body': 'Non-critical finding, skipping'}

    # 취약한 이미지 정보 추출
    resources = detail.get('resources', [])
    for resource in resources:
        if resource.get('type') == 'AWS_ECR_CONTAINER_IMAGE':
            details = resource.get('details', {}).get('awsEcrContainerImage', {})
            repository_name = details.get('repositoryName')
            image_digest = details.get('imageDigest')
            image_tags = details.get('imageTags', [])

            print(f"CRITICAL vulnerability in {repository_name}:{image_tags} ({image_digest})")

            # 이미지에 quarantine 태그 추가
            try:
                ecr_client.put_image(
                    repositoryName=repository_name,
                    imageTag='QUARANTINED',
                    imageManifest=get_image_manifest(repository_name, image_digest)
                )
                print(f"Image quarantined: {repository_name}@{image_digest}")
            except Exception as e:
                print(f"Failed to quarantine image: {e}")

    return {'statusCode': 200, 'body': 'Processing complete'}

def get_image_manifest(repository_name, image_digest):
    response = ecr_client.batch_get_image(
        repositoryName=repository_name,
        imageIds=[{'imageDigest': image_digest}]
    )
    return response['images'][0]['imageManifest']
```

---

## 4. .trivyignore (예외 처리)

```
# .trivyignore
# 취약점 CVE ID를 나열하면 해당 CVE 무시 (반드시 사유 주석 필수)

# CVE-2023-12345: OpenSSL 취약점
# 사유: 패치 버전이 아직 배포 안 됨, 2024-03-01 재검토 예정
# 영향 분석: 외부 네트워크 접근 없는 내부 서비스, 익스플로잇 조건 미충족
CVE-2023-12345

# CVE-2023-67890: libssl 취약점
# 사유: Alpine 패키지 upstream 패치 대기 중
# Ticket: JIRA-1234
CVE-2023-67890
```

---

## 5. Dockerfile 보안 베스트 프랙티스

```dockerfile
# Dockerfile - 보안 강화 예시

# 1. 특정 버전 고정 (latest 금지)
FROM node:20.11.0-alpine3.19 AS builder

WORKDIR /app

# 2. 루트가 아닌 사용자로 실행
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# 3. 의존성 먼저 설치 (캐시 활용)
COPY package*.json ./
RUN npm ci --only=production

# 4. 소스 코드 복사
COPY --chown=appuser:appgroup . .

# 5. Multi-stage build로 빌드 도구 제외
FROM node:20.11.0-alpine3.19

# 6. 불필요한 패키지 제거 (취약점 표면 축소)
RUN apk del apk-tools && \
    rm -rf /var/cache/apk/*

WORKDIR /app

COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist

# 7. 비루트 사용자로 전환
USER appuser

# 8. 비특권 포트 사용 (1024 이상)
EXPOSE 8080

# 9. read-only filesystem (securityContext에서 설정)
# readOnlyRootFilesystem: true

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/health/live || exit 1

CMD ["node", "dist/main.js"]
```

---

## 6. 트러블슈팅

### 증상 1: Trivy 스캔이 항상 CRITICAL로 실패하지만 실제 영향 없음

**원인**: OS 기본 패키지(musl, busybox 등)의 CVE가 CRITICAL이지만 실제 익스플로잇 경로 없음.

**해결 방법**:
```bash
# 상세 정보로 실제 영향 분석
docker run --rm aquasec/trivy:latest image \
  --format json \
  --severity CRITICAL \
  my-app:latest | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
for result in data.get('Results', []):
    for vuln in result.get('Vulnerabilities', []):
        if vuln['Severity'] == 'CRITICAL':
            print(f\"CVE: {vuln['VulnerabilityID']}\")
            print(f\"Package: {vuln['PkgName']}@{vuln['InstalledVersion']}\")
            print(f\"Fixed: {vuln.get('FixedVersion', 'N/A')}\")
            print(f\"Description: {vuln.get('Description', '')[:200]}\")
            print()
"

# 패치 버전이 있으면 Dockerfile에서 base image 업그레이드
# 없으면 .trivyignore에 사유와 함께 추가
```

---

### 증상 2: ECR Inspector 스캔 결과가 push 후 즉시 나타나지 않음

**원인**: Inspector Enhanced Scanning은 ECR push 후 이미지 분석에 수분 소요.

**해결 방법**:
```bash
# 스캔 결과 확인 (CLI)
aws inspector2 list-findings \
  --filter-criteria '{
    "resourceType": [{"comparison": "EQUALS", "value": "AWS_ECR_CONTAINER_IMAGE"}],
    "ecrImageRepositoryName": [{"comparison": "EQUALS", "value": "my-app"}]
  }' \
  --region ap-northeast-2 \
  --query 'findings[*].{Severity:severity,Title:title,CVE:packageVulnerabilityDetails.vulnerabilityId}'
```

---

## 7. 모니터링 및 알람

```bash
# Inspector 취약점 대시보드 (AWS Console)
# Inspector > Dashboard > ECR Repositories

# CLI로 CRITICAL 취약점 현황 조회
aws inspector2 get-findings-statistics \
  --resource-type ECR_IMAGE \
  --region ap-northeast-2

# ECR 취약점 개수 CloudWatch 커스텀 지표 발행
aws cloudwatch put-metric-data \
  --namespace "Security/ECR" \
  --metric-name "CriticalVulnerabilities" \
  --value <COUNT> \
  --unit Count \
  --dimensions RepositoryName=my-app \
  --region ap-northeast-2
```

---

## 8. TIP

**TIP 1**: `trivy image --ignore-unfixed` 옵션으로 패치가 없는 취약점은 스캔 결과에서 제외. CI 실패를 줄이되 수정 가능한 취약점에 집중.

**TIP 2**: Trivy DB를 GitHub Actions 캐시에 저장하면 매 빌드마다 DB 다운로드 불필요 (빌드 시간 단축, 외부 네트워크 의존도 감소).

```yaml
- name: Cache Trivy DB
  uses: actions/cache@v4
  with:
    path: ~/.cache/trivy
    key: trivy-db-${{ github.run_id }}
    restore-keys: trivy-db-

- name: Run Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: my-app:scan-target
    cache-dir: ~/.cache/trivy
```

**TIP 3**: 정기 스캔 (Weekly Cron)을 별도 워크플로우로 설정하여 기존 ECR 이미지의 신규 CVE 감지. 새로운 취약점은 이미지 재빌드 없이도 발견 가능.

```yaml
on:
  schedule:
    - cron: '0 9 * * 1'  # 매주 월요일 오전 9시 (KST 기준 UTC 0시)
```

**관련 문서**
- [Secrets 관리](./secrets-in-pipeline.md)
- [이미지 태깅 전략](../01-cicd/cicd-image-tagging-strategy.md)
- [GitHub Actions + ECR 파이프라인](../01-cicd/cicd-github-actions-ecr.md)
