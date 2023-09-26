---
title: Repository 개념과 Repository 인터페이스
author: vectorch9
date: 2023-09-27 00:05:00 +09:00
categories: [Spring, JPA]
tags: [Spring, JPA, DDD]
pin: false
img_path: '/assets/img/posts/repository/'
---
## Repository란

먼저 Repository란 뭘까? `@Repository`의 [공식 문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/stereotype/Repository.html)를 찾아보자.

> Indicates that an annotated class is a "Repository", originally defined by Domain-Driven Design (Evans, 2003) as "a mechanism for encapsulating storage, retrieval, and search behavior which emulates a collection of objects".
> 
> Teams implementing traditional Jakarta EE patterns such as "Data Access Object" may also apply this stereotype to DAO classes, though care should be taken to understand the distinction between Data Access Object and DDD-style repositories before doing so. This annotation is a general-purpose stereotype and individual teams may narrow their semantics and use as appropriate.
> 
> A class thus annotated is eligible for Spring [`DataAccessException`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/dao/DataAccessException.html "class in org.springframework.dao") translation when used in conjunction with a [`PersistenceExceptionTranslationPostProcessor`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/dao/annotation/PersistenceExceptionTranslationPostProcessor.html "class in org.springframework.dao.annotation"). The annotated class is also clarified as to its role in the overall application architecture for the purpose of tooling, aspects, etc.
> 
> As of Spring 2.5, this annotation also serves as a specialization of [`@Component`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/stereotype/Component.html "annotation interface in org.springframework.stereotype"), allowing for implementation classes to be autodetected through classpath scanning.

Repository란 DDD에서 나온 개념으로, 객체의 모음(Collection)을 에뮬레이트(포현)하여 저장, 획득, 검색하는 행동을 지원하는 메커니즘을 의미한다.

DAO를 구현한 자바 EE팀은 해당 어노테이션을 DAO 클래스에 적용하였지만, DAO와 DDD의 Repository는 차이가 있으며 이를 잘 이해하여야 한다. `@Repository`는 general-purpose하며 각 팀내에서 의미를 축소하고 적절히 사용하여야 한다.

`PersistenceExceptionTranslationPostProcessor`와 결합된 경우엔 관련된 예외가 `DataAccessException`으로 번역되어 처리된다.

> `PersistenceExceptionTranslationPostProcessor`는 빈 후처리기로 `@Repository`가 붙은 클래스에 `PersistenceExceptionTranslationAdvisor`를 어드바이저로 등록한다. 해당 어드바이저는 `PersistenceExceptionTranslationInterceptor`를 어드바이스로 가진다.(포인트컷은 당연히 어노테이션을 이용한다.) 어드바이스는 발생하는 예외를 `DataAccessException`으로 번역해준다. 번역 과정엔 `PersistenceExceptionTranslator`가 이용된다.

스프링 2.5부턴 `@Component`의 특수한 형태로, 컴포넌트 스캔 시 자동으로 감지되어 빈으로 등록된다.

> 참고로 `@Controller`, `@Service`는 단순히 `@Component`의 특수한 형태로 DDD의 컨트롤러, 서비스를 **표시**하는 의미만 가진다. `@Repository`처럼 특수한 로직이 추가되진 않으나, 사람이 구분하기 쉬우며 나중에 커스텀 어드바이저를 등록하는 것에 활용될 수 있다.

## Repository VS DAO
문서에도 언급되어있는, Repository와 DAO의 차이점은 무엇일까?

DAO는 Data Access Object로 이름그대로 **데이터 영속성 계층**을 추상화한다. 반면 Repository는 위에서 설명하였듯이 **객체의 모음**을 추상화한다. DAO는 더 Low-level인 DB에 가까운 기술이며, Repository는 상대적으로 High-level인 도메인에 인접한 개념이다.

Repository는 상대적으로 더 추상적인 개념으로, 구현 시 DAO를 사용할 수 있다. 물론 Repository는 DB에 국한되지 않는 개념으로 DAO가 아닌 다른 방식(ex. 인메모리, 자바컬렉션, Redis..)을 택할 수도 있다.

> [Stack Overflow](https://stackoverflow.com/questions/8550124/what-is-the-difference-between-dao-and-repository-patterns)에서 잘 설명해준다.

코드를 통해 보면 아래와 같이 작성할 수 있을 것이다.

```java
class UserDAO {

	// DB와 관련된 필드...
	
	public User save(User user) {
		final String sql = "INSERT INTO users(...) VALUES(...)";
		// 이후 DB와의 연결 및 쿼리 작업...
	}
}

interface UserRepository {
	User create(User user);
}

class UserRepositoryDBImpl implements UserRepository {

	// 주입 받았다고 가정
	private final UserDAO userDAO;
	
	@Override
	public User create(User user){
		userDAO.save(user);
	}
}
```

위는 DAO와 Repository를 모두 사용한 방식이다. Repository의 구현 내부에 DAO가 이용된다. Repository는 DB에 국한된 개념이 아니므로 다른 방식으로 구현 할 수도 있다. 예를들어 자바 컬렉션을 통해 구현하려면 아래와 같은 코드를 작성한다.

```java
class UserRepositoryInMemoryImpl implements UserRepository {

	// 주입 받았다고 가정
	private final List<User> users = ArrayList<>();
	
	@Override
	public User create(User user) {
		users.add(user);
	}
}
```

이 처럼 Repository는 데이터의 집합을 의미하므로 실제로 어떻게 저장할지는 내부구현에 맡기게 된다. DB를 사용하지만 DAO 계층을 굳이 나누기 싫다면 아래와 같이 구현할 수도 있을 것이다.

```java
class UserRepositoryDBImpl implements UserRepository {

	// DB와 관련된 필드...
	
	@Override
	public User create(User user) {
		final String sql = "INSERT INTO users(...) VALUES(...)";
		// 이후 DB와의 연결 및 쿼리 작업...
	}
}
```

## 스프링의 Repository
Spring Data 프로젝트는 단순한 마커 인터페이스인 `Repository<T, ID>` 를 시작으로 `CrudRepository<T,ID>`와 같은 다양한 Repository 관련 인터페이스가 존재한다.

실제 CRUD 관련 메서드가 있는 `CrudRepository`의 [문서](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html)를 살펴보자.
![](1.png)
Repository기 때문에 DB와 관련된 언급은 일절 없으며, 여러 CRUD 메서드에 대한 스펙만 정의한다.

`JpaRepository<T,ID>`는 Spring Data JPA 프로젝트에서 지원하는 인터페이스다. (앞선 Spring Data를 포함하지만 **다른** 프로젝트다!) JPA와 관련된 기능이 추가되어, `flush()`, `deleteAllInBatch()`와 같이 실제 영속성 컨텍스트와 관련된 메서드도 정의되어 있다. 

> 이는 엄밀한 DDD의 Repository 개념엔 맞지 않으나, `Repository`를 상속하는 모든 인터페이스가 `*Repository`이므로 네이밍 컨벤션 상 `JpaRepository`라는 이름이 된 것이 아닐까 생각한다.

## Spring Data JPA의 Repository
스프링을 써봤다면 알듯이, `Repository` (자식포함)을 상속하면 `@Repository`를 붙이지 않아도 정상적으로 실행된다. 이는 **컴포넌트 스캔** 없이도 누군가 해당 인터페이스를 구현하고 빈으로 등록해준다는 의미이다. JPA에서 `Repository` 인터페이스는 누가 구현해줄까?

![](2.png)
[공식문서](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.create-instances)에 따르면, `@EnableJpaRepositoires`를 통해 `Repository` 구현을 활성화한다.
내부적으론 `@EnableJpaRepositoires`를 붙여주면 `JpaRepositoriesRegistar`가 로드되는데, 이 `JpaRepositoriesRegistar`가 실제로 Repsotiory의 구현체를 만들어주는 역할을 한다.

> 스프링부트의 경우 자동 설정이 있어 해당 어노테이션 없이도 동작한다.

❗`JpaRepositoriesRegistar`의 내부 로직도 파악해보려 했으나, 관련 자료도 너무 적고 요청 위임의 연속이라 파악할 수 없었다. 정리된 문서를 찾게되면 업데이트 하겠다.

### NoRepositoryBean
위의 기준대로 `Repository` 인터페이스의 자식 인터페이스가 모두 구현체를 가진다면 문제가 생긴다. 예를들어 `CrudRepository`가 구현체를 가질 수 있을까? `CrudRepository`는 `<T, ID>`인 제네릭 타입을 가지며 공통 부분을 묶어 추상화한 인터페이스다. 개발자는 작성한 최종 인터페이스가 구현되길 원하지 이러한 중간 단계 인터페이스가 구현되길 원하진 않는다.

스프링은 이를 해결하기 위해 `@NoRepositoryBean`이라는 어노테이션을 사용한다. (잘보면 위의 `CrudRepository` 문서 위에도 붙어있다.)
![](3.png)
[공식 문서](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/NoRepositoryBean.html)에 따르면

> 지정된 `Repository` 인터페이스가 선택되어 인스턴스(구현체)로 만들어지는 것을 막는다.
> 기본 `Repository`를 확장하는 중간단계(intermediate)의 인터페이스가 구현되어 빈으로 등록되는 것을 막기위해 사용할 수 있다.

요약하면, *해당 어노테이션이 붙은 인터페이스를 구현체로 만들지 말라*는 의미를 가진다.

## 마치며
Repository의 개념과 JPA와 관련된 사항들을 간단히 알아보았다. `@NoRepositoryBean`이라는 어노테이션을 발견한 후 이 어노테이션은 뭐지?라는 생각에서 시작한 글이다. `@NoRepositoryBean`를 알아보던 중 Repository에 대한 정확한 정의가 무엇일까라는 생각이 들어 글이 더 길어졌다.
