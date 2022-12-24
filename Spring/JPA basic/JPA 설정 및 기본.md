기본편에서는 Maven 으로 강의 만듬.

의존성 추가 2가지
- JPA 하이버네이트
- H2 데이터베이스

![[Pasted image 20221224194120.png]]

- javax 는 자파 표준을 사용하기 때문에 공통 부분
- hibernate는 하이버네이트 전용 옵션.

## 데이터베이스 방언

~~~java
<property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>
~~~

- JPA는 ==특정 데이터베이스에 종속되지 않는다.==
- 각각의 데이터베이스가 제공하는 SQL 문법과 함수는 조금씩 다름
	- eg1) 가변문자 : MySQL은 VARCHAR, Oracle은 VARCHAR2
	- eg2) 문자열을 자르는 함수 : SQL 표준은 SUBSTRING(), Oracle은 SUBSTR()
	- eg3) 페이징 : MySQL은 LIMIT, Oracle은 ROWNUM
- 방언 : SQL 표즌을 지키지 않는 특정 데이터베이스만의 고유한 기능
- hibernate.dialect 속성에 지정
	- eg) H2 : org.hibernate.dialect.H2Dialect
	- eg) Oracle 10g : org.hibernate.dialect.Oracle10gDialect
	- eg) MySQL : org.hibernate.dialect.MySQL5InnoDBDialect
- 하이버네이트는 40가지 이상의 데이터베이스 방언 지원

![[Pasted image 20221224194711.png]]


## 객체와 테이블을 생성하고 매핑하기

![[Pasted image 20221224205021.png]]

- @Entity : JPA가 관리할 객체
- @id : 데이터베이스 pk와 매핑

## 회원 등록

![[Pasted image 20221224205151.png]]

### persist 만으로 저장 가능


## 회원 수정


![[Pasted image 20221224205259.png]]

- 회원 정보를 찾아서 set 으로 변경만 해주면 update 쿼리가 날아감


## 삭제
em.remove() 사용

## !! 주의
- 엔티티 매니저 팩토리(emf)는 하나만 생성해서 애플리케이션 전체에서 공유
- 엔티디 매니저는 쓰레드간에 공유하지 않는다.
- ==JPA의 모든 데이터 변경은 트랜잭션 안에서 실행==


## JPQL소개
- 조회의 가장 단순한 방법은 em.find()
- 하지만 조건, 나이가 18살 이상인 회원을 모두 검색하고 싶을땐??
~~~ java
List<Member> result = em.createQuery("select m from Member as m", Member.class).getresultList();
~~~
-  쿼리문에 where도 가능

- JPA를 사용하면 엔티디 객체를 중심으로 개발, 문제는 검색 쿼리
- 검색을 할 때도 테이블이 아닌 엔티티 객체를 대상으로 검색
- 모든 DB데이터를 객체로 변환해서 검색하는 것은 불가능.
- 애플리케이션이 필요한 데이터만 DB에서 불러오려면 결국 검색 조건이 포함된 SQL이 필요
- JPA은 SQL을 추상화한 JPQL이라는 객체 지향 쿼리 언어 제공
- SQL 과 문법이 유사하다. SELECT, FROM, WHERE, GROUP BY, HAVING, JOIN 지원
- ==JPQL은 엔티디 객체==를 대상으로 쿼리
- ==SQL은 데이터베이스 테이블==을 대상으로 쿼리
- 테이블이 아닌 ==객체를 대상으로 검색하는 객체지향 쿼리==
- SQL을 추상화해서 특정 데이터베이스 SQL에 의존하지 않음
- JPQL은 객체 지향 SQL