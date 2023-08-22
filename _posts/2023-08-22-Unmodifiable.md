---
title: 컬렉션의 Unmodifiable (List.of() 내부 로직)
author: vectorch9
date: 2023-08-22 16:00:00 +09:00
categories: [Java]
tags: [Java, Collection]
pin: false
img_path: '/assets/img/posts/unmodifiable/'
---

자바의 *Unmodifiable*는 무엇을 의미할까? 이는 불변과 유사한 개념으로 컬렉션 프레임워크에서 사용되는 용어다. 번역한 그대로, 특정 컬렉션의 추가, 삭제, 수정(replace)연산을 허용하지 않는다. 언뜻보면 불변과 같아 보이는데 어떤 차이가 있을까?

가장 많이 사용하는 컬렉션인 `List`를 예로 들면, `Collections.unmodifiableList(...)`나 `List.of(...)`사용 시 반환 되는 객체가 바로 Unmodifiable List다.

이 둘은 겉으로 보이는 기능은 동일하지만, 내부적인 구현에 차이가 있다.

`List` [문서](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/List.html#unmodifiable)의 설명에 따르면 Unmodifiable List는 변경(add, remove, replace)을 허용하지 않는 리스트다. 하지만 컬렉션에 포함되는 객체가 불변이 아니라면, 해당 객체가 변하지 않음을 보장하진 못한다. 
또한 원소로 `null`을 허용하지 않는다.
> 엄밀한 의미의 불변(Immutable)은 해당 객체내에 다른 객체를 포함한다면, 포함되는 모든 객체도 불변이어야 한다. 따라서 Immutable이 아닌 Unmodifiable라는 용어를 사용한 것이다. 실제로 문서내에 Immutable이라는 단어는 단 한번도 언급되지 않는다.

`Collections` [문서](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#unmodifiableCollection-java.util.Collection-)의 설명에 따르면 Unmodifiable Collection은 특정 컬렉션의 *Unmodifiable view*를 의미한다. 특정 컬렉션의 래퍼 클래스를 만들어 *read-only* View를 제공한다. 주의할 사항으로 단순히 View를 제공하기 때문에 내부 컬렉션이 변경되면 View의 내용도 변한다.

요약하면, Unmodifiable은 외부에서 변경하는 것을 방지하는 불변과 유사한 성질을 가진다.

실제 코드로 살펴보자.
### of 실제 구현
먼저 `List.of()`의 실제 구현을 살펴본다.
![](0.png)
위와 같이 `ImmutableCollections`의 `List12` 또는 `ListN`을 호출한다. (`List12`는 객체가 1,2개인 리스트를 `ListN`은 N개인 리스트를 의미한다. 아마 내부적인 최적화를 위해 나눠놓은듯 하다.)
![|500](1.png)
`AbstractImmutableList`를 포함하고 있으며 `@ValueBased` 어노테이션을 갖는다. `List12`기 때문에 하나 또는 두개의 원소만 가져 `E e0`와 `Object e1`을 갖는다. 
> 두번째 원소인 `e1`이 `Object`인 이유는 1개의 원소만 가지는 경우 `null`대신 미리 선언된 `EMPTY` 객체를 할당하기 위함이다.
> (`List.of()`는 자바9때 추가된 문법이라 `Optional`을 사용해도 됐을텐데 왜 안썼을까..?)

> Value based class에 대해선 [해당 문서](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/doc-files/ValueBased.html)를 참고하자.


> 엄밀한 의미의 Immutable은 아니지만 내부적 네이밍엔 Immutable을 사용한 모습이다.
> 이처럼 용어의 정확한 의미를 따지기보단 프로젝트의 컨벤션을 따르자. `List.of()`와 관련된 객체는 일관적으로 `Unmodifiable`이 아닌 `Immutable` 네이밍을 사용한다.

![](2.png)
`ListN`은 내부적으로 변하지 않음을 보장하기 때문에 일반 배열을 통해 관리된다. (원수의 수가 줄거나 늘지 않으므로 고정 길이인 배열을 사용한다.)

`List12`와 `ListN`의 부모인 `AbstractImmutableList`를 살펴보자.
![](3.png)
`List`를 구현하고 있으며, 변경과 관련된 메서드를 호출하면 모두 예외를 던진다.

### unmodifiableList 실제 구현
![](4.png)
리스트를 파라매터로 받아 (변경할 수 없는) 리스트의 **View**를 반환한다. 코드를 살펴보면 `RandomAccess`의 구현체인지 여부에 따라 다른 객체가 생성된다.

> `RandomAccess`는 마커 인터페이스로 최적의 속도로 임의접근이 가능한 자료구조를 의미한다.(즉 O(1)인지) 예를들어 `Vector`, `ArrayList`는 내부 구현이 배열이기 때문에 해당 속성을 가지며 `LinkedList`는 해당 속성을 가지지 않는다.

![](5.png)
원본 리스트가 `RandomAccess`인 경우엔, View는 `UnmodifiableList`를 상속하며 `RandomAccess` 마커를 갖는다. 이는 원본 객체가 `RandomAccess`기 때문에 당연한 결과다.
부모인 `UnmodifiableList`를 살펴보자.
![](6.png)
구조를 보면 알겠지만 단순한 **래퍼 클래스** 구조를 가진다. 리스트에 변경을 가하는 메서드를 호출하면 예외를 던진다.
다시 부모인 `UnmodifiableCollection`을 살펴보자.
![](7.png)
![](8.png)
`List`에 국한되지 않는 컬렉션의 공통 메서드들에 대해서도 예외처리를 해준다.

이처럼 `UnmodifiableList`는 단순한 래퍼 클래스이기 때문에 내부의 객체가 변하면 외부의 View도 변하게 되는 것이다. 실제로 아래의 테스트를 수행하면 성공한다.
![](9.png)

왜 `List.of`는 내부적으로 `Collections.unmodifiableList`를 이용하지 않을까? 추측해본 이유는 다음과 같다.

`List.of`는 각 원소, 배열을 파라매터로 받고 `unmodifiableList`는 파라매터로 리스트가 필요하다. `List.of`에서 `Collections.unmodifiableList`를 사용한다면, 배열을 리스트로 전환한 후 이를 래핑하는 과정이 필요하다. 즉 *배열 -> 리스트 -> Unmodifiable 리스트*라는 두번의 변환 과정이 필요하여 상당히 번거로워 자체 구현한 클래스를 사용하는 것이 아닐까?라고 생각한다.