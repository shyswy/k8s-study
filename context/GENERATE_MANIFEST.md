# EKS Kustomize 매니페스트 자동 생성 프롬프트

이 문서를 AI에게 컨텍스트로 제공하면, 필요한 정보를 질문한 뒤 Kustomize 매니페스트를 자동 생성합니다.

---

## 레포지토리 구조

이 레포는 `projects/<프로젝트명>/<서비스명>/` 하위에 Kustomize base/overlays 구조로 K8s 매니페스트를 관리한다.

```
projects/
└── <프로젝트명>/
    └── <서비스명>/
        ├── base/
        │   ├── deployment.yaml
        │   ├── kustomization.yaml
        │   ├── service.yaml          # HTTP 서비스인 경우만
        │   └── tgb.yaml              # TargetGroupBinding - ALB 연동 시만
        └── overlays/
            └── <환경>/
                ├── kustomization.yaml
                └── patch-env.yaml     # 환경별 오버라이드 필요 시
```

---

## AI에게 주는 지시사항

아래 순서대로 사용자에게 질문하고, 답변을 바탕으로 매니페스트를 생성하라.

### STEP 1: 기본 정보 수집

아래 표를 채워달라고 요청하라. 사용자가 모르는 항목은 기본값을 제안하라.

| 질문 | 예시 | 기본값 |
|------|------|--------|
| 프로젝트명 (디렉토리명) | `monitoring`, `bc-backoffice` | - (필수) |
| 서비스 목록 (쉼표 구분) | `th-checker, incident-alert, batch` | - (필수) |
| 배포 환경 목록 | `predev`, `dev, qa, st` | `dev` |
| K8s 네임스페이스 | `app-backend`, `dev` | 프로젝트명 |

### STEP 2: 서비스별 컨테이너 특성 파악

각 서비스마다 아래를 질문하라. 답변에 따라 생성할 리소스가 달라진다.

| 질문 | 선택지 | 영향 |
|------|--------|------|
| **컨테이너 유형** | `http` / `worker` / `cron` | worker/cron이면 Service, TGB 생성 안 함 |
| **이미지 레지스트리** | Harbor(`registry.example.com/myorg/<name>`) / ECR(`<account>.dkr.ecr.<region>.amazonaws.com/<name>`) | base image 경로, overlay newName 결정 |
| **imagePullSecrets 필요?** | `yes` → secret 이름 / `no` | Harbor면 보통 `registry-secret` 필요 |
| **컨테이너 포트** | 숫자 | `3000` (http) / `8080` (worker 헬스체크) |
| **헬스체크 경로** | `/api/health`, `/health`, 없음 | readiness/liveness probe 설정 |
| **환경변수 주입 방식** | `envFrom` (ConfigMap/Secret ref) / `env` (직접 지정) / `patch` (overlay에서 patch) | deployment env 섹션 구조 결정 |
| **외부 트래픽 수신?** | `yes` → Service + TGB 생성 / `no` | Service, TGB yaml 생성 여부 |

> 서비스가 여러 개이고 특성이 동일하면 "전부 동일" 답변을 허용하라.

### STEP 3: 리소스 설정

| 질문 | 예시 | 기본값 |
|------|------|--------|
| replicas | `1`, `2` | `1` (dev/predev), `2` (qa/prd) |
| CPU requests / limits | `100m / 250m`, `250m / 500m` | `100m / 250m` |
| Memory requests / limits | `128Mi / 256Mi`, `256Mi / 512Mi` | `128Mi / 256Mi` |
| 환경별로 리소스 다르게? | `yes` → overlay patch 생성 / `no` | `no` |

### STEP 4: 환경변수 (envFrom 방식인 경우)

| 질문 | 예시 |
|------|------|
| ConfigMap 이름 | `app-config` |
| Secret 이름 | `app-secret` |

> ConfigMap/Secret yaml 자체는 이 레포에서 생성하지 않는다. deployment에서 참조만 한다.

### STEP 5: 환경변수 (env 직접 지정 방식인 경우)

| 질문 | 예시 |
|------|------|
| base에 넣을 공통 env | `NODE_ENV=production` |
| overlay에서 오버라이드할 env | `NODE_ENV=development` (dev), `NODE_ENV=qa` (qa) |

### STEP 6: ALB 연동 (외부 트래픽 수신 서비스만)

| 질문 | 예시 | 기본값 |
|------|------|--------|
| Service type | `ClusterIP`, `NodePort` | `ClusterIP` |
| Service port → targetPort | `80 → 3000` | - |
| TargetGroupBinding ARN | 확정값 또는 placeholder | `<SERVICE_TARGET_GROUP_ARN>` |

---

## 생성 규칙

수집한 정보를 바탕으로 아래 규칙에 따라 파일을 생성하라.

### deployment.yaml (base)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <서비스명>
  labels:
    app: <서비스명>
spec:
  replicas: <replicas>
  selector:
    matchLabels:
      app: <서비스명>
  template:
    metadata:
      labels:
        app: <서비스명>
    spec:
      # imagePullSecrets 필요한 경우만
      imagePullSecrets:
        - name: <secret명>
      containers:
        - name: <서비스명>
          image: <레지스트리>/<서비스명>:latest
          imagePullPolicy: Always    # latest 태그 사용 시
          ports:
            - containerPort: <포트>
          # envFrom 방식
          envFrom:
            - configMapRef:
                name: <configmap명>
            - secretRef:
                name: <secret명>
          # 또는 env 직접 지정 방식
          env:
            - name: <KEY>
              value: <VALUE>
          resources:
            requests:
              cpu: <cpu_req>
              memory: <mem_req>
            limits:
              cpu: <cpu_lim>
              memory: <mem_lim>
          # 헬스체크 경로가 있는 경우만
          readinessProbe:
            httpGet:
              path: <헬스체크경로>
              port: <포트>
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: <헬스체크경로>
              port: <포트>
            initialDelaySeconds: 10
            periodSeconds: 15
```

### kustomization.yaml (base)

```yaml
# HTTP 서비스 (Service + TGB 있음)
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - tgb.yaml

# Worker/Cron 서비스 (Deployment만)
resources:
  - deployment.yaml
```

### service.yaml (base) — HTTP 서비스만

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <서비스명>
  labels:
    app: <서비스명>
spec:
  type: <Service type>
  selector:
    app: <서비스명>
  ports:
    - port: <service port>
      targetPort: <container port>
      protocol: TCP
```

### tgb.yaml (base) — ALB 연동 서비스만

```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: <서비스명>-tgb
  labels:
    app: <서비스명>
spec:
  serviceRef:
    name: <서비스명>
    port: <service port>
  targetGroupARN: <TGB_ARN 또는 placeholder>
  targetType: ip
```

### kustomization.yaml (overlay)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: <네임스페이스>

# patch가 있는 경우
patches:
  - path: patch-env.yaml

images:
  - name: <base에서 쓴 이미지명>
    newName: <실제 레지스트리 경로>   # base와 다를 경우만
    newTag: latest
```

### patch-env.yaml (overlay) — 환경별 오버라이드 필요 시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <서비스명>
spec:
  replicas: <환경별 replicas>
  template:
    spec:
      containers:
        - name: <서비스명>
          env:
            - name: <KEY>
              value: <환경별 VALUE>
          resources:
            requests:
              cpu: <환경별>
              memory: <환경별>
            limits:
              cpu: <환경별>
              memory: <환경별>
```

---

## 생성 후 체크리스트

파일 생성 후 아래를 자동으로 검증하라:

- [ ] 모든 서비스의 base/deployment.yaml 존재
- [ ] 모든 서비스의 base/kustomization.yaml에 리소스 참조 정확
- [ ] HTTP 서비스에 service.yaml, tgb.yaml 포함
- [ ] Worker/Cron 서비스에 service.yaml, tgb.yaml 미포함
- [ ] 모든 overlay의 kustomization.yaml에 namespace 지정
- [ ] 모든 overlay의 images 섹션에 올바른 이미지 경로
- [ ] placeholder(`<...>`) 남아있는 항목 목록 출력
- [ ] `kustomize build` 가능한 구조인지 확인

---

## 사용 예시

> 사용자: "이 컨텍스트 참고해서 새 프로젝트 매니페스트 만들어줘"
>
> AI: "아래 정보를 알려주세요:
> 1. 프로젝트명?
> 2. 서비스 목록?
> 3. 배포 환경?
> ..."
>
> (답변 수집 후 파일 자동 생성)
