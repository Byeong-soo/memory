## 회원 도메인 설계의 문제점


![[Pasted image 20221218215818.png]]

- 다른 저장소로 변경할 때 OCP 원칙을 잘 준수할까?
- DIP를 잘 지키고 있을까?
- **의존관계가 인터페이스 뿐만 아니라 구현까지 모두 의존하는 문제점이 있음**


### 주문 도메인 전체 로직

  ![[Pasted image 20221219142651.png]]

- **역할과 구현을 분리** 해서 자유롭게 구현 객체를 조립할 수 있게 설계했다. 덕분에 회원 저장소는 물론이고, 할인 정책도 유연하게 변경할 수 있다.


## OCP DIP 규칙

![[Pasted image 20221219162902.png]]

![[Pasted image 20221219163922.png]]


- 클리이언트 코드인 OrderServiceImpl은 DiscountPolicy의 인터페이스 뿐만 아니라 구체 클래스도 함꼐 의존
- 그래서 구체 클래스를 변경할 때 클라이언트 코드도 함께 변경해야 한다.
- **==DIP** 위반!!!== -> 추상에만 의존하도록 변경(인터페이스에만 의존)
- DIP를 위반하지 않도록 인터페이스에만 의존하도록 의존관계를 변경하면 된다.


## AppConfig

![[Pasted image 20221219174658.png]]
- 애플리케이션의 전체 동작 방식을 구성(config) 하기 위해, 구현 객체를 생성하고, 연결하는 책임을 가지는 별도의 설정 클래스

![[Pasted image 20221219185337.png]]
- 중복을 제거하고, 역할에 따른 구현이 보이도록 리팩터링.
- 역할과 구현클래스가 한눈에 들어옴. 애플리케이션 전체 구성이 어떻게 되어있는지 빠르게 파악 할 수 있다.


## 사용,구성의 분리

![[Pasted image 20221219185453.png]]
- DiscountPolicy 로 변경하더라도 구성 영역만 영향을 받고, 사용 영역은 전혀 영향을 받지 않는다.