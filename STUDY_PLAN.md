# K8s / EKS 학습 계획서

> **목적**: 이 문서는 `sample-k8s-deploy` 프로젝트를 담당 개발자로서 자신 있게 운영하고,
> 이후 Kubernetes / EKS 전반에 대한 전문성을 쌓기 위한 단계별 학습 로드맵이다.
>
> **사용법**: 이 파일을 Claude와의 학습 세션에서 로드하면, 프로젝트 맥락과 학습 진도를 바탕으로
> 이어서 학습할 수 있다. 완료한 항목은 `[x]`로 표시하고 저장하면 된다.

---

## 0. 현재 프로젝트 컨텍스트 (Context for Learning Sessions)

> 이 섹션은 학습 세션 시작 시 Claude가 프로젝트를 즉시 파악할 수 있도록 작성된 요약이다.

### 프로젝트 개요

**이름**: `sample-k8s-deploy`
**역할**: GitOps 방식으로 AWS EKS 클러스터에 Swagger Hub 앱을 배포하는 **배포 전용 레포지토리**
(소스 코드 없음 — Kubernetes 매니페스트만 존재)

**핵심 스택**:

| 구성요소 | 기술 | 역할 |
|---------|------|------|
| 실행 환경 | AWS EKS | 관리형 Kubernetes 클러스터 |
| 배포 자동화 | ArgoCD | Git 변경 감지 → 클러스터 자동 동기화 |
| 설정 관리 | Kustomize | base/overlay 패턴으로 환경별 YAML 관리 |
| 이미지 레지스트리 | Harbor | 사내 Private Docker 이미지 저장소 |
| 외부 트래픽 | Nginx Ingress | 도메인 기반 HTTP 라우팅 |

### 디렉토리 구조 (현재 상태)

```
sample-k8s-deploy/
└── apps/
    └── swagger-hub/
        ├── base/                      ← 모든 환경에 공통 적용되는 설정
        │   ├── kustomization.yaml     ← 포함할 리소스 목록 선언
        │   ├── deployment.yaml        ← 앱 실행 방법 (이미지, 포트, 복제 수)
        │   ├── service.yaml           ← 클러스터 내부 네트워크 접근점
        │   └── ingress.yaml           ← 외부 도메인 → 서비스 연결
        └── overlays/
            └── dev/                   ← dev 환경 전용 설정
                └── kustomization.yaml ← 네임스페이스 + 이미지 태그 오버라이드
```

### 현재 프로젝트 파일 내용

**`apps/swagger-hub/base/deployment.yaml`** — 핵심 파일:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: swagger-hub
spec:
  replicas: 1
  selector:
    matchLabels:
      app: swagger-hub
  template:
    spec:
      imagePullSecrets:
        - name: registry-secret       # Harbor Private 레지스트리 인증 시크릿
      containers:
        - name: swagger-hub
          image: registry.example.com/myorg/swagger-hub:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000   # Node.js 앱 포트
```
> **현재 미비사항**: resources(CPU/메모리 제한), livenessProbe, readinessProbe 없음

**`apps/swagger-hub/base/service.yaml`**:
```yaml
apiVersion: v1
kind: Service
spec:
  selector:
    app: swagger-hub
  ports:
    - port: 80          # 서비스 포트 (외부에서 접근하는 포트)
      targetPort: 3000  # Pod 포트로 포워딩
```

**`apps/swagger-hub/base/ingress.yaml`**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: swagger-hub
                port:
                  number: 80
```

**`apps/swagger-hub/overlays/dev/kustomization.yaml`**:
```yaml
resources:
  - ../../base

namespace: swagger-dev   # base 리소스 전체에 이 네임스페이스 적용

images:
  - name: registry.example.com/myorg/swagger-hub
    newTag: bd467fae     # CI/CD가 커밋 해시로 자동 업데이트
```

### 트래픽 흐름 (완성 시)

```
[사용자 브라우저]
      ↓ HTTPS
app.example.com
      ↓
[Nginx Ingress Controller]  ← 클러스터 외부 트래픽 진입점
      ↓
[Service: swagger-hub]  (ClusterIP, port:80)
      ↓
[Pod: swagger-hub]  (container port:3000)
      ↓
Node.js 앱 응답
```

### GitOps 배포 흐름

```
[개발자] → 소스 레포에 코드 Push
      ↓
[GitLab CI] → Docker 빌드 → Harbor에 이미지 Push
      ↓
[GitLab CI] → 이 레포(sample-k8s-deploy)의 overlays/dev/kustomization.yaml
               의 newTag 값을 새 커밋 해시로 자동 업데이트 & Push
      ↓
[ArgoCD] → 이 레포의 변경 감지 (polling 또는 webhook)
      ↓
[ArgoCD] → kustomize build apps/swagger-hub/overlays/dev/ 실행
      ↓
[ArgoCD] → 결과 YAML을 EKS 클러스터에 kubectl apply
      ↓
[EKS] → 새 Pod 생성 (새 이미지 사용), 구 Pod 종료
```

---

## 학습 단계 개요

```
[ Phase 1 ] 실용 레벨 ── 현장에서 바로 쓸 수 있는 수준
  1단계: 컨테이너 & K8s 핵심 개념
  2단계: 현재 프로젝트 파일 완전 해독
  3단계: kubectl 실전 조작
  4단계: Kustomize 패턴
  5단계: GitOps & ArgoCD 운영
  6단계: 일상 운영 & 트러블슈팅

[ Phase 2 ] 전문가 레벨 ── 큰 그림이 보이는 수준
  7단계: Kubernetes 내부 동작 원리
  8단계: AWS EKS 심화
  9단계: 네트워킹 심화
  10단계: 보안 & RBAC
  11단계: 프로덕션 하드닝
  12단계: 관찰 가능성 (Observability)
  13단계: 고급 GitOps & 운영 패턴
```

---

# Phase 1: 실용 레벨

> **목표**: 현재 프로젝트를 혼자 운영·수정할 수 있고, 장애 시 원인을 파악해 해결할 수 있는 수준

---

## 1단계: 컨테이너 & Kubernetes 핵심 개념

> **학습 목표**: "왜 Kubernetes가 필요한가"부터 시작하여 현재 프로젝트에 등장하는
> 모든 K8s 리소스 종류의 역할을 설명할 수 있다.

### 1-1. 왜 Kubernetes인가 (개념 이해)

**문제**: 컨테이너 1개를 단순히 실행하면 되지 않나?

| 문제 상황 | Kubernetes의 해결책 |
|----------|-------------------|
| 서버가 죽으면 앱도 죽는다 | Pod 자동 재시작 (ReplicaSet) |
| 트래픽 증가 시 서버 1대로 감당 불가 | 수평 확장 (HPA, 다중 replica) |
| 새 버전 배포 시 잠깐 다운타임 발생 | Rolling Update |
| 여러 서버에 수동으로 앱 배포 복잡 | 선언적(Declarative) 배포 — "이런 상태이어야 한다" 선언 |
| 여러 컨테이너의 내부 통신 관리 어려움 | Service, DNS 기반 서비스 디스커버리 |

**핵심 철학**: Kubernetes는 **원하는 상태(Desired State)**를 선언하면,
현재 상태(Current State)를 원하는 상태로 맞추려고 지속적으로 동작한다.

### 1-2. Kubernetes 아키텍처 개요

```
┌─────────────────────────────────────────────────────┐
│                    EKS Cluster                      │
│                                                     │
│  ┌──────────────────────┐  ┌────────────────────┐  │
│  │   Control Plane      │  │   Worker Node(s)   │  │
│  │   (AWS 관리)         │  │   (EC2 인스턴스)   │  │
│  │                      │  │                    │  │
│  │  ┌────────────────┐  │  │  ┌──────────────┐  │  │
│  │  │ API Server     │  │  │  │  kubelet     │  │  │
│  │  │ (kubectl 대상) │  │  │  │  (Pod 관리)  │  │  │
│  │  └────────────────┘  │  │  └──────────────┘  │  │
│  │  ┌────────────────┐  │  │  ┌──────────────┐  │  │
│  │  │ etcd           │  │  │  │  kube-proxy  │  │  │
│  │  │ (상태 저장소)  │  │  │  │  (네트워킹)  │  │  │
│  │  └────────────────┘  │  │  └──────────────┘  │  │
│  │  ┌────────────────┐  │  │  ┌──────────────┐  │  │
│  │  │ Scheduler      │  │  │  │  Pod들       │  │  │
│  │  │ (Pod 배치 결정)│  │  │  │  (컨테이너)  │  │  │
│  │  └────────────────┘  │  │  └──────────────┘  │  │
│  │  ┌────────────────┐  │  └────────────────────┘  │
│  │  │ Controller Mgr │  │                          │
│  │  │ (상태 유지)    │  │                          │
│  │  └────────────────┘  │                          │
│  └──────────────────────┘                          │
└─────────────────────────────────────────────────────┘
```

- **Control Plane**: EKS에서 AWS가 관리. 직접 건드릴 일 없음
- **Worker Node**: 실제 앱이 돌아가는 EC2 서버
- **kubectl**: Control Plane의 API Server에 명령을 보내는 CLI

### 1-3. 현재 프로젝트에 나오는 K8s 리소스

**Pod** — 컨테이너의 실행 단위

```
Pod
└── Container (swagger-hub 앱)
    └── Image: registry.example.com/myorg/swagger-hub:bd467fae
    └── Port: 3000
```

- 가장 작은 배포 단위
- Pod가 죽으면 IP가 바뀜 → 직접 접근하지 않고 Service를 통해 접근
- Deployment가 Pod를 생성/관리함

**Deployment** — Pod의 배포 관리자

```yaml
# deployment.yaml이 하는 일:
replicas: 1  →  "Pod를 1개 유지해라"
selector  →  "app: swagger-hub 라벨이 붙은 Pod를 관리해라"
template  →  "Pod를 만들 때 이 사양으로 만들어라"
```

- Pod가 죽으면 자동으로 새로 만든다
- 새 버전 배포 시 Rolling Update 수행 (무중단 배포)
- 이전 버전으로 Rollback 가능

**Service** — Pod에 대한 고정 주소

```yaml
# service.yaml이 하는 일:
selector: app: swagger-hub  →  이 라벨의 Pod로 트래픽 보내라
port: 80       →  서비스 포트 (Service에 접속할 때 쓰는 포트)
targetPort: 3000  →  Pod의 실제 포트
```

- Pod는 죽으면 IP가 바뀌지만, Service IP/DNS는 고정
- `ClusterIP`: 클러스터 내부에서만 접근 가능 (현재 설정)
- `swagger-hub.swagger-dev.svc.cluster.local` 로 내부 DNS 접근 가능

**Ingress** — 외부 트래픽의 진입로

```yaml
# ingress.yaml이 하는 일:
host: app.example.com  →  이 도메인으로 오면
path: /                        →  이 경로의 요청을
backend: service swagger-hub:80  →  이 서비스로 보내라
```

- L7(HTTP) 레이어 라우팅
- 도메인, 경로 기반으로 여러 서비스에 트래픽 분배 가능
- Nginx Ingress Controller가 실제 로드밸런서 역할 수행

**Namespace** — 리소스의 논리적 격리

```
EKS Cluster
├── namespace: argocd       ← ArgoCD 시스템 Pod들
├── namespace: swagger-dev  ← swagger-hub 개발 환경 Pod들
└── namespace: kube-system  ← K8s 시스템 Pod들
```

- 같은 이름의 리소스를 다른 namespace에 공존 가능
- 팀/환경/프로젝트별 격리에 사용
- 현재: `swagger-dev` (overlays/dev/kustomization.yaml에서 지정)

**Secret** — 민감 정보 저장

```yaml
# deployment.yaml에서:
imagePullSecrets:
  - name: registry-secret  ← Harbor 로그인 정보를 담은 Secret
```

- Harbor Private 레지스트리에서 이미지를 pull할 때 필요한 인증 정보
- base64 인코딩된 형태로 etcd에 저장
- 이 Secret은 클러스터에 미리 생성되어 있어야 함 (별도 생성 필요)

### 1-4. 학습 체크

- [x] "왜 Kubernetes가 필요한가"를 3가지 이유로 설명할 수 있다
- [x] Pod / Deployment / Service / Ingress의 역할 차이를 설명할 수 있다
- [x] Service가 Pod를 찾는 방법(label selector)을 설명할 수 있다
- [x] `registry-secret` Secret이 왜 필요한지 설명할 수 있다
- [x] Namespace의 역할과 현재 프로젝트에서의 사용 방식을 설명할 수 있다

---

## 2단계: 현재 프로젝트 파일 완전 해독

> **학습 목표**: 프로젝트의 모든 YAML 파일을 한 줄씩 읽고 "이 줄은 이 역할을 한다"고
> 설명할 수 있다. 수정 시 어떤 파일의 어떤 필드를 바꿔야 하는지 즉각 판단 가능.

### 2-1. YAML 읽는 법

```yaml
apiVersion: apps/v1         # API 그룹/버전 — 어느 K8s API를 쓰는지
kind: Deployment            # 리소스 종류
metadata:
  name: swagger-hub         # 리소스 이름 (클러스터 내 고유)
  namespace: swagger-dev    # 어느 namespace에 속하는지
  labels:                   # 태그 (다른 리소스가 이걸로 찾음)
    app: swagger-hub
spec:                       # 원하는 상태 상세 명세
  ...
```

### 2-2. Deployment 상세 해독

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: swagger-hub
spec:
  replicas: 1              # ① Pod를 몇 개 유지할 것인가
  selector:
    matchLabels:
      app: swagger-hub     # ② 어떤 Pod를 이 Deployment가 관리하는가
  template:                # ③ Pod를 만들 때의 설계도
    metadata:
      labels:
        app: swagger-hub   # ④ Pod에 붙는 라벨 (②의 selector와 일치해야 함!)
    spec:
      imagePullSecrets:
        - name: registry-secret  # ⑤ 이미지 pull 시 사용할 인증 정보
      containers:
        - name: swagger-hub
          image: registry.example.com/myorg/swagger-hub:latest
          #       ↑ 레지스트리 주소  ↑ 프로젝트  ↑앱명  ↑태그
          imagePullPolicy: IfNotPresent  # ⑥ 이미지가 이미 있으면 pull 생략
          ports:
            - containerPort: 3000  # ⑦ 앱이 실제로 리스닝하는 포트 (정보 목적)
```

> **⚠️ 중요**: `containerPort`는 문서화 목적. 실제 포트 매핑은 Service에서 한다.

### 2-3. Service 상세 해독

```yaml
apiVersion: v1
kind: Service
metadata:
  name: swagger-hub
spec:
  type: ClusterIP          # ① 서비스 타입 (생략 시 기본값이 ClusterIP)
  selector:
    app: swagger-hub       # ② 이 라벨을 가진 Pod로 트래픽 전달
  ports:
    - port: 80             # ③ Service 포트 (다른 서비스가 이 서비스에 접근할 때)
      targetPort: 3000     # ④ Pod의 실제 포트 (③을 받아서 ④로 포워딩)
      protocol: TCP
```

**Service 타입 비교**:

| 타입 | 접근 범위 | 언제 쓰나 |
|------|---------|---------|
| `ClusterIP` | 클러스터 내부만 | 내부 서비스간 통신, Ingress 뒤에 붙을 때 |
| `NodePort` | 노드 IP + 포트로 외부 접근 | 개발/테스트용 |
| `LoadBalancer` | 외부 LB IP로 접근 | 클라우드 LB가 필요할 때 |

### 2-4. Ingress 상세 해독

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: swagger-hub
  annotations:
    kubernetes.io/ingress.class: nginx  # ① Nginx Ingress Controller 사용 지정
spec:
  rules:
    - host: app.example.com      # ② 이 도메인으로 오는 요청만 처리
      http:
        paths:
          - path: /                       # ③ 모든 경로 (/api, /v2 등 포함)
            pathType: Prefix              # ④ 접두어 매칭 (Exact는 정확히 일치)
            backend:
              service:
                name: swagger-hub         # ⑤ 어느 Service로 보낼지
                port:
                  number: 80              # ⑥ Service의 포트 (③의 80과 일치)
```

### 2-5. Kustomize 파일 해독

**base/kustomization.yaml**:
```yaml
resources:          # 이 base에 포함할 파일 목록
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

**overlays/dev/kustomization.yaml**:
```yaml
resources:
  - ../../base      # base를 통째로 포함

namespace: swagger-dev   # base의 모든 리소스에 이 namespace 적용

images:                  # 이미지 태그 오버라이드
  - name: registry.example.com/myorg/swagger-hub  # base의 image 이름과 일치
    newTag: bd467fae     # 이 태그로 교체
```

> **동작 원리**: Kustomize는 base의 deployment.yaml을 가져와서
> `registry.example.com/myorg/swagger-hub:latest`를
> `registry.example.com/myorg/swagger-hub:bd467fae`로 바꿔서 출력한다.

### 2-6. 학습 체크

- [x] deployment.yaml에서 `selector.matchLabels`와 `template.metadata.labels`가 왜 일치해야 하는지 설명할 수 있다
- [x] service.yaml의 `port`와 `targetPort`의 차이를 설명할 수 있다
- [x] Ingress와 Service의 역할 차이를 설명할 수 있다
- [x] Kustomize의 `images` 오버라이드가 어떻게 동작하는지 설명할 수 있다
- [x] 새 이미지 태그로 배포하려면 어떤 파일의 어떤 줄을 바꾸면 되는지 말할 수 있다

---

## 3단계: kubectl 실전 조작

> **학습 목표**: 현재 클러스터 상태를 조회하고, 문제 발생 시 즉각 원인을 파악하며,
> 간단한 수정을 CLI로 수행할 수 있다.

### 3-1. 클러스터 접근 설정

```bash
# EKS 클러스터에 kubeconfig 설정 (최초 1회)
aws eks update-kubeconfig \
  --name <cluster-name> \
  --region ap-northeast-2

# 현재 어떤 클러스터에 연결되어 있는지 확인
kubectl config current-context
kubectl config get-contexts  # 등록된 모든 컨텍스트 목록
```

### 3-2. 현황 조회 (가장 많이 쓰는 명령)

```bash
# swagger-dev namespace의 모든 리소스 조회
kubectl get all -n swagger-dev

# 리소스별 조회
kubectl get pods -n swagger-dev
kubectl get deployments -n swagger-dev
kubectl get services -n swagger-dev
kubectl get ingress -n swagger-dev

# 상세 정보 (이벤트 포함 — 문제 파악할 때 필수)
kubectl describe pod <pod-name> -n swagger-dev
kubectl describe deployment swagger-hub -n swagger-dev

# Pod 로그 보기
kubectl logs <pod-name> -n swagger-dev
kubectl logs -f <pod-name> -n swagger-dev     # 실시간 follow
kubectl logs --previous <pod-name> -n swagger-dev  # 이전 크래시 로그

# Pod 안으로 접속 (디버깅용)
kubectl exec -it <pod-name> -n swagger-dev -- /bin/sh
```

### 3-3. 배포 관련

```bash
# 현재 배포 상태 확인
kubectl rollout status deployment/swagger-hub -n swagger-dev

# 배포 히스토리 확인
kubectl rollout history deployment/swagger-hub -n swagger-dev

# 이전 버전으로 롤백
kubectl rollout undo deployment/swagger-hub -n swagger-dev

# YAML 파일로 적용
kubectl apply -f apps/swagger-hub/base/deployment.yaml

# Kustomize 빌드 결과 미리보기 (적용 안 함)
kubectl kustomize apps/swagger-hub/overlays/dev/

# Kustomize 적용
kubectl apply -k apps/swagger-hub/overlays/dev/
```

### 3-4. 유용한 조회 옵션

```bash
# Pod의 어느 노드에 배치됐는지
kubectl get pods -n swagger-dev -o wide

# YAML 형식으로 현재 상태 조회 (현재 적용된 설정 확인)
kubectl get deployment swagger-hub -n swagger-dev -o yaml

# 라벨로 필터링
kubectl get pods -n swagger-dev -l app=swagger-hub

# 모든 namespace에서 조회
kubectl get pods --all-namespaces

# 짧게: -n 대신 namespace를 앞에 쓸 수도 있음
kubectl -n swagger-dev get pods
```

### 3-5. Secret 관련

```bash
# 시크릿 목록 조회
kubectl get secrets -n swagger-dev

# registry-secret 시크릿 생성 (Private 레지스트리 인증)
kubectl create secret docker-registry registry-secret \
  --docker-server=registry.example.com \
  --docker-username=<username> \
  --docker-password=<password> \
  -n swagger-dev
```

### 3-6. 학습 체크

- [x] `kubectl get all -n swagger-dev`로 현재 실행 중인 리소스를 볼 수 있다
- [x] Pod 이름을 찾아 로그를 출력할 수 있다
- [x] `kubectl describe pod`에서 Events 섹션을 찾아 읽을 수 있다
- [x] `kubectl kustomize` 로 최종 YAML을 미리보기할 수 있다
- [x] 배포 후 `kubectl rollout status`로 성공 여부를 확인할 수 있다

---

## 4단계: Kustomize 패턴

> **학습 목표**: Kustomize의 base/overlay 패턴을 완전히 이해하고,
> staging/production 환경 추가, 설정 변경 등을 직접 수행할 수 있다.

### 4-1. Kustomize가 하는 일

Kustomize는 **YAML 파일들을 조합·변환하는 도구**다. 템플릿 언어 없이 순수 YAML로 동작한다.

```
base/deployment.yaml  ─┐
base/service.yaml     ─┤  kustomize build  →  최종 YAML (kubectl apply에 전달)
base/ingress.yaml     ─┘       ↑
overlays/dev/kustomization.yaml ─ 오버라이드 적용
```

### 4-2. 주요 오버라이드 패턴

**패턴 1: 이미지 태그 변경** (현재 프로젝트에서 사용 중)
```yaml
images:
  - name: registry.example.com/myorg/swagger-hub
    newTag: bd467fae
```

**패턴 2: 네임스페이스 지정** (현재 프로젝트에서 사용 중)
```yaml
namespace: swagger-dev
```

**패턴 3: 전략적 병합 패치 (Strategic Merge Patch)** — 특정 필드만 변경
```yaml
# overlays/dev/kustomization.yaml
patches:
  - path: patch-deployment.yaml

# overlays/dev/patch-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: swagger-hub   # 어떤 리소스를 패치할지 지정
spec:
  replicas: 2         # 이 필드만 base를 덮어씀
  template:
    spec:
      containers:
        - name: swagger-hub
          resources:          # 리소스 제한 추가
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

**패턴 4: ConfigMap / Secret 생성**
```yaml
configMapGenerator:
  - name: swagger-config
    literals:
      - APP_ENV=development
      - LOG_LEVEL=debug
```

**패턴 5: 공통 라벨 추가**
```yaml
commonLabels:
  environment: dev
  managed-by: argocd
```

### 4-3. 실습: staging 환경 추가

현재 `overlays/dev/`만 있는 구조에서 `overlays/staging/`을 추가하는 방법:

```bash
# 1. 디렉토리 생성
mkdir -p apps/swagger-hub/overlays/staging

# 2. kustomization.yaml 작성
# apps/swagger-hub/overlays/staging/kustomization.yaml
```
```yaml
resources:
  - ../../base

namespace: swagger-staging

images:
  - name: registry.example.com/myorg/swagger-hub
    newTag: latest

patches:
  - path: patch-deployment.yaml
```
```yaml
# apps/swagger-hub/overlays/staging/patch-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: swagger-hub
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: swagger-hub
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
```

### 4-4. 빌드 결과 검증

```bash
# dev 최종 YAML 미리보기
kubectl kustomize apps/swagger-hub/overlays/dev/

# staging 최종 YAML 미리보기
kubectl kustomize apps/swagger-hub/overlays/staging/

# diff로 dev vs staging 차이 확인
diff <(kubectl kustomize apps/swagger-hub/overlays/dev/) \
     <(kubectl kustomize apps/swagger-hub/overlays/staging/)
```

### 4-5. 학습 체크

- [x] base/overlay 패턴의 목적을 한 문장으로 설명할 수 있다
- [x] Strategic Merge Patch로 replica 수를 환경별로 다르게 설정할 수 있다
- [x] Kustomize와 Helm의 접근 방식 차이를 설명할 수 있다
- [x] staging 환경 overlay를 직접 작성할 수 있다
- [x] `kubectl kustomize`로 최종 결과를 검증할 수 있다

---

## 5단계: GitOps & ArgoCD 운영

> **학습 목표**: GitOps의 철학을 이해하고, ArgoCD UI/CLI를 통해
> 배포 상태를 모니터링하고 sync/rollback을 수행할 수 있다.

### 5-1. GitOps란?

**핵심 원칙**: Git 저장소가 인프라/앱의 **유일한 진실 원천(Single Source of Truth)**

```
전통적 방식:
  개발자 → kubectl apply → 클러스터 변경
  (클러스터 실제 상태를 Git이 모름)

GitOps 방식:
  개발자 → Git Push → ArgoCD → kubectl apply → 클러스터 변경
  (클러스터 상태 = Git 상태, 항상 동기화)
```

**장점**:
- 모든 변경이 Git 히스토리로 추적됨 (누가 언제 뭘 바꿨는지)
- 클러스터 상태를 직접 수정해도 ArgoCD가 Git 상태로 되돌림 (Self-Heal)
- 롤백 = `git revert`

### 5-2. ArgoCD 핵심 개념

**Application**: ArgoCD의 핵심 단위. "이 Git 경로를 저 K8s 클러스터/namespace에 동기화해라"

```yaml
# ArgoCD Application 예시
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: swagger-hub-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: http://gitlab.example.com/myorg/sample-k8s-deploy.git
    targetRevision: main
    path: apps/swagger-hub/overlays/dev    # ← 이 경로를
  destination:
    server: https://kubernetes.default.svc
    namespace: swagger-dev                  # ← 여기에 동기화
  syncPolicy:
    automated:
      prune: true       # Git에서 삭제된 리소스는 클러스터에서도 삭제
      selfHeal: true    # 클러스터 수동 변경 시 Git 상태로 되돌림
```

**Sync 상태**:

| 상태 | 의미 |
|------|------|
| `Synced` | Git과 클러스터 상태 일치 (정상) |
| `OutOfSync` | Git과 클러스터 상태 불일치 (배포 필요 또는 누군가 수동으로 변경) |
| `Progressing` | 동기화 진행 중 |
| `Degraded` | 동기화했으나 Pod가 정상이 아님 |
| `Unknown` | 상태 확인 불가 |

**Health 상태**:

| 상태 | 의미 |
|------|------|
| `Healthy` | 모든 리소스 정상 |
| `Progressing` | 롤아웃 진행 중 |
| `Degraded` | 문제 있음 (Pod CrashLoop 등) |
| `Missing` | 리소스가 클러스터에 없음 |

### 5-3. ArgoCD 주요 CLI 명령

```bash
# ArgoCD CLI 로그인
argocd login <argocd-server-url>

# 앱 목록 조회
argocd app list

# 특정 앱 상태 조회
argocd app get swagger-hub-dev

# 수동 동기화
argocd app sync swagger-hub-dev

# 동기화 히스토리 조회
argocd app history swagger-hub-dev

# 이전 버전으로 롤백
argocd app rollback swagger-hub-dev <revision-id>

# diff 확인 (Git vs 클러스터 현재 상태)
argocd app diff swagger-hub-dev
```

### 5-4. 배포 흐름 완전 추적

CI/CD가 새 이미지를 배포할 때 실제 일어나는 일:

```
1. GitLab CI: 소스 레포 코드 변경 감지
2. GitLab CI: Docker 빌드 → registry.example.com/myorg/swagger-hub:abc12345 Push
3. GitLab CI: 이 레포의 overlays/dev/kustomization.yaml에서
              newTag: bd467fae → newTag: abc12345 로 변경 후 Push
              (커밋 메시지: "chore: deploy swagger-hub abc12345 to dev")
4. ArgoCD: main 브랜치의 변경 감지 (polling 3분 또는 webhook)
5. ArgoCD: kustomize build apps/swagger-hub/overlays/dev/ 실행
6. ArgoCD: 출력된 YAML의 이미지 태그가 클러스터와 다름 → OutOfSync
7. ArgoCD: automated sync → kubectl apply 실행
8. EKS: 새 Pod 생성 (abc12345 이미지), 구 Pod 종료 (Rolling Update)
9. ArgoCD: Healthy + Synced 상태로 복귀
```

### 5-5. 학습 체크

- [x] GitOps와 전통적 배포 방식의 차이를 설명할 수 있다
- [x] ArgoCD의 Synced / OutOfSync / Degraded 상태의 의미를 알고 있다
- [x] `selfHeal: true`가 무엇을 하는지 설명할 수 있다
- [x] CI/CD 파이프라인에서 이 레포가 어떤 역할을 하는지 설명할 수 있다
- [x] ArgoCD CLI로 현재 동기화 상태를 확인할 수 있다

---

## 6단계: 일상 운영 & 트러블슈팅

> **학습 목표**: Pod 장애 유형을 보고 즉각 원인을 진단하고 해결할 수 있다.

### 6-1. Pod 상태 해독

```bash
kubectl get pods -n swagger-dev

NAME                           READY   STATUS             RESTARTS   AGE
swagger-hub-7d4b9f8c6-xkpzm    0/1     ImagePullBackOff   0          2m
swagger-hub-7d4b9f8c6-xkpzm    0/1     CrashLoopBackOff   5          10m
swagger-hub-7d4b9f8c6-xkpzm    1/1     Running            0          1m
swagger-hub-7d4b9f8c6-xkpzm    0/1     Pending            0          5m
swagger-hub-7d4b9f8c6-xkpzm    0/1     OOMKilled          1          3m
```

### 6-2. 에러별 진단 가이드

**`ImagePullBackOff` — 이미지를 가져오지 못함**

```bash
kubectl describe pod <pod-name> -n swagger-dev
# Events 섹션에서:
# Failed to pull image "...": ... unauthorized

# 원인 체크리스트:
# 1. 이미지 태그가 실제로 Harbor에 있는가?
# 2. registry-secret Secret이 존재하는가?
kubectl get secrets -n swagger-dev
# 3. Secret의 Harbor 자격증명이 유효한가?
```

**`CrashLoopBackOff` — 컨테이너가 시작 후 반복 종료**

```bash
# 현재 로그
kubectl logs <pod-name> -n swagger-dev

# 이전 크래시의 로그 (앱 에러 메시지 확인)
kubectl logs --previous <pod-name> -n swagger-dev

# 원인: 앱 자체의 런타임 에러, 잘못된 환경변수, 포트 충돌 등
```

**`Pending` — Pod가 노드에 배치되지 못하고 대기**

```bash
kubectl describe pod <pod-name> -n swagger-dev
# Events:
# 0/3 nodes are available: insufficient cpu.
# → 노드의 CPU 자원 부족

# 또는: node(s) didn't match node affinity
# → nodeSelector/affinity 설정 문제
```

**`OOMKilled` — 메모리 초과로 강제 종료**

```bash
kubectl describe pod <pod-name> -n swagger-dev
# Last State: Terminated, Reason: OOMKilled

# 해결: resources.limits.memory 값 증가
# 또는 앱의 메모리 누수 점검
```

### 6-3. 트러블슈팅 체크리스트

```bash
# Pod 장애 시 순서대로 확인

# Step 1: 어떤 상태인가?
kubectl get pods -n swagger-dev

# Step 2: 무슨 이벤트가 있었나?
kubectl describe pod <pod-name> -n swagger-dev
# → Events 섹션이 핵심

# Step 3: 앱 로그에 에러가 있나?
kubectl logs <pod-name> -n swagger-dev
kubectl logs --previous <pod-name> -n swagger-dev

# Step 4: 서비스 연결이 되나?
kubectl get endpoints -n swagger-dev
# → swagger-hub endpoint에 IP가 있어야 함

# Step 5: Ingress가 올바른가?
kubectl describe ingress swagger-hub -n swagger-dev
```

### 6-4. 자주 하는 운영 작업

```bash
# 이미지 태그만 빠르게 변경 (권장하지 않음, GitOps 원칙 위반 — 테스트용)
kubectl set image deployment/swagger-hub \
  swagger-hub=registry.example.com/myorg/swagger-hub:new-tag \
  -n swagger-dev

# Deployment 재시작 (Pod 강제 재생성)
kubectl rollout restart deployment/swagger-hub -n swagger-dev

# 특정 Pod 강제 삭제 (Deployment가 새 Pod 자동 생성)
kubectl delete pod <pod-name> -n swagger-dev

# 실시간 이벤트 스트림 모니터링
kubectl get events -n swagger-dev --watch
```

### 6-5. 학습 체크

- [ ] `ImagePullBackOff` 발생 시 3가지 원인을 점검할 수 있다
- [ ] `CrashLoopBackOff` 발생 시 로그로 앱 에러를 확인할 수 있다
- [ ] `kubectl describe`의 Events 섹션을 읽고 해석할 수 있다
- [ ] `kubectl rollout restart`로 Pod를 재시작할 수 있다
- [ ] `kubectl get endpoints`로 Service-Pod 연결을 확인할 수 있다

---

### Phase 1 최종 체크리스트

- [ ] 현재 EKS 클러스터의 swagger-dev namespace에서 모든 리소스를 조회할 수 있다
- [ ] 프로젝트의 모든 YAML 파일을 한 줄씩 설명할 수 있다
- [ ] 이미지 태그를 변경해서 배포하는 전체 흐름을 설명할 수 있다
- [ ] Pod 장애 발생 시 혼자 원인을 파악하고 해결할 수 있다
- [ ] staging 환경 overlay를 직접 추가할 수 있다
- [ ] ArgoCD에서 Sync 상태를 모니터링할 수 있다
- [ ] resources, livenessProbe, readinessProbe를 deployment.yaml에 추가할 수 있다

---

# Phase 2: 전문가 레벨

> **목표**: "왜 이렇게 설계됐는가"를 이해하고, 새로운 상황에서 올바른 선택을 할 수 있으며,
> 다른 팀원에게 설명해줄 수 있는 수준

---

## 7단계: Kubernetes 내부 동작 원리

> **학습 목표**: 선언적 배포의 내부 메커니즘을 이해한다.
> "kubectl apply를 하면 실제로 무슨 일이 일어나는가?"

### 7-1. Control Loop 패턴

Kubernetes의 모든 컨트롤러는 같은 패턴으로 동작한다:

```
while true:
  desired = etcd에서 원하는 상태 읽기
  current = 클러스터의 실제 상태 조회
  if desired != current:
    필요한 조치 실행 (Pod 생성/삭제/업데이트)
  sleep(일정 시간)
```

이것이 **Reconciliation Loop** (조정 루프).

### 7-2. kubectl apply → 클러스터 변경까지의 흐름

```
kubectl apply -f deployment.yaml
      ↓
API Server (인증/인가 확인)
      ↓
Admission Controllers (validating, mutating — 리소스 검증/변환)
      ↓
etcd (원하는 상태 저장)
      ↓
Deployment Controller (감시 중 → 변경 감지)
      ↓
ReplicaSet Controller (Deployment가 ReplicaSet 생성)
      ↓
Scheduler (Pod를 어느 노드에 배치할지 결정)
      ↓
kubelet (해당 노드의 에이전트 — 실제 컨테이너 실행)
      ↓
Container Runtime (Docker/containerd — 이미지 pull, 컨테이너 시작)
```

### 7-3. Deployment → ReplicaSet → Pod 관계

```
Deployment (swagger-hub)
  └── ReplicaSet (swagger-hub-7d4b9f8c6)   ← 현재 버전
  │     └── Pod (swagger-hub-7d4b9f8c6-xkpzm)
  └── ReplicaSet (swagger-hub-6c8f5b7d4)   ← 이전 버전 (replicas: 0)
        └── (Pod 없음)
```

- 이미지 태그가 바뀌면 새 ReplicaSet 생성 + 구 ReplicaSet 축소 (Rolling Update)
- 구 ReplicaSet은 rollback용으로 보존됨
- `kubectl rollout undo`는 구 ReplicaSet을 다시 scale up

### 7-4. 서비스 디스커버리와 kube-proxy

```
Pod A가 Service swagger-hub에 접근하고 싶을 때:
  1. DNS 조회: swagger-hub.swagger-dev.svc.cluster.local → 10.96.x.x (ClusterIP)
  2. kube-proxy의 iptables/IPVS 규칙이 10.96.x.x → Pod IP로 DNAT
  3. 실제 Pod IP로 패킷 전달
```

CoreDNS는 K8s 클러스터 내부 DNS 서버. Service 이름 → ClusterIP 해석.

### 7-5. Rolling Update 동작 원리

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # 동시에 최대 1개 Pod 다운 허용
      maxSurge: 1         # 동시에 최대 1개 Pod 초과 허용
```

```
초기 상태: [v1] [v1] [v1]   replicas: 3

Step 1: [v1] [v1] [v1] [v2]   (maxSurge: 새 Pod 추가)
Step 2: [v1] [v1] [v2]        (구 Pod 1개 제거)
Step 3: [v1] [v1] [v2] [v2]   (새 Pod 추가)
Step 4: [v1] [v2] [v2]        (구 Pod 제거)
Step 5: [v1] [v2] [v2] [v2]
Step 6: [v2] [v2] [v2]        완료
```

### 7-6. etcd — Kubernetes의 데이터베이스

- 모든 K8s 리소스 상태가 저장되는 분산 Key-Value 저장소
- `kubectl get` = etcd에서 읽기
- `kubectl apply` = etcd에 쓰기
- EKS에서는 AWS가 관리 (직접 접근 불가)

### 7-7. 학습 체크

- [ ] Reconciliation Loop의 개념을 설명할 수 있다
- [ ] `kubectl apply` 후 실제 컨테이너가 실행되기까지의 단계를 열거할 수 있다
- [ ] Deployment가 ReplicaSet을 통해 Pod를 관리하는 이유를 설명할 수 있다
- [ ] Rolling Update가 무중단인 이유를 설명할 수 있다
- [ ] Service가 kube-proxy/iptables를 통해 동작하는 방식을 설명할 수 있다

---

## 8단계: AWS EKS 심화

> **학습 목표**: EKS가 vanilla Kubernetes와 무엇이 다른지 이해하고,
> AWS 서비스들과의 통합 방식을 파악한다.

### 8-1. EKS가 관리해주는 것 vs 직접 해야 하는 것

| 구분 | EKS가 관리 | 개발자/운영자가 관리 |
|------|-----------|-------------------|
| Control Plane | API Server, etcd, Scheduler, Controller Manager | - |
| 업그레이드 | Control Plane | Node Group, Add-on |
| HA | Control Plane 다중 AZ 배포 | Worker Node 다중 AZ 배포 |
| 보안 | Control Plane 네트워크 격리 | IAM, Security Group, RBAC |
| 스토리지 | - | EBS CSI Driver, EFS CSI Driver |

### 8-2. EKS Node Group 종류

**Managed Node Group** (권장):
- AWS가 EC2 인스턴스의 provisioning, 업데이트, 종료를 자동화
- 노드 업그레이드 시 자동 drain & cordon
- `eksctl`로 관리

**Self-Managed Node Group**:
- EC2 인스턴스를 직접 관리
- 더 많은 커스터마이징 가능

**Fargate**:
- 노드 없이 Pod를 서버리스로 실행
- 노드 관리 완전히 불필요, 비용은 Pod 단위 과금

### 8-3. EKS 인증 체계 (IAM + K8s RBAC)

```
개발자 → kubectl 명령
      ↓
AWS IAM 인증 (kubeconfig의 aws eks get-token)
      ↓
K8s API Server
      ↓
K8s RBAC 인가 (Role, ClusterRole, RoleBinding)
      ↓
작업 허용/거부
```

**aws-auth ConfigMap** (구 방식):
```yaml
# kube-system namespace의 aws-auth configmap
mapRoles:
  - rolearn: arn:aws:iam::123456789:role/eks-developer
    username: developer
    groups:
      - system:masters  # cluster-admin 권한
```

**EKS Access Entry** (신규 방식, EKS 1.29+):
- IAM principal을 K8s 접근 정책에 직접 매핑
- aws-auth ConfigMap 없이 관리

### 8-4. IRSA (IAM Roles for Service Accounts)

Pod에서 AWS 서비스(S3, DynamoDB 등)에 접근할 때 IAM 권한을 주는 방법:

```
전통적 방법: 노드 EC2 인스턴스에 IAM Role → 모든 Pod가 같은 권한
IRSA: K8s ServiceAccount → IAM Role 매핑 → 특정 Pod만 특정 권한

Pod (ServiceAccount: my-app-sa)
  → OIDC Provider가 JWT 토큰 발급
  → AWS STS가 IAM Role로 교환
  → S3/DynamoDB 접근 가능
```

### 8-5. EKS Add-on

EKS 클러스터에 추가로 설치하는 운영 컴포넌트:

| Add-on | 역할 |
|--------|------|
| CoreDNS | 클러스터 내부 DNS |
| kube-proxy | 서비스 네트워킹 |
| VPC CNI | Pod 네트워킹 (AWS VPC IP 할당) |
| EBS CSI Driver | EBS 볼륨을 PersistentVolume으로 사용 |
| AWS Load Balancer Controller | ALB/NLB를 Ingress/Service로 사용 |

### 8-6. 학습 체크

- [ ] EKS와 self-managed K8s의 관리 책임 범위 차이를 설명할 수 있다
- [ ] IAM과 K8s RBAC가 어떻게 연계되는지 설명할 수 있다
- [ ] IRSA가 왜 필요한지 설명할 수 있다
- [ ] Managed Node Group과 Fargate의 트레이드오프를 설명할 수 있다
- [ ] VPC CNI가 Pod에 IP를 할당하는 방식을 설명할 수 있다

---

## 9단계: 네트워킹 심화

> **학습 목표**: 클러스터 내/외부 트래픽 흐름을 패킷 레벨에서 이해한다.

### 9-1. K8s 네트워킹 모델

K8s 네트워킹 4가지 규칙:
1. 모든 Pod는 NAT 없이 다른 Pod와 통신 가능
2. 모든 노드는 NAT 없이 모든 Pod와 통신 가능
3. Pod가 보는 자신의 IP = 외부에서 보는 Pod IP (같음)
4. Service는 안정적인 가상 IP(ClusterIP) 제공

### 9-2. AWS VPC CNI 특성

EKS는 AWS VPC CNI를 사용 → Pod IP = VPC 서브넷의 실제 IP

```
VPC: 10.0.0.0/16
  서브넷: 10.0.1.0/24 (ap-northeast-2a)
  서브넷: 10.0.2.0/24 (ap-northeast-2b)

Node: 10.0.1.5
  Pod A: 10.0.1.20  ← VPC IP 직접 할당
  Pod B: 10.0.1.21
Node: 10.0.2.6
  Pod C: 10.0.2.30
```

장점: Pod간 통신이 VPC 라우팅으로 직접 처리 (오버레이 네트워크 불필요)

### 9-3. Ingress 동작 심화

```
인터넷
  ↓
AWS Load Balancer (or NodePort)
  ↓
Nginx Ingress Controller Pod (kube-system namespace)
  ↓ (Host 헤더, 경로 기반 라우팅)
Service ClusterIP
  ↓ (kube-proxy iptables DNAT)
App Pod
```

**현재 프로젝트**: `kubernetes.io/ingress.class: nginx` → Nginx Ingress Controller 사용

**ALB Ingress** (AWS 전용 대안):
```yaml
annotations:
  kubernetes.io/ingress.class: alb
  alb.ingress.kubernetes.io/scheme: internet-facing
```
→ AWS ALB가 직접 Ingress 처리 (Nginx Controller 불필요)

### 9-4. Network Policy

Pod 간 통신을 제한하는 방화벽 규칙:

```yaml
# swagger-hub Pod는 ingress에서 오는 트래픽만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: swagger-hub-netpol
  namespace: swagger-dev
spec:
  podSelector:
    matchLabels:
      app: swagger-hub
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
```

### 9-5. 학습 체크

- [ ] K8s 네트워킹 4가지 규칙을 설명할 수 있다
- [ ] AWS VPC CNI에서 Pod가 VPC IP를 받는 방식을 설명할 수 있다
- [ ] Nginx Ingress와 ALB Ingress의 차이를 설명할 수 있다
- [ ] Network Policy로 Pod 간 통신을 제어하는 예시를 작성할 수 있다

---

## 10단계: 보안 & RBAC

> **학습 목표**: 클러스터 보안의 여러 레이어를 이해하고 최소 권한 원칙을 적용한다.

### 10-1. K8s 보안 레이어

```
레이어 1: 인증 (Authentication) — "누구인가?"
  → AWS IAM, kubeconfig 인증서, ServiceAccount 토큰

레이어 2: 인가 (Authorization) — "무엇을 할 수 있는가?"
  → RBAC (Role-Based Access Control)

레이어 3: Admission Control — "이 요청이 정책에 맞는가?"
  → OPA/Gatekeeper, Kyverno (Pod Security 정책)

레이어 4: 런타임 보안 — "실행 중 이상한 행동을 하지 않는가?"
  → Falco, seccomp, AppArmor
```

### 10-2. RBAC 핵심

```yaml
# Role: namespace 내 권한 정의
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: swagger-dev
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

# RoleBinding: Role을 사용자/서비스계정에 연결
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: swagger-dev
subjects:
  - kind: User
    name: developer@example.com
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Role vs ClusterRole**: Role은 특정 namespace, ClusterRole은 클러스터 전체

### 10-3. Pod Security

```yaml
# SecurityContext: 컨테이너 보안 설정
spec:
  securityContext:
    runAsNonRoot: true      # root로 실행 금지
    runAsUser: 1000
  containers:
    - name: swagger-hub
      securityContext:
        allowPrivilegeEscalation: false  # 권한 상승 금지
        readOnlyRootFilesystem: true     # 파일시스템 읽기전용
        capabilities:
          drop:
            - ALL                        # Linux capability 모두 제거
```

### 10-4. Secret 관리 베스트 프랙티스

현재 프로젝트는 `registry-secret` Secret을 사용. 운영 시 고려사항:

```
현재 방식: kubectl로 Secret 직접 생성 → 클러스터에만 존재

개선 방안 1: AWS Secrets Manager + External Secrets Operator
  → Secrets Manager에 저장 → Operator가 K8s Secret으로 동기화

개선 방안 2: Sealed Secrets
  → 암호화된 형태로 Git에 저장 가능 → 복호화는 클러스터에서만

현재 문제: Git에 Secret이 없으면 클러스터 재구축 시 수동으로 다시 생성해야 함
```

### 10-5. 학습 체크

- [ ] RBAC의 Role / ClusterRole / RoleBinding 구조를 설명할 수 있다
- [ ] 최소 권한 원칙을 적용해서 특정 namespace의 pod read-only Role을 작성할 수 있다
- [ ] `runAsNonRoot`, `readOnlyRootFilesystem`이 왜 중요한지 설명할 수 있다
- [ ] Secret을 Git에 안전하게 관리하는 방법 2가지를 설명할 수 있다

---

## 11단계: 프로덕션 하드닝

> **학습 목표**: 현재 프로젝트의 미비한 프로덕션 설정을 이해하고 적용할 수 있다.

### 11-1. 현재 프로젝트에 없는 것들 (필수 추가 항목)

**① Resources (리소스 제한)** — 왜 없으면 안 되나?
```
없을 경우: Pod 하나가 노드 전체 CPU/메모리를 독점 → 다른 Pod 죽음
          Scheduler가 Pod 배치 최적화 불가

requests: 스케줄러가 노드 선택 시 참고 (최소 보장량)
limits:   실제 사용 상한선 (초과 시 CPU throttle, 메모리는 OOMKill)
```

```yaml
containers:
  - name: swagger-hub
    resources:
      requests:
        cpu: 100m       # 0.1 vCPU 보장 요청
        memory: 128Mi
      limits:
        cpu: 500m       # 최대 0.5 vCPU
        memory: 512Mi
```

**② Liveness Probe** — 앱이 죽었는지 감지
```yaml
livenessProbe:
  httpGet:
    path: /             # swagger-hub의 루트 경로 (또는 /health 엔드포인트)
    port: 3000
  initialDelaySeconds: 15  # 앱 시작 대기 시간
  periodSeconds: 30         # 30초마다 체크
  failureThreshold: 3       # 3번 연속 실패 시 Pod 재시작
```

**③ Readiness Probe** — 트래픽 받을 준비됐는지 감지
```yaml
readinessProbe:
  httpGet:
    path: /
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
# 실패하면: Service 엔드포인트에서 제거 (트래픽 안 보냄)
# 성공하면: 다시 엔드포인트에 추가
```

Liveness vs Readiness 차이:
- `livenessProbe` 실패 → Pod 재시작
- `readinessProbe` 실패 → 트래픽 차단 (Pod는 살아있음)

**④ HPA (Horizontal Pod Autoscaler)**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: swagger-hub
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: swagger-hub
  minReplicas: 2      # 최소 2개 (HA 보장)
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # CPU 70% 초과 시 스케일 아웃
```

**⑤ PodDisruptionBudget (PDB)**
```yaml
# 노드 유지보수/업그레이드 시에도 최소 가용성 보장
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: swagger-hub-pdb
spec:
  minAvailable: 1     # 항상 1개 이상 Pod는 실행 중이어야 함
  selector:
    matchLabels:
      app: swagger-hub
```

### 11-2. 환경별 설정 비교 (목표 상태)

| 항목 | dev | staging | production |
|------|-----|---------|-----------|
| replicas | 1 | 2 | 3 (HPA: min3, max10) |
| CPU request | 100m | 250m | 500m |
| Memory request | 128Mi | 256Mi | 512Mi |
| CPU limit | 500m | 1000m | 2000m |
| Memory limit | 512Mi | 1Gi | 2Gi |
| ArgoCD sync | 자동 | 자동 | 수동 승인 |
| Log level | debug | info | warn |

### 11-3. 학습 체크

- [ ] resources requests vs limits의 차이와 각각의 역할을 설명할 수 있다
- [ ] livenessProbe와 readinessProbe의 차이를 실제 시나리오로 설명할 수 있다
- [ ] HPA가 동작하려면 metrics-server가 필요한 이유를 설명할 수 있다
- [ ] PodDisruptionBudget이 필요한 상황을 설명할 수 있다
- [ ] 현재 프로젝트에 resources와 probe를 추가하는 PR을 작성할 수 있다

---

## 12단계: 관찰 가능성 (Observability)

> **학습 목표**: 클러스터와 앱의 상태를 메트릭/로그/트레이스로 모니터링하는 방법을 이해한다.

### 12-1. 관찰 가능성 3대 요소

```
Metrics (메트릭) ── 시계열 숫자 데이터
  예: CPU 사용률, 요청 수/초, 에러율, 응답 시간
  도구: Prometheus + Grafana

Logs (로그) ── 이벤트 텍스트 기록
  예: kubectl logs, 앱 에러 로그
  도구: Fluent Bit → CloudWatch / Elasticsearch

Traces (트레이스) ── 요청의 전체 경로 추적
  예: 요청이 어느 서비스를 몇 ms씩 걸쳤는지
  도구: Jaeger, AWS X-Ray
```

### 12-2. Prometheus + Grafana 스택

```
App Pod ──→ /metrics 엔드포인트 (Prometheus 형식)
             ↑ scrape (수집)
         Prometheus
             ↓ 쿼리 (PromQL)
           Grafana ── 대시보드 시각화
```

**K8s 기본 메트릭 (kube-state-metrics)**:
- `kube_deployment_status_replicas_ready` — Ready Pod 수
- `kube_pod_container_resource_limits` — 리소스 제한
- `container_cpu_usage_seconds_total` — 실제 CPU 사용량

### 12-3. EKS 로그 관리

```
Pod 로그 → Fluent Bit DaemonSet → CloudWatch Logs
                                 └ /aws/containerinsights/<cluster>/application

노드 로그 → CloudWatch Logs
           └ /aws/containerinsights/<cluster>/host

API Server 감사 로그 → CloudWatch Logs (EKS Control Plane 로그 활성화 필요)
```

### 12-4. ArgoCD + 모니터링 통합

ArgoCD 자체도 Prometheus 메트릭 제공:
- `argocd_app_info` — Application 정보
- `argocd_app_sync_total` — Sync 횟수
- `argocd_app_health_status` — Health 상태

### 12-5. 학습 체크

- [ ] Metrics / Logs / Traces의 차이와 각각 언제 쓰는지 설명할 수 있다
- [ ] Prometheus scrape 방식을 설명할 수 있다
- [ ] EKS에서 Pod 로그를 CloudWatch로 수집하는 방법을 설명할 수 있다
- [ ] PromQL로 기본 메트릭 쿼리를 작성할 수 있다

---

## 13단계: 고급 GitOps & 운영 패턴

> **학습 목표**: 대규모/다중 팀/다중 환경에서의 GitOps 패턴을 이해한다.

### 13-1. ArgoCD ApplicationSet

여러 앱/환경을 하나의 CRD로 자동 관리:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: sample-app-apps
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: http://gitlab.example.com/myorg/sample-k8s-deploy.git
        revision: main
        directories:
          - path: apps/*/overlays/*   # apps/아무앱/overlays/아무환경
  template:
    metadata:
      name: '{{path.basename}}-{{path[1]}}'  # ex: dev-swagger-hub
    spec:
      source:
        repoURL: http://gitlab.example.com/myorg/sample-k8s-deploy.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path[1]}}-{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

→ `apps/` 하위에 디렉토리만 추가하면 ArgoCD Application 자동 생성

### 13-2. 멀티 클러스터 전략

```
deploy-manifests/
├── apps/
│   └── swagger-hub/
│       ├── base/
│       └── overlays/
│           ├── dev/        → cluster: eks-dev
│           ├── staging/    → cluster: eks-staging
│           └── production/ → cluster: eks-production
└── argocd/
    └── applicationset.yaml → 각 overlay를 해당 클러스터로 배포
```

### 13-3. 배포 전략 (Deployment Strategy)

**Canary (카나리)**: 소수 트래픽으로 먼저 검증

```
100% → v1
90% v1 + 10% v2  (카나리 배포)
관찰 → 문제 없음 → 50/50 → 100% v2
```

**Blue-Green**: 두 환경을 완전히 유지, 트래픽 전환

```
Blue(v1): 100% 트래픽 수신 중
Green(v2): 대기 중
전환: Service selector를 Blue → Green으로 변경
문제 시: 즉시 Blue로 복귀
```

**도구**: Argo Rollouts (Canary/Blue-Green 자동화)

### 13-4. GitOps 거버넌스

**Branch 전략**:
```
main ← 언제나 프로덕션 상태
  dev: 개발 환경 반영
  staging: 스테이징 환경 반영
  (또는 단일 main 브랜치에서 overlay로 환경 분리)
```

**PR 기반 배포**:
- dev → staging 프로모션: PR + Approval → merge
- staging → production 프로모션: PR + 다중 Approval + 릴리즈 노트

### 13-5. 학습 체크

- [ ] ApplicationSet의 Git Generator가 어떻게 동작하는지 설명할 수 있다
- [ ] 멀티 클러스터에서 ArgoCD가 어떻게 배포를 관리하는지 설명할 수 있다
- [ ] Canary와 Blue-Green 배포의 트레이드오프를 설명할 수 있다
- [ ] 프로덕션 배포에 수동 승인을 추가하는 ArgoCD 설정을 작성할 수 있다

---

## Phase 2 최종 체크리스트

- [ ] `kubectl apply` → 컨테이너 실행까지의 전체 흐름을 각 컴포넌트 역할과 함께 설명할 수 있다
- [ ] EKS와 self-managed K8s의 관리 책임 경계를 명확히 알고 있다
- [ ] 새 팀원이 최소 권한으로 kubectl을 사용할 수 있도록 RBAC을 설계할 수 있다
- [ ] 현재 프로젝트에 resources, probe, HPA, PDB를 추가하는 production 환경 overlay를 작성할 수 있다
- [ ] ArgoCD ApplicationSet으로 새 앱 추가를 자동화하는 구성을 작성할 수 있다
- [ ] 장애 발생 시 Prometheus 메트릭과 CloudWatch 로그로 원인을 추적하는 흐름을 설명할 수 있다

---

## 학습 문서 시스템

> 매 학습 세션이 끝날 때마다 해당 단계의 내용을 `docs/` 하위 MD 파일로 저장한다.
> 생성된 문서들은 복습 자료로 활용한다.

### 디렉토리 구조

```
docs/
├── README.md                          ← 학습 문서 전체 인덱스
├── phase1-practical/                  ← Phase 1 학습 노트
│   ├── 01-container-k8s-basics.md
│   ├── 02-project-yaml-deep-dive.md
│   ├── 03-kubectl-hands-on.md
│   ├── 04-kustomize-patterns.md
│   ├── 05-gitops-argocd.md
│   └── 06-troubleshooting.md
└── phase2-expert/                     ← Phase 2 학습 노트
    ├── 07-k8s-internals.md
    ├── 08-eks-deep-dive.md
    ├── 09-networking.md
    ├── 10-security-rbac.md
    ├── 11-production-hardening.md
    ├── 12-observability.md
    └── 13-advanced-gitops.md
```

### 학습 문서 작성 규칙

각 챕터 파일의 구성:

```
# [챕터 제목]

## 학습 일자
YYYY-MM-DD

## 핵심 개념 정리
배운 개념들을 내 언어로 재서술

## 현재 프로젝트 연결
이 개념이 프로젝트의 어느 부분과 연결되는지

## 현재 프로젝트 파일 해독 (주석 포함)
학습에서 다룬 YAML/설정 파일을 한 줄씩 주석으로 해독한 전체 코드 포함.
실제 프로젝트 파일 기준으로 작성하며, 환경별 차이가 있으면 비교표도 포함.

## 핵심 명령어 / 코드
실습에서 쓴 kubectl, kustomize 명령어 모음

## 헷갈렸던 점 & 해결
학습 중 막혔던 부분과 이해한 방식

## 복습 포인트
핵심 내용 5줄 이상. 단순 요약이 아니라 "이것만 보면 전체 내용이 떠오르는" 수준으로 상세하게 작성.
```

### 학습 문서 생성 시점

- 해당 단계의 **학습 체크** 항목을 70% 이상 통과했을 때 Claude에게 요청
- 요청 방법: `"이 단계 학습 내용을 docs/ 에 정리해줘"`

---

## 학습 세션 가이드

> **Claude와 함께 학습하는 방법**

### 세션 시작 방법

학습 세션 시작 시 Claude에게:
```
STUDY_PLAN.md를 기반으로 학습 세션을 시작할게.
현재 [단계명]까지 완료했고, [다음 단계]부터 학습하고 싶어.
```

### 학습 세션 유형

**개념 학습 세션**: 특정 단계의 이론 학습
```
예: "7단계의 Reconciliation Loop에 대해 예시를 들어 설명해줘"
예: "Service가 kube-proxy와 iptables를 어떻게 활용하는지 패킷 레벨로 설명해줘"
```

**실습 세션**: 직접 코드/설정 작성
```
예: "현재 프로젝트의 deployment.yaml에 resources와 probe를 추가해줘"
예: "staging 환경 overlay를 직접 작성하는 걸 도와줘"
```

**질문 세션**: 특정 개념의 깊이 있는 이해
```
예: "왜 K8s는 IP 기반이 아닌 라벨 셀렉터로 Pod를 선택하는가?"
예: "ArgoCD의 selfHeal이 무한루프에 빠지지 않는 이유는?"
```

**체크 세션**: 학습 내용 검증
```
예: "이 단계의 학습 체크 항목들을 나에게 질문해줘. 내가 답하면 맞는지 피드백해줘."
질문 개수는 5개에 국한하지 않고, 해당 단계의 내용 범위와 난이도에 따라 유동적으로 출제.
```

**문서화 세션**: 학습 내용을 docs/에 저장
```
예: "1단계 학습 내용을 docs/에 정리해줘"
```

### 현재 학습 진도

> 아래에 완료 날짜를 기록하세요

- Phase 1
  - [x] 1단계: 컨테이너 & K8s 핵심 개념 — 완료일: 2025-07-15
  - [x] 2단계: 현재 프로젝트 파일 완전 해독 — 완료일: 2025-07-15
  - [x] 3단계: kubectl 실전 조작 — 완료일: 2025-07-15
  - [x] 4단계: Kustomize 패턴 — 완료일: 2025-07-15
  - [x] 5단계: GitOps & ArgoCD 운영 — 완료일: 2025-07-15
  - [x] 6단계: 일상 운영 & 트러블슈팅 — 완료일: 2025-07-15

- Phase 2
  - [ ] 7단계: Kubernetes 내부 동작 원리 — 완료일: ______
  - [ ] 8단계: AWS EKS 심화 — 완료일: ______
  - [ ] 9단계: 네트워킹 심화 — 완료일: ______
  - [ ] 10단계: 보안 & RBAC — 완료일: ______
  - [ ] 11단계: 프로덕션 하드닝 — 완료일: ______
  - [ ] 12단계: 관찰 가능성 (Observability) — 완료일: ______
  - [ ] 13단계: 고급 GitOps & 운영 패턴 — 완료일: ______

---

## 참고 자료

### 공식 문서
- [Kubernetes 공식 문서 (한글)](https://kubernetes.io/ko/docs/home/)
- [Kustomize 공식 문서](https://kustomize.io/)
- [ArgoCD 공식 문서](https://argo-cd.readthedocs.io/en/stable/)
- [AWS EKS 공식 문서](https://docs.aws.amazon.com/eks/latest/userguide/)
- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)

### 한국어 학습 자료
- [subicura - 쿠버네티스 안내서](https://subicura.com/k8s/)
- [EKS Workshop (한글)](https://catalog.us-east-1.prod.workshops.aws/workshops/46236689-b414-4571-a734-eb04f5a25571/ko-KR)

### 실습 환경
- [Play with Kubernetes (무료 브라우저 K8s)](https://labs.play-with-k8s.com/)
- [Killercoda K8s 시나리오](https://killercoda.com/playgrounds/scenario/kubernetes)

### 이 프로젝트 관련
- 소스 레포: `http://gitlab.example.com/myorg/sample-k8s-deploy.git`
- Harbor 레지스트리: `registry.example.com`
- 앱 도메인: `app.example.com`
- 배포 네임스페이스: `swagger-dev`
