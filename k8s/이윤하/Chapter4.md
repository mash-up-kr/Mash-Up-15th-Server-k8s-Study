![](https://velog.velcdn.com/images/ayeon0/post/6e209701-7a47-46b3-8e00-e1a2a44c17c3/image.png)
## 인트로
컨테이너에 환경별로 설정값, 환경변수(시크릿)을 주입하기 위해서는 **컨피그맵**과 **비밀값**이 있다. 이 데이터는 클러스터 속에서 다른 리소스와 독립적인 장소에 보관된다.
파드 정의에서 컨피그맵과 비밀값의 데이터를 읽어 오도록 할 수 있다.
이번 4장에서는 k8s의 설정 관리를 다루어보자.

## 설정이 애플리케이션에 저장되는 과정
컨피그맵과 비밀값 역시 타 리소스처럼 k8s CLI(create)나 매니페스트 파일로 정의하여 생성할 수 있다. 타 리소스와 달리 이 둘은 스스로 어떤 기능을 하지 않고 적은 양의 데이터만 저장한다.
이 리소스는 파드로 전달되어 컨테이너 환경의 일부가 되고, 컨테이너는 전달된 데이터를 읽어들일 수 있다.

## 환경 변수
컨피그맵과 비밀값을 알기 전에 환경 변수를 알아보자
> 환경 변수는 컴퓨터 단위로 설정되며, 모든 애플리케이션이 읽을 수 있다.

환경 변수를 주입하기 위해서는 디플로이먼트 매니페스트 파일 정의에 추가하면 된다.
```yaml
spec:
      containers:
        - name: sleep
          image: kiamol/ch03-sleep
          env:
          - name: KIAMOL_CHAPTER # 환경 변수의 이름
            value: "04"          # 변수의 값
```

* 환경 변수는 파드 생애 주기 내내 변경되지 않음
* 파드 실행 중에는 환경 변수 값 수정 불가, 수정하기 위해서는 파드를 수정된 버전으로 대체(apply)해아함.

## 컨피그맵
간단한 설정은 디플로이먼트에 정의할 수 있겠으나 실제로는 많고 복잡한 설정값을 **컨피그맵**으로 관리한다.
> 파드에서 읽어  들이는 데이터를 저장하는 리소스

*  key-value, text, binary 파일까지 다양한 형태를 가지고 있다.

파드 하나에 여러 개의 컨피그맵을 전달할 수 있고, 반대로도 가능하다.

```yaml
  env:
	- name: POSTGRES_DB
    	valueFrom:
        	configMapKeyRef:
            	name: postgres-config # ConfigMap의 이름
                key: DB_DATABASE # ConfigMap에서 가져올 키
```

`postgres-config`라는 컨피그맵의 `DB_DATABASE`이라는 키에 해당하는 값을 읽어와 `POSTGRES_DB` 환경 변수로 할당하라는 정의다.
파드 정의에서 컨피그맵을 참조하게 되면 해당 컨피그맵이 존재해야 클러스터에 배치할 수 있다.
CLI로 만들어보자.

```bash
# 컨피그맵 생성
$ kubectl create configmap sleep-config-literal --from-literal=kiamol.section='4.1'

# 컨피그맵에 들어 있는 데이터 확인
$ kubectl get cm sleep-config-literal # cm => configMap과 같다.

# 컨피그맵의 상세 정보를 보기 좋게 출력
$ kubectl describe cm sleep-config-literal

# 정의가 수정된 파드 배치
$ kubectl apply -f sleep/sleep-with-configMap-env.yaml

# 환경 변수 적용 확인
$ kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'
```

### 컨피그맵에 설정 파일 사용하기
`.env`파일로 컨피그맵을 만들 수 있다.
```
KIAMOL_CHAPTER=ch04
KIAMOL_SECTION=ch04-4.1
KIAMOL_EXERCISE=try it now
```
```bash
# 환경 파일 내용으로 컨피그맵 생성
kubectl create configmap sleep-config-env-file --from-env-file=sleep/ch04.env

# 컨피그맵의 상세 정보 확인
kubectl get cm sleep-config-env-file

# 새로운 컨피그맵의 설정을 적용해 파드 업데이트
kubectl apply -f sleep/sleep-with-configMap-env-file.yaml

# 컨테이너에 적용된 환경 변수의 값 확인
kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'
```
* 만약 env 항목과 envFrom 항목에서 컨피그맵 두 개를 읽어들이고, 같은 환경 변수를 갖고 있다면 env 항목에서 정의된 값이 envFrom 보다 우선된다.
### 설정값에 우선순위 부여
환경에 따라 설정값의 우선 순위가 달라질 수 있다.
기본 설정값은 도커 이미지에 포함된 JSON 파일에서 읽어 들이지만, 애플리케이션이 그 외 위치를 찾아 설정 파일이 발견될 경우, 해당 파일의 설정값이 기본값을 대체한다. 거기에 환경 변수는 모든 JSON 설정 파일보다 우선한다. 정리해보자
> 환경 변수 > JSON 설정 > 도커 이미지에 포함된 기본 JSON 설정

### 매니페스트 파일로 컨피그맵 정의
매니페스트 파일로 컨피그맵을 정의해보자
```yaml
apiVersion: v1
kind: ConfigMap # 리소스 유형

metadata:
  name: postgres-config # 컨피그맵 이름

data:
  config.json: |- # 키 이름이 파일 이름
    {
      "ConfigController": { # 파일 내용은 어떤 포맷이든 가능
        "Enabled" : true
      }
    }
```

해당 컨피그맵을 배치하고 수정된 정의로 파드를 업데이트하면 /config 페이지를 확인할 수 있다.

간단하게 생성하는 방법은 이런 방법도 있다.
```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: nest-config # ConfigMap 이름

data:
  DB_DATABASE: 'mash-up-db'
```

이 경우 위에서 본 것처럼
```yaml
  env:
	- name: POSTGRES_DB
    	valueFrom:
        	configMapKeyRef:
            	name: postgres-config # ConfigMap의 이름
                key: DB_DATABASE # ConfigMap에서 가져올 키
```
로 간단하게 값을 가져올 수 있다.

## 컨피그맵에 담긴 설정값 데이터 주입
환경 변수 외에 설정값을 전달하는 다른 방법은 컨테이너 파일 시스템 속 파일로 설정값을 주입하는 것이다. k8s는 컨피그맵은 디렉터리로, 각 항 목은 파일 형태로 컨테이너 파일 시스템에 추가한다.
이 과정에서 파드 정의의 두 가지 항목고 관련된 기능이 관여한다.
* 볼륨: 컨피그맵에 담긴 데이터를 파드로 전달
* 볼륨 마운트: 볼륨을 파드 컨테이너의 특정 경로에 위치시킴

```yaml
    spec:
      containers:
        - name: web
          image: kiamol/ch04-todo-list
          volumeMounts: # 컨테이너에 볼륨을 마운트
            - name: config # 마운트할 볼륨의 이름
              mountPath: "/app/config" # 볼륨이 마운트될 경로
              readOnly: true # 볼륨을 읽기 전용으로 마운트
      volumes: # 볼륨
        - name: config # 볼륨의 이름
          configMap: # 볼륨의 원본은 컨피그맵
            name: todo-web-config-dev # 내용을 읽어올 컨피그맵 이름
```

컨피그맵이 디렉터리로 취급되고 컨피그맵 속의 각각의 항목이 파일이 된다.
이 예제에서는 /app/appsettings.json에서 기본 설정을 읽어 오며, /app/config/config.json 파일을 찾아 해당 파일의 설정값을 우선 적용한다.

## 컨피그맵과 볼륨 마운트 사용시 주의사항
아래와 같이 마운트 경로가 이미 컨테이너 이미지에 있는 경로라면 컨피그맵 디렉터리가 기존 디렉터리를 덮어 쓸 수 있다.

...
    spec:
      containers:
        - name: web
          image: kiamol/ch04-todo-list
          volumeMounts:
            - name: config
              mountPath: "/app" # 잘못된 경로!
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: todo-web-config-dev
...
해당 정의를 보면 컨피그맵 볼륨은 /app 디렉토리에 마운트되게 된다.
이 경우, /app 디렉터리에 파일이 병합되지 않고 오히려 해당 디렉터리가 증발한다.

## 비밀값으로 설정값 다루기
민감한 정보는 컨피그맵 말고 비밀값으로 다루자
```bash
# 평문 리터럴로 비밀값 생성
$ kubectl create secret generic sleep-secret-literal --from-literal=secret=shh...

# 비밀값의 상세 정보 확인
$ kubectl describe secret sleep-secret-literal

# 비밀값의 평문 확인(base64 인코딩됨)
$ kubectl get secret sleep-secret-literal -o jsonpath='{.data.secret}'

# 비밀값의 평문 확인
$ kubectl get secret sleep-secret-literal -o jsonpath='{.data.secret}' | base64 -d
```
생성된 비밀값은 describe로 확인하려 해도 출력되지 않는다.
다른 명령어를 통해 데이터를 출력해도 Base64로 인코딩된 값만 출력된다.
```yaml
    spec:
      containers:
        - name: sleep
          image: kiamol/ch03-sleep
          env:
          - name: KIAMOL_SECRET
            valueFrom:
              secretKeyRef: # 비밀값에서 데이터 가져오기
                name: sleep-secret-literal # 비밀값 이름
                key: secret # 비밀값 항목 이름
```
```bash
$ kubectl apply -f sleep/sleep-with-secret.yaml
deployment. apps/sleep configured

$ kubectl exec deploy/sleep -- printenv KIAMOL_SECRET
s h h . . .
```
환경 변수는 컨테이너에서 동작하는 모든 프로세스에서 접근할 수 있다.
일부 애플리케이션은 오류가 발생하면 환경 변수를 전체를 로그로 남기는 경우도 있기 때문에 비밀값의 데이터가 노출될 수 있다. 애플리케이션이 설정 파일을 지원한다면 파일 권한 설정으로 정보를 지키자.
* 비밀값은 yaml로 관리하지 말자.
* 대신 자리만 배치해두고 애플리케이션을 배치할 때 채워넣는 방식으로 구현하자.

## k8s 애플리케이션 설정 관리
쿠버네티스에 애플리케이션을 배포할 때 설계 단계에서 고려해야할 점은 크게 두 가지다.

1. 애플리케이션 중단 없이 설정 변경
2. 민감 정보를 관리 방식
### 무중단 설정 변경
파드가 교체되지 않는 무중단 업데이트를 위해서는 환경 변수를 활용할 수 없다.
대신 볼륨 마운트를 이용해 설정 파일을 수정해야 합니다.
이 경우에도 기존 컨피그맵이나 비밀값을 업데이트하는 방식이어야 한다.

컨피그맵이나 비밀값을 업데이트하지 않고 설정을 변경하기 위해서는 설정 객체에 버전을 명시하고 애플리케이션을 업데이트할 때 새로운 설정 객체를 배치한 후 이 설정 객체를 가리키도록 애플리케이션을 수정해야 합니다.
애플리케이션을 수정한다는 것에 무중단 배포는 불가해지지만, 설정값 변경 이력이 남고 이전 설정으로 돌아가는 선택지가 생긴다.

### 민간 정보 관리 방식
설정 파일 배포를 관리하는 팀이 있다면 컨피그맵과 비밀값의 버전 관리 정책이 적합하다.
아니면, 형상 관리에 저장된 yaml 템플릿 파일로 컨피그맵과 비밀값이 업데이트 되는 자동화된 배치를 사용할 수 있다.
이 경우 yaml 템플릿에는 민간 정보가 들어갈 빈칸을 만들어 둔 뒤, 다른 보안 저장소에서 해당 값을 채워 yaml 파일을 완성하는 방식을 채택해야 한다.