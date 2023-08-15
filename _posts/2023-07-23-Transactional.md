---
title: Transactional 내부 동작(AOP를 곁들인)
author: vectorch9
date: 2023-08-15 00:50:00 +09:00
categories: [Spring, JPA]
tags: [Transactional, JPA, Spring, Internal]
pin: false
img_path: '/assets/img/posts/transactional/'
---

## 문제 상황
현재 멀티모듈로 진행중인 프로젝트에서 `application` 모듈은 최소한의 의존성을 갖는 규칙이 있다. `application` 모듈에선 JPA에 대한 의존성 없이 순수한 비지니스 로직을 작성한 서비스만 존재한다. `@Transactional`은 주로 서비스의 클래스, 메서드에 작성되는데 JPA에 대한 의존성이 없기 때문에 사용이 불가능했다.

따라서 커스텀 `@Transactional`을 작성할까?라는 고민을 하게되었고 그 과정에서 발생한 코드 탐구를 기록한다. 스프링의 `@Transactional`은 기본적으로 AOP를 통해서 처리되기 때문에, 실제로 적용된 AOP에 대해서도 배울 수 있다.

>이 글에선 트랜잭션, AOP에 대한 기본 개념은 설명하지 않는다.

## 코드 탐구
![](code0.png)
먼저 `@EnableTransactionManagement` 어노테이션에 대해 살펴보자. 해당 어노테이션은 트랜잭션 관련 기능을 활성화하기 위해 필요하다. `mode`의 기본값이 `PROXY`인걸 기억하자. `@Import`를 통해 로드하는 `TransactionManagermentConfigurationSelect`를 살펴보자.

> 스프링부트에선 해당 어노테이션을 추가하지 않아도 자동으로 활성화된다.

![](code1.png)
앞서 설정한 `mode`값을 기준으로 설정을 진행한다. 기본값인 `PROXY`인 경우에 호출하는 Configuration의 내용은 다음과 같다.

![](code2.png)
총 3가지 빈을 등록한다.

`TransactionAttributeSource`로 등록되는 `AnnotationTransactionAttributeSource`에 대해 먼저 살펴보자.

![](code3.png)
`AnnotationTransactionAttributeSource`는 `@Transactional` 어노테이션에 할당된 정보들을 파싱하는 역할을 한다. `org.springframework.transaction.annotation.@Transactional`을 처리하기 위한 파서가 기본적으로 등록되며 의존성 여부에 따라 그 외 어노테이션(ex. `jakarta.transaction.@Transactional`)을 처리하기 위한 파서도 등록된다.

모든 파서의 파싱 결과는 `TransactionAttribute`로 동일하다. 
> 스프링 빈 등록시, 어떤 방식(e.x. xml, 어노테이션)으로 빈을 등록하든 결과적으로  `BeanDefinition`이 반환되는데 이와 비슷한 원리다.

다음은 AOP의 핵심인 어드바이저에 대해 살펴본다.
![](code4.png)
어드바이스로 `TransactionInterceptor`를 등록하는데 이는 밑에서 설명한다. 자세히 살펴보면 포인트컷은 설정하지 않고 `TransactionAttributeSource`만 주입해주는 모습이다.

![](code5.png)
이는 설정한 `Advisor` 클래스가 내부적으로 포인트컷을 직접 선언하기 때문이다. 해당 클래스는 어떻게 어드바이스 적용 대상을 판별할까?

![](code6.png)
`TransactionAttributeSourcePointcut`은 위에서 주입된 `TransactionAttributeSource`를 통해 매칭 여부를 판단한다. 파싱 결과가 `null`인지를 검사하는데, 파서 내부에서 조건에 맞는 어노테이션이 없다면 반환 값이 `null`이 되기 때문이다.

### 어드바이스
이제 실제로 `@Transactional`을 처리하는 어드바이스, `TransactionInterceptor`에 대해 알아보자.

![](code7.png)
AOP를 사용하기 때문에 `Advice`의 자식인 `MethodInterceptor`를 구현하고, `TransactionAspectSupport`를 상속한다. `MethodInterceptor`를 구현한 인터셉터기 때문에 `invoke()` 메서드가 있음을 유추할 수 있다. 해당 메서드를 살펴보자.

![](code8.png)
`invokeWithinTransaction()`을 호출하는데 이는 부모인 `TransactionAspectSupport`에 선언된 메서드다.

`invokeWithinTransaction()`는 상당히 긴 메서드로 일부만 살펴본다.

![](code9.png)
`TransactionAttributeSource`를 통해 어노테이션 정보를 파싱하고 `TransactionAttribute`를 받아온다. 이후 사용할 `TransactionManager`에 대한 참조도 받아온다.

![](code10.png)
이후 우리가 AOP와 트랜잭션에 아는 내용대로 흘러간다. 트랜잭션을 만든다. -> `invocation`을 통해 실제 메서드를 호출한다. -> 결과에 따라 `completeTransactionAfterThrowing` 또는 `commitTransactionAfterReturning`을 호출한다. 각 메서드는 내부에 롤백, 커밋하는 메서드가 각각 들어있다.

실제 DB에 트랜잭션 관련 요청, 응답을 송수신하는 코드는 `TransactionManager`를 통해 처리된다. 상황에 맞는 여러 구현체가 선언되어 있으며 원하는 경우 직접 커스터마이징도 가능하다. 예를들면, JDBC의 경우엔 `DataSourceTransactionManager`를 통해 실제 트랜잭션 처리를 하게 된다. H2 인메모리 DB의 경우엔 `HibernateTransactionManager` 구현체를 사용한다.

> 트랜잭션 매니저에 대한 더 자세한 글은 [다른 블로그](https://dhsim86.github.io/web/2017/11/04/spring_custom_transactionmanager-post.html)를 참조하자.

### 트랜잭션 매니저
위의 `ProxyTransactionManagementConfiguration`를 다시 살펴보자.
![](code11.png)
에서 `interceptor.setTransactionManager(this.txManager);`를 찾을 수 있다. `this.txManager`는 부모인 `AbstractTransactionManagementConfiguration`에서 선언됐다.

![](code12.png)
위 메서드가 실행되며 결과적으로 `txManager`가 설정된다. `TransactionManagermentConfigurer`는 이름 그대로 `TransactionManager`를 등록하기 위한 설정으로, 커스텀 `TransactionManager`를 등록하기 위해선 해당 인터페이스를 상속받아 사용하면 된다. 스프링부트는 기본 설정으로 위에서 살펴본 트랜잭션 매니저를 등록해준다.

![](code13.png)
디버거를 통해 살펴보면 실제로 `@Transactional` 메서드 호출 시 해당 인터셉터를 타고 들어온다. 현재 MySQL과 Jpa를 사용하고 있기 때문에 트랜잭션 매니저는 `JpaTransactionManager`구현체가 사용되는 모습이다.

## 결론
이렇게 스프링의 `@Transactional`이 어떻게 AOP를 통해 호출되는지 요청 흐름을 살펴보았다. 모르는 사이에도 가장 많이 사용하고 있었을 어드바이저의 내부 원리에 대해 알게되었다. 코드를 살펴보는 과정에서 역시 스프링답게 다형성을 고려한 설계가 돋보였다. 글에서 언급하진 않았지만 관련된 인터페이스, 추상클래스가 정말정말 많다.

글 초반에 언급하였듯 커스텀 `@Transactional`을 적용하기 위해 공부한 과정이었지만, 다른 방법을 택하기로 결정하였다. 단순히 `@Transactional`만 구현하고 끝날 것이 아닌 관련된 `TransactionAttributeSource`까지 구현하고 등록해야하기 때문이다. 해당 프로젝트에선 트랜잭션을 위한 **최소한의 의존성**인 `spring-tx`만 추가하도록 결정했다. "순수한 애플리케이션 모듈"에는 맞지 않지만 실용성과의 타협이 필요했다. 클린아키텍처를 위해 실용성을 포기하는 것은 주객전도라고 생각한다. (물론 최대한 지키는 것이 좋지만, 언제나 그렇듯 **적절히**가 중요하다고 생각한다.)