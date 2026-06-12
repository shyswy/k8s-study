# 3단계: kubectl 실전 조작

## 학습 일자
2025-07-15

## 핵심 개념 정리

kubectl은 K8s API Server에 명령을 보내는 CLI 도구.
kubeconfig(`~/.kube/config`)에 클러스터 접속 정보가 저장되어 있고,
`aws eks update-kubeconfig`로 EKS 클러스터를 등록한다.

## 현재 프로젝트 연결

- 클러스터: `eks-cluster-name` (ap-northeast-2)
- 노드: 2대 (v1.31.14, Amazon Linux 2023, containerd)
- Namespace: `argocd`(ArgoCD), `kube-system`(시스템), `default`(테스트)
- `swagger-dev` namespace는 아직 미생성 (swagger-hub 미배포 상태)
- 접속 방법: `ssh -i ~/.ssh/eks-predev.pem ubuntu@52.78.108.45`

## 현재 클러스터 상태 (실습 시점)

```
argocd namespace:
  - argocd-application-controller (StatefulSet)
  - argocd-applicationset-controller
  - argocd-dex-server
  - argocd-notifications-controller
  - argocd-redis
  - argocd-repo-server
  - argocd-server
  → 전부 Running ✅

kube-system namespace:
  - aws-load-balancer-controller (2개)
  - aws-node (DaemonSet, 노드당 1개)
  - coredns (2개)
  - kube-proxy (DaemonSet, 노드당 1개)
  - metrics-server (2개)

default namespace:
  - test-container (2개) → CrashLoopBackOff ❌ (MODULE_NOT_FOUND)
```

---

## 핵심 명령어 정리

### 클러스터 접근 설정

```bash
# EKS 클러스터에 kubeconfig 등록 (최초 1회)
aws eks update-kubeconfig --name <cluster-name> --region ap-northeast-2

# 현재 연결된 클러스터 확인
kubectl config current-context

# 등록된 모든 클러스터(컨텍스트) 목록
kubectl config get-contexts

# 다른 클러스터로 전환
kubectl config use-context <context-name>
```

kubeconfig 구조:
```
~/.kube/config
├── clusters: [클러스터 API 서버 주소 목록]
├── users: [인증 방법 목록]
└── contexts: [cluster + user 조합]
```

### 조회

```bash
kubectl get all -n <namespace>          # 전체 리소스 목록
kubectl get pods -n <ns>                # Pod 목록
kubectl get pods -o wide                # 노드/IP 포함
kubectl get pods -l app=<name>          # 라벨 필터링
kubectl get pods -A                     # 모든 namespace
kubectl get rs -n <ns>                  # ReplicaSet 목록

kubectl describe pod/<name> -n <ns>     # 상세 + Events (트러블슈팅 핵심)
kubectl logs <pod> -n <ns>              # 현재 로그
kubectl logs -f <pod>                   # 실시간 follow
kubectl logs --previous <pod>           # 이전 크래시 로그
kubectl logs --tail=100 <pod>           # 최근 100줄만
kubectl exec -it <pod> -- /bin/sh       # Pod 내부 접속
kubectl exec <pod> -- env               # 환경변수 확인
```

### 배포

```bash
kubectl apply -f <file.yaml>            # 매니페스트 적용
kubectl apply -k <overlay-dir>/         # Kustomize overlay 적용
kubectl apply -k <dir> --dry-run=client # 적용 안 하고 결과만 확인
kubectl kustomize <overlay-dir>/        # Kustomize 빌드 결과 미리보기 (적용 안 함)

kubectl set image deployment/<name> <container>=<image>:<tag>  # 이미지 변경
kubectl rollout status deployment/<name>    # 배포 완료 대기 (blocking)
kubectl rollout history deployment/<name>   # 배포 히스토리
kubectl rollout undo deployment/<name>      # 직전 버전 롤백
kubectl rollout undo deployment/<name> --to-revision=1  # 특정 리비전으로 롤백
kubectl rollout restart deployment/<name>   # Pod 순차 재생성 (무중단)
```

### 삭제

```bash
kubectl delete deployment/<name> -n <ns>   # Deployment + ReplicaSet + Pod 전부 삭제
kubectl delete pod/<name> -n <ns>          # Pod 1개만 삭제 (Deployment가 재생성)
kubectl delete -k <overlay-dir>/           # Kustomize로 만든 리소스 전체 삭제
```

### Secret

```bash
kubectl get secrets -n <ns>
kubectl describe secret registry-secret -n <ns>
kubectl create secret docker-registry registry-secret \
  --docker-server=registry.example.com \
  --docker-username=<username> \
  --docker-password=<password> \
  -n <ns>
```

---

## 리소스 지정 문법

`<리소스타입>/<리소스이름>` 형식. 리소스 이름은 `kubectl get <타입>`의 NAME 컬럼에서 확인.

```bash
kubectl describe deployment/nginx-demo      # Deployment 이름
kubectl logs pod/nginx-demo-7f99764bd4-x9hmf  # Pod 이름
kubectl delete service/test-container
```

**어떤 명령에 어떤 레벨의 리소스를 쓰는가:**
- `rollout` 계열 → Deployment 이름 (배포 전략은 Deployment가 담당)
- `logs`, `exec` → Pod 이름 (개별 컨테이너 대상)
- `describe` → 둘 다 가능 (보고 싶은 리소스에 따라)

---

## kubectl get all 출력 해독

```
Pod:        이름, READY(준비된 컨테이너/전체), STATUS, RESTARTS, AGE
Service:    이름, TYPE(ClusterIP 등), CLUSTER-IP, PORT(S)
Deployment: 이름, READY(준비된 Pod/전체), UP-TO-DATE, AVAILABLE, AGE
ReplicaSet: 이름, DESIRED, CURRENT, READY, AGE
```

---

## ReplicaSet 심화 이해

### ReplicaSet이란

**"이 사양(Pod 템플릿)으로 Pod를 N개 유지해라"라는 명세서.**
직접 만드는 게 아니라 Deployment가 내부적으로 자동 생성한다.

```
Deployment (swagger-hub)
  └── ReplicaSet (swagger-hub-75974df9bc)  ← Deployment가 자동 생성
        └── Pod (swagger-hub-75974df9bc-4qgwl)  ← ReplicaSet이 자동 생성
        └── Pod (swagger-hub-75974df9bc-rqb6s)
```

### 이미지가 바뀔 때마다 새 ReplicaSet이 생긴다

```
처음 배포 (nginx:1.24):
  Deployment
    └── ReplicaSet-A (nginx:1.24, replicas: 2) ← Pod 2개 실행 중

이미지 변경 (nginx:1.25):
  Deployment
    ├── ReplicaSet-A (nginx:1.24, replicas: 0) ← Pod 0개 (비활성, 삭제 안 함)
    └── ReplicaSet-B (nginx:1.25, replicas: 2) ← Pod 2개 새로 실행

rollout undo:
  Deployment
    ├── ReplicaSet-A (nginx:1.24, replicas: 2) ← 다시 scale up!
    └── ReplicaSet-B (nginx:1.25, replicas: 0) ← 비활성화
```

### ReplicaSet이 보존하는 것

**오직 Pod 템플릿(spec.template)만:**
- 컨테이너 이미지, 태그
- 포트 (containerPort)
- 환경변수
- resources (CPU/메모리)
- probe 설정
- volume mounts

### ReplicaSet이 보존하지 않는 것

| 리소스 | 보존 여부 | 이유 |
|--------|-----------|------|
| Service | ❌ | 별도 리소스. Deployment와 무관하게 존재 |
| Ingress | ❌ | 별도 리소스 |
| ConfigMap/Secret | ❌ | 별도 리소스 |
| replicas 수 | ❌ | Deployment가 관리 |

### 구 ReplicaSet은 언제 삭제되나

`revisionHistoryLimit` 설정에 따라 결정. **기본값: 10개 보존.**

```yaml
spec:
  revisionHistoryLimit: 10  # 최근 10개까지 보존, 나머지 자동 삭제
```

- 배포 11번 하면 가장 오래된 ReplicaSet 1개 자동 삭제
- 0으로 설정하면 구 ReplicaSet 즉시 삭제 → 롤백 불가능

### ReplicaSet 확인 명령

```bash
kubectl get rs -n default -l app=nginx-demo
# NAME                    DESIRED   CURRENT   READY
# nginx-demo-75974df9bc   2         2         2      ← 활성 (현재 버전)
# nginx-demo-7f64f47d66   0         0         0      ← 비활성 (이전 버전, 보존 중)
```

---

## rollout undo의 한계

### rollout undo가 되돌리는 것
- ✅ 컨테이너 이미지/태그
- ✅ 환경변수, 포트, resources, probe 등 Pod spec 전체

### rollout undo가 되돌리지 않는 것
- ❌ Service 변경
- ❌ Ingress 변경
- ❌ ConfigMap/Secret 내용 변경
- ❌ replicas 수 변경

### 문제 시나리오

포트를 3000 → 8080으로 변경하면서 deployment + service를 함께 수정한 경우:

```
rollout undo 실행 시:
  ✅ Deployment: containerPort 8080 → 3000 (되돌아감)
  ❌ Service: targetPort 8080 그대로 (안 되돌아감)

  결과: Service가 8080으로 보내는데, Pod는 3000에서 리스닝 → 연결 실패!
```

### 완전한 롤백 방법

| 방법 | 설명 | 안전성 |
|------|------|--------|
| `git revert` (GitOps) | 모든 파일이 이전 커밋 상태로 복원 → ArgoCD가 전체 리소스 동기화 | ✅ 가장 안전 |
| 수동 복원 | Service, Ingress 등도 직접 이전 상태로 kubectl apply | ⚠️ 실수 가능성 높음 |

**결론: 여러 리소스에 걸친 변경의 완전한 롤백은 GitOps(git revert)가 유일하게 안전한 방법.**

---

## 실습 내용 (실제 클러스터)

### CrashLoopBackOff 진단 실습

```bash
# 1. 상태 확인
kubectl get pods -n default
# test-container-796d77669d-2pwz2   0/1   CrashLoopBackOff   1855   6d14h

# 2. Events 확인
kubectl describe pod test-container-796d77669d-2pwz2 -n default
# Events:
#   Warning  BackOff  kubelet  Back-off restarting failed container

# 3. 로그 확인
kubectl logs test-container-796d77669d-2pwz2 -n default
# code: 'MODULE_NOT_FOUND'
# → Node.js 앱에서 @bc/shared-common 모듈 의존성 누락
```

### 배포 전체 사이클 실습

```bash
# 1. nginx:1.24 배포
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.24
          ports:
            - containerPort: 80
EOF

# 2. 배포 완료 확인
kubectl rollout status deployment/nginx-demo -n default
# deployment "nginx-demo" successfully rolled out

# 3. 이미지 업데이트 (Rolling Update)
kubectl set image deployment/nginx-demo nginx=nginx:1.25 -n default
kubectl rollout status deployment/nginx-demo -n default
# Waiting for deployment rollout to finish: 1 out of 2 new replicas have been updated...
# → 순차적으로 교체됨 (무중단)

# 4. ReplicaSet 확인 (구버전 보존됨)
kubectl get rs -n default -l app=nginx-demo
# nginx-demo-75974df9bc   0   0   0   ← v1.24 (비활성, 보존)
# nginx-demo-7f64f47d66   2   2   2   ← v1.25 (활성)

# 5. 롤백
kubectl rollout undo deployment/nginx-demo -n default
# → ReplicaSet-A(v1.24)를 다시 scale up, ReplicaSet-B(v1.25)를 scale down

# 6. rollout restart (이미지 변경 없이 Pod만 재생성)
kubectl rollout restart deployment/nginx-demo -n default
# → 새 ReplicaSet 생성, Pod 순차 교체

# 7. 정리
kubectl delete deployment/nginx-demo -n default
```

---

## 이미지 명시 위치

Deployment의 `spec.template.spec.containers[].image` 필드:

```yaml
spec:
  template:
    spec:
      containers:
        - name: swagger-hub
          image: registry.example.com/myorg/swagger-hub:latest  # ← 여기
```

현재 프로젝트에서는 overlay의 kustomization.yaml이 이 태그를 덮어씀:

```yaml
# overlays/dev/kustomization.yaml
images:
  - name: registry.example.com/myorg/swagger-hub
    newTag: bd467fae  # ← base의 latest를 이걸로 교체
```

---

## kubectl apply를 직접 하면 안 되는 이유 (GitOps)

ArgoCD가 Git 상태를 클러스터에 동기화하는데, 직접 apply하면:
1. 직접 apply한 변경이 클러스터에 반영됨
2. ArgoCD가 "Git 상태와 클러스터 상태가 다르네?" → **OutOfSync** 감지
3. `selfHeal: true`면 → ArgoCD가 Git 상태로 되돌려버림 (내가 한 변경 사라짐)
4. `selfHeal: false`면 → OutOfSync 상태로 남아있고, Git에 기록도 안 남음

---

## Kustomize 빌드 미리보기

```bash
kubectl kustomize apps/swagger-hub/overlays/dev/
```

- `kustomization.yaml`이 있는 **디렉토리**를 지정 (파일이 아님)
- 해당 디렉토리의 kustomization.yaml을 읽고, base + overlay를 합쳐서 최종 YAML을 stdout으로 출력
- 클러스터에 아무것도 적용하지 않음 (검증 용도)

---

## delete deployment vs delete pod vs rollout restart

| 명령 | 하는 일 | 결과 |
|------|---------|------|
| `delete deployment/<name>` | Deployment 자체 삭제 | ReplicaSet + Pod 전부 사라짐. **앱이 없어짐** |
| `delete pod/<name>` | Pod 1개만 삭제 | Deployment가 새 Pod 자동 생성 |
| `rollout restart deployment/<name>` | Deployment 유지, Pod 순차 교체 | 무중단 재시작 |

---

## 헷갈렸던 점 & 해결

- **Q**: `deployment/nginx-demo` vs Pod 이름 — rollout에서 왜 Deployment 이름을 쓰나?
- **A**: rollout은 "배포 전략"을 제어하는 명령이고, 그건 Deployment가 담당. Pod는 결과물일 뿐.

- **Q**: `delete deployment` vs `delete pod` 차이
- **A**: delete deployment = 앱 제거. delete pod = 1개만 죽이고 Deployment가 재생성.

- **Q**: ReplicaSet은 이전 배포 형상의 "템플릿"인가?
- **A**: 맞다. Pod를 찍어낼 수 있는 설계도(spec.template)를 그대로 보존하고 있음. rollout undo = 구 ReplicaSet을 다시 scale up.

- **Q**: rollout undo로 완전한 롤백이 가능한가?
- **A**: 아니다. Pod spec만 되돌림. Service, Ingress 등 다른 리소스는 안 되돌아감. 완전한 롤백은 git revert + ArgoCD sync가 유일하게 안전.

- **Q**: 구 ReplicaSet은 영원히 남아있나?
- **A**: `revisionHistoryLimit` (기본 10)만큼만 보존. 초과하면 가장 오래된 것부터 자동 삭제.

---

## 복습 포인트

1. `kubectl get all -n <ns>` → 전체 현황 파악의 시작점. Pod STATUS와 RESTARTS 숫자가 핵심 단서.
2. 트러블슈팅 순서: `get pods` → `describe pod` (Events) → `logs` (--previous). 이 3단계로 대부분 원인 파악 가능.
3. `rollout status`는 배포 완료까지 blocking으로 대기. CI/CD에서 성공/실패 판단에 사용.
4. `rollout undo`가 빠른 이유: 새 Pod를 만드는 게 아니라 기존 ReplicaSet을 다시 scale up하는 것. 단, Pod spec만 되돌리므로 Service/Ingress 등은 별도 복원 필요.
5. GitOps 환경에서 `kubectl apply`는 긴급 상황에서만. 정상 흐름은 Git Push → ArgoCD sync. 직접 apply하면 selfHeal이 되돌려버림.
6. 이미지 태그 변경의 정상 경로: `overlays/dev/kustomization.yaml`의 `newTag` 수정 → Git Push → ArgoCD 감지 → 자동 배포.
7. ReplicaSet = Pod 템플릿 보존소. `revisionHistoryLimit`(기본 10)개까지 보존되며, 이게 있어야 rollout undo가 가능.
8. 여러 리소스에 걸친 변경의 완전한 롤백은 `git revert`가 유일하게 안전한 방법. rollout undo는 Deployment(Pod spec)만 되돌린다.
9. `kubectl kustomize <dir>` = 적용 전 최종 YAML 검증. 디렉토리를 지정하면 그 안의 kustomization.yaml을 자동으로 찾아 빌드.
