# 쿠버네티스 내부의 네트워크 트래픽 라우팅

- 파드는 고유한 IP 주소를 갖지만, 파드가 재시작되면 IP가 바뀜.
- 그래서 다른 파드가 해당 파드로 통신하려고 하면 문제가 생길 수 있음.
- 쿠버네티스는 이런 문제를 해결하려고 DNS를 사용함.
- 서비스 리소스를 생성하면 내부 DNS 서버에 등록되고, 고정된 서비스 이름을 통해 파드 IP가 바뀌어도 항상 접근 가능함.

# 파드와 파드 간 통신
- 파드 간 통신을 할 땐 보통 ClusterIP 서비스를 사용함.
- ClusterIP는 쿠버네티스 클러스터 내부에서만 접근 가능한 가상 IP를 제공함.
- 서비스는 셀렉터(label selector)를 통해 여러 파드에 연결됨.


### 이렇게 하면 사용자 요청이 자동으로 여러 파드로 로드 밸런싱 됨.

📄 서비스 정의 예시 (ClusterIP)
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
      targetPort: 80
```

🧪 서비스 배포 및 테스트

### 서비스 생성
kubectl apply -f sleep/sleep2-service.yaml

### 서비스 확인
kubectl get svc sleep-2

#### sleep-1 파드에서 sleep-2로 ping 테스트
kubectl exec deploy/sleep-1 -- ping -c 1 sleep-2
💡 참고: ping은 ICMP를 사용하는데, 서비스 리소스는 TCP/UDP용이기 때문에 잘 안 될 수도 있음.

# 외부 트래픽을 파드로 전달하기
- 클러스터 바깥에서 파드로 접근하려면 두 가지 방법이 있음
- NodePort: 노드의 특정 포트를 통해 서비스에 접근.
- LoadBalancer: 클라우드 환경에서 로드밸런서를 생성해서 서비스 연결.

📄 NodePort 예시
```yaml

apiVersion: v1
kind: Service
metadata:
  name: sleep-2
spec:
  type: NodePort
  selector:
    app: sleep-2
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

💡 NodePort는 외부에서 [노드IP]:[NodePort] 형태로 접근해야 함.

### 쿠버네티스 클러스터 외부로 트래픽 전달하기 (Ingress)
- 여러 서비스를 하나의 도메인 주소로 접근하게 만들려면 Ingress 리소스를 사용함.
- Ingress는 경로 기반 또는 호스트 기반 라우팅을 제공함.
- 일반적으로 NGINX 같은 Ingress Controller가 필요함.

📄 Ingress 리소스 예시
```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sleep-2
                port:
                  number: 80
```

💡 Ingress를 쓰면 도메인과 경로에 따라 여러 서비스를 라우팅할 수 있어서 진짜 유용함.

### 쿠버네티스 서비스의 DNS 해석 과정
- 서비스 이름은 내부 DNS로 등록돼서 파드끼리 이름으로 접근 가능함.
- 예: sleep-2.default.svc.cluster.local
- 도메인은 일반적으로 [서비스이름].[네임스페이스].svc.cluster.local 형식임.
- DNS 서버는 CoreDNS가 기본으로 설치되어 있고, 서비스 이름을 IP로 변환해 줌.
- 파드가 삭제되고 새로 만들어져도 서비스 이름은 고정이니까, 통신 문제가 없음.

---

# 기억하면 좋은 포인트 요약

| 항목             | 내용                                                       |
|------------------|------------------------------------------------------------|
| ClusterIP        | 클러스터 내부 통신용 기본 서비스 타입                      |
| NodePort         | 노드의 포트를 열어 외부에서 접근 가능하게 함               |
| LoadBalancer     | 클라우드 환경에서 퍼블릭 IP 부여                           |
| Ingress          | HTTP/HTTPS 경로 기반 라우팅. 여러 서비스 통합 가능         |
| 서비스 DNS       | 서비스 이름은 DNS로 자동 등록되어 접근 가능                |
| 파드 IP          | 재시작 시 변경됨 → 서비스로 추상화 필요                    |
| 서비스 선택자     | 레이블 셀렉터로 대상 파드 그룹 지정함                      |


---

# 논의해 볼만한 질문
- 서비스 타입 중에서 NodePort와 LoadBalancer는 언제 각각 쓰는 게 더 적절할까?
- Ingress를 구성할 때 경로 기반 라우팅과 서브도메인 기반 라우팅 중 어떤 게 더 관리하기 쉬울까?

