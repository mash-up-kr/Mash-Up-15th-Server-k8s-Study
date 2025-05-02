![](https://velog.velcdn.com/images/ayeon0/post/919a033b-d1cb-4b22-b225-bad9ccd06dff/image.png)

# 개요
파드가 교체되더라도 데이터가 손실되지 않도록, 클러스터 전체에서 접근할 수 있는 저장소, 어떤 노드에 있는 파드라도 동일하게 접근할 수 있는 데이터 저장소는 반드시 필요하다.

이번 5장은 k8s가 스토리지를 어떤 방식으로 추상화했는지 알아보자.

# 컨테이너 파일 시스템의 구축 과정
파드 속 컨테이너의 파일 시스템은 여러 가지의 출처를 합쳐 구성된다.
1. 컨테이너 이미지가 초기 내용 제공
2. 기록 가능한 레이어
이미지에 들어 있던 파일을 수정하거나, 새로운 파일을 기록하는 작업이 기록 가능한 레이어에서 일어난다.

컨테이너에서 동작하는 애플리케이션에는 이런 레이어 구조를 알 수 없고, 읽기 쓰기가 가능한 하나의 파일 시스템으로 보이기 때문에 k8s로 이주시키기 매우 편리하다.

> 파드 속 컨테이너의 생애 주기는 해당 컨테이너 생애 주기를 따른다.
파드의 재시작은 그 안에 들어 있는 컨테이너의 재시작이다.

## 볼륨
4장에서 **볼륨**을 구성하고 파일 시스템의 지정한 경로로 마운트했다.
가장 단순한 유형인 **공디렉토리**에 대해 알아보자.
공디렉토리는 컨테이너 안에서 빈 디렉토리로 초기화 되고, 파드 수준의 저장소라 파드와 같은 생애주기를 가진다.
```yaml
    spec:
      containers:
        - name: sleep
          image: kiamol/ch03-sleep
          volumeMounts:
            - name: data # 이름이 data인 볼륨을 마운트
              mountPath: /data # 볼륨은 /data 경로에 마운트
      volumes:
        - name: data # 볼륨 data 정의
          emptyDir: {} # 유형은 EmptyDir
```
파드를 죽이고 재생성시키더라도 이전 컨테이너에서 생성했던 파일은 그대로 남아있는 모습을 볼 수 있다.
이런 특성을 활용해 공디렉토리를 캐싱의 목적으로도 사용할 수 있다.
## 볼륨과 마운트로 노드에 데이터 저장하기
공디렉토리 볼륨보다 더 오래 유지하기 위해 캐시를 노드에 저장하는 방법이 있다.
파드에서 해당 노드의 디렉토리에 접근하ㅣㄱ 위해 노드의 디스크를 가르키는 볼륨인 **호스트경로**볼륨을 사용할 수 있다.
컨테이너가 마운트 경로 디렉터리에 데이터를 기록하면, 실제 데이터는 노드의 디스크에 기록된다.
### 문제점
이 경우 노드가 2개 이상인 클러스터에서는 문제가 생긴다. 그리고 보안 취약점이 있다. 애플리케이션이 침투당하면 노드 디스크 전체가 장악당할 수 있다.
```yaml
spec:
containers:
- name: sleep
image: kiamol/ch03-sleep
volumeMounts:
- name: node-root
mountPath: /node-root
volumes:
- name: node-root
hostPath:
path: / # 노드 파일 시스템의 루트 디렉터리
type: Directory  # 경로에 디렉터리가 존재해야한다
```
hostpath를 루트 디렉토리로 한 경우 누구든지 이 파드가 동작 중인 노드의 파일 시스템에 접근할 수 있다.
안전하게 정의된 호스트경로 볼륨의 예시를 보자
```yaml
    spec:
      containers:
        - name: sleep
          image: kiamol/ch03-sleep
          volumeMounts:
            - name: node-root	# 마운트할 볼륨의 이름
              mountPath: /pod-logs # 마운트할 경로
              subPath: var/log/pods # 마운트할 볼륨의 경로
            - name: node-root
              mountPath: /container-logs
              subPath: var/log/containers
      volumes:
        - name: node-root
          hostPath:
            path: /
            type: Directory
```
볼륨은 여전히 노드의 루트지만, 컨테이너에서 볼륨에 접근하는 유일한 경로인 볼륨 마운트가 하위 디렉터리를 대상으로 하고 있다.

# 영구볼륨과 클레임
스토리지는 애플리케이션에 제공할 수 있는 또 다른 유형의 리소스다.
스토리지 계층의 추상으로는 **영구볼륨(PersistentVolume)**과 **영구볼륨클레임(PersistentVolumeClaim)**이 있다.
## 영구볼륨

> 사용 가능한 스토리지의 조각을 정의한 k8s 리소스

영구볼륨에는 이를 구현하는 스토리지 시스템에 대한 볼륨 정의가 포함되어 있다.
```yaml
apiVersion: v1
kind: PersistentVolume 
metadata:
  name: pv01 # 볼륨의 이름
spec:
  capacity:
    storage: 50Mi # 볼륨 용량
  accessModes: # 파드 접근 유형
    - ReadWriteOnce # 파드 하나에서만 읽기/쓰기
  nfs: # NFS 스토리지를 사용
    server: nfs.my.network # NFS 서버의 도메인 네임
    path: "/kubernetes-volumes" # 스토리지 경로
```
해당 정의는 NFS 스토리지를 사용하는 정의다. 실습 예제에서는 로컬 볼륨을 사용한다.
클러스터에 영구 볼륨을 생성해도, 파드는 영구볼륨클레임을 사용해서 볼륨 사용을 요청해야한다.

## 영구볼륨클레임
> 파드가 사용하는 스토리지의 추상

```yaml
apiVersion: v1 
kind: PersistentVolumeClaim 
metadata:
  name: postgres-pvc
spec:
  accessModes: # 접근 유형은 필수 설정
    - ReadWriteOnce
  resources:
    requests:
      storage: 40Mi # 요청하는 스토리지 용량
  storageClassName: "" # 스토리지 유형을 지정하지 않음
```
정의할 때는 접근 유형과 스토리지 용량, 스토리지 유형을 지정한다.
스토리지 유형을 지정하지 않으면 현존하는 영구 볼륨 중 요구사항과 일치하는 것을 찾아준다.
영구볼륨과 영구볼륨클레임은 1대1 매핑되어있다.
## 파드에서 사용하기
파드가 PVC를 사용하려면 PVC가 PV와 연결되어 있어야 한다.
연결되지 않으면 연결될 때까지 Pending이 되어 실행되지 않는다.
```yaml
    spec:
      containers:
        - name: db
          image: postgres:11.6-alpine
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim: # PVC를 볼륨으로 사용
            claimName: postgres-pvc # 사용할 PVC 이름
```
PostgreSQL DB 파드 정의의 일부다. PVC를 볼륨으로 사용한 걸 볼 수 있다.
# 스토리지 유형과 동적 볼륨 프로비저닝
지금까지의 방식은 명시적으로 PV와 PVC를 생성해서 연결하는 정적인 방식이지만, 대부분의 k8s 플랫폼에서는 **동적 볼륨 프로비저닝**이라는 간단한 방식을 더 자주 사용한다.
> PVC만 생성하면 그에 맞는 PV를 클러스터에서 동적으로 생성해주는 방식

클러스터에는 다양한 요구 사항에 대응할 수 있는 여러 스토리지 유형을 설정할 수 있는데, 기본 유형을 다시 지정할 수 있다.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
      # storageClassName 필드가 없으면 기본으로 유형이 쓰인다.
```
## 스토리지 유형
스토리지 유형은 표준 k8s 리소스로 생성되는데, 유형의 정의에 3가지 필드로 어떻게 스토리지가 동작할지 지정할 수 있다.
* provisioner: PV가 필요해질 때 만드는 주체, 플랫폼에 따라 주체가 달라진다.
* reclaimPolicy: 연결되있던 pVC가 삭제되었을 때 남은 PV를 어떻게 처리할지 지정한다.
* volumeBindingMode: PVC가 생성되자마자 바로 PV를 생성해서 연결할지, 해당 클레임을 사용하는 파드가 생성될 때 영구볼륨을 생성할지 선택할 수 있다.
