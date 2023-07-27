---
title: Getting Started
author: cotes
date: 2023-07-23 20:55:00 +09:00
categories: [Spring, DB]
tags: [getting started]
pin: true
img_path: '/assets/img/posts/transactional/'
---

진행중인 토이 프로젝트에서 의존성 문제로 `@Transactional` 어노테이션을 직접 사용할 수 없는 상황이라 커스텀 어노테이션을 통해 처리할까 고민 중 이다. 커스터마이징을 위해선 기본 로직에 대한 이해도가 있어야한다. 실제로 적용된 AOP도 공부할 겸, 직접 코드를 살펴보기로 했다.

> `@Transactional`은 스프링 내부에서 AOP를 통해 처리된다. 이 글에선 트랜잭션, AOP에 대한 기본 개념은 설명하지 않는다.

![](code1.png)
먼저 AOP기 때문에 어드바이저가 등록되는 Config을 살펴본다. 해당 Config은 `@EnableTransactionManagement`가 있을 경우 로드되며, `@Transactional`을 프록시 방식으로 처리하기 위해 사용할 빈들을 등록한다.

> `@EnableTransactionManagement`는 스프링부트의 경우 따로 추가할 필요없다.

![](code2.png)
총 3가지 빈을 등록한다.

`TransactionAttributeSource`로 등록되는 `AnnotationTransactionAttributeSource`에 대해 먼저 살펴보자.

![](code3.png)
`AnnotationTransactionAttributeSource`는 `@Transactional` 어노테이션에 할당된 정보들을 파싱하는 역할을 한다. `org.springframework.transaction.annotation.@Transactional`을 처리하기 위한 파서가 기본적으로 등록되며 의존성 여부에 따라 그 외 어노테이션(ex. `jakarta.transaction.@Transactional`)을 처리하기 위한 파서도 등록된다.

모든 파서의 파싱 결과는 `TransactionAttribute`로 동일하다. 
> 스프링 빈 등록시, 어떤 방식(e.x. xml, 어노테이션)으로 빈을 등록하든 결과적으로  `BeanDefinition`이 반환되는데 이와 같은 원리다.

다음은 AOP의 핵심인 어드바이저에 대해 살펴본다.
![](code4.png)
어드바이스로 `TransactionInterceptor`를 등록하는데 이는 밑에서 설명한다. 자세히 살펴보면 포인트컷은 설정하지 않고 `TransactionAttributeSource`만 주입해주는 모습이다.

![](code5.png)
해당 클래스는 내부적으로 포인트컷을 직접 선언하기 때문이다. 해당 클래스는 어떻게 어드바이스 적용 대상을 판별할까?

![](code6.png)
`TransactionAttributeSourcePointcut`은 위에서 주입된 `TransactionAttributeSource`를 통해 매칭 여부를 판단한다. 파싱 결과가 `null`인지를 검사하는데, 파서 내부에서 조건에 맞는 어노테이션이 없다면 반환 값이 `null`이 되기 때문이다.

---
이제 본격적으로 `@Transactional`을 처리하는 어드바이스, `TransactionInterceptor`에 대해 알아보자.

![](code7.png)
AOP를 사용하기 때문에 `Advice`의 자식인 `MethodInterceptor`를 구현하고, `TransactionAspectSupport`를 상속한다. `MethodInterceptor`를 구현하기 때문에 `invoke()` 메서드가 있음을 유추할 수 있다. 해당 메서드를 살펴보자.

![](code8.png)
`invokeWithinTransaction()`을 호출하는데 이는 부모인 `TransactionAspectSupport`에 선언된 메서드다.

`invokeWithinTransaction()`는 상당히 긴 메서드기 때문에 일부만 설명한다.

![](code9.png)
`TransactionAttributeSource`를 통해 어노테이션 정보를 파싱하고 `TransactionAttribute`를 받아온다. 이후 사용할 `TransactionManager`에 대한 참조도 받아온다.

![](code10.png)
이후 우리가 AOP와 트랜잭션에 아는 내용대로 흘러간다. 트랜잭션을 만든다. -> `invocation`을 통해 실제 메서드를 호출한다. -> 결과에 따라 `completeTransactionAfterThrowing` 또는 `commitTransactionAfterReturning`을 호출한다. 각 메서드는 내부에 롤백, 커밋하는 메서드가 각각 들어있다.

실제 DB에 트랜잭션 관련 요청, 응답을 송수신하는 코드는 `TransactionManager`를 통해 처리된다. 상황에 맞는 여러 구현체가 선언되어 있으며 원하는 경우 직접 커스터마이징도 가능하다. 예를들면, JDBC의 경우엔 `DataSourceTransactionManager`를 통해 실제 트랜잭션 처리를 하게 된다. H2 인메모리 DB의 경우엔 `HibernateTransactionManager` 구현체를 사용한다.

> 트랜잭션 매니저에 대한 더 자세한 글은 [다른 블로그](https://dhsim86.github.io/web/2017/11/04/spring_custom_transactionmanager-post.html)를 참조하자.

위의 `ProxyTransactionManagementConfiguration`를 다시 살펴보면
```java
@Bean  
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)  
public TransactionInterceptor transactionInterceptor(TransactionAttributeSource transactionAttributeSource) {  
	TransactionInterceptor interceptor = new TransactionInterceptor();  
	interceptor.setTransactionAttributeSource(transactionAttributeSource);  
	if (this.txManager != null) {  
	interceptor.setTransactionManager(this.txManager);  
	}  
	return interceptor;  
}
```
에서 `interceptor.setTransactionManager(this.txManager);`를 찾을 수 있다. `this.txManager`는 `ProxyTransactionManagementConfiguration`의 부모인 `AbstractTransactionManagementConfiguration`에서 선언된다.

![](code11.png)
위 메서드가 실행되며 맨밑에서 `txManager`가 설정된다. `TransactionManagermentConfigurer`는 이름 그대로 `TransactionManager`를 등록하기 위한 설정으로 커스텀 `TransactionManager`를 등록하기 위해선 해당 인터페이스를 상속받아 사용하면 된다. 스프링부트는 기본 설정으로 위에서 살펴본 트랜잭션 매니저를 등록해준다.

이렇게 스프링의 `@Transactional`이 어떻게 AOP를 통해 호출되는지 요청 흐름을 살펴보았다. 

> 글 초반에 언급하였듯 커스텀 `@Transactional`을 적용하기 위해 공부한 과정이었지만, 다른 방법을 택하기로 결정하였다. 이유는 `AnnotationTransactionAttributeSource`내에서 어노테이션을 실제로 파싱하는 파서때문인데, 직접 구현하기엔 로직이 너무 많았고 코드를 그대로 복사하기엔 좋은 방식이 아니라고 생각했다.