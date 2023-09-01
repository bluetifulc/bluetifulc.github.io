---
title: DispatcherServlet 내부 처리 과정
author: vectorch9
date: 2023-09-01 13:00:00 +09:00
categories: [Spring]
tags: [Spring, Internal]
pin: false
img_path: '/assets/img/posts/dispatcher/'
---

스프링의 모든 요청 과정이 시작되는 `DispatcherServlet`에 대해 살펴보자.

![](logic.png)
전반적인 요청 흐름은 위와 같다.

> 필터는 스프링이 아닌 서블릿의 기술로, 이 글에선 다루지 않는다.

## DispatcherServlet
![](0.png)
`DispatcherServlet`은 이름에서 알 수 있듯이 `Servlet`으로, 요청 진입 시 `service()`가 호출된다.

> `service()`에서는 `ServletRequest`, `ServletResponse`를 `HttpServletRequest`, `HttpServletResponse`로 캐스팅한다. 이후 HTTP 메서드에 따라 `doGet()`, `doPost()`등의 메서드로 분기하는데, 각 메서드 내에선 큰 처리를 해주진 않고 `processRequest()`를 호출한다.

`processRequest()`내에선 요청와 관련된 여러 전처리 이후 `doService()`를 호출하게 된다.

부모인 `FrameworkServlet`에선 `doService()`를 추상 메서드로 선언하며, 실제 메서드 내용은 `DispatcherServlet`에서 작성된다. 즉, **템플릿 메서드 패턴**이 적용됐다.

요약하면 `service()` -> `do***()` -> `processRequest()` -> `doService()`의 호출과정을 거친다.

`doService()`내 에선 다시 이런저런 세팅을 해준 뒤 `doDispatch()`를 호출한다. 여기부턴 코드로 살펴보자.
### doDispatch()
![](1.png)
`doDispatch()`내에선  `getHandler()`를 통해 요청을 실제로 처리할 `HandlerMethod`를 찾으며 이에 대한 예외처리도 해주고 있다. 

>`HandlerMethod`는 실제로 요청을 처리해줄 핸들러(즉, `Controller`)의 메서드와 빈에 대한 정보를 갖고있다.

>❓잘보면 `doDispatch()`가 Deprecated되어있는데 이유를 잘 모르겠다. 아는 사람은 댓글로 이유를 달아주면 감사하겠다..

### getHandler()
![](1_5.png)
`getHandler()`에선 `HandlerMapping`인터페이스를 통해 요청한 엔드포인트에 맞는 핸들러를 찾는다. 대부분의 경우 `@RequestMapping`을 사용하기 때문에 `RequestMappingHandlerMapping` 구현체가 `HandlerMethod`를 찾아준다.

`HandlerExecutionChain`을 반환하는데, 내부적으로 요청을 처리해줄 핸들러(`HandlerMethod`)와 인터셉터를 갖고있다.

### getHandlerAdapter()
![](2.png)
`HandlerMethod`를 찾았다면 `getHandlerAdapter()`를 통해 이에 맞는 `HandlerAdapter`도 찾는다. **어댑터 패턴**이 적용되어 `HandlerMethod` 구현체에 따라 다를 방식(ex 파라매터, 리턴값)에 대한 중간 처리를 해준다.

> `applyPreHandle()`, `applyPostHandle()`를 통해 스프링의 **인터셉터**를 호출하는 모습도 보인다.

이후 `ha.handle()`을 통해 요청을 찾은 핸들러 어댑터에게 위임한다.

## HandlerAdapter
마찬가지로 대부분 `@RequestMapping`을 통해 요청을 처리하므로 이를 처리해주는 핸들러어댑터인 `RequestMappingHandlerAdapter`를 살펴보자.

`RequestMappingHandlerAdapter`에선 `handle()` -> `handleInternal()` -> `invokeHandlerMethod()`의 메서드 흐름을 가진다.

### invokeHandlerMethod()
![](3.png)
해당 메서드에선 정말 다양한 객체들을 주입, 설정해주고 있다. 잘 찾아보면 `ArgumentResolvers`를 주입하는 것이 눈에 띈다. `HandlerMethod`를 `ServletInvocableHanlderMethod`로 만드는데 이는 아래에서 설명하겠다.

![](4.png)
생략되었지만 위의 코드 외에도 많은 설정을 해준 뒤, 최종적으로 `invokeAndHandle()`이 호출된다. 이는 `ServletInvocableHandlerMethod`에 있는 메서드다.

## HandlerMethod
그럼 `ServletInvocableHandlerMethod`는 무엇일까?

![](5.png)


[공식 문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ServletInvocableHandlerMethod.html)에 따르면,
- `InvocableHandlerMethod`: `HandlerMethod`의 확장으로, 메서드 invoke할 때 argument 값을 resolve 해준다. 
- `ServletInvocableHandlerMethod`: 요청 위임 후 return value(반환 값)을 처리하며 `@ResponseStatus`에 대한 처리도 해준다.

### invokeAndHandle()
![](6.png)
`ServletInvocableHandlerMethod`의 `invokeAndHandle()` 코드다. 주요 로직만 살펴보자.

코드의 끝 부분을 살펴보면 `handlerReturnValue()`를 통해 리턴 값을 처리해준다. (공식문서와 설명이 일치한다.)
해당 메서드 내에선 `HandlerMethodReturnValueHandlerComposite`가 기본적으로 사용되며 이름대로 **컴포지트 패턴**이 적용되어있다.

> 많이 사용하는 `ResponseEntity`나 `HttpEntity`는 컴포지트를 거쳐 `HttpEntityMethodProcessor`에 의해 처리된다.

> `HttpEntityMethodProcessor`내 에선 `HttpMessageConverter`를 통해 원하는 형태의 응답으로 파싱한다. 흔히 말하는 잭슨 라이브러리(ObjectMapper)가 JSON 메시지컨버터에서 사용된다.

코드 윗 부분에서 `invokeForRequest()`를 호출하는데 이는 부모인 `InvocableHandlerMethod`에 있는 메서드다.
### invokeForRequest()
![](7.png)
`getMethodArgumentValues()`내 에서 앞서 주입된 `ArgumentResolver`를 통해 argument를 처리 해준 뒤 `doInvoke()`를 호출한다.(공식문서의 설명과 일치한다.)

`ArgumentResolver`의 구현체로 `HandlerMethodArgumentResolverComposite`를 호출하는데 마찬가지로 **컴포지트 패턴**이 적용되어 있다.
> 컴포지트를 거쳐 `RequestBodyArgumentResolver`나 `RequestParamArgumentResolver`와 같은 리졸버에 의해 처리된다. 이름이 매우 직관적이다.

> `@RequestParam`이나 `@ModelAttribute`와 같은 어노테이션을 처리하는 `ArgumentResolver`내에선 필요한 경우 `ConversionService`를 통해 타입 변환을 시도한다. 이는 브레이크 포인트를 찍어 디버깅하여 확인할 수 있다.

이후 `doInvoke()`는 우리가 작성한 `Controller`의 메서드에 요청을 위임한다. 드디어 HTTP 요청이 DispatcherServlet의 많은 과정을 거쳐 우리의 컨트롤러로 왔다.

## 반환 후
![](8.png)
이제 다시 `DispatcherServlet`으로 돌아왔다. 위 코드는 `doDispatch()`의 뒷부분이다. `processDispatchResult()` 메서드 내에선 이런저런 처리를 해준 뒤 **View**를 사용하는 경우, `render()`메서드를 호출한다. `render()`내에선 `LocaleResolver`, `ViewResolver`를 통해 뷰와 관련된 처리를 진행한다.

> `triggerAfterCompletion()`은 인터셉터를 호출한다.

이제 다시 사용자의 요청에 대한 모든 처리가 끝난 뒤 응답을 전송한다. 코드를 많이 생략했음에도 불구하고, 정말 많은 전처리가 발생한다. `DispatcherServlet`은 **프론트 컨트롤러**로, 이러한 전처리가 중복되지 않도록 제일 먼저 요청을 받아 공통적으로 처리해주는 역할을 가진다. 만약 `DispatcherServlet`가 없었다면 이러한 처리를 개발자가 직접 작성해야 할 것이다 (정말 끔찍하다..)

굉장히 많은 요청의 위임과 인터페이스가 등장한다. 스프링 답게 객체 지향과 다형성, 확장성에 대한 정말 많은 고민이 보인다.