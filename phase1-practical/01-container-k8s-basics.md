# 1단계: 컨테이너 & Kubernetes 핵심 개념

> **상태**: ✅ 완료
> **학습 일자**: 2025-07-15
> **참고**: STUDY_PLAN.md 1단계

---

## 핵심 개념 정리

### 왜 K8s인가 (docker-compose가 아니라)

docker-compose는 단일 서버용 도구. K8s는 분산 환경 오케스트레이션:
- 여러 노드를 하나의 클러스터로 추상화
- 선언한 상태를 지속적으로 유지 (Reconciliation)
- 무중단 배포 (Rolling Update), 자동 확장 (HPA) 내장

**핵심 철학**: 원하는 상태(Desired State)를 선언하면, K8s가 현재 상태를 거기에 맞춘다.

### K8s 아키텍처 (EKS 기준)

- **Control Plane**: AWS가 관리 (API Server, etcd, Scheduler, Controller Manager)
- **Worker Node**: 실제 앱이 돌아가는 EC2 (kubelet, kube-proxy, Pod들)
- **kubectl**: API Server에 명령을 보내는 CLI

### 6가지 핵심 리소스

| 리소스 | 역할 | 한 줄 요약 |
|--------|------|-----------|
| Pod | 컨테이너 실행 단위 | K8s 배포 최소 단위. 죽으면 IP 바뀜 |
| Deployment | Pod 관리자 | Pod를 몇 개 유지하고, 어떻게 업데이트할지 정의 |
| Service | 클러스터 내부 고정 주소 | label selector로 Pod를 찾아 트래픽 분배 |
| Ingress | 외부 트래픽 진입점 | 도메인/경로 기반으로 어느 Service로 보낼지 L7 라우팅 |
| Namespace | 논리적 격리 공간 | 리소스 격리, RBAC, 리소스 쿼터, Network Policy 단위 |
| Secret | 민감 정보 저장 | Private 레지스트리 인증 등 |

### Label Selector — K8s의 핵심 연결 메커니즘

리소스끼리 연결되는 방식은 이름이 아니라 **라벨(label)**:
- Deployment: `selector.matchLabels`로 관리할 Pod를 찾음
- Service: `selector`로 트래픽 보낼 Pod를 찾음
- Pod의 라벨은 `Deployment.spec.template.metadata.labels`에서 정의됨
- Pod가 늘어나도 같은 라벨이면 Service가 자동 인식 → 트래픽 자동 분배

### Service vs Ingress 차이

| | Ingress | Service |
|---|---|---|
| 레이어 | L7 (HTTP) | L4 (TCP) |
| 하는 일 | 도메인/경로 보고 **어느 Service로** 보낼지 | 받은 트래픽을 **어느 Pod로** 보낼지 |
| 비유 | 건물 안내 데스크 | 층별 접수 창구 |

### Namespace 특성

- 같은 namespace: 서비스 이름만으로 통신 가능 (`curl http://backend:80`)
- 다른 namespace: FQDN 필요 (`curl http://backend.swagger-dev.svc.cluster.local:80`)
- namespace별 RBAC, 리소스 쿼터, Network Policy 설정 가능

### HPA vs resources

| | resources | HPA |
|---|---|---|
| 대상 | Pod 1개의 자원 울타리 | Pod 개수 자동 조절 |
| 없으면 | Pod가 노드 자원 독점 가능 | 수동으로만 replicas 변경 |
| 관계 | HPA가 동작하려면 resources.requests 필수 | |

---

## 현재 프로젝트 연결

### bc-backoffice/backend (API 서버)
- Deployment + Service + TGB 구성
- Service 필요: 외부에서 HTTP 요청을 받아야 하므로
- resources, probe 설정됨. HPA는 미설정 (replicas: 2 고정)

### monitoring 앱들 (MSK Consumer)
- Deployment만 존재. Service 없음
- MSK에서 메시지를 "가져오는" 쪽이라 아무도 이 Pod에 접속할 일 없음
- 수평확장 시 Kafka Consumer Group이 파티션을 자동 리밸런싱

### MSK + EKS 수평확장 흐름
```
K8s: "언제, 몇 개" Pod를 만들지 결정 (HPA)
  ↓
새 Pod 생성 → MSK에 같은 group.id로 연결
  ↓
Kafka: "누가 어떤 파티션" 담당할지 리밸런싱
```
- 제약: Pod 수 > 파티션 수이면 남는 Pod는 유휴 상태

---

## 헷갈렸던 점 & 해결

1. **Pod의 라벨은 어디서 정의?**
   → `Deployment.spec.template.metadata.labels`에서 정의. template이 Pod 설계도.

2. **Service ↔ Ingress 역할 혼동**
   → Ingress는 "어느 Service로", Service는 "어느 Pod로". 레이어가 다름.

3. **MSK Consumer 수평확장 시 Service 없이 트래픽 분배가 되나?**
   → Service는 "들어오는 트래픽 분배"용. Consumer는 "나가서 가져오는 것"이라 Kafka가 분배 담당. 방향이 반대.

---

## 복습 포인트

1. K8s = 선언적 상태 관리. "이런 상태여야 한다" 선언하면 K8s가 맞춤
2. Label selector가 K8s의 핵심 연결 메커니즘 (Deployment↔Pod, Service↔Pod)
3. Service는 클러스터 내부 고정 주소 + 트래픽 분배. 외부 노출은 Ingress
4. Service가 필요한지는 "누가 이 Pod에 접근해야 하는가?"로 판단
5. MSK Consumer 수평확장: K8s가 Pod 수 결정, Kafka가 파티션 분배
