---
title: "트랜잭션 전파"
author: vectorch9
date: 2023-10-28 00:18:00 +09:00
categories: [Spring, JPA]
tags: [Spring, JPA, DB]
pin: false
img_path: '/assets/img/posts/propagation/'
---

## 트랜잭션
트랜잭션은 `BEGIN`으로 시작해서 `COMMIT` 또는 `ROLLBACK`으로 끝난다. 입구는 한개지만 출구는 두개인 셈이다.

> 트랜잭션에 대한 개념은 다루지 않는다. 다른 글을 참조하길 바란다.

스프링은 `TransactionManager`라는 추상화된 트랜잭션 관리 인터페이스를 제공한다. 제공하는 메서드는 다음과 같다.
- `txManager.getTransaction()`
- `txManager.commit()`
- `txManager.rollback()`

커밋과 롤백은 메서드 이름에서부터 그 역할이 드러난다. 트랜잭션을 시작하는 메서드는 왜 `begin()`이 아닌 `getTransaction()`일까?

트랜잭션은 전파 속성과 고립 수준을 기준으로 트랜잭션을 새로 시작할 수도, 기존의 트랜잭션을 사용할 수도 있기 때문이다. 

오늘은 트랜잭션의 전파 속성에 대해 알아보자.

## 트랜잭션 전파
스프링의 트랜잭션은 **트랜잭션 전파, Transaction Propagation**라는 속성을 제공한다. 많이 사용하는 `@Transactional` 어노테이션(선언적 트랜잭션)에도 트랜잭션 전파 속성이 포함된다. 트랜잭션 전파는 이름그대로, 이미 진행중인 트랜잭션이 있을 때는 어떻게 전파할 지를 의미한다.

```java
 public @interface Transactional {
	// ... 그 외 속성
	Propagation propagation() default Propagation.REQUIRED;
}
```

`Propagation`은 Enum으로 선언되어 있으며 `REQUIRED, SUPPROTS, MANDATORY, REQUIRES_NEW, NOT_SUPPORTED, NEVER, NESTED`를 설정 가능하다.

구분을 위해 실제 DB에서 사용하는 트랜잭션 단위를 *물리 트랜잭션*, 스프링에서 사용하는 가상의 트랜잭션 단위를 *논리 트랜잭션*이라고 부른다. 하나의 물리 트랜잭션은 반드시 하나 이상의 논리 트랜잭션을 가져야하며 여러 논리 트랜잭션을 가질 수도 있다.

트랜잭션 전파의 기본값인 `REQUIRED`를 먼저 살펴보자.

### REQUIRED
`REQUIRED`는 2개의 논리 트랜잭션을 묶어 하나의 물리 트랜잭션에 포함한다. 두번째 트랜잭션은 첫번째 트랜잭션의 내부에 참여하는 형태가 된다. 논리 트랜잭션 커밋은 2번 발생하며, 내부 트랜잭션 커밋시엔 실제 커밋이 발생하지 않는며 외부 트랜잭션까지 커밋된 뒤에야 실제 커밋이 발생한다.

> 이름이 `REQUIRED`인 이유는 *트랜잭션이 필요하다*는 의미로 트랜잭션이 없다면 새로 만들기 때문이다. 이미 존재한다면 해당 트랜잭션에 참여한다. 말그대로 트랜잭션을 필요로만 하지 새로 요구하진 않기 때문이다.

아래 코드는 `AbstractPlatformTransactionManager`의 `processCommit()` 메서드의 일부다.

> `AbstractPlatformTransactionManager`는 `TransactionManager`의 자식인 추상클래스로, 각 플랫폼에 알맞는 트랜잭션 매니저에게 일종의 템플릿을 제공한다. 즉, 템플릿 메서드 패턴이 적용되어 있다. 트랜잭션 매니저는 [트랜잭션 매니저](/posts/Transactional) 문서를 참고하자.

```java
if (status.isNewTransaction()) {
    if (status.isDebug()) {  
       logger.debug("Initiating transaction commit");  
    }    unexpectedRollback = status.isGlobalRollbackOnly();  
    doCommit(status);  
}
```

`isNewTransaction()`은 아래와 같다.

```java
/**  
 * Return whether there is an actual transaction active. */
public boolean hasTransaction() {  
    return (this.transaction != null);  
}  
  
public boolean isNewTransaction() {  
    return (hasTransaction() && this.newTransaction);  
}
```

코드를 요약하면, 새 (논리)트랜잭션이거나 실제 트랜잭션을 소유한 경우에만 커밋이 진행된다. 

실제 트랜잭션을 보유 중인 조건은 이해하겠는데, 새 트랜잭션은 무슨 의미일까? 메서드 호출 시 사용되는 콜스택을 생각해보자. 가장 먼저 시작한 메서드는 가장 마지막에 종료된다. *새 트랜잭션*은 가장 먼저 시작된 논리 트랜잭션으로, 동시에 가장 마지막에 끝나는 논리 트랜잭션이다. 때문에, 새 트랜잭션이 끝날 땐 모든 논리 트랜잭션이 이미 끝난 상태이므로 물리 트랜잭션 커밋을 수행할 수 있다.

요약하면 참여중인 모든 논리 트랜잭션이 끝났을 때 물리 트랜잭션을 커밋시켜야하며, 새 트랜잭션이 끝났다는 것은 모든 논리 트랜잭션이 끝났다는 것과 동치이다.

롤백은 커밋과 거의 동일하게 처리된다. 다만 한가지 더 고려해야 할 사항이 있다. 커밋은 모든 트랜잭션이 커밋되었을 때 실제 커밋을 요청한다. 롤백은 만약 하나의 논리 트랜잭션이라도 롤백을 원하는 경우 물리 트랜잭션이 롤백되어야 한다. All와 Any의 관계라고 생각하면 된다. 스프링은 롤백의 Any 요구사항을 처리하기 위해 **글로벌 롤백 마킹**을 이용한다.
	
아래는 `AbstractPlatformTransactionManager`의 `processRollback()` 메서드 일부다.

```java
if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {  
    doSetRollbackOnly(status);  
}
```

`isGlobalRollbackOnParticipationFailure()`는 글로벌 롤백 마킹 설정으로 기본값이 `true`다. 예외 발생 시 `RollbackOnly` 마킹을 진행하겠다는 의미로, 실제 위 코드에서도 if문 통과 시 `doSetRollbackOnly()`를 통해 마킹을 진행한다.

이후 새 트랜잭션이 끝날 때 아래 코드에 의해 롤백된다.

```java
// processRollback()의 끝부분
if (unexpectedRollback) { // 롤백 마킹이 되어 있다면 예외를 던져 강제로 롤백시킨다.
    throw new UnexpectedRollbackException(  
          "Transaction rolled back because it has been marked as rollback-only");  
}
```

참여하는 논리 트랜잭션 중 하나라도 예외가 발생하여 롤백 마킹이 되었다면, 해당 물리 트랜잭션은 무조건 롤백된다. 그래서 이름이 `RollbackOnly`다. (커밋을 하지 않고 롤백만 가능하다.)

주의할점으로 위 `processRollback()` 메서드는 각 논리 트랜잭션 메서드가 끝날 때 수행된다. 아래의 코드를 살펴보자.

```java
public class ParentService {  
    @Transactional  
    public void parentMethod() {  
        memberRepository.save(new Member("member1"));  
        try {  
            childService.childMethod();  
        } catch (IllegalArgumentException e) {  
            // do nothing
        }  
        memberRepository.save(new Member("member3"));  
    } 
}

@Service  
public class ChildService {  
    @Transactional  
    public void childMethod() {  
        memberRepository.save(new Member("member2"));  
        throw new IllegalArgumentException();  
    }  
}
```

트랜잭션의 성질을 잘못 이해하였다면 `member1`과 `member3`는 정상적으로 저장될 것만 같다. 하지만 디버그 아웃풋은 아래의 예외를 출력하며 롤백된다.

```
2023-10-28T14:51:40.374+09:00 DEBUG 12832 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Failed to complete request: org.springframework.transaction.UnexpectedRollbackException: Transaction silently rolled back because it has been marked as rollback-only
```

`rollback-only`가 마킹되어 있기 때문에 롤백된다는 디버그 메시지다.

`childMethod()`가 예외로 인해 종료되는 순간 롤백 마킹이 진행되며, 부모 메서드에서 try-catch를 해주더라도 이미 마킹된 롤백은 사라지지 않는다. 그리고 위에서 살펴본 `processRollback()`메서드의 끝부분에 의해 `UnexpectedRollbackException`가 던져지며 물리 트랜잭션까지 롤백된다.

이 *롤백 마킹*을 잘 기억해두길 바란다.
### REQUIRES_NEW
두 개의 논리 트랜잭션에 두 개의 물리 트랜잭션을 각각 부여한다. 즉 내부 트랜잭션이 외부에 영향을 주지 않는다. 트랜잭션이 전파되지 않고 항상 새로 만드는 것이다.

트랜잭션이 전파되지 않는다는 *예외가 전파되지 않는다와 동일한 의미가 아니다*. 내부 트랜잭션 메서드에서 던져진 예외를 적절히 처리(try-catch)해주지 않는다면 이 예외는 외부 메서드까지 전해진다. 

내부의 예외에 의해 외부 메서드까지 롤백되는 셈이다. 두 개의 트랜잭션으로 분리되었지만 두 개의 트랜잭션이 모두 롤백되기 때문에 겉으로 보기엔 `REQUIRED`와 동일한 결과가 나타난다. 의도한대로 동작하길 원한다면 자식 메서드의 예외를 처리해주어야 한다.

다시 코드 예제를 보자.

```java
public class ParentService {  
    @Transactional  
    public void parentMethod() {  
        memberRepository.save(new Member("member1"));
		childService.childMethod();
        memberRepository.save(new Member("member3"));  
    } 
}

@Service  
public class ChildService {  
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void childMethod() {  
        memberRepository.save(new Member("member2"));  
        throw new IllegalArgumentException();
    }  
}
```

전파 속성에 의해 트랜잭션이 분리되어 있으므로 `member1`과 `member3`는 정상적으로 저장되어야 할 것만 같다. 하지만 롤백이 된다. 위에서 설명한대로 자식에서 던져진 `IllegalArgumentException`는 부모까지 영향을 주기 때문이다.

부모 메서드를 아래와 같이 수정해보자.

```java
@Transactional  
public void parentMethod() {  
	memberRepository.save(new Member("member1"));  
	try {  
		childService.childMethod();  
	} catch (IllegalArgumentException e) {  
		// do nothing
	}  
	memberRepository.save(new Member("member3"));  
} 
```

자식은 예외를 던지며 롤백되었으나, 부모에선 이를 try-catch를 통해 처리해주었다. 이제 자식의 예외는 부모 메서드에 영향을 주지 않는다. 

전파 속성을 통해 트랜잭션이 분리되었으며, 자식에서 던져진 예외도 처리해주었으므로 `member1`과 `member3`는 정상적으로 저장된다.

> `REQUIRED`일때 와의 차이는 무엇일까? `REQUIRED` 전파 속성일 땐 자식 메서드가 예외에 의해 끝나며 롤백 마킹이 진행되었다. 이미 마킹이 되었기 때문에 try-catch문을 통해 예외를 처리해주어도 롤백이 진행됐다. 하지만 `REQUIRES_NEW` 속성에선 롤백 마킹을 진행하지 않는다. 서로 다른 트랜잭션이기 때문이다.

```
MariaDB [test]> select * from members;
+----+---------+---------+
| id | name    | team_id |
+----+---------+---------+
| 22 | member1 |    NULL |
| 24 | member3 |    NULL |
+----+---------+---------+
```

실제 DB를 살펴보아도 정상적으로 저장되어 있다.

주의 사항으로, 내부 트랜잭션이 진행 중일 땐 외부 트랜잭션을 *대기, Suspend*시킨다. 물리 트랜잭션을 분리한다는 의미는 DB 커넥션을 하나 더 사용한단 의미로, 하나의 스레드에 여러 DB 커넥션이 할당된 상태로 대기시킬 수 있다. 커넥션이 빠르게 고갈될 수 있으므로 주의하자.
### 그 외 전파속성
그 외 전파 속성은 위 속성들을 이해했으면 쉽게 이해가능하다.

- SUPPORTS
  이미 트랜잭션이 있다면 참여하고, 없다면 트랜잭션 없이 실행한다.
- MANDATORY
  이미 트랜잭션이 있다면 참여하고 없다면 에러가 발생한다. 즉 내부 트랜잭션으로만 실행 가능하다.
- NOT_SUPPRORTED
  트랜잭션을 사용하지 않는다. 트랜잭션이 있다면 대기시킨다.
- NEVER
  트랜잭션을 사용하지 않는다. 트랜잭션이 있다면 에러가 발생한다.
- NESTED
  내부 중첩 트랜잭션을 생성한다. 내부 트랜잭션은 외부의 영향을 받으나, 외부 트랜잭션은 내부 트랜잭션에 영향을 받지않는다. 내부 트랜잭션이 롤백되어도 외부 트랜잭션 커밋이 가능하지만, 외부 트랜잭션이 롤백시엔 내부 트랜잭션도 롤백된다.

NESTED 옵션은 JDBC의 `savepoint`라는 기능을 이용하는데, JPA에선 사용 불가능하며 DB 드라이버마다 지원 여부가 다르다.
  
간략히 짚고넘어가자면, 일부 생략되었던 `processCommit()` 메서드는 아래와 같이 `savepoint`에 따른 분기도 작성되어 있다.

```java
if (status.hasSavepoint()) {  // - (1) 생략되었던 if절
    if (status.isDebug()) {  
       logger.debug("Releasing transaction savepoint");  
    }    unexpectedRollback = status.isGlobalRollbackOnly();  
    status.releaseHeldSavepoint();  
}
else if (status.isNewTransaction()) { // - (2) 아까 설명한 코드
    if (status.isDebug()) {  
       logger.debug("Initiating transaction commit");  
    }    unexpectedRollback = status.isGlobalRollbackOnly();  
    doCommit(status);  
}
```

## 마치며
DB의 락과 트랜잭션은 항상 조심히 다뤄야 한다. 버그가 발생하면 발견하기 까다로우며 서비스에 전체에 큰 영향을 줄 수 있기 때문이다.

스프링은 추상화를 통해 트랜잭션을 알아서 처리해준다. 개발할 땐 편하겠지만, 버그 발생 시엔 디버깅하기 더 어려워진다는 의미로 전파 속성과 같은 내부 원리도 잘 학습해두어야 한다.