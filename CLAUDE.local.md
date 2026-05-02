# CLAUDE.local.md — 로컬 개인 설정

> 이 파일은 .gitignore에 등록되어 있으며 개인 환경에만 적용됩니다.
> 팀 공유 설정은 CLAUDE.md를 수정하세요.

---

## 로컬 AWS 환경

```bash
# 기본 사용 AWS 프로파일
AWS_PROFILE=default
AWS_DEFAULT_REGION=ap-northeast-2

# 멀티 계정 프로파일 (필요 시 수정)
# AWS_PROFILE_CICD=cicd-account
# AWS_PROFILE_PROD=prod-account
# AWS_PROFILE_STAGING=staging-account
```

## 로컬 클러스터 접근 설정

```bash
# kubeconfig 경로
KUBECONFIG=~/.kube/config

# EKS 클러스터 컨텍스트 업데이트 (필요 시)
# aws eks update-kubeconfig --region ap-northeast-2 --name <CLUSTER_NAME> --profile prod-account
```

## 개인 작업 스타일 선호

- 문서 초안은 항상 한국어로 작성 후 영어 원문 병기
- CLI 예시는 `ap-northeast-2` 리전 기준으로 작성
- Terraform 예시는 provider `~> 5.0` 기준
- GitHub Actions 예시는 `ubuntu-latest` 기준
- ArgoCD 예시는 `v2.x` 기준

## 로컬 도구 경로

```bash
# 로컬 설치 도구 (필요 시 수정)
# TERRAFORM_BIN=/usr/local/bin/terraform
# KUBECTL_BIN=/usr/local/bin/kubectl
# HELM_BIN=/usr/local/bin/helm
# ARGOCD_BIN=/usr/local/bin/argocd
# TRIVY_BIN=/usr/local/bin/trivy
```

## 환경별 계정 ID (플레이스홀더)

```bash
# 실제 계정 ID로 교체 필요
CICD_ACCOUNT_ID=<CICD_ACCOUNT_ID>
PROD_ACCOUNT_ID=<PROD_ACCOUNT_ID>
STAGING_ACCOUNT_ID=<STAGING_ACCOUNT_ID>
```

## 작업 메모 (임시)

