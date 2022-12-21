
![[Pasted image 20221220123314.png]]

### BeanFactory

- 스프링 컨테이너의 최상위 인터페이스이다.
- 스프링 빈을 관리하고 조회하는 역할을 담당한다.
- getBean() 을 제공한다.

### ApplicationContext

- BeanFactory 기능을 모두 상속받아 제공
- 애플리케이션을 개발할 떄는 빈은 관리하고 조회하는 기능은 기본이고, 많은 부가기능이 필요
- ApplicationContext 는 빈 관리기능 + 편리한 부가 기능을 제공한다.
- BeanFactory를 직접 사용하는 일은 X . 대부분 부가기능이 포함된 ApplicationContext를 사용
- BeanFactory나 ApplicationContext를 스프링 컨테이너라고 한다.


![[Pasted image 20221220123510.png]]

### ApplicationContext 가 제공하는 부가 기능

- ### 메시지 소스를 활용한 국제화 기능
	- 한국에서 들어오면 한국어, 영어권에서 들어오면 영어로 출력
- ### 환경변수
	- 로컬, 개발, 운영등을 구분해서 처리
- ### 애플리케이션 이벤트
	- 이벤트를 발행하고 구독하는 모델을 편리하게 지원
- ### 편리한 리소스 조회
	- 파일, 클래스패스, 외부 등에서 리소스를 편리하게 조회


 