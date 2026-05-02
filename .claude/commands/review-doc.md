# /review-doc — 문서 품질 검토

지정한 문서를 검토하고 품질 개선 사항을 제시하는 커맨드.

## 사용법

```
/review-doc {파일 경로}
```

## 예시

```
/review-doc docs/cicd/cicd-multi-account-architecture.md
/review-doc docs/gitops/argocd-eks-deployment.md
/review-doc docs/security/devsecops-image-scan.md
```

## 검토 항목

이 커맨드를 받으면 아래 기준으로 문서를 검토합니다:

### 구조 검토

- [ ] 5개 필수 섹션 포함 여부 (개요, 설명, 트러블슈팅, 모니터링, TIP)
- [ ] 섹션 순서 올바른지
- [ ] 관련 문서 링크 포함 여부

### 내용 검토

- [ ] 코드 블록에 언어 태그 있는지
- [ ] 플레이스홀더 `<YOUR_VALUE>` 형식 사용 여부
- [ ] 실행 가능한 코드 예시 포함 여부
- [ ] 트러블슈팅 최소 2개 이상 포함 여부
- [ ] 모니터링 섹션에 실제 명령어/설정 포함 여부

### 보안 검토 (`rules/security-checklist.md` 기준)

- [ ] Access Key 하드코딩 없는지
- [ ] OIDC 방식 권장 여부
- [ ] 최소 권한 IAM Policy 예시 사용 여부
- [ ] `latest` 태그 사용 금지 준수 여부
- [ ] 보안 고려사항 섹션 포함 여부

### 문체 검토 (`rules/doc-writing.md` 기준)

- [ ] 명사형 종결어미 사용 (`~함`, `~사용`)
- [ ] 추측성 표현 없는지 (`~할 수 있습니다` 금지)
- [ ] 기술 용어 영어 원문 병기 여부

## 출력 형식

```markdown
## 검토 결과: {파일 경로}

### 🔴 필수 수정 (Critical)
- {항목}: {구체적인 문제와 개선안}

### 🟡 개선 권장 (Warning)
- {항목}: {개선 제안}

### 🟢 잘된 점 (Good)
- {항목}: {칭찬 내용}

### 종합 점수: {X}/10

### 수정 제안 코드
\`\`\`markdown
{수정된 섹션 예시}
\`\`\`
```
