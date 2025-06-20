# CH6 컨트롤러 리소스를 이용한 어플리케이션의 스케일링

> 쿠버네티스에서 동일한 어플리케이션이 돌아가는 파드를 레플리카라고 한다. 그리고 노드가 여러 개인 클러스터에서 레플리카는 여러 노드에 분산 배치된다.
> - 이러한 방식으로 더 ㅁ낳은 요청을 처리하고 일부가 고장을 일으키더라도 나머지가 그대로 동작하여 노은 가용성을 확보하는 스케일링의 이점을 누릴 수 있다.

## 6.1 쿠버네티스는 어떻게 어플리케이션을 스케일링하는가
- 파드는 쿠버네티스에서 컴퓨팅의 단위다.
- 파드를 관리하는 다른 리소스를 정의하여 사용하고, 컨트롤러라고 부른다.
- 컨트롤러 중에서도 디플로이먼트를 주로 사용했다.
  - 컨트롤러 리소스 정의는 파드의 템플릿을 포함하고 컨트롤러 리소스는 이 템플릿을 사용하여 똑같은 파드의 레플리카를 여러 개 만들 수 있다.

=> 사실 디플로이먼트는 파드를 직접 관리하지 않는다. 레플리카셋의 역할이다.

디플로이먼트 -> 레플리카셋 -> 파드 -> 컨테이너

- 레플리카셋 역시 컨트롤러 객체이고, 레플리카셋이 관리하는 파드를 삭제하면 이를 대체하는 새로운 파드를 만든다.
  - 레플리카셋은 항상 **제어 루프**를 돌며 관리 중인 리소스 수와 필요한 리소스 수를 확인하기에 즉각 삭제된 파드를 대체할 수 있다.
  - 어플리케이션 스케일링도 마찬가지이다.
    - 제어 루프 중에 부족한 파드 수를 확인하고 템플릿에서 새로운 파드를 생성한다.

- 어떻게 어플리케이션에 스케일링이 이렇게 빨리 적용되고, 어떤 원리로 HTTP 응답이 다른 파드에서 올까?
  - 1. 단일 노드 클러스터 실습 환경에서 모든 파드가 하나의 노드에서 실행된다.
  - 해당 노드에는 이미 어플리케이션의 도커 이미지가 있다.
    - 운영 클러스터에서 스케일링을 지시하면, 이미지를 아직 내려받지 않은 노드에서 파드가 실행될 가능성이 크다.
    - 이 노드에서 파드를 실행하려면 이미지를 먼저 내려받아야 하고, 스케일링이 적용되는 데 걸리는 시간은 노드에서 이미지를 내려받는 시간보다 빠를 수 없다.
    - **이미지 최적화가 필요한 이유이다.**
  - 2. 똑같은 서비스 리소스를 향해 보낸 HTTP 요청이 서로 다른 파드에서 처리될 수 있는 이유는 서비스와 파드 간 느슨한 결합 때문이다.
    - 서비스의 레이블 셀렉터와 일치하는 파드 수가 늘어나고, 쿠버네티스는 이들 파드에 요청을 고르게 분배한다.

>> 레이블 셀렉터를 이용한 추상화
- 느슨한 결합 : 파드를 삭제하고 다시 띄워도 레이블만 같으면 Service와 연결 유지 가능
- 탄력적 확장 : 레플리카셋이 파드를 스케일 아웃해도 Service는 자동 연결
- 마치 DIP와 비슷하다고 이해함
  - 레이블이라는 인터페이스(추상화)에 의존하고 실제 구현체 (파드의 구체적인 정보)에 의존하지 않음.

즉, 필요한 만큼 파드를 실행하되 모두 같은 서비스 아래 연결한다. 그리고 서비스에 요청이 들어오면 서비스 아래의 파드에 고르게 요청을 분배해준다.
네트워킹과 컴퓨팅을 분리하는 추상화. 이것이 쿠버네티스 스케일링의 핵심이다.
클러스터 IP 주소가 파드의 IP 주소에 대한 추상이라고 설명했다. 파드가 교체되더라도 어플리케이션은 동일한 서비스 주소를 이용하여 새 파드에 접근할 수 있다. 이번에는 서비스가 여러 파드의 추상 역할을 한다. 클러스터 내 어떤 노드에서 요청이 오더라도 동일한 네트워크 계층에서 여러 파드에 고르게 트래픽을 분배할 수 있다.

클러스터 IP => 파드 IP의 추상화
- 외부에선 파드의 구체적인 정보를 알지 못한다. 서비스가 여러 파드의 추상 역할을 하여 여러 파드에 트래픽을 적절하게 분배.


## 6.2 디플로이먼트와 레플리카셋을 이용한 부하 스케일링
>> 레플리카셋을 이용하면 스케일링이 매우 쉬워진다.

- 디플로이먼트는 여러 개의 레플리카셋을 관리할 수 있다.
  - 디플로이먼트에서 파드 정의를 변경했다면 대체 레플리카셋을 생성한 후 기존 레플리카셋의 레플리카 수를 0으로 만든다.
- 디플로이먼트에서 레플리카 수를 조정하면, 레플리카셋의 레플리카 수가 그에 맞춰서 변경된다.
  - 새로운 레플리카셋이 완전히 준비된 후 기존 레플리카의 수가 감소한다.

- 외부 노출을 담당하는 컴포넌트는 로드 밸런서 서비스를 이용하는 프로시이다. (인입되는 트래픽을 주로 다루니 리버스 프록시이다.)
- 

## 6.3 데몬셋을 이용한 스케일링으로 고가용성 확보하기
- 데몬셋은 리눅스의 백그라운드에서 단일 인스턴스로 동작하며 시스템 관련 기능을 제공하는 프로세스인 데몬에서 따온 이름이다.
  - 각 노드에서 정보를 수집해서 중앙의 수집 모듈에 전달하거나 하는 인프라 수준의 관심사와 관련된 목적으로 많이 쓰인다.
  - 각 노드마다 파드가 하나씩 동작하면서 해당 노드에 대한 정보를 수집하는 것이다.
- 한 노드에 여러 레플리카를 둘 정도의 부하는 없지만 단순히 고가용성을 확보하려는 목적에서 데몬셋을 활용할 수 있다. 리버스 프록시가 좋은 예다.

## 6.4 쿠버네티스의 객체 간 오너십
- 컨트롤러 리소스는 레이블을 통해서 자신의 관리 대상 리소스를 결정한다. 
- 쿠버네티스에는 관리 주체가 사라진 객체를 찾아 제거하는 가비지 컬렉터가 있다. 파드는 레플리카셋의 관리를 받고, 레플리카셋은 다시 디플로이먼트의 관리를 받는다.
  - 리소스 간 의존 관계를 잘 관리하지만, 관계가 형성되는 수단은 레이블 셀렉터 뿐이다. 