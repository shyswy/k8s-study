# 4단계: Kustomize 패턴

## 학습 일자
2025-07-15

## 핵심 개념 정리

Kustomize는 한마디로 **"YAML 조합기"**. Helm처럼 템플릿 문법(`{{ }}`)을 쓰지 않고, 순수 YAML만으로 환경별 설정을 관리한다.

### base/overlay 패턴

- **base**: 모든 환경에 공통 적용되는 설정 (deployment, service, ingress)
- **overlay**: 환경별 차이점만 덮어쓰기 (namespace, 이미지 태그, replicas 등)
- 핵심: **base를 중복 복사하지 않고**, 변경점만 overlay에서 관리

```
base의 YAML 3개 → kustomize build → dev overlay가 namespace, 이미지 태그 덮어씀 → 최종 YAML 출력
```

현재 dev overlay가 하는 일은 딱 2가지:
1. 모든 리소스에 `namespace: swagger-dev` 적용
2. 이미지 태그를 `latest` → `bd467fae`(커밋 해시)로 교체

### 주요 오버라이드 패턴 5가지

| 패턴 | 용도 | 현재 프로젝트 사용 |
|------|------|-------------------|
| `namespace` | 전체 리소스에 네임스페이스 지정 | ✅ |
| `images` | 이미지 이름/태그 교체 | ✅ |
| `patches` (Strategic Merge Patch) | 특정 필드만 덮어쓰기 | ❌ |
| `configMapGenerator` | ConfigMap 자동 생성 | ❌ |
| `commonLabels` | 모든 리소스에 공통 라벨 추가 | ❌ |

가장 실무에서 중요한 건 **patches** 패턴. 환경별로 replicas, resources, probe 등을 다르게 설정할 때 쓴다.

### Strategic Merge Patch

base 전체를 복사하지 않고, **바꾸고 싶은 필드만** 적으면 나머지는 base 값 유지.

```yaml
# overlays/staging/patch-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: swagger-hub      # kind + name으로 base의 대상 리소스 매칭
spec:
  replicas: 2            # 이 필드만 덮어씀
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

### Kustomize vs Helm

| | Kustomize | Helm |
|--|-----------|------|
| 접근 방식 | 순수 YAML 패치/병합 | Go 템플릿 (`{{ .Values.replicas }}`)으로 변수 주입 |
| 학습 곡선 | 낮음 | 높음 (템플릿 문법) |
| 유연성 | 단순한 오버라이드에 적합 | 복잡한 조건부 로직 가능 |
| kubectl 내장 | ✅ (`kubectl apply -k`) | ❌ (별도 설치) |
| 현재 프로젝트 | ✅ 사용 중 | 미사용 |

---

## 현재 프로젝트 연결

### 현재 프로젝트 구조

```
apps/swagger-hub/
├── base/                      ← 공통 설정 (모든 환경에 적용)
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── overlays/
    └── dev/                   ← dev 환경 전용 오버라이드
        └── kustomization.yaml
```

### 현재 프로젝트 파일 해독 (주석 포함)

**base/kustomization.yaml** — 리소스 목록 선언:
```yaml
resources:          # 이 base에 포함할 파일 목록 (이것만 가져감, 디렉토리 전체가 아님)
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

**overlays/dev/kustomization.yaml** — 오버라이드 선언:
```yaml
resources:
  - ../../base           # base의 kustomization.yaml을 읽어서 거기 선언된 리소스만 가져옴
                         # (base 디렉토리 하위 파일을 무조건 전부 긁어오는 게 아님!)

namespace: swagger-dev   # 최종 YAML의 모든 리소스에 이 namespace를 텍스트로 주입
                         # Kustomize는 클러스터에 namespace가 있는지 모름
                         # 없으면 kubectl apply 시점에 에러 발생

images:                  # base에서 name이 일치하는 이미지를 찾아서 치환
  - name: registry.example.com/myorg/swagger-hub  # 매칭 대상 (이 이름을 찾아라)
    newTag: bd467fae     # 태그만 교체 (newName 없으니 주소는 유지)
```

### 다른 프로젝트 예시 (ECR 사용 시)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: dev

patches:
  - path: patch-env.yaml    # 패치 파일 경로 지정

images:
  - name: backend                                                    # 매칭 대상
    newName: <AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/backend  # 주소 자체 교체
    newTag: latest                                                    # 태그도 교체
```

### 양쪽 kustomization.yaml이 필요한 이유

| 파일 | 역할 |
|------|------|
| `base/kustomization.yaml` | "이 디렉토리에 어떤 리소스 파일들이 있는지" 목록 선언 |
| `overlays/dev/kustomization.yaml` | "base를 가져와서 어떤 변경을 적용할지" 오버라이드 선언 |

```
base/kustomization.yaml        → "deployment, service, ingress 3개가 있다"
overlays/dev/kustomization.yaml → "base 전부 가져와서 namespace=dev, 이미지태그=xxx로 바꿔라"
```

base의 kustomization.yaml이 없으면 overlay에서 `resources: - ../../base`를 해도 Kustomize가 base 디렉토리에서 뭘 읽어야 할지 모른다. **양쪽 다 필수.**

---

## 핵심 명령어 / 코드

```bash
# Kustomize 빌드 결과 미리보기 (적용 안 함)
kubectl kustomize apps/swagger-hub/overlays/dev/

# Kustomize로 클러스터에 적용
kubectl apply -k apps/swagger-hub/overlays/dev/

# 환경별 diff 비교
diff <(kubectl kustomize apps/swagger-hub/overlays/dev/) \
     <(kubectl kustomize apps/swagger-hub/overlays/staging/)
```

### staging overlay 추가 방법

```bash
mkdir -p apps/swagger-hub/overlays/staging
```

```yaml
# apps/swagger-hub/overlays/staging/kustomization.yaml
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

---

## 헷갈렸던 점 & 해결

- **Q**: patch-deployment.yaml / kustomization 등의 네이밍이 정확히 일치해야 되는 거야? 아니면 kind만 보고 알아서 하는 거?
- **A**: 두 가지를 나눠서 봐야 한다:
  - `kustomization.yaml` → **이름 고정 필수**. Kustomize가 디렉토리를 읽을 때 이 이름의 파일을 자동으로 찾음. 다른 이름이면 인식 못함. (`kustomization.yml`, `Kustomization`도 허용)
  - 패치 파일 → **이름 자유**. `abc.yaml`이든 `my-patch.yaml`이든 상관없음. kustomization.yaml에서 `patches: - path: 파일명`으로 경로를 명시적으로 지정하고, 파일 내부의 `kind` + `metadata.name`으로 base의 어떤 리소스를 패치할지 매칭.
  - 실무에서는 `patch-deployment.yaml`, `patch-ingress.yaml` 같이 뭘 패치하는지 알기 쉬운 이름을 관례적으로 씀.

- **Q**: `resources: - ../../base`가 base 디렉토리의 모든 파일을 가져오나?
- **A**: 아니다. base 디렉토리의 `kustomization.yaml`을 읽어서, 거기에 선언된 resources 목록만 가져온다. base에 `readme.md`가 있어도 무시됨.

- **Q**: namespace가 클러스터에 없으면 어떻게 되나?
- **A**: Kustomize는 단순히 최종 YAML 출력에 `namespace: dev`를 텍스트로 주입할 뿐. 실제 클러스터에 존재하는지는 모름. 없으면 `kubectl apply` 시점에 `Error: namespaces "dev" not found` 에러 발생. 그래서 보통 namespace를 미리 만들어두거나, kustomization.yaml에 namespace 리소스를 포함시킴.

- **Q**: images의 name / newName / newTag 차이?
- **A**:
  - `name` → 매칭 대상 (base에서 이 이미지 이름을 찾아라)
  - `newName` → 이미지 **주소 자체**를 교체 (예: `backend` → `123456.dkr.ecr.../backend`)
  - `newTag` → **태그만** 교체 (예: `latest` → `abc123`)
  - 둘 다 쓰면 둘 다 교체

- **Q**: base에도 kustomization 있고, overlay에도 있는데 이유?
- **A**: 역할이 다르다. base의 것은 "어떤 리소스 파일들이 있는지" 목록 선언, overlay의 것은 "base를 가져와서 어떤 변경을 적용할지" 오버라이드 선언. base의 kustomization.yaml이 없으면 overlay에서 뭘 읽어야 할지 모르므로 양쪽 다 필수.

---

## 학습 체크 결과 (4/5 통과)

| 질문 | 결과 | 피드백 |
|------|------|--------|
| base/overlay 패턴의 목적을 한 문장으로 | ✅ | "공통 설정(base)은 한 벌만 유지하고, 환경별 차이점만 overlay에서 덮어쓰는 것" |
| base 수정 없이 staging replicas 2로 | ✅ | overlay에서 patch yaml 만들고 kustomization.yaml의 patches에 등록 |
| Kustomize와 Helm의 차이 | 🔶 | Kustomize = 순수 YAML 패치/병합은 맞지만, Helm = Go 템플릿 문법으로 변수 주입이라는 설명이 빠짐 |
| resources: - ../../base가 뭘 가져오나 | ✅ | base의 kustomization.yaml에 선언된 resources 목록만 가져옴 |
| newName과 newTag의 차이 | 🔶 | name(매칭 대상) / newName(주소 교체) / newTag(태그 교체) 구분이 살짝 섞임 |

---

## 복습 포인트

1. Kustomize의 핵심 = base(공통)는 한 벌만 유지, overlay(환경별 차이)에서 덮어쓰기. base를 복사하지 않는다.
2. `kustomization.yaml`은 이름 고정 필수. 패치 파일은 이름 자유 — kustomization.yaml에서 path로 지정하고, 파일 내부 `kind` + `metadata.name`으로 대상 매칭.
3. `resources: - ../../base`는 base 디렉토리의 kustomization.yaml에 선언된 리소스만 가져온다. 디렉토리 전체를 긁어오는 게 아님.
4. images 섹션: `name`은 매칭 대상, `newName`은 주소 교체, `newTag`는 태그 교체. 현재 프로젝트는 newTag만 사용 (CI/CD가 커밋 해시로 자동 업데이트).
5. Strategic Merge Patch는 바꾸고 싶은 필드만 적으면 나머지는 base 값 유지. replicas, resources, probe 등 환경별 차이를 관리하는 핵심 패턴.
6. namespace 필드는 Kustomize가 텍스트로 주입할 뿐. 클러스터에 해당 namespace가 없으면 apply 시 에러 발생.
7. Kustomize vs Helm: Kustomize는 순수 YAML 패치, Helm은 Go 템플릿 변수 주입. Kustomize는 kubectl에 내장되어 별도 설치 불필요.
8. base와 overlay 양쪽에 kustomization.yaml이 필요한 이유: base는 "어떤 파일이 있는지" 목록, overlay는 "어떤 변경을 적용할지" 선언. 역할이 다르고 둘 다 필수.
