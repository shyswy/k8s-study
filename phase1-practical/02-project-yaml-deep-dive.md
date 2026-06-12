# 2단계: 현재 프로젝트 파일 완전 해독

> **상태**: ✅ 완료
> **학습 일자**: 2025-07-15
> **참고**: STUDY_PLAN.md 2단계

---

## 핵심 개념 정리

### Deployment 주요 필드

| 필드 | 역할 |
|------|------|
| `replicas` | Pod 유지 개수 |
| `selector.matchLabels` | 관리할 Pod를 라벨로 지정 (template.labels와 반드시 일치) |
| `template` | Pod 설계도 |
| `env` / `envFrom` | 환경변수 주입 (직접 지정 or ConfigMap/Secret 참조) |
| `resources.requests` | 스케줄러가 노드 선택 시 참고하는 최소 보장량 |
| `resources.limits` | 실제 사용 상한선 (CPU throttle, 메모리 OOMKill) |
| `readinessProbe` | 트래픽 받을 준비 체크. 실패 시 트래픽 차단 |
| `livenessProbe` | 앱 생존 체크. 실패 시 Pod 재시작 |

### Probe 3가지 방식

| 방식 | 용도 | 예시 |
|------|------|------|
| `httpGet` | API 서버 (HTTP 엔드포인트 있을 때) | `path: /api/health, port: 3000` |
| `tcpSocket` | 포트만 열려있으면 OK (DB, Consumer 등) | `port: 8080` |
| `exec` | 커스텀 스크립트로 판단 | `command: ["cat", "/tmp/healthy"]` |

- `initialDelaySeconds`: liveness > readiness로 설정 (앱 시작 중 무한 재시작 방지)

### Service port vs targetPort

```
Service port:80 = 이 Service에 접근하는 모든 것이 쓰는 포트
  - 같은 namespace Pod, 다른 namespace Pod, Ingress, TGB 모두 이 포트로 접근
targetPort:3000 = Pod의 실제 포트로 포워딩
```

Ingress는 Service가 아니다. "어느 Service의 어느 port로 보내라"는 라우팅 규칙.

### TGB (TargetGroupBinding)

- K8s 공식 리소스가 아닌 **AWS Load Balancer Controller의 CRD**
- Ingress 방식: K8s가 ALB를 생성/관리 (올인원)
- TGB 방식: 인프라팀이 ALB를 Terraform으로 별도 관리 + K8s는 연결만 (역할 분리)

### Kustomize images 오버라이드

- base는 레지스트리에 독립적으로 유지 (`image: backend:latest`)
- overlay에서 `newName`으로 레지스트리 주소, `newTag`로 태그 교체
- CI/CD는 overlay의 `newTag` 한 줄만 바꾸면 배포 완료
- 환경별 레지스트리가 다를 수 있으므로 base에 하드코딩하면 안 됨

### base / overlay 차이

- base 수정 → 모든 환경에 반영
- overlay 수정 → 해당 환경만 반영
- patch는 `metadata.name`으로 대상 리소스를 지정하고, 명시한 필드만 덮어씀

---

## 현재 프로젝트 파일 해독 (주석 포함)

### base/deployment.yaml

```yaml
apiVersion: apps/v1              # K8s API 버전. Deployment는 apps/v1 사용
kind: Deployment                 # 리소스 종류: Pod의 배포/관리자
metadata:
  name: backend                  # Deployment 이름 (namespace 내 고유)
  labels:
    app: backend                 # Deployment 자체에 붙는 라벨 (조회/필터용)
spec:
  replicas: 2                    # Pod를 항상 2개 유지. 죽으면 자동 재생성
  selector:
    matchLabels:
      app: backend               # "이 라벨의 Pod를 내가 관리한다" (template.labels와 반드시 일치)
  template:                      # ── 여기서부터 Pod 설계도 ──
    metadata:
      labels:
        app: backend             # Pod에 붙는 라벨. Service도 이 라벨로 Pod를 찾음
    spec:
      containers:
        - name: backend          # 컨테이너 이름
          image: backend:latest  # 컨테이너 이미지 (overlay에서 ECR 주소+태그로 오버라이드됨)
          ports:
            - containerPort: 3000  # 앱이 리스닝하는 포트 (문서화 목적, 실제 매핑은 Service가 함)
          env:
            - name: NODE_ENV
              value: production    # 환경변수 직접 주입 (overlay patch로 환경별 변경)
          resources:
            requests:
              cpu: 250m            # 스케줄러에게 "최소 0.25 vCPU 확보해줘"
              memory: 256Mi        # 스케줄러에게 "최소 256MB 확보해줘"
            limits:
              cpu: 500m            # 최대 0.5 vCPU (초과 시 CPU throttle)
              memory: 512Mi        # 최대 512MB (초과 시 OOMKill → Pod 재시작)
          readinessProbe:          # "트래픽 받을 준비 됐나?" — 실패 시 Service에서 제외 (트래픽 차단)
            httpGet:
              path: /api/health    # 이 경로로 GET 요청
              port: 3000           # 이 포트로 요청
            initialDelaySeconds: 5 # 컨테이너 시작 후 5초 뒤부터 체크 시작
            periodSeconds: 10      # 10초마다 반복 체크
          livenessProbe:           # "앱이 살아있나?" — 실패 시 Pod 재시작
            httpGet:
              path: /api/health
              port: 3000
            initialDelaySeconds: 10  # readiness보다 길게 (시작 중 무한 재시작 방지)
            periodSeconds: 15        # 15초마다 반복 체크
```

### base/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend                  # Service 이름 = 클러스터 내부 DNS 이름
  labels:                        #   같은 ns: curl http://backend:80
    app: backend                 #   다른 ns: curl http://backend.<ns>.svc.cluster.local:80
spec:
  type: ClusterIP                # 클러스터 내부에서만 접근 가능 (기본값)
  selector:
    app: backend                 # 이 라벨의 Pod로 트래픽 전달 (Deployment의 template.labels와 일치)
  ports:
    - port: 80                   # Service 포트 — Pod/Ingress/TGB 등 모든 접근자가 쓰는 포트
      targetPort: 3000           # Pod의 실제 포트로 포워딩 (port:80 → targetPort:3000)
      protocol: TCP
```

### base/tgb.yaml

```yaml
apiVersion: elbv2.k8s.aws/v1beta1    # AWS Load Balancer Controller의 CRD (K8s 공식 아님)
kind: TargetGroupBinding
metadata:
  name: backend-tgb
  labels:
    app: backend
spec:
  serviceRef:
    name: backend                      # 이 Service를
    port: 80                           # 이 포트로
  targetGroupARN: <BACKEND_TARGET_GROUP_ARN>   # 이 AWS Target Group에 연결
  targetType: ip                       # Pod IP를 직접 Target Group에 등록
  # Ingress와 차이: ALB를 K8s가 만드는 게 아니라, 이미 있는 ALB의 TG에 연결만 함
  # → 인프라팀이 Terraform으로 ALB 관리, K8s는 연결만 담당 (역할 분리)
```

### base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:                # 이 base에 포함할 파일 목록
  - deployment.yaml       # Pod 배포 정의
  - service.yaml          # 클러스터 내부 네트워크 접근점
  - tgb.yaml              # AWS ALB Target Group 연결
  # HPA 없음 → Pod 수 고정 (replicas 값 그대로)
```

### overlays/dev/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base                    # base 폴더의 모든 리소스를 가져옴

namespace: dev                    # base의 모든 리소스에 namespace: dev 적용

patches:
  - path: patch-env.yaml          # base Deployment를 환경별 설정으로 덮어씀

images:
  - name: backend                 # base에서 image: backend 를 찾아서
    newName: <AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/backend
    #        ↑ ECR 레지스트리 주소로 교체 (base는 레지스트리 독립적으로 유지)
    newTag: latest                # ← CI/CD가 커밋 해시로 자동 업데이트하는 부분
```

### overlays/dev/patch-env.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend               # 이 이름의 Deployment를 패치 대상으로 지정
spec:
  replicas: 1                  # base의 2 → 1로 덮어씀 (dev는 Pod 1개면 충분)
  template:
    spec:
      containers:
        - name: backend
          env:
            - name: NODE_ENV
              value: development   # base의 production → development로 덮어씀
          resources:
            requests:
              cpu: 100m            # base의 250m → 100m (dev는 자원 적게)
              memory: 128Mi        # base의 256Mi → 128Mi
            limits:
              cpu: 250m            # base의 500m → 250m
              memory: 256Mi        # base의 512Mi → 256Mi
# 명시하지 않은 필드(selector, labels, probe 등)는 base 그대로 유지됨
```

### 환경별 차이 비교 (patch-env.yaml)

| 항목 | base (원본) | dev | qa | st |
|------|-------------|-----|----|----|
| replicas | 2 | **1** | 2 | 2 |
| NODE_ENV | production | **development** | **qa** | **staging** |
| CPU req/limit | 250m/500m | **100m/250m** | 250m/500m | 250m/500m |
| Memory req/limit | 256Mi/512Mi | **128Mi/256Mi** | 256Mi/512Mi | 256Mi/512Mi |

### monitoring 앱 (MSK Consumer) — Service 없는 구조

```yaml
# monitoring/incident-alert/base/kustomization.yaml
resources:
  - deployment.yaml    # Deployment만 존재. Service 없음, TGB 없음
  # → MSK에서 메시지를 "가져오는" 쪽이라 아무도 이 Pod에 접속할 일 없음
```

```yaml
# monitoring/incident-alert/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: incident-alert
spec:
  replicas: 1
  # ...
  containers:
    - name: incident-alert
      image: registry.example.com/myorg/incident-alert:latest
      imagePullPolicy: Always          # 항상 최신 이미지 pull (Harbor 사용)
      ports:
        - containerPort: 8080          # 관리/디버깅용 포트 (외부 트래픽 아님)
      envFrom:
        - configMapRef:
            name: app-config   # ConfigMap의 모든 key=value를 환경변수로 주입
        - secretRef:
            name: app-secret   # Secret의 모든 key=value를 환경변수로 주입
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 250m
          memory: 256Mi
      # ⚠️ probe 미설정 — 앱이 데드락 걸려도 K8s가 "정상"으로 판단
      # 프로덕션에서는 tcpSocket 또는 exec 방식 probe 추가 권장
```

### kustomize build 실행 순서

```
1. base 리소스 로드        (deployment, service, tgb)
2. patch 적용              (환경별 replicas, env, resources 덮어씀)
3. images 적용             (레지스트리 주소 + 태그 교체)
4. namespace 적용          (모든 리소스에 namespace 추가)
5. 최종 YAML 출력          → ArgoCD가 이걸 클러스터에 적용
```

---

## 헷갈렸던 점 & 해결

1. **Service port는 "다른 서비스 접근용"만이 아니다**
   → Pod, Ingress, TGB 등 Service에 접근하는 모든 주체가 쓰는 포트

2. **TGB vs Ingress 차이**
   → Ingress는 K8s가 ALB까지 관리. TGB는 이미 있는 ALB에 연결만. 역할 분리 목적

3. **images 오버라이드를 왜 쓰나 (하드코딩 대비)**
   → 환경별 레지스트리가 다를 수 있고, CI/CD가 newTag 한 줄만 바꾸면 되는 구조

4. **API 서버가 아닌 앱의 probe**
   → httpGet 외에 tcpSocket, exec 방식 사용 가능. monitoring 앱들은 현재 probe 미설정

---

## 복습 포인트

1. Deployment의 selector.matchLabels ↔ template.metadata.labels 반드시 일치
2. Service port = 접근 포트, targetPort = Pod 실제 포트
3. TGB = AWS CRD, 이미 있는 ALB Target Group과 Service를 연결 (Ingress와 역할 분리)
4. base는 레지스트리 독립적 유지, overlay에서 images로 환경별 교체
5. 배포 = overlay의 newTag 변경 → Git push → ArgoCD 자동 동기화
6. patch는 명시한 필드만 덮어쓰고, 나머지는 base 그대로 유지
7. MSK Consumer 같은 앱은 Service 불필요 (아무도 접근할 일 없음)
