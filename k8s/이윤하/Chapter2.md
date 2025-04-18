# 2장 파드와 디플로이먼트로 컨테이너 실행하기

## 2-1 쿠버네티스는 어떻게 컨테이너를 실행하고 관리하는가?
### 파드란 무엇인가? 
* 컨테이너는 애플리케이션 구성 요소 하나를 실행하는 가상화된 환경을 가리킨다.
* 쿠버네티스는 이 컨테이너를 또 다른 가상환경인 **파드**로 감싼다.
* 파드는 쿠버네티스에서 컴퓨팅의 **최소 단위**다.
* 파드 1개는 보통 컨테이너 1개를 포함하지만, 예외적으로 여러 개의 컨테이너를 포함할 수 있다. 
![Pod](https://velog.velcdn.com/images/ayeon0/post/e18dc783-0133-4c9d-baa3-d0a1169f050e/image.JPG)

### 파드 만들어보기
* 책에서는 CLI로 실습했지만, 바로 yaml 파일로 만들어보자
1. yaml 파일 생성하기(일단 따라해보자)
* nginx-pod.yaml
```yaml
apiVersion: v1 
kind: Pod 
metadata:
  name:
spec:
  containers:
    - name: nginx-container
      image: nginx
      ports:
        - containerPort: 80
```
* 들여쓰기를 할 때 **Tab** 말고 **띄어쓰기**를 활용해야한다.
2. yaml 파일 기반 파드 생성하기
```bash
$ kubectl apply -f nginx-pod.yaml
```
3. 파드 조회하기
```bash
$ kubectl get pods
```
4. 파드 상세 정보 조회
```bash
$ kubectl describe pod ${pod 이름}
```
* 쿠버네티스는 직접 컨테이너를 실행하지 않고, 파드를 통해 실행한다.
* 컨테이너는 노드에 설치된 컨테이너 런타임(ex. 도커)을 통해 실행된다.
> 쿠버네티스는 파드를 관리, 컨테이너는 쿠버네티스 외부에서 관리된다.

### 파드에 접근하기
* 파드로 띄운 프로그램(Nginx)에 접속이 안 된다?
![Nginx](https://velog.velcdn.com/images/ayeon0/post/f1f9c6f0-ad96-4495-94e8-fbee84236f57/image.png)
* 도커에서는 컨테이너 내부와 컨테이너 외부의 네트워크가 서로 독립적으로 분리되어있지만, 쿠버네티스에서는 **파드 내부의 네트워크를 컨테이너가 공유한다.**
* **파드의 네트워크는 로컬 컴퓨터의 네트워크와 분리**되어 있다. 그러니 이 예제에서 Nginx로 요청을 보내도 응답이 없던 것이다.
> 포트 포워딩을 해보자
```bash
$ sudo kubectl port-forward pod/${파드의 이름} ${로컬 포트}:${파드의 포트}
```
![결과](https://velog.velcdn.com/images/ayeon0/post/ddc8493e-5e73-400b-a8a2-350e4b55eb70/image.png)
성공적으로 요청이 보내졌다.
## 2-2 컨트롤러 객체와 함께 파드 실행하기
> 객체란? 
쿠버네티스에서는 서비스(Service), 디플로이먼트(Deployment), 파드(Pod)와 같은 리소스를 보고 객체(오브젝트, Object)라고 부른다

* 컨트롤러 객체를 사용하면 파드를 쉽게 관리할 수 있다.
* 디플로이먼트(Deployment)는 파드를 묶음으로 쉽게 관리할 수 있는 컨트롤러 객체
### 디플로이먼트의 장점
* 관리 및 생성할 파드의 수를 지정할 수 있음
* 파드를 주시하다 비정상적으로 종료될 경우, 자동으로 새 파드를 생성한다.
* 여러 파드를 한 번에 중지, 삭제, 업데이트 할 수 있다.
### 디플로이먼트의 구조
![구조](https://velog.velcdn.com/images/ayeon0/post/6d492b6e-907d-40cf-87c2-ee0fe194fe47/image.png)
> 디플로이먼트는 레플리카셋을 관리, 레플리카셋이 파드들을 관리한다.

실습은 밑에서 yaml 파일을 만들면서 해보자.

## 2-3 애플리케이션 매니페스트에 배포 정의하기
* 쿠버네티스 API의 정식 스크립트 포맷은 JSON이지만, 가독성이 뛰어나고 주석도 작성 가능하고, 형상 관리가 되는 YAML을 많이 쓴다.
### 매니페스트로 파드 생성해보기
* 위에서 작성한 nginx-pod.yaml을 다시 보자
```yaml
apiVersion: v1 # Pod를 생성할 때는 v1이라고 기재한다. (공식 문서)
kind: Pod # Pod를 생성한다고 명시

# 메타데이터에는 이름(필수)와 레이블(필수x)가 있다.
metadata:
  name: nginx-pod # Pod에 이름 붙이는 기능
  
# 리소스의 실제 정의 내용, 실행할 컨테이너를 정의하는 곳이다.
spec:
  containers:
    - name: nginx-container # 생성할 컨테이너의 이름
      image: nginx # 컨테이너를 생성할 때 사용할 Docker 이미지
      ports:
        - containerPort: 80 # 해당 컨테이너가 어떤 포트를 사용하는 지 명시적으로 표현
```

### 디플로이먼트로 NestJS를 띄워보자
1. Dockerfile 작성하기
```bash
FROM node

WORKDIR /app

COPY . .

RUN npm install

RUN npm run build

EXPOSE 3000

ENTRYPOINT [ "node", "dist/main.js" ]
```

2. .dockerignore 작성
```bash
node_modules
```
3. Dockerfile을 바탕으로 이미지 빌드하기
```bash
$ docker build -t nest-server:1.0 .
```
4. 디플로이먼트를 매니페스트로 생성하기
nest-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment

# Deployment 기본 정보
metadata:
  name: nest-deployment # Deployment 이름

# Deployment 세부 정보
spec:
  replicas: 3 # 생성할 파드의 복제본 개수
  selector:
    matchLabels:
      app: backend-app # 아래에서 정의한 Pod 중 'app: backend-app'이라는 값을 가진 파드를 선택

  # 배포할 Pod 정의
  template:
    metadata:
      labels: # 레이블 (= 카테고리)
        app: backend-app
    spec:
      containers:
        - name: nest-container # 컨테이너 이름
          image: nest-server:1.0 # 컨테이너를 생성할 때 사용할 이미지
          imagePullPolicy: IfNotPresent # 로컬에서 이미지를 먼저 가져온다. 없으면 레지스트리에서 가져온다.
          ports:
            - containerPort: 3000  # 컨테이너에서 사용하는 포트를 명시적으로 표현
```
5. 디플로이먼트 배포하기
```bash
$ kubectl apply -f nest-deployment.yaml
```
6. 잘 생성되었는지 확인
```bash
$ kubectl get deployment
$ kubectl get replicas
$ kubectl get pods
```

## 2-4 파드에서 실행 중인 애플리케이션에 접근하기
* 실제 애플리케이션은 컨테이너 속에서 동작한다. kubectl로 파드 안 컨테이너에 접근할 수 있다.
* 파드 내부로 직접 들어가보자
* 위에서 만든 nginx-pod.yaml을 apply 해보았다.
```bash
$ kubectl exec -it ${pod 이름} -- bash

# 파드 내부
$ curl localhost:80
```
![결과](https://velog.velcdn.com/images/ayeon0/post/78323be0-191b-4bb2-ae6a-5e9e3be24b77/image.png)
도커에서 컨테이너로 접속하는 명령어와 유사하게 접근 가능하다.
> bash로 접속이 안 된다면 ```bash``` 대신 ```sh```로 접근하면 된다.
컨테이너 종류에 따라 ```bash``` 대신 ```sh```가 설치되어있을 수 있다.
-- 와 쉘 사이에 띄어쓰기가 있음에 유의하자

* 컨테이너의 로그 또한 확인할 수 있다. ```--tail=2``` 옵션으로 최근 2줄만 출력하게 할 수 있다.
```bash
# kubectl logs [파드명]
$ kubectl logs nginx-pod # 파드 로그 확인하기
```
* 쉘로 파드에 접근한 경우 파일 시스템에 접근할 수 있다. 추후 secret 관리 때 사용하겠지만 ```env``` 명령어로 환경변수를 열람할 수 있고, ```cat```이나 ```cp```등 리눅스 명령어도 사용할 수 있다.

## 2-5 쿠버네티스의 리소스 관리 이해하기
* 디플로이먼트는 레플리카셋에서 정의한 개수만큼의 파드를 관리한다.
* 만약 수동으로 파드를 삭제한 경우, 그만큼의 파드가 자동으로 생성된다.
```bash
$ kubectl delete pod --all # 모든 유형의 리소스를 삭제
```
* 컨트롤러 객체(여기서는 디플로이먼트)를 삭제하면 자신이 관리하던 리소스(파드)를 제거하고 삭제된다.

## 나온 명령어들 정리
파드 조회
```bash
$ kubectl get pods
```
파드 포트포워딩
```bash
# kubectl port-forward pod/[파드명] [로컬에서의 포트]/[파드에서의 포트]
$ kubectl port-forward pod/nginx-pod 80:80
```
파드 삭제
```bash
# kubectl delete pod [파드명]
$ kubectl delete pod nginx-pod # nginx-pod라는 파드 삭제
```
파드 디버깅
```bash
# kubectl describe pods [파드명]
$ kubectl describe pods nginx-pod # nginx-pod 파드의 세부 정보 조회
```

파드 로그 확인하기
```bash
# kubectl logs [파드명]
$ kubectl logs nginx-pod # 파드 로그 확인하기
```
파드 내부 접속하기
```bash
# kubectl exec -it [파드명] -- bash
$ kubectl exec -it nginx-pod -- bash

# kubectl exec -it [파드명] -- sh
$ kubectl exec -it nginx-pod -- sh
```
매니페스트로 정의한 리소스 생성하기
```bash
# kubectl apply -f [파일명]
$ kubectl apply -f nginx-pod.yaml
```