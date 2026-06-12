# Kubernetes / EKS Study

AWS EKS 기반 GitOps 배포 시스템을 담당 개발자로서 운영하고, Kubernetes 전반의 전문성을 쌓기 위한 학습 레포지토리.

## 목적

- Kubernetes / AWS EKS 인프라 이해 및 운영 능력 확보
- GitOps(ArgoCD + Kustomize) 배포 흐름 완전 이해
- 장애 대응, 네트워킹, 보안, 프로덕션 하드닝까지 커버

## 학습 구조

```
Phase 1 (실용 레벨)
 └── 컨테이너 기초 → YAML 해독 → kubectl → Kustomize → GitOps/ArgoCD → 트러블슈팅

Phase 2 (전문가 레벨)
 └── K8s 내부 동작 → EKS 심화 → 네트워킹 → 보안/RBAC → 프로덕션 → Observability → 고급 GitOps
```

## 디렉토리 구조

```
├── STUDY_PLAN.md               # 학습 로드맵 & 진도 관리
├── EKS_DEPLOY_STRATEGY.md      # EKS 배포 전략 정리
├── context/                    # 학습 컨텍스트 자료
├── phase1-practical/           # Phase 1: 실용 레벨 (6 챕터)
├── phase2-expert/              # Phase 2: 전문가 레벨 (7 챕터)
└── projects/                   # 실무 프로젝트 매니페스트
    ├── bc-backoffice/          # B2B 백오피스 배포
    └── monitoring/             # 모니터링 시스템
```

## 현재 진도

- **Phase 1**: 전체 완료 (6단계 트러블슈팅까지)
- **Phase 2**: 미시작

## 기술 스택

- AWS EKS (ap-northeast-2)
- ArgoCD (GitOps)
- Kustomize (매니페스트 관리)
- Nginx Ingress Controller
- Harbor (Private Registry)

## 학습 방법

`STUDY_PLAN.md`를 AI 학습 세션에서 로드하면, 진도를 기반으로 이어서 학습할 수 있도록 설계됨.
