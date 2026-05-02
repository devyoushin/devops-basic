# DevOps 보안 체크리스트 (DevOps Security Checklist)

CI/CD 파이프라인, GitOps, 컨테이너 운영 시 반드시 확인해야 할 보안 항목.

---

## 1. IAM / 접근 제어

### cicd-user 체크리스트

- [ ] ECR push 전용 최소 권한만 부여 (EKS, EC2, S3 등 접근 불가)
- [ ] Access Key는 GitHub Secrets에만 저장 (코드에 하드코딩 금지)
- [ ] 90일 주기 Access Key 로테이션
- [ ] OIDC 방식으로 마이그레이션 계획 수립 (Access Key 제거 목표)
- [ ] CloudTrail에서 cicd-user 활동 로그 감사

### ArgoCD IRSA 체크리스트

- [ ] IRSA Role Trust Policy에 정확한 ServiceAccount 조건 명시
  - `system:serviceaccount:argocd:argocd-server` (네임스페이스:SA 이름)
- [ ] IAM Policy는 `eks:DescribeCluster`, `eks:ListClusters`만 포함
- [ ] aws-auth ConfigMap 또는 EKS Access Entry에 IRSA Role 등록 확인
- [ ] kubeconfig Secret 방식 미사용 (IRSA로 대체)

### GitHub OIDC 체크리스트

- [ ] Trust Policy의 `sub` 조건이 특정 리포지토리/브랜치로 제한됨
  - 와일드카드 `repo:*:*` 금지, 정확한 리포지토리 명시
- [ ] 각 용도별 별도 IAM Role (배포용, Terraform용 분리)
- [ ] `aud` 조건 = `sts.amazonaws.com` 확인

---

## 2. 컨테이너 이미지 보안

### 빌드 시점

- [ ] Base image 버전 고정 (`node:20.11.0-alpine3.19`, `latest` 금지)
- [ ] Multi-stage build로 빌드 도구 최종 이미지에서 제외
- [ ] 루트가 아닌 사용자로 컨테이너 실행 (`USER 1001`)
- [ ] `readOnlyRootFilesystem: true` 설정 (임시 파일이 필요하면 emptyDir 마운트)
- [ ] 불필요한 패키지 설치 금지 (curl, wget 등 공격 도구 제외)

### 이미지 태그

- [ ] `latest` 태그 운영 환경 사용 금지
- [ ] git-sha 기반 고정 태그 사용
- [ ] ECR Lifecycle Policy로 이미지 수 관리 (스토리지 절약, 노출 최소화)

### 취약점 스캔

- [ ] CI 파이프라인에서 이미지 빌드 후 Trivy/ECR Inspector 스캔
- [ ] CRITICAL 취약점 발견 시 빌드 실패 처리 (`exit-code: 1`)
- [ ] `.trivyignore` 예외 등록 시 반드시 사유 + 재검토 일정 주석 포함
- [ ] 주간 정기 스캔으로 기존 이미지의 신규 CVE 감지

---

## 3. Secrets 관리

### 저장 및 전달

- [ ] GitHub Actions Secrets에 자격 증명 저장 (코드 파일 내 하드코딩 금지)
- [ ] `.env` 파일 `.gitignore` 등록 확인
- [ ] Terraform State에 민감 정보 포함 시 S3 버킷 암호화 + 접근 제한
- [ ] K8s Secret은 etcd 암호화 활성화 또는 External Secrets Operator 사용

### 파이프라인 Secrets

- [ ] OIDC 방식으로 AWS 인증 (Access Key 불필요 환경에서는 완전 제거)
- [ ] GitHub Actions 로그에서 Secrets 값 노출 여부 확인
  - `echo ${{ secrets.xxx }}` 패턴 코드 검색
- [ ] Secrets 로테이션 주기 문서화 (Access Key 90일, DB 비밀번호 90일)

### Git 리포지토리

- [ ] `git-secrets` 또는 `gitleaks` pre-commit hook 설치
- [ ] GitHub Secret Scanning 활성화 (Organization 레벨)
- [ ] 비밀 노출 감지 시 즉시 로테이션 절차 (Runbook) 보유

---

## 4. GitOps 보안

### 리포지토리 접근 제어

- [ ] GitOps 리포지토리와 소스 리포지토리 분리
- [ ] `CODEOWNERS` 파일로 prod overlay 승인자 지정
- [ ] Branch Protection 활성화 (main/prod 브랜치에 직접 push 금지)
- [ ] PR 승인 없이 merge 불가 설정 (Required reviewers: 1+)

### ArgoCD 보안

- [ ] ArgoCD admin 비밀번호 초기 설정 후 즉시 변경
- [ ] SSO (GitHub OIDC, SAML) 연동으로 개인 계정 로그인 대체
- [ ] RBAC Policy로 팀별 애플리케이션 접근 제한
- [ ] ArgoCD API 서버 외부 노출 금지 (Internal ALB 또는 VPN 뒤)
- [ ] `selfHeal: true` 설정으로 수동 변경 자동 복구

---

## 5. EKS / Kubernetes 보안

### 클러스터 레벨

- [ ] Kubernetes API Server 퍼블릭 엔드포인트 비활성화 또는 IP 제한
- [ ] EKS 최신 버전 유지 (End of Support 전 업그레이드)
- [ ] `aws-auth` ConfigMap 또는 EKS Access Entry 정기 감사

### Pod 레벨

- [ ] NetworkPolicy로 Pod 간 기본 격리 (`Default Deny All`)
- [ ] Pod Security Admission으로 특권 컨테이너 금지
  - `pod-security.kubernetes.io/enforce: restricted`
- [ ] `runAsRoot: false` 강제
- [ ] Resource Requests/Limits 필수 설정 (OOMKilled 방지)
- [ ] ServiceAccount Token 자동 마운트 비활성화 (불필요한 SA)
  ```yaml
  automountServiceAccountToken: false
  ```

### IRSA / 권한

- [ ] IRSA Role은 최소 권한 (해당 Pod가 필요한 AWS 서비스만)
- [ ] ServiceAccount 어노테이션에 정확한 Role ARN 지정
- [ ] 사용하지 않는 ServiceAccount 삭제

---

## 6. IaC (Terraform) 보안

- [ ] Terraform State S3 버킷: 퍼블릭 접근 차단, KMS 암호화
- [ ] State에 민감 정보 포함 시 접근 IAM Policy 최소화
- [ ] `prevent_destroy = true` 중요 리소스에 설정
- [ ] Terraform Plan 결과 코드 리뷰에 포함 (PR comment)
- [ ] `terraform validate` + `tflint` CI 통합

---

## 7. 감사 및 모니터링

- [ ] CloudTrail 전체 리전 활성화 (S3에 90일+ 보관)
- [ ] ECR push/pull 이벤트 CloudTrail 기록 확인
- [ ] ArgoCD sync 이벤트 로그 보관
- [ ] 비정상 배포 패턴 알람 (off-hours 배포, 알 수 없는 이미지 태그)
- [ ] AWS Inspector CRITICAL 취약점 EventBridge → Slack 알람

---

## 8. 인시던트 대응

### 자격 증명 유출 시

```
1. 유출된 키 즉시 비활성화 (30분 이내)
   aws iam update-access-key --status Inactive

2. CloudTrail에서 유출 키 사용 이력 조회 (무엇을 했는지 확인)

3. 피해 범위 확인 (영향받은 리소스, 데이터)

4. 새 키 생성 및 파이프라인 업데이트

5. 구키 삭제

6. 인시던트 보고서 작성 (원인, 영향, 재발 방지)
```

### 컨테이너 이미지 CRITICAL 취약점 발견 시

```
1. 영향받은 이미지 버전 파악

2. 취약점 익스플로잇 가능성 평가 (CVSS 스코어, 접근 경로)

3. 패치 버전 확인

4. Dockerfile base image 업데이트 및 재빌드

5. 취약한 버전 ECR에서 삭제 또는 QUARANTINED 태그 처리

6. 배포 환경 이미지 업데이트 (GitOps repo image tag 변경)
```
