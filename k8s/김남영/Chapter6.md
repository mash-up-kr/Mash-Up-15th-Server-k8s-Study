# 6장: 스케일링과 파드 컨트롤러

## 개요
- Kubernetes에서 스케일링은 동일한 애플리케이션을 실행하는 파드를 늘리고 줄이는 작업
- 각 파드를 레플리카(replica)라고 하며, 여러 노드에 분산 배치되어 높은 가용성 확보 가능

## 스케일링 방법

### 1. ReplicaSet
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: whoami-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: whoami-web
  template:
    metadata:
      labels:
        app: whoami-web
    spec:
      containers:
        - name: web
          image: your-image
```

ReplicaSet은 컨트롤 루프를 돌며 현재 파드 수와 원하는 수를 맞춤

파드가 삭제되면 자동으로 대체 파드를 생성

### 2. Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pi-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pi-web
  template:
    metadata:
      labels:
        app: pi-web
    spec:
      containers:
        - name: web
          image: your-image
```

Deployment는 ReplicaSet을 관리하며, 롤링 업데이트·롤백 기능 제공

replicas 필드 변경 시 기존 ReplicaSet의 replica 수만 조정

파드 템플릿이 업데이트되면 새 ReplicaSet 생성 후 구 버전의 replica 수를 0으로 설정

### 3. 명령형 스케일링

```kubectl scale deployment/pi-web --replicas=5```  
즉각적인 대응이 필요할 때 유용

이후 선언적 YAML에도 동일한 설정 반영 필요

ReplicaSet 재사용 메커니즘
파드 이름 뒤에 붙는 해시 값은 템플릿 정의의 해시

이전과 동일한 템플릿이면 기존 ReplicaSet 이름과 해시가 일치 → 레플리카 수만 증가시켜 재사용

볼륨 공유 문제
스케일링된 파드들 간 캐시용 볼륨 공유 시, 개별 emptyDir는 파드마다 분리됨

해결방법:
- 호스트경로 볼륨(hostPath) 사용
- StatefulSet/DaemonSet 이용해 공유 구조 고려

#### 데몬셋(DaemonSet)을 이용한 스케일링
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: pi-proxy
spec:
  selector:
    matchLabels:
      app: pi-proxy
  template:
    metadata:
      labels:
        app: pi-proxy
    spec:
      containers:
        - name: proxy
          image: nginx
      nodeSelector:
        role: worker
```
클러스터 내 모든 노드(또는 특정 레이블 매칭 노드)에 파드 1개씩 배치

로그 수집, 모니터링 에이전트, 리버스 프록시 등에 활용

객체 오너십과 가비지 컬렉션
컨트롤러 리소스(Deployment, ReplicaSet, DaemonSet)는 레이블 셀렉터로 관리 대상 결정

관리 대상 리소스는 metadata.ownerReferences에 소유자 정보 기록

컨트롤러 삭제 시 가비지 컬렉터가 종속 리소스도 자동 삭제

### 연습 문제
ReplicationController로 정의된 웹 컴포넌트를 Deployment로 전환

단일 인스턴스만 필요한 API 컴포넌트를 DaemonSet으로 정의

```yaml
# 예시: Deployment 변환
apiVersion: apps/v1
kind: Deployment
metadata:
  name: numbers-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: numbers-web
  template:
    metadata:
      labels:
        app: numbers-web
    spec:
      containers:
        - name: web
          image: kiamol/ch03-numbers-web

# 예시: DaemonSet 선언
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: numbers-api
spec:
  selector:
    matchLabels:
      app: numbers-api
  template:
    metadata:
      labels:
        app: numbers-api
    spec:
      containers:
        - name: api
          image: kiamol/ch03-numbers-api
      nodeSelector:
        rng: hw
```