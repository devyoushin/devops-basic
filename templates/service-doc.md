# {서비스/주제 이름} ({English Name})

<!--
사용법:
1. 이 파일을 docs/{카테고리}/{서비스}-{주제}.md로 복사
2. {} 항목을 실제 내용으로 교체
3. 해당 없는 섹션은 "해당 없음" 또는 삭제
-->

## 1. 개요

{무엇인지, 왜 필요한지 2-3문장으로 설명}

**{비교 대상이 있을 경우} 비교**

| 항목 | {방식 A} | {방식 B} |
|------|----------|----------|
| {항목 1} | {값} | {값} |
| {항목 2} | {값} | {값} |
| 권장 환경 | {상황} | {상황} |

---

## 2. 아키텍처 다이어그램 (선택)

```
{ASCII art 다이어그램 - 복잡한 플로우가 있을 경우}
```

---

## 3. 사전 요구사항

- {필요한 IAM 권한}
- {필요한 도구 버전}
- {필요한 설정}

---

## 4. 설정 방법

### 4-1. {첫 번째 단계}

```bash
# {단계 설명}
{명령어}
```

### 4-2. Terraform 코드

```hcl
# terraform/{환경}/main.tf

resource "aws_{리소스}" "{이름}" {
  # {설명}
  {속성} = "{값}"

  tags = {
    Environment = "<ENV>"
    ManagedBy   = "terraform"
  }
}
```

### 4-3. GitHub Actions 코드 (해당 시)

```yaml
# .github/workflows/{workflow-name}.yaml

name: {워크플로우 이름}

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  {job-name}:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-2
```

### 4-4. Kubernetes 매니페스트 (해당 시)

```yaml
# {리소스 설명}

apiVersion: {apiVersion}
kind: {Kind}
metadata:
  name: {이름}
  namespace: {네임스페이스}
spec:
  # {주요 설정}
```

---

## 5. 검증 방법

```bash
# {정상 동작 확인 방법}
{명령어}

# 기대 출력
# {예상 출력 값}
```

---

## 6. 트러블슈팅

### 증상 1: {증상 제목}

```
{실제 에러 메시지}
```

**원인**: {근본 원인 설명}

**해결 방법**:
```bash
# {해결 명령어}
{명령어}
```

---

### 증상 2: {증상 제목}

**원인**: {근본 원인 설명}

**해결 방법**:
```bash
{명령어}
```

---

## 7. 모니터링 및 알람

```bash
# {모니터링 명령어 또는 설정}
{명령어 또는 Terraform 코드}
```

```hcl
# CloudWatch 알람 (해당 시)
resource "aws_cloudwatch_metric_alarm" "{이름}" {
  alarm_name          = "{알람 이름}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "{지표 이름}"
  namespace           = "{네임스페이스}"
  period              = 300
  statistic           = "Average"
  threshold           = {임계값}
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

---

## 8. TIP

**TIP 1**: {현장 팁 1 - 실제 운영 경험 기반}

**TIP 2**: {현장 팁 2}

**관련 문서**
- [{관련 문서 제목}]({상대 경로}.md)
- [{관련 문서 제목}](../{카테고리}/{파일}.md)
