# 쿠버네티스 교과서 4장 요약: 컨피그맵과 비밀값으로 애플리케이션 설정하기

## 4.1 설정값을 쿠버네티스에서 다루는 방식

쿠버네티스에서는 컨테이너 이미지를 재빌드하지 않고도 설정값을 관리할 수 있도록 `ConfigMap`과 `Secret` 리소스를 제공함. 이 두 가지는 외부 환경에 따라 설정을 다르게 주입할 수 있게 도와줌.

- **ConfigMap**: 설정파일, 문자열, ENV 값 등 비민감 정보를 저장
- **Secret**: 비밀번호, 인증 토큰 등 민감 정보를 base64로 인코딩해서 저장

*설정과 코드는 분리하는 게 좋은 아키텍처. 환경마다 설정이 다르면 코드에 하드코딩하지 말고 이 둘을 활용*

---

## 4.2 ConfigMap 사용법

### 사용 시점
- 환경마다 설정이 다름 (`dev`, `prod`)
- 민감하지 않은 설정값 (예: 로그 레벨, 외부 API 주소)

### ConfigMap 생성 방법

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "DEBUG"
  API_URL: "https://api.example.com"
```

위 설정은 kubectl apply -f 명령으로 생성할 수 있음.

## 4.3 ConfigMap 적용 방법

### 환경 변수로 주입
```yaml
envFrom:
  - configMapRef:
      name: app-config
```
→ 컨테이너 안에 LOG_LEVEL, API_URL이 환경 변수로 들어감

### 파일처럼 주입 (볼륨 마운트)
```yaml
volumeMounts:
  - name: config-volume
    mountPath: "/etc/config"

volumes:
  - name: config-volume
    configMap:
      name: app-config
```
→ 컨피그맵 값이 컨테이너 안에 파일로 들어감 (/etc/config/LOG_LEVEL 등)

파일로 마운트하면 애플리케이션이 설정 파일을 직접 읽게 만들 수 있음.

## 4.4 Secret 사용법
사용 시점
- 사용자 이름, 비밀번호, API 키, 인증서 등을 다룰 때

Secret 생성 예시

```yaml

apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: bXl1c2Vy  # base64 인코딩된 "myuser"
  password: bXlwYXNz  # base64 인코딩된 "mypass"
```
echo -n 'myuser' | base64 로 직접 인코딩 가능

### 환경 변수로 Secret 주입
```yaml

env:
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: username
```

→ 이 값은 printenv 같은 명령으로 노출될 수 있음. 보안에 주의

## 4.5 설정값의 라이프사이클
설정값이 바뀌었을 때 바로 반영되게 하려면 볼륨 마운트를 선호함

환경 변수로 주입하면 파드를 다시 시작해야 적용됨

컨피그맵과 시크릿은 Deployment와 분리 관리되므로 CI/CD 파이프라인에서 유용함

---

### 스터디 토론 질문
- ConfigMap과 Secret을 환경 변수로 쓸 때와 볼륨으로 마운트할 때의 차이는?
- Secret이 base64로 인코딩되어 있지만 진짜 보안이 되는 걸까?