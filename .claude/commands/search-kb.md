# /search-kb — 지식 베이스 검색

키워드로 지식 베이스를 검색하고 관련 문서와 내용을 반환하는 커맨드.

## 사용법

```
/search-kb {키워드1} {키워드2} ...
```

## 예시

```
/search-kb ArgoCD IRSA
/search-kb ECR cross-account push
/search-kb 이미지 태그 전략
/search-kb GitHub Actions OIDC
/search-kb Terraform state lock
/search-kb 롤백 방법
```

## 실행 지침

이 커맨드를 받으면 아래 절차를 따릅니다:

1. 키워드를 기반으로 `docs/` 디렉토리 전체 문서 탐색
2. 관련 문서 목록과 각 문서에서 해당 내용이 있는 섹션 반환
3. 가장 관련성 높은 내용을 요약하여 제시
4. 추가 참조 문서 링크 제공

## 출력 형식

```markdown
## 검색 결과: {키워드}

### 관련 문서

| 문서 | 관련 섹션 | 관련도 |
|------|-----------|--------|
| [cicd-multi-account-architecture.md](docs/cicd/cicd-multi-account-architecture.md) | 3-3. GitHub OIDC 방식 | ★★★ |
| [cicd-github-actions-ecr.md](docs/cicd/cicd-github-actions-ecr.md) | 2. OIDC 방식 | ★★★ |
| [secrets-in-pipeline.md](docs/security/secrets-in-pipeline.md) | 3. OIDC로 AWS 인증 | ★★ |

### 핵심 내용 요약

{키워드 관련 핵심 내용을 2-3단락으로 요약}

### 바로 사용 가능한 코드

\`\`\`{언어}
{가장 관련성 높은 코드 예시}
\`\`\`

### 추가 참조

- {관련 주제 및 문서 링크}
```

## 검색 범위

| 카테고리 | 키워드 예시 |
|----------|-------------|
| CI/CD | GitHub Actions, ECR push, OIDC, cicd-user, 이미지 빌드 |
| GitOps | ArgoCD, IRSA, sync, App of Apps, ApplicationSet |
| 배포 | Rolling, Blue/Green, Canary, Helm, Argo Rollouts |
| IaC | Terraform, state, plan, apply, Atlantis |
| 보안 | Trivy, Inspector, Secrets, CVE, 취약점 |
