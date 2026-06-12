# 5단계: GitOps & ArgoCD 운영

## 학습 일자
2025-07-15

## 핵심 개념 정리

### GitOps란

한 문장으로: **"Git이 곧 클러스터의 상태다"**

Git 저장소가 인프라/앱의 유일한 진실 원천(Single Source of Truth).

```
전통적:  개발자 → kubectl apply → 클러스터 변경 (Git은 모름)
GitOps:  개발자 → Git Push → ArgoCD가 감지 → 클러스터 자동 변경
```

장점:
- **변경 추적**: 누가 언제 뭘 바꿨는지 Git 히스토리로 전부 남음
- **롤백 = git revert**: 이전 커밋으로 되돌리면 클러스터도 되돌아감
- **Self-Heal**: kubectl로 직접 수정해도 ArgoCD가 Git 상태로 원복

### ArgoCD Application

ArgoCD의 핵심 단위. "이 Git 경로를 저 클러스터에 동기화해라"라는 선언.

현재 프로젝트 기준:
```
ArgoCD Application 설정:
  source:  apps/swagger-hub/overlays/dev/    ← Git의 이 경로를
  destination:  swagger-dev namespace        ← 이 클러스터/네임스페이스에 동기화
```

```yaml
# ArgoCD Application 예시
spec:
  source:
    repoURL: http://gitlab.example.com/myorg/sample-k8s-deploy.git
    path: apps/swagger-hub/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: swagger-dev
  syncPolicy:
    automated:
      prune: true       # Git에서 삭제한 리소스 → 클러스터에서도 삭제
      selfHeal: true    # kubectl 직접 수정 → Git 상태로 되돌림
```

### Sync 상태 — 이것만 알면 된다

| 상태 | 의미 | 대응 |
|------|------|------|
| Synced + Healthy | Git = 클러스터, 정상 | 할 일 없음 ✅ |
| OutOfSync | Git ≠ 클러스터 | 새 배포가 있거나 누가 수동 변경함 |
| Progressing | 동기화 진행 중 | 기다리면 됨 |
| Synced + Degraded | YAML 적용 성공, Pod 비정상 | Pod 로그/이벤트 확인 ⚠️ |

### syncPolicy 주요 옵션

```yaml
syncPolicy:
  automated:
    prune: true       # Git에서 삭제한 리소스 → 클러스터에서도 삭제
    selfHeal: true    # 누가 kubectl로 직접 수정 → Git 상태로 되돌림
```

- `prune: false`면 → Git에서 ingress.yaml을 삭제해도 클러스터의 Ingress는 남아있음 ("유령 리소스")
- `selfHeal: false`면 → kubectl로 replicas를 5로 바꾸면 그대로 유지됨

---

## 현재 프로젝트 연결

### 실제 배포 흐름 (전체 추적) — 가장 중요

```
① 개발자: 소스 레포에 코드 Push
     ↓
② GitLab CI: Docker 빌드 → Harbor에 이미지 Push (예: 태그 abc12345)
     ↓
③ GitLab CI: 이 배포 레포(sample-k8s-deploy)의
             overlays/dev/kustomization.yaml에서
             newTag: bd467fae → newTag: abc12345 로 변경 후 Push
     ↓
④ ArgoCD: Git 변경 감지 (polling 3분 또는 webhook)
     ↓
⑤ ArgoCD: kustomize build → 최종 YAML 생성 → 클러스터와 비교 → OutOfSync
     ↓
⑥ ArgoCD: 자동 sync (automated 설정 시) → kubectl apply
     ↓
⑦ EKS: 새 Pod 생성 (abc12345 이미지) → 구 Pod 종료 (Rolling Update)
     ↓
⑧ ArgoCD: Synced + Healthy 상태로 복귀
```

**핵심 포인트**: 개발자가 직접 kubectl을 치는 순간이 없다. 소스 코드만 Push하면 CI/CD + ArgoCD가 전부 처리.

### 이 배포 레포의 역할

- 소스 코드 없음 — Kubernetes 매니페스트만 존재
- CI/CD가 이미지 태그(newTag)를 자동 업데이트
- ArgoCD가 이 레포를 감시하여 클러스터에 동기화
- 모든 배포 변경이 Git 커밋으로 추적됨

### bastion 서버 ArgoCD 환경 확인 (실습 시도)

bastion 접속하여 확인한 결과:
```
✅ kubectl 연결됨 (eks-cluster-name 클러스터)
✅ ArgoCD 클러스터에 설치됨 (모든 Pod Running)
   - argocd-application-controller
   - argocd-applicationset-controller
   - argocd-dex-server
   - argocd-notifications-controller
   - argocd-redis
   - argocd-repo-server
   - argocd-server
❌ ArgoCD CLI 미설치 → 추후 설치 후 실습 예정
```

접속 방법: `ssh -i ~/.ssh/eks-predev.pem ubuntu@52.78.108.45`

---

## 핵심 명령어 / 코드

### ArgoCD CLI 명령어

```bash
# ArgoCD CLI 로그인
argocd login <argocd-server-url>

# 앱 상태 확인 (가장 많이 씀)
argocd app get swagger-hub-dev

# 앱 목록 조회
argocd app list

# 수동 동기화 (automated가 아닐 때)
argocd app sync swagger-hub-dev

# Git vs 클러스터 차이 확인
argocd app diff swagger-hub-dev

# 배포 히스토리
argocd app history swagger-hub-dev

# 롤백
argocd app rollback swagger-hub-dev <revision-id>
```

### ArgoCD CLI 설치 (bastion에서 추후 실행)

```bash
curl -sSL -o /tmp/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 /tmp/argocd /usr/local/bin/argocd
argocd version --client
```

---

## 헷갈렸던 점 & 해결

- **Q**: Synced인데 Degraded면 뭐가 문제인가?
- **A**: Synced = Git의 YAML을 클러스터에 **성공적으로 적용**한 상태. Degraded = 적용된 리소스가 **정상 동작하지 않는** 상태. 즉 "YAML 적용은 성공했지만, Pod가 제대로 뜨지 않는 상황". 예:
  - 이미지 태그가 Harbor에 없어서 `ImagePullBackOff`
  - 앱 자체 버그로 `CrashLoopBackOff`
  - 메모리 부족으로 `OOMKilled`
  - ArgoCD/Kustomize 문제가 아니라 **앱/이미지 문제**임.

- **Q**: prune: true가 없으면 구체적으로 뭐가 문제?
- **A**: Git에서 리소스 파일을 삭제해도 클러스터에는 그대로 남아있음. 구체적 시나리오: base에서 `ingress.yaml`을 삭제하고 Git Push → ArgoCD가 sync → `prune: false`면 클러스터의 Ingress 리소스는 **그대로 남아있음** → Git에는 없는 "유령 리소스"가 클러스터에 계속 살아있게 됨.

- **Q**: selfHeal: true일 때 kubectl로 replicas를 5로 바꾸면?
- **A**: Git에 `replicas: 1`로 되어있으니 ArgoCD가 diff를 감지하고 Git 상태(replicas: 1)로 되돌린다.

---

## 학습 체크 결과 (4/5 통과)

| 질문 | 결과 | 피드백 |
|------|------|--------|
| GitOps의 Single Source of Truth와 전통적 방식 차이 | ✅ | Git을 근거로 sync를 맞춘 형상이 배포됨 |
| selfHeal: true일 때 kubectl scale 실행하면? | ✅ | Git과 diff 생기니까 Git 상태로 되돌림 |
| Synced인데 Degraded인 상황 | 🔶 | "sync 되지 않음"이 아니라 "YAML 적용은 성공했지만 Pod가 비정상". ImagePullBackOff, CrashLoopBackOff 등 |
| 새 이미지 배포 전체 흐름 | ✅ | 소스 Push → CI가 Harbor에 이미지 Push + 배포 레포 overlay 업데이트 → ArgoCD 감지 → 재배포 |
| prune: true 없으면 어떤 문제? | 🔶 | "Git에서 제거된 내용이 남아있을 수 있다"는 맞지만, 구체적 시나리오: ingress.yaml 삭제 후에도 클러스터의 Ingress가 유령 리소스로 남음 |

---

## 복습 포인트

1. GitOps = Git이 Single Source of Truth. 클러스터 상태는 항상 Git과 동기화. 직접 kubectl apply 하면 selfHeal이 되돌려버림.
2. ArgoCD Application = "이 Git 경로를 저 클러스터/namespace에 동기화해라"라는 선언. source(Git) + destination(클러스터)으로 구성.
3. **Synced + Degraded = YAML 적용 성공 + Pod 비정상.** ArgoCD 문제가 아니라 앱/이미지 문제. Pod 로그와 Events 확인 필요. (체크에서 틀린 부분)
4. selfHeal: true → kubectl 직접 수정을 Git 상태로 되돌림. prune: true → Git에서 삭제한 리소스를 클러스터에서도 삭제. **prune: false면 유령 리소스가 남음** (예: ingress.yaml 삭제해도 클러스터의 Ingress 살아있음).
5. 현재 프로젝트 배포 흐름: 소스 Push → GitLab CI가 Harbor에 이미지 Push + 배포 레포 newTag 업데이트 → ArgoCD 감지 → 자동 sync → Rolling Update. **개발자가 kubectl을 직접 치지 않음.**
6. 롤백 = git revert가 가장 안전. ArgoCD rollback도 가능하지만, Git 상태와 불일치가 생길 수 있음.
7. ArgoCD가 변경을 감지하는 방식: polling(기본 3분) 또는 webhook. OutOfSync 감지 후 automated sync가 설정되어 있으면 자동 적용.
8. bastion 서버에 ArgoCD CLI 미설치 상태. 추후 설치 후 `argocd app get`, `argocd app sync` 등 실습 필요.
