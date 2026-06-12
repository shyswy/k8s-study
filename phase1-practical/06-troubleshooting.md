# 6단계: 일상 운영 & 트러블슈팅

## 학습 일자
2025-07-15

## 핵심 개념 정리

### Pod 상태 해독

| 상태 | 의미 | 심각도 |
|------|------|--------|
| `Running` | 정상 동작 중 | ✅ |
| `Pending` | 노드에 배치되지 못함 | ⚠️ 리소스 부족 |
| `ImagePullBackOff` | 이미지를 못 가져옴 | 🔴 배포 실패 |
| `CrashLoopBackOff` | 시작 후 반복 종료 | 🔴 앱 에러 |
| `OOMKilled` | 메모리 초과로 강제 종료 | 🔴 리소스 설정 문제 |
| `Evicted` | 노드 메모리 압박으로 축출됨 | 🔴 노드 레벨 문제 |

### 트러블슈팅 순서 (항상 이 순서로)

```
Step 1: 상태 확인     → kubectl get pods -n swagger-dev
Step 2: 이벤트 확인   → kubectl describe pod <pod-name> -n swagger-dev  ← Events 섹션이 핵심
Step 3: 로그 확인     → kubectl logs <pod-name> -n swagger-dev
Step 4: 연결 확인     → kubectl get endpoints -n swagger-dev
```

### 에러별 원인 정리

**ImagePullBackOff — 원인 3가지:**
1. 이미지 태그가 Harbor에 실제로 존재하지 않음 (CI push 실패)
2. `registry-secret` Secret이 해당 namespace에 없음
3. Secret의 자격증명(비밀번호)이 만료됨

**CrashLoopBackOff — 흔한 원인:**
- 앱 자체 런타임 에러 (DB 연결 실패, 환경변수 누락 등)
- 포트 충돌
- 잘못된 entrypoint/command
- `kubectl logs --previous`로 이전 크래시 로그 확인 필수

**Pending — 흔한 원인:**
- 노드의 CPU/메모리 자원 부족
- nodeSelector/affinity 조건에 맞는 노드 없음
- PersistentVolume 바인딩 대기

**OOMKilled vs Eviction:**

| 상황 | 누가 kill | 결과 |
|------|----------|------|
| Pod가 자기 `limits.memory` 초과 | 커널 (cgroup OOM killer) | OOMKilled, Pod 재시작 |
| 노드 전체 메모리 부족 | kubelet eviction | Pod Evicted, 다른 노드에 재스케줄링 |

Eviction 축출 순서: `BestEffort` (requests/limits 없음) → `Burstable` → `Guaranteed` (requests=limits)

### Eviction 시 요청 처리

| 조건 | 요청 영향 |
|------|----------|
| replicas: 1, probe 없음 | 요청 유실 (502/503) |
| replicas: 1, readinessProbe 있음 | 여전히 유실 (Pod가 1개뿐) |
| replicas: 2+, readinessProbe 있음 | **영향 없음** — 나머지 Pod가 처리 |

→ 프로덕션: `replicas >= 2` + `readinessProbe` + `PodDisruptionBudget` 세트로 설정

### 부하 과다 시 대응 전략

**EKS 자체로 해결 가능:**
- HPA — CPU/메모리 기준 Pod 수 자동 증가
- Cluster Autoscaler / Karpenter — Pending Pod 발생 시 노드 자동 추가

**메시지 큐가 필요한 경우:**
- 순간 스파이크 (Pod/노드 추가에 수십 초~수 분 소요, 그 사이 유실)
- 처리 시간이 긴 작업 (동기 요청 시 타임아웃)
- 다운스트림 장애 (DB, 외부 API 병목)
- 요청 순서 보장 필요

현재 프로젝트(swagger-hub)는 읽기 위주 서빙이라 HPA + replicas 증가로 충분.

---

## 현재 프로젝트 연결

### 발생 가능한 시나리오

| 시나리오 | 증상 | 원인 | 해결 |
|---------|------|------|------|
| CI가 이미지 push 실패 | ImagePullBackOff | Harbor에 태그 없음 | CI 파이프라인 재실행 |
| registry-secret 만료 | ImagePullBackOff | Secret 자격증명 만료 | Secret 재생성 |
| Node.js 앱 버그 | CrashLoopBackOff | 앱 에러 | 소스 레포에서 수정 후 재배포 |
| 노드 리소스 부족 | Pending | CPU/메모리 부족 | 노드 스케일 아웃 또는 requests 조정 |
| 메모리 누수 | OOMKilled | limits 초과 | limits 증가 또는 앱 수정 |

### 현재 프로젝트의 취약점

- `replicas: 1` → Eviction/Rolling Update 시 다운타임 발생
- `resources` 미설정 → BestEffort QoS → Eviction 시 가장 먼저 축출됨
- `readinessProbe` 미설정 → Pod 종료 시 트래픽이 죽는 Pod로 갈 수 있음

---

## 핵심 명령어 / 코드

```bash
# 상태 확인
kubectl get pods -n swagger-dev
kubectl get pods -n swagger-dev -o wide   # 노드 배치 확인

# 이벤트 확인 (가장 중요)
kubectl describe pod <pod-name> -n swagger-dev

# 로그 확인
kubectl logs <pod-name> -n swagger-dev
kubectl logs --previous <pod-name> -n swagger-dev   # 이전 크래시 로그
kubectl logs -f <pod-name> -n swagger-dev            # 실시간 follow

# Service → Pod 연결 확인
kubectl get endpoints swagger-hub -n swagger-dev

# Pod 재시작
kubectl rollout restart deployment/swagger-hub -n swagger-dev   # 전체 순차 재시작
kubectl delete pod <pod-name> -n swagger-dev                     # 특정 Pod만 삭제

# 실시간 이벤트 모니터링
kubectl get events -n swagger-dev --watch

# Secret 확인
kubectl get secrets -n swagger-dev | grep harbor
kubectl get secret registry-secret -n swagger-dev -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

# 노드 상태 확인 (Eviction 의심 시)
kubectl describe node <node-name>   # Conditions: MemoryPressure 확인

# Evicted Pod 확인
kubectl get pods -n swagger-dev | grep Evicted
```

---

## 헷갈렸던 점 & 해결

- **Q**: Endpoints가 비어있으면 Ingress 문제인가?
- **A**: 아니다. Endpoints 비어있음 = **Service → Pod 연결 끊김**. Ingress와 무관. 원인: Pod가 없거나(CrashLoopBackOff, Pending), Pod label이 Service selector와 불일치, 또는 readinessProbe 실패로 endpoints에서 제외됨.

- **Q**: OOMKilled와 Eviction의 차이?
- **A**: OOMKilled = Pod가 자기 limits.memory를 초과 → 커널이 해당 컨테이너만 kill. Eviction = 노드 전체 메모리 부족 → kubelet이 우선순위 낮은 Pod부터 축출. 둘 다 메모리 문제지만 레벨이 다름 (컨테이너 vs 노드).

- **Q**: rollout restart vs delete pod?
- **A**: `rollout restart` = Deployment의 모든 Pod를 순차적으로 재생성 (Rolling Update 방식). `delete pod` = 특정 Pod 1개만 삭제 → Deployment가 새 Pod 1개 자동 생성.

---

## 학습 체크 결과 (4/7 완전 정답, 3/7 부분 정답)

| 질문 | 결과 | 보완 |
|------|------|------|
| ImagePullBackOff 원인 3가지 | 🔶 | 태그 없음, Secret 없음, 자격증명 만료 |
| --previous 옵션 | ✅ | |
| describe의 Events 섹션 | ✅ | |
| rollout restart vs delete pod | ✅ | |
| endpoints 비어있음의 의미 | 🔶 | Service→Pod 연결 끊김 (Ingress 무관) |
| Pending 해결 방법 2가지 | 🔶 | 노드 추가 또는 requests 값 낮춤 |
| OOMKilled 수정 필드 | ✅ | resources.limits.memory |

---

## 복습 포인트

1. **트러블슈팅 순서**: get pods → describe pod (Events 핵심) → logs (--previous) → get endpoints. 이 순서를 몸에 익혀야 함.
2. **ImagePullBackOff 원인 3가지**: ①태그가 Harbor에 없음 ②registry-secret Secret 없음 ③자격증명 만료. describe의 Events에 `unauthorized` 또는 `not found` 메시지로 구분 가능.
3. **OOMKilled vs Eviction**: OOMKilled = Pod가 자기 limits 초과 → 커널이 컨테이너 kill. Eviction = 노드 메모리 부족 → kubelet이 QoS 낮은 Pod부터 축출. resources 미설정 시 BestEffort로 가장 먼저 축출됨.
4. **Endpoints 비어있음 = Service→Pod 연결 끊김**. Ingress 문제가 아님. Pod가 없거나, label 불일치거나, readinessProbe 실패가 원인.
5. **replicas:1의 위험**: Eviction/Rolling Update/OOMKilled 어떤 상황이든 순간 다운타임 발생. 프로덕션은 반드시 replicas >= 2 + readinessProbe + PDB 세트.
6. **부하 과다 대응**: EKS 내에서는 HPA + Cluster Autoscaler로 스케일링. 순간 스파이크나 무거운 작업은 메시지 큐(SQS, Kafka)로 비동기 처리 필요. 현재 프로젝트는 읽기 위주라 HPA로 충분.
7. **rollout restart vs delete pod**: restart는 전체 Pod 순차 재생성(Rolling), delete pod는 특정 1개만 삭제 후 자동 재생성.
