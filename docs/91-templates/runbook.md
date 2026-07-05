# Runbook: {작업/장애 제목}

<!--
Runbook 사용 가이드:
- 실제 장애 또는 운영 작업 시 이 템플릿 복사하여 사용
- 모든 단계를 순서대로 수행하며 체크박스 체크
- 작업 완료 후 실제 소요 시간 및 메모 기록
-->

| 항목 | 내용 |
|------|------|
| 작업 유형 | {장애 대응 / 정기 유지보수 / 긴급 배포 / 롤백} |
| 대상 환경 | {prod / staging / dev} |
| 예상 소요 시간 | {30분} |
| 다운타임 | {있음 / 없음 / 예상: X분} |
| 작성자 | {이름} |
| 최종 수정일 | {YYYY-MM-DD} |
| 승인자 | {이름 (장애 대응 시 생략 가능)} |

---

## 사전 조건

### 필요 권한

- [ ] AWS Console 또는 CLI 접근 권한 (`<AWS_PROFILE>` 프로파일)
- [ ] kubectl 접근 권한 (`kubectl config use-context <CONTEXT_NAME>`)
- [ ] ArgoCD 접근 권한 (admin 또는 `<PROJECT>` 프로젝트 sync 권한)
- [ ] GitHub 접근 권한 (GitOps repo write)

### 필요 도구

```bash
# 도구 버전 확인
aws --version          # 2.x 이상
kubectl version        # 1.29+ 클라이언트
helm version           # v3.x
argocd version         # 2.x
```

### 알림

- [ ] Slack `#devops` 채널에 작업 시작 알림
- [ ] PagerDuty 또는 온콜 담당자에게 사전 공지 (장애 대응 시)

---

## 실행 절차

### Phase 1: 사전 확인

- [ ] **1.1** 현재 서비스 상태 확인
  ```bash
  # EKS 배포 상태
  kubectl get pods -n <NAMESPACE> -o wide
  kubectl get deployment <DEPLOYMENT_NAME> -n <NAMESPACE>
  ```
  현재 상태 메모: _______________

- [ ] **1.2** ArgoCD 동기화 상태 확인
  ```bash
  argocd app get <APP_NAME>
  ```

- [ ] **1.3** 현재 이미지 태그 확인 (롤백 기준점)
  ```bash
  kubectl get deployment <DEPLOYMENT_NAME> -n <NAMESPACE> \
    -o jsonpath='{.spec.template.spec.containers[0].image}'
  ```
  현재 이미지: _______________

- [ ] **1.4** 최근 배포 이력 확인
  ```bash
  kubectl rollout history deployment/<DEPLOYMENT_NAME> -n <NAMESPACE>
  argocd app history <APP_NAME>
  ```

---

### Phase 2: 본 작업

- [ ] **2.1** {첫 번째 작업 단계}
  ```bash
  {명령어}
  ```
  결과 확인: _______________

- [ ] **2.2** {두 번째 작업 단계}
  ```bash
  {명령어}
  ```

- [ ] **2.3** ArgoCD Sync (해당 시)
  ```bash
  argocd app sync <APP_NAME>
  argocd app wait <APP_NAME> --health --timeout 300
  ```

---

### Phase 3: 검증

- [ ] **3.1** Pod 정상 기동 확인
  ```bash
  kubectl get pods -n <NAMESPACE> -l app=<APP_NAME>
  # 기대: STATUS=Running, RESTARTS 증가 없음
  ```

- [ ] **3.2** 서비스 헬스 체크
  ```bash
  # 내부 헬스 체크
  kubectl exec -it <POD_NAME> -n <NAMESPACE> -- \
    wget -qO- http://localhost:8080/health/ready

  # 외부 엔드포인트 확인
  curl -sf https://<DOMAIN>/health
  ```

- [ ] **3.3** 로그 오류 확인
  ```bash
  kubectl logs -n <NAMESPACE> -l app=<APP_NAME> --tail=100 \
    | grep -E "ERROR|CRITICAL|Exception"
  ```

- [ ] **3.4** 주요 메트릭 확인
  - CloudWatch Dashboard: {대시보드 URL}
  - 확인 항목: {에러율, 레이턴시, CPU, 메모리}

---

### 롤백 절차 (이상 발생 시)

- [ ] **R.1** ArgoCD 즉시 이전 버전으로 롤백
  ```bash
  # ArgoCD 이전 revision 확인
  argocd app history <APP_NAME>

  # 롤백 실행
  argocd app rollback <APP_NAME> <PREV_REVISION>
  argocd app wait <APP_NAME> --health --timeout 180
  ```

- [ ] **R.2** kubectl 롤백 (ArgoCD 미사용 또는 긴급 시)
  ```bash
  kubectl rollout undo deployment/<DEPLOYMENT_NAME> -n <NAMESPACE>
  kubectl rollout status deployment/<DEPLOYMENT_NAME> -n <NAMESPACE>
  ```

- [ ] **R.3** 롤백 후 GitOps repo 동기화
  ```bash
  # GitOps repo에서도 이전 이미지 태그로 revert
  cd gitops-repo
  git revert HEAD  # 또는 특정 커밋 revert
  git push origin main
  ```

- [ ] **R.4** ArgoCD auto-sync 재활성화 (수동 롤백 후)
  ```bash
  argocd app set <APP_NAME> --sync-policy automated --self-heal
  ```

---

## 완료 보고

### 작업 결과

| 항목 | 내용 |
|------|------|
| 시작 시간 | {HH:MM} |
| 종료 시간 | {HH:MM} |
| 실제 소요 시간 | {분} |
| 다운타임 | {있음/없음, 시간} |
| 결과 | {성공 / 실패 / 부분 성공} |

### 확인 사항

- [ ] 서비스 정상 운영 확인
- [ ] 모니터링 알람 없음 확인
- [ ] Slack 작업 완료 알림

### 이슈 및 메모

```
{작업 중 발생한 이슈, 다음 번에 개선할 사항}
```

### 관련 링크

- ArgoCD 앱: {URL}
- GitHub Actions 실행: {URL}
- CloudWatch 대시보드: {URL}
- Slack 스레드: {URL}

---

## 버전 이력

| 날짜 | 변경 내용 | 작성자 |
|------|-----------|--------|
| {YYYY-MM-DD} | 최초 작성 | {이름} |
