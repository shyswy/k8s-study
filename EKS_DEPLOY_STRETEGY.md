Design Document: EKS + ArgoCD + Kustomize 배포 전략
Overview
본 설계 문서는 AWS EKS 클러스터에 ArgoCD와 Kustomize를 활용한 GitOps 기반 배포 파이프라인을 구축하기 위한 아키텍처와 구성 요소를 정의한다.

핵심 구성 요소 역할
EKS	실행 환경	컨테이너화된 애플리케이션이 실행되는 관리형 Kubernetes 클러스터
ArgoCD	배포 엔진	Git 저장소를 감시하고 클러스터 상태를 자동으로 동기화하는 GitOps CD 도구
Kustomize	구성 관리	환경별 Kubernetes 매니페스트를 base/overlay 패턴으로 관리하는 도구
GitOps 배포 흐름
개발자 코드 푸시 → CI 빌드/테스트 → 이미지 빌드 & 푸시 → 매니페스트 이미지 태그 업데이트 → Git 커밋
                                                                                              ↓
                                                                              ArgoCD가 변경 감지 → Kustomize로 매니페스트 렌더링 → EKS 클러스터에 배포
Architecture
전체 아키텍처 다이어그램
graph TB
   subgraph "개발자"
       DEV[개발자] -->|코드 푸시| APP_REPO[App 소스 저장소]
   end
 
   subgraph "CI Pipeline - GitLab CI"
       APP_REPO -->|트리거| CI[CI Pipeline]
       CI -->|빌드 & 푸시| ECR[ECR<br/>Container Registry]
       CI -->|이미지 태그 업데이트| DEPLOY_REPO[배포 매니페스트 저장소]
   end
 
   subgraph "GitOps - ArgoCD"
       DEPLOY_REPO -->|폴링/웹훅| ARGOCD[ArgoCD]
       ARGOCD -->|kustomize build| KUSTOMIZE[Kustomize 렌더링]
       KUSTOMIZE -->|매니페스트 적용| EKS
   end
 
   subgraph "AWS EKS Cluster"
       EKS[EKS Control Plane]
       EKS --> NG_DEV[Node Group - Dev]
       EKS --> NG_STG[Node Group - Staging]
       EKS --> NG_PRD[Node Group - Production]
   end
 
   subgraph "모니터링"
       ARGOCD -->|상태 알림| NOTIFY[Slack/Teams 알림]
   end
저장소 분리 전략
GitOps에서는 애플리케이션 소스 저장소와 배포 매니페스트 저장소를 분리하는 것을 권장한다.

App Source Repo (프로젝트별)	애플리케이션 소스 코드, Dockerfile, CI 파이프라인	개발팀
Deploy Manifest Repo (단일)	전체 프로젝트의 Kustomize base/overlay, ArgoCD ApplicationSet	DevOps팀
매니페스트 저장소 운영 방식: 모노레포
배포 매니페스트 저장소는 모노레포(Mono-repo) 방식으로 다수의 프로젝트를 단일 저장소에서 관리한다.

모노레포를 선택한 이유:
- 한 곳에서 전체 배포 현황을 파악할 수 있음
- 공통 리소스(네임스페이스 정책, 네트워크 정책 등)를 프로젝트 간 쉽게 공유
- ArgoCD ApplicationSet의 Git Generator를 활용하면 디렉토리만 추가하면 자동으로 Application이 생성됨
- 배포 표준(리소스 제한, 라벨링 등)을 일관되게 적용 가능

프로젝트 추가 시 작업:
1. apps/{프로젝트명}/base/ 에 공통 매니페스트 작성
2. apps/{프로젝트명}/overlays/{환경}/ 에 환경별 패치 작성
3. ApplicationSet이 자동으로 ArgoCD Application을 생성 → 별도 CRD 작성 불필요

Components and Interfaces
1. EKS 클러스터
역할: 컨테이너 워크로드의 실행 환경을 제공한다.

구성 요소:
- Control Plane: AWS가 관리하는 Kubernetes API Server, etcd, Controller Manager, Scheduler
- Node Group: EC2 인스턴스 기반의 워커 노드 그룹 (Managed Node Group 사용 권장)
- Add-ons: CoreDNS, kube-proxy, VPC CNI, EBS CSI Driver

인터페이스:
- kubectl / kubeconfig를 통한 클러스터 접근
- AWS IAM을 통한 인증 (aws-auth ConfigMap 또는 EKS Access Entry)
- VPC 내부 네트워크를 통한 서비스 간 통신

# EKS 클러스터 구성 예시 (eksctl)
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
 
metadata:
  name: my-eks-cluster
  region: ap-northeast-2
  version: "1.29"
 
vpc:
  id: vpc-xxxxxxxxx
  subnets:
    private:
      ap-northeast-2a: { id: subnet-aaa }
      ap-northeast-2b: { id: subnet-bbb }
      ap-northeast-2c: { id: subnet-ccc }
 
managedNodeGroups:
  - name: app-nodegroup
    instanceType: m5.xlarge
    desiredCapacity: 3
    minSize: 2
    maxSize: 10
    privateNetworking: true
    labels:
      role: app
    tags:
      Environment: production
2. ArgoCD
역할: Git 저장소의 선언적 매니페스트를 Kubernetes 클러스터와 지속적으로 동기화한다.

구성 요소:

argocd-server	API 서버 및 웹 UI 제공
argocd-repo-server	Git 저장소 클론, 매니페스트 렌더링 (Kustomize/Helm)
argocd-application-controller	Application CRD 감시, 클러스터 상태 비교 및 동기화
argocd-redis	캐싱 레이어
argocd-dex-server	SSO/OIDC 인증 (선택)
동작 흐름:

sequenceDiagram
   participant Git as Git Repository
   participant Repo as Repo Server
   participant Ctrl as App Controller
   participant K8s as EKS Cluster
 
   Ctrl->>Git: 주기적 폴링 (3분 기본) 또는 Webhook
   Git-->>Ctrl: 변경 감지
   Ctrl->>Repo: 매니페스트 렌더링 요청
   Repo->>Git: 저장소 클론
   Repo->>Repo: kustomize build 실행
   Repo-->>Ctrl: 렌더링된 매니페스트 반환
   Ctrl->>K8s: 현재 상태 조회
   Ctrl->>Ctrl: Desired vs Live 상태 비교
   alt Auto-Sync 활성화
       Ctrl->>K8s: kubectl apply 실행
   else Manual Sync
       Ctrl->>Ctrl: OutOfSync 상태 표시
   end
ArgoCD Application CRD 예시:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.example.com/devops/deploy-manifests.git
    targetRevision: main
    path: overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
3. Kustomize
역할: 템플릿 없이 base/overlay 패턴으로 환경별 Kubernetes 매니페스트를 관리한다.

핵심 개념:

Base	모든 환경에 공통으로 적용되는 기본 매니페스트
Overlay	환경별로 base 위에 덮어쓰는 패치 및 설정
Patch	기존 리소스의 특정 필드만 변경하는 부분 수정
kustomization.yaml	Kustomize 구성 파일로, 포함할 리소스와 변환 규칙을 정의
Kustomize vs Helm 비교:

접근 방식	오버레이 (패치)	템플릿 (Go template)
학습 곡선	낮음	중간
복잡한 로직	제한적	조건문, 반복문 지원
kubectl 통합	내장 (kubectl -k)	별도 설치 필요
가독성	순수 YAML	템플릿 문법 혼재
Data Models
디렉토리 구조 (모노레포)
deploy-manifests/
├── apps/
│   ├── site-business/
│   │   ├── base/
│   │   │   ├── kustomization.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── configmap.yaml
│   │   │   └── hpa.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   ├── kustomization.yaml
│   │       │   ├── namespace.yaml
│   │       │   └── patch-deployment.yaml
│   │       ├── staging/
│   │       │   ├── kustomization.yaml
│   │       │   ├── namespace.yaml
│   │       │   └── patch-deployment.yaml
│   │       └── production/
│   │           ├── kustomization.yaml
│   │           ├── namespace.yaml
│   │           ├── patch-deployment.yaml
│   │           └── patch-hpa.yaml
│   ├── device-lifecycle/
│   │   ├── base/
│   │   │   └── ...
│   │   └── overlays/
│   │       ├── dev/
│   │       ├── staging/
│   │       └── production/
│   └── content-service/
│       ├── base/
│       │   └── ...
│       └── overlays/
│           ├── dev/
│           ├── staging/
│           └── production/
├── argocd/
│   └── applicationset.yaml       ← Git Generator 기반 자동 Application 생성
└── common/
    ├── namespace-policies/        ← 전체 프로젝트 공통 정책
    └── network-policies/
Base 매니페스트 예시
apps/site-business/base/kustomization.yaml:

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
 
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
  - hpa.yaml
 
commonLabels:
  app: site-business
  managed-by: kustomize
base/deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
base/service.yaml:

apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  selector:
    app: my-app
Overlay 예시
overlays/dev/kustomization.yaml:

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
 
namespace: my-app-dev
 
resources:
  - ../../base
  - namespace.yaml
 
patches:
  - path: patch-deployment.yaml
 
images:
  - name: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app
    newTag: dev-abc1234
overlays/dev/patch-deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: my-app
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
          env:
            - name: APP_ENV
              value: development
            - name: LOG_LEVEL
              value: debug
overlays/production/kustomization.yaml:

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
 
namespace: my-app-production
 
resources:
  - ../../base
  - namespace.yaml
 
patches:
  - path: patch-deployment.yaml
  - path: patch-hpa.yaml
 
images:
  - name: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app
    newTag: v1.2.3
overlays/production/patch-deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: my-app
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: "2"
              memory: 2Gi
          env:
            - name: APP_ENV
              value: production
            - name: LOG_LEVEL
              value: warn
환경별 설정 비교
Replicas	1	2	3+ (HPA)
CPU Request	100m	250m	500m
Memory Request	128Mi	256Mi	512Mi
CPU Limit	250m	500m	2000m
Memory Limit	256Mi	512Mi	2Gi
Log Level	debug	info	warn
Sync Mode	Auto	Auto	Manual
HPA	미사용	미사용	min:3, max:10
Correctness Properties
A property is a characteristic or behavior that should hold true across all valid executions of a system-essentially, a formal statement about what the system should do.
Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

본 프로젝트는 인프라 구성(IaC) 및 매니페스트 관리 프로젝트로, 대부분의 요구사항이 ArgoCD/EKS의 내장 기능에 의존한다. 코드 레벨에서 검증 가능한 correctness properties는 Kustomize 매니페스트 구성과 CI 스크립트 동작에 집중한다.

Property 1: Base 리소스 포함 보장
For any Kustomize overlay 환경과 base에 정의된 리소스에 대해, kustomize build를 실행하면 base의 모든 리소스가 최종 출력에 포함되어야 한다.

Validates: Requirements 3.1

Property 2: Overlay 패치 병합 정확성
For any base 매니페스트와 overlay 패치 조합에 대해, kustomize build를 실행하면 패치에 명시된 필드가 base 값을 올바르게 오버라이드한 결과를 생성해야 한다.

Validates: Requirements 3.2

Property 3: Kustomize 출력 YAML 유효성
For any 유효한 base/overlay 구성에 대해, kustomize build의 출력은 파싱 가능한 유효한 YAML이어야 하며, 각 문서는 apiVersion, kind, metadata 필드를 포함해야 한다.

Validates: Requirements 3.3

Property 4: Application CRD 스키마 유효성
For any ArgoCD Application CRD 정의에 대해, 해당 YAML은 필수 필드(source.repoURL, source.path, destination.server, destination.namespace)를 포함해야 한다.

Validates: Requirements 4.1

Property 5: 이미지 태그 업데이트 일관성
For any 이미지 태그 값에 대해, 이미지 태그 업데이트 스크립트를 실행하면 kustomization.yaml의 images 섹션에 해당 태그가 정확히 반영되어야 한다.

Validates: Requirements 5.2

Property 6: 환경별 리소스 독립성
For any 두 개의 서로 다른 환경 overlay에 대해, 각각의 kustomize build 결과에서 리소스 요청/제한 값은 해당 환경의 overlay에 정의된 값과 일치해야 한다.

Validates: Requirements 6.4

Error Handling
Kustomize 빌드 오류
리소스 미발견	overlay에서 존재하지 않는 base 리소스 참조	kustomize build 시 명확한 오류 메시지 출력, CI에서 사전 검증
YAML 문법 오류	잘못된 YAML 형식	CI 파이프라인에서 YAML lint 단계 추가
패치 충돌	동일 필드에 대한 중복 패치	strategic merge patch 우선순위 규칙 적용
ArgoCD 동기화 오류
SyncFailed	매니페스트 적용 실패 (권한, 스키마 등)	ArgoCD UI에서 오류 상세 확인, 매니페스트 수정 후 재동기화
ComparisonError	Git 저장소 접근 불가	저장소 연결 설정 확인, 인증 토큰 갱신
Degraded	Pod 헬스 체크 실패	애플리케이션 로그 확인, 필요 시 롤백
CI/CD 파이프라인 오류
이미지 빌드 실패	Dockerfile 오류, 의존성 문제	CI 로그 확인, 로컬 빌드 테스트
이미지 푸시 실패	ECR 인증 만료, 권한 부족	IAM 역할 및 ECR 정책 확인
매니페스트 업데이트 실패	Git 충돌, 권한 부족	Git 토큰 갱신, 브랜치 보호 규칙 확인
Testing Strategy
이중 테스트 접근법
본 프로젝트는 인프라 구성 프로젝트이므로, 테스트는 매니페스트 유효성 검증과 스크립트 동작 검증에 집중한다.

Unit Tests
Kustomize 매니페스트 구조 검증 (각 환경별 kustomize build 성공 여부)
ArgoCD Application CRD 스키마 검증
CI 스크립트의 이미지 태그 업데이트 로직 검증
디렉토리 구조 표준 준수 여부 검증
Property-Based Tests
테스트 프레임워크: fast-check (TypeScript/JavaScript 기반 property-based testing 라이브러리)
각 property-based test는 최소 100회 반복 실행
각 테스트는 설계 문서의 correctness property를 명시적으로 참조
태그 형식: **Feature: eks-argocd-kustomize-deployment, Property {number}: {property_text}**
테스트 범위
YAML 유효성	모든 매니페스트 파일	js-yaml, kubeval
Kustomize 빌드	각 환경별 overlay	kustomize CLI
CRD 스키마	ArgoCD Application 정의	JSON Schema 검증
이미지 태그 업데이트	CI 스크립트	fast-check (PBT)
디렉토리 구조	저장소 전체	Node.js fs 모듈
CI/CD 파이프라인 설계
GitLab CI 파이프라인 흐름
graph LR
   subgraph "App Source Repo CI"
       A[코드 푸시] --> B[빌드 & 테스트]
       B --> C[Docker 이미지 빌드]
       C --> D[ECR 푸시]
       D --> E[매니페스트 저장소<br/>이미지 태그 업데이트]
   end
 
   subgraph "Deploy Manifest Repo"
       E --> F[Git 커밋 & 푸시]
       F --> G[ArgoCD 감지]
   end
 
   subgraph "ArgoCD Sync"
       G --> H{환경?}
       H -->|dev| I[자동 동기화]
       H -->|staging| J[자동 동기화]
       H -->|production| K[수동 승인 후 동기화]
   end
GitLab CI 예시 (.gitlab-ci.yml)
stages:
  - test
  - build
  - update-manifest
 
variables:
  ECR_REGISTRY: "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com"
  APP_NAME: "my-app"
  DEPLOY_REPO: "https://gitlab.example.com/devops/deploy-manifests.git"
 
test:
  stage: test
  image: node:24
  script:
    - npm ci
    - npm test
 
build-and-push:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    IMAGE_TAG: "${CI_COMMIT_SHORT_SHA}"
  before_script:
    - aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin $ECR_REGISTRY
  script:
    - docker build -t $ECR_REGISTRY/$APP_NAME:$IMAGE_TAG .
    - docker push $ECR_REGISTRY/$APP_NAME:$IMAGE_TAG
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_BRANCH == "main"'
 
update-manifest-dev:
  stage: update-manifest
  image: alpine/git
  variables:
    IMAGE_TAG: "${CI_COMMIT_SHORT_SHA}"
  script:
    - git clone $DEPLOY_REPO /tmp/deploy
    - cd /tmp/deploy
    - |
      cat overlays/dev/kustomization.yaml | \
        sed "s/newTag: .*/newTag: $IMAGE_TAG/" > /tmp/kustomization.yaml
      mv /tmp/kustomization.yaml overlays/dev/kustomization.yaml
    - git add .
    - git commit -m "chore: update dev image tag to $IMAGE_TAG"
    - git push
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
ArgoCD 설치 및 구성 가이드
1. ArgoCD 설치
# ArgoCD 네임스페이스 생성
kubectl create namespace argocd
 
# ArgoCD 설치 (Stable 버전)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
 
# ArgoCD CLI 설치 (macOS)
brew install argocd
 
# 초기 admin 비밀번호 확인
argocd admin initial-password -n argocd
2. ArgoCD 접근 설정
# LoadBalancer로 외부 노출 (개발 환경)
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
 
# 또는 Ingress 설정 (운영 환경 권장)
# ingress.yaml 참조
3. ArgoCD 로그인 및 저장소 등록
# 로그인
argocd login <ARGOCD_SERVER> --username admin --password <PASSWORD>
 
# Git 저장소 등록
argocd repo add https://gitlab.example.com/devops/deploy-manifests.git \
  --username <GIT_USER> \
  --password <GIT_TOKEN>
 
# 클러스터 등록 (외부 클러스터인 경우)
argocd cluster add <CLUSTER_CONTEXT_NAME>
4. Application 생성
# Dev 환경 Application 생성
argocd app create my-app-dev \
  --repo https://gitlab.example.com/devops/deploy-manifests.git \
  --path overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace my-app-dev \
  --sync-policy automated \
  --auto-prune \
  --self-heal
 
# Production 환경 Application 생성 (수동 동기화)
argocd app create my-app-production \
  --repo https://gitlab.example.com/devops/deploy-manifests.git \
  --path overlays/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace my-app-production
구축 단계별 로드맵
1	EKS 클러스터 프로비저닝 (VPC, 서브넷, 노드 그룹)	1-2일
2	ArgoCD 설치 및 초기 구성	0.5일
3	배포 매니페스트 저장소 구성 (base/overlay)	1일
4	ArgoCD Application CRD 작성 (dev/staging/prod)	0.5일
5	CI 파이프라인 구성 (이미지 빌드, 태그 업데이트)	1일
6	Dev 환경 E2E 테스트	0.5일
7	Staging/Production 환경 구성 및 검증	1일
8	모니터링 및 알림 설정	0.5일
