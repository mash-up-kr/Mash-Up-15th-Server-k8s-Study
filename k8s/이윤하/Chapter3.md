# 네트워크를 통해 서비스에 파드 연결하기

## 함께 나눌 질문
1. 3장 본문에서는 LoadBalancer와 NodePort를 외부 트래픽을 파드에 연결하는 방법으로 소개하고 있다.
   듣고 있는 인강에서도 NodePort로 실습을 진행하고 있다보니, 실제 프로덕션 환경에서 외부 접근을 관리할 때 Ingress를 더 일반적으로 사용하는 이유는 무엇이며, LoadBalancer, NodePort, Ingress 각각 차이가 무엇이고,
   어떤 상황에서 각 방식을 선택하는 것이 가장 적절할 지 궁금합니다.
2. 서비스와 파드는 label selector를 통해 느슨하게 연결되는 아키텍쳐를 갖고 있는데, 이런 느슨한 연결이 가지는 장단점이 궁금합니다.

### ✅ 서비스(Service)란?

서비스(Service) : 외부로부터 요청을 받는 역할 / 외부로부터 들어오는 트래픽을 받아, 파드에 균등하게 분배해주는 로드밸런서 역할을 하는 기능

실제 서비스에서 파드(Pod)에 요청을 보낼 때, 포트 포워딩(`port-forward`)이나 파드 내로 직접 접근(`kubectl exec …`)해서 요청을 보내진 않는다. 서비스(Service)를 통해 요청을 보내는 게 일반적이다.

k8s는 파드에 IP를 부여하는데, 파드가 사라지고 생기면서 IP가 계속 바뀌는 문제가 있다. k8s는 **서비스**에 **Address Discovery**를 통해 이 문제를 해결했다.

## 서비스란?
파드에는 2가지 중요 성질이 있다.
1. 파드는 k8s에서 부여한 IP를 가진 가상 환경임
2. 파드는 컨트롤러 객체(2장에서 본 디플로이먼트 등)에 의해 생애주기가 관리되면서 쓰고 버리는 리소스임.

파드가 생기고 없어지면서 IP가 바뀔 때, 해당 IP 주소를 찾는 것은 쉽지 않다. 새 IP 주소는 k8s API를 통해 파악할 수 있다.

### 파드의 IP를 확인해보기
```bash
% kubectl apply -f sleep/sleep1.yaml -f sleep/sleep2.yaml
deployment.apps/sleep-1 created
deployment.apps/sleep-2 created


% kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
sleep-1-5b88cbd4bf-vbkw7   1/1     Running   0          2m20s
sleep-2-7f69798f94-x548h   1/1     Running   0          2m20s


% kubectl wait --for=condition=Ready pod -l app=sleep-2
pod/sleep-2-7f69798f94-x548h condition met


# JSON Path 질의는 sleep-2 파드 IP 주소를 반환합니다.
% kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'
10.1.0.32                    


# sleep-1 파드에서 sleep-2 파드로 ping을 날려봅니다.
% kubectl exec deploy/sleep-1 -- ping -c 2 $(kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}')
PING 10.1.0.32 (10.1.0.32): 56 data bytes
64 bytes from 10.1.0.32: seq=0 ttl=64 time=0.443 ms
64 bytes from 10.1.0.32: seq=1 ttl=64 time=0.239 ms

--- 10.1.0.32 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.239/0.341/0.443 ms

# sleep-2 파드를 없애봅니다. 디플로이먼트로 만든 파드였기 때문에 새로 생깁니다.
% kubectl delete pods -l app=sleep-2
pod "sleep-2-7f69798f94-x548h" deleted


# 새로 생긴 파드의 IP 주소는 바뀌어있습니다.
% kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'
10.1.0.33
```
‘언제든지 다른 것으로 바뀔 수 있는 리소스에 접근하기 위한 고정된 주소’ 는 새로운 문제는 아니다. **인터넷에서 IP 주소에 이름을 붙이는 도메인 네임을 도입하여, 이 문제를 해결**했었다. k8s에서도 같은 해결책을 도입해서, k8s 클러스터에는 전용 DNS 서버가 존재하여, 서비스의 이름과 IP 주소를 매핑해준다.

서비스는 파드와 파드가 가진 네트워크 주소를 추상화한 것이다. 디플로이먼트처럼 서비스는 label selector 방식(밑에 매니페스트를 보면 이해된다)으로 파드와 느슨한 연결을 갖는다.
Consumer 컴포넌트가 서비스의 IP 주소로 요청을 보내면 k8s가 서비스와 연결된 파드의 실 IP 주소로 요청을 연결해준다.

### 서비스 만들어보기
다음과 같은 매니페스트 파일을 만들어 서비스를 생성할 수 있다.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sleep-2
spec:
  selector:
    app: sleep-2
  ports:
    - port: 80
```

sleep/sleep2-service.yaml
```bash
% kubectl apply -f sleep/sleep2-service.yaml
service/sleep-2 created

# 서비스를 조회합니다.
% kubectl get service sleep-2
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
sleep-2   ClusterIP   10.110.59.233   <none>        80/TCP    9s

# svc로 줄여서 쓸 수도 있다.
# 서비스에서는 클러스터 어디에서든 접근할 수 있는 cluster-ip가 있다.
% kubectl get svc sleep-2
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
sleep-2   ClusterIP   10.110.59.233   <none>        80/TCP    19s

# sleep1에서 sleep2로 ping을 날려보자. 전송이 안되는 것을 볼 수 있는데
# 왜냐하면 서비스 리소스가 ping이 사용하는 ICMP 프로토콜을 지원하지 않기 때문이다.
younhalee@younhalee-MacBookAir ch03 % kubectl exec deploy/sleep-1 -- ping -c 1 sleep-2
PING sleep-2 (10.110.59.233): 56 data bytes

--- sleep-2 ping statistics ---
1 packets transmitted, 0 packets received, 100% packet loss
command terminated with exit code 1
```
## 파드끼리 통신하기
서비스의 유형 중 가장 기본은 클러스터IP(ClusterIP)다.
클러스터IP는 클러스터 전체에서 통용되는 IP 주소를 생성하는데, 이 주소는 파드가 어떤 노드에 있느냐에 상관없이 접근이 가능하다. 단 클러스터 내부에서만 사용할 수 있다.
> 쿠버네티스 내부에서만 통신할 수 있는 IP 주소를 부여. 외부에서는 요청할 수 없다.

라고 기억해놓자.
### 프론트와 백엔드를 배포해서 통신해보기
백엔드 디플로이먼트를 생성하자.
```yaml
apiVersion: apps/v1
kind: Deployment
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
```

다음은 프론트 디플로이먼트다.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: numbers-web
spec:
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
```

이제 디플로이먼트를 생성하자.
```yaml
% kubectl apply -f numbers/api.yaml -f numbers/web.yaml
deployment.apps/numbers-api created
deployment.apps/numbers-web created

# 준비 상태가 될 때까지 기다립니다.
% kubectl wait --for=condition=Ready pod -l app=numbers-web
pod/numbers-web-76447f6964-g4vwc condition met

# 포트포워딩을 시켜줍니다.
% kubectl port-forward deploy/numbers-web 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
Handling connection for 8080
```

이렇게 했을 때 다음과 같은 오류가 발생한다.
API의 도메인 이름은 numbers-api 인데, 이 도메인 네임이 k8s 내부 DNS 서버에 등록되어있지 않기 때문이다.
![](https://velog.velcdn.com/images/ayeon0/post/63556994-d5a8-4913-aa08-c25a9f58b0d9/image.png)

서비스로 이 문제를 해결해보자
```yaml
apiVersion: v1
kind: Service
metadata:
  name: numbers-api
spec:
  ports:
    - port: 80
  selector:
    app: numbers-api
    
  # 서비스의 유형을 지정합니다. default값이 ClusterIP이므로 생략 가능합니다.
  # 단 의미를 분명히 하기 위해 명시하는 것이 좋습니다.
  type: ClusterIP 
```
numbers/api-service.yaml
```bash
% kubectl apply -f numbers/api-service.yaml
service/numbers-api created

% kubectl get svc numbers-api
NAME          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
numbers-api   ClusterIP   10.105.8.38   <none>        80/TCP    7s

% kubectl port-forward deploy/numbers-web 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```

![](https://velog.velcdn.com/images/ayeon0/post/fe1d4cba-efd4-4861-9daa-1a5d8296c386/image.png)
잘 동작함을 확인할 수 있다.
이렇듯 애플리케이션의 모든 컴포넌트를 yaml에 정의할 수 있다.

### 파드를 지워도 될까?
```bash
# API 파드의 IP를 확인해보자
% kubectl get pod -l app=numbers-api -o custom-columns=NAME:metadata.name,POD_IP:status.podIP
NAME                           POD_IP
numbers-api-545b9d9ccd-bhj7r   10.1.0.34

# 이 파드를 지웠다.
% kubectl delete pod -l app=numbers-api
pod "numbers-api-545b9d9ccd-bhj7r" deleted

# 다시 확인해보면 IP가 바뀐 것을 알 수 있다.
% kubectl get pod -l app=numbers-api -o custom-columns=NAME:metadata.name,POD_IP:status.podIP
NAME                           POD_IP
numbers-api-545b9d9ccd-k8v6z   10.1.0.36

# 포트포워딩을 해서 잘 되는지 확인해보자.
% kubectl port-forward deploy/numbers-web 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```

여기서는 파드를 의도적으로 삭제했지만, 실제 prod 환경에서는 애플리케이션을 업데이트하면서 파드가 교체된다. 이렇게 파드가 교체되더라도 서비스가 파드를 연결해주기 때문에 통신이 계속 될 수 있는 것이다.
포트포워딩 말고 파드에 사용될 서비스를 배포해보자

## 외부 트래픽과 연결하기
서비스는 클러스터 외부와 내부(파드)를 연결하는 2가지 방법이 있다.
### 방법 1. 로드밸런서(LoadBalancer)
type을 LoadBalancer로 하면 외부의 로드밸런서(AWS의 로드밸런서 등)를 활용해 외부에서 접속할 수 있도록 연결한다.

로드밸런서 서비스가 커버하는 범위는 클러스터 전체이므로, 다른 노드에 있는 파드도 트래픽을 전달받을 수 있다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: numbers-web
spec:
  ports:
    - port: 8080
      targetPort: 80
  selector:
    app: numbers-web
  type: LoadBalancer
```

numbers/web-serivce.yaml

```bash
% kubectl get deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
numbers-api   1/1     1            1           6d13h
numbers-web   1/1     1            1           6d13h

% kubectl apply -f numbers/web-service.yaml
service/numbers-web created

% kubectl get svc numbers-web
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
numbers-web   LoadBalancer   10.111.222.240   localhost     8080:31382/TCP   9s

% kubectl get svc numbers-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080'
http://localhost:8080
```

### 방법2. 노드포트(NodePort)
쿠버네티스 내부에서 해당 서비스에 접속하기 위한 포트를 열고 외부에서 접속 가능하도록 한다.

이 서비스는 서비스에서 설정한 포트가 모든 노드에서 개방되어있어야 하기 때문에 로드밸런서 서비스만큼 유연하지는 않다. 또 다중 노드 클러스터에서 로드밸런싱 효과를 얻을 수 없고, 노드포트 서비스에서 지원하는 분산 수준도 로드밸런서 서비스와 차이가 있다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: numbers-web-node
spec:
  ports:
    - port: 8080 # 다른 파드가 서비스에 접근하기 위한 포트
      targetPort: 80 # 대상 파드에 트래픽을 전달하는 포트
      nodePort: 30080 # 서비스가 외부에 공개되는 포트
  selector:
    app: numbers-web
  type: NodePort
```
NodePort는 클러스터 환경에 따라 동일하게 동작하지 않고, 실제로 사용할 일은 별로 없다. 라고 하는데, 내가 듣는 인강에서는 NodePort로 실습했고, 로드밸런서는 외부 ELB같은 도구를 사용할 때 쓴다고 했다.

## 내부 트래픽을 외부로 전달하기
때로는 k8s 외부에 있는 컴포넌트 예를 들면 DB와도 통신할 필요가 있을 것이다. 클러스터 외부를 가리키는 도메인 네임을 찾는데에도 서비스를 활용할 수 있다.

### 방법1. ExternalName Service
ExternalName Service는 어떤 도메인 네임에 대한 별명이다.
파드에서는 로컬 네임을 사용, k8s DNS에서 해당 로컬 네임을 조회하면 외부 도메인으로 매핑하는 방식이다.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: numbers-api
spec:
  type: ExternalName
  externalName: raw.githubusercontent.com # 로컬 도메인 네임과 매핑할 외부 도메인
```
numbers-services/api-service-externalName.yaml

```bash
# 서비스의 유형을 변경하려면 서비스를 삭제하고, 다시 생성해야한다.
% kubectl delete svc numbers-api
service "numbers-api" deleted

% kubectl apply -f numbers-services/api-service-externalName.yaml
service/numbers-api created

# 새로운 서비스는 깃허브 도메인을 external ip로 가지고 있다.
% kubectl get svc numbers-api
NAME          TYPE           CLUSTER-IP   EXTERNAL-IP                 PORT(S)   AGE
numbers-api   ExternalName   <none>       raw.githubusercontent.com   <none>    7s
```

![](https://velog.velcdn.com/images/ayeon0/post/e05aa79d-afee-4098-8d75-05efac9f2b6e/image.png)
같은 숫자만 뜬다.

프론트에서는 http://numbers-api/sixeyed/kiamol/master/ch03/numbers/rng로 API요청을 보내고 있는데, 여기서 numbers-api는 클러스터 내부 도메인 이름이었다.
근데 ExternalName을 이용해서 numbers-api가 raw.githubusercontent.com로 매핑되도록 설정했으므로  https://raw.githubusercontent.com/sixeyed/kiamol/master/ch03/numbers/rng로 요청을 보내게 되는 것이다.
k8s는 DNS의 표준 기능 중 하나인 CNAME을 사용하여 ExternalName Service를 구현한다. 웹 애플리케이션 파드가 도메인 네임 numbers-api를 조회하면 쿠버네티스 DNS는 CNAME인 raw.githubusercontent.com을 반환하여, 깃허브 서버로 API 요청이 전달되는 것이다.
k8s 서비스는 클러스터 전체를 커버하는 가상 네트워크의 일부이므로, 모든 파드가 서비스를 이용할 수 있다.

ExternalName 서비스에는 한계가 있다. 도메인 URL을 바꿔주므로 당연히 요청 IP주소는 달라지지만, 헤더까지 바꾸지는 못한다는 것이다.
단순 TCP를 이용하는 DB에는 문제가 없지만, HTTP 헤더에는 호스트명이 들어가는데, ExternalName 서비스를 통해 호스트가 달라지면 HTTP 요청은 실패한다. 따라서 HTTP를 사용하지 않는 TCP 서비스에는 이 방법이 최선이다.

### 방법2. HeadLess Service
이 방법 또한 ExternalName 서비스가 가지는 한계를 극복하지는 못한다. ExternalName 서비스가 도메인 네임을 매핑시켜줬다면, 헤드리스 서비스는 IP 주소를 매핑시켜준다.
헤드리스 서비스는 클러스터IP의 형태로 정의되지만, 레이블 셀렉터가 없는 대신에 자신이 제공해야할 IP 주소의 목록이 담긴 엔드포인트 리소스를 가진다.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: numbers-api
spec:
  type: ClusterIP      # 일반적인 ClusterIP 서비스와 다르게, label selector가 없다.
  ports:
    - port: 80
---                    # --- 을 이용하면 한 파일에 여러 개의 리소스를 정의할 수 있다.
kind: Endpoints
apiVersion: v1
metadata:
  name: numbers-api
subsets:
  - addresses:         # IP 주소 목록을 가진다.
      - ip: 192.168.123.234
    ports:
      - port: 80
```
numbers-services/api-service-headless.yaml
```bash
% kubectl delete svc numbers-api 
service "numbers-api" deleted

% kubectl apply -f numbers-services/api-service-headless.yaml
service/numbers-api created
endpoints/numbers-api created

# 헤드리스 서비스 또한 일반 서비스와 마찬가지로 
# 서비스의 유형은 ClusterIP로, 클러스터 내부의 가상 IP주소를 가진다.
% kubectl get svc numbers-api
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
numbers-api   ClusterIP   10.105.225.35   <none>        80/TCP    8s

# 서비스가 가진 엔드포인트에서는 클러스터 IP가 연결하는 주소(192.168.123.234:80)를
# 볼 수 있지만 여기서는 실재하는 주소는 아니다.
% kubectl get endpoints numbers-api
NAME          ENDPOINTS            AGE
numbers-api   192.168.123.234:80   18s

% kubectl get svc
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes    ClusterIP      10.96.0.1        <none>        443/TCP          13d
numbers-api   ClusterIP      10.105.225.35    <none>        80/TCP           9m42s
numbers-web   LoadBalancer   10.111.222.240   localhost     8080:31382/TCP   11h
sleep-2       ClusterIP      10.110.59.233    <none>        80/TCP           7d1h

% kubectl exec deploy/sleep-1 -- sh -c 'nslookup numbers-api | grep "^[^*]"'
Server:		10.96.0.10
Address:	10.96.0.10:53
Name:	numbers-api.default.svc.cluster.local
Address: 10.105.225.35 # 왜 주소가 엔드포인트의 IP 주소가 아니라 서비스의 주소인지??
```

![](https://velog.velcdn.com/images/ayeon0/post/48c71438-20da-4e17-9856-bd286a756406/image.png)
실재하지 않는 주소로 매핑되어 실패했다.

왜 DNS가 반환하는 주소가 엔드포인트의 IP가 아니라 서비스의 주소일까?
답은 k8s 서비스의 동작 원리에 있다.

## k8s 서비스 작동원리는?
복잡하다고 생각되면 클러스터IP 유형을 주로 사용한다고 생각하자.

1. 도메인 네임 조회
   파드 속 컨테이너가 요청한 도메인 네임의 조회는 k8s DNS가 응답한다.
   조회 대상이 서비스면 DNS는 클러스터 내부 IP 혹은 외부 도메인 네임을 반환한다.

2. 라우팅
   파드에서 나오는 모든 트래픽은 네트워크 프록시가 라우팅을 담당한다. 이 프록시는 각각의 노드에서 동작하며, 모든 서비스의 엔드포인트에 대한 최신정보를 유지하고, 운영체제가 제공하는 네트워크 패킷 필터(리눅스는 IPVS 또는 iptables)를 사용하여 트래픽을 라우팅한다.

여기서 클러스터 IP는 네트워크에 실재하지 않는 가상 IP 주소다.
파드는 노드마다 있는 네트워크 프록시를 거쳐 네트워크에 접근하며, 네트워크 프록시는 패킷 필터링을 적용해 가상 IP 주소를 실제 엔드포인트로 연결한다.
서비스는 삭제 전까지 IP 주소가 바뀌지 않으므로 오래 지속되며, 서비스에도 컨트롤러가 있어 파드가 변경되어도 엔드포인트를 최신으로 유지할 수 있다.

### k8s 네임스페이스
네임스페이스는 여러 리소스를 하나로 묶기 위한 리소스다.
k8s 클러스터를 파티션으로 나누는 역할을 한다.
우리가 사용하는 클러스터에는 이미 여러 개의 네임스페이스가 있다.
기본적으로 있는 default 네임스페이스가 있고, k8s DNS나 k8s API 같은 내장 컴포넌트는 kube-system 네임스페이스에 속한 파드에서 동작한다.
지금까지 만든 리소스는 특별히 지정하지 않아 default 에 해당한다.
```bash
# --namespace (-n) 을 이용하여 네임스페이스를 지정할 수 있다.
% kubectl get svc --namespace default
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes    ClusterIP      10.96.0.1        <none>        443/TCP          13d
numbers-api   ClusterIP      10.105.225.35    <none>        80/TCP           3h43m
numbers-web   LoadBalancer   10.111.222.240   localhost     8080:31382/TCP   15h
sleep-2       ClusterIP      10.110.59.233    <none>        80/TCP           7d4h

# kube-system 네임스페이스에 해당하는 서비스 조회
% kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   13d

# 네임스페이스까지 들어간 완전한 도메인 네임으로 DNS 조회하기
% kubectl exec deploy/sleep-1 -- sh -c 'nslookup numbers-api.default.svc.cluster.local | grep "^[^*]"'
Server:		10.96.0.10
Address:	10.96.0.10:53
Name:	numbers-api.default.svc.cluster.local
Address: 10.105.225.35

# 쿠버네티스 시스템 네임스페이스의 완전한 도메인 네임으로 DNS 조회하기
% kubectl exec deploy/sleep-1 -- sh -c 'nslookup kube-dns.kube-system.svc.cluster.local | grep "^[^*]"'
Server:		10.96.0.10
Address:	10.96.0.10:53
Name:	kube-dns.kube-system.svc.cluster.local
Address: 10.96.0.10
```
네임스페이스를 활용하면 보안을 해치지 않고 클러스터의 활용도를 높일 수 있다.

## 서비스와 디플로이먼트 삭제하기
```bash
# 모든 디플로이먼트를 삭제합니다. 
# 네임스페이스를 지정하지 않았으므로, default에 속한 것만 삭제됩니다.
% kubectl delete deploy --all
deployment.apps "numbers-api" deleted
deployment.apps "numbers-web" deleted
deployment.apps "sleep-1" deleted

# 모든 서비스를 삭제합니다.
% kubectl delete svc --all
service "kubernetes" deleted
service "numbers-api" deleted
service "numbers-web" deleted
service "sleep-2" deleted

# 남아있는 리소스를 확인합니다.
% kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   7s
```
