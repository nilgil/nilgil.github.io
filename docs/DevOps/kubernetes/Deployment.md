---
title: Deployment
parent: Kubernetes
last_modified_at: 2024-11-12
---

# Deployment
Pod 배포 자동화를 위한 오브젝트이다. (ReplicaSet + 배포전략)  
새로운 Pod을 롤아웃/롤백할 때 (Pod Template Image 변경시) ReplicaSet 생성을 대신해준다.  
다양한 배포 전략을 제공하고 이전 파드에서 새로운 파드로의 전환 속도 제어가 가능하다.

---
### Pod 배포를 위한 3가지 정보
1. selector : 어떤 Pod 집합을 대상으로 Replication 해야 하는지
2. replicas : 몇 개의 Pod을 생성할지
3. Pod Template Image : 어떤 컨테이너를 실행할지

---
### 배포 전략
- Recreate
  - 새로운 버전을 배포하기 전에 이전 버전이 즉시 종료됨
    - 컨테이너가 정상적으로 시작되기 전까지 서비스 불가 (실제 운영에 부적합)
  - replicas 수만큼의 컴퓨팅 리소스 필요
  - 개발단계에서 유용
- RollingUpdate
  - 새로운 버전을 배포하면서 이전 버전을 종료
    - 새로운 Pod 생성과 이전 Pod 종료가 동시에 일어나는 방식
  - 서비스 다운 타임 최소화
    - Pod이 존재하지 않는 시점이 없다.

---
### 배포 속도 제어 옵션
- maxUnavailable : 롤링 업데이트를 수행하는 동안 Scale Down 할 수 있는 최대 비율 지정
  - 즉 유지하고자 하는 최소 Pod의 비율을 지정하는 것과도 같다.
  - 배포시 해당 비율만큼 Pod을 제거하고 시작할 수 있다.
    - 100 - maxUnavailable = 최소 Pod 유지 비율
    - Ex) Pod이 100개인 경우 maxUnavailable이 30%인 경우 최소 70개의 Pod은 유지되어야 한다.
- maxSurge : 롤링 업데이트를 수행하는 동안 Scale Up 할 수 있는 최대 비율 지정
  - 즉 총 Pod의 개수가 100 + maxSurge %를 넘지 않도록 유지한다.
  - 배포시 해당 비율만큼 Pod을 미리 생성할 수 있다.

---
### Manifest
```yaml
apiVersion: apps/         # Kubernetes API 버전
kind: Deployment          # 오브젝트 타입
metadata:                 # 오브젝트를 유일하게 식별하기 위한 정보
  name: my-app            # 오브젝트 이름
spec:                     # 사용자가 원하는 Pod의 바람직한 상태
  selector:               # ReplicaSet을 통해 관리할 Pod을 선택하기 위한 label query
    matchLabels:
      app: my-app
  replicas: 3             # 실행하고자 하는 Pod 복제본 개수 선언
  template:               # Pod 실행 정보 - Pod Template과 동일 (metadata, spec, ...)
    metadata:
      labels:
        app: my-app       # selector에 정의한 label을 포함해야 한다.
    spec:
      containers:
      - name: my-app
        image: my-app:1.0
```

---
### Rollback
Deployment는 롤아웃 히스토리를 Revision #으로 관리한다.  
이 Revision에는 그 당시 배포된 Pod Template에 대한 상세 정보가 기록되어있다.

```bash
kubectl rollout undo deployment <deployment-name> --to-revision=#
```
#에 롤백하려는 Revision 번호를 적어주면 된다.
