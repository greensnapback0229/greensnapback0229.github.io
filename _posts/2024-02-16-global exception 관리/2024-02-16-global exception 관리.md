---
categories: [Spring, ExceptionHandle]
tags: [exception_handle]
---

### Question
프로젝트를 진행하면서 생각보다 내가 생각한 것 보다 많은 예외가 발생할 수 있는 상황들이 있을 수 있음을 체감했다. 아무것도 몰라서 `try-catch`만으론 한계를 느꼈다. 
Spring 프레임워크에서 제공하는 에러처리 기능들에 대해서 살펴보고 내 프로젝트에 적용 해보려고한다.

# 전역 예외 처리(Global Exception Handle)
모든 일에는 예외가 있다. 프로젝트 규모가 커질수록 발생할 수 있는 예외나 에러가 점점 많아지고 발생빈도도 증가한다. 이런 예외들을 줄이고 최대한 단순화시키기 위해서 사용하는게 **`Global Exception Handler`** 매커니즘이다.  
예외들을 한곳으로 모으고 관리하여 Client에 http 에러 코드를 반환하지 않고 정상 http status 정상 코드를 반환하고 커스텀한 예외처리 메세지를 보낸다. 이렇게 되면 Client에서의 처리를 도울 수 있고 서버에서도 어떤 부분이 잘못된지 정확히 명시할 수 있다. 

<!-- 스프링에서 크게 2가지 상황의 에러로 나눌 수 있는데 Controller단에서 발생하는 예외와 Service단 즉 비즈니스 로직상에서 발생하는 예외 2가지로 나누어서 살펴보려고 한다. 
# Controller Exceptioin 처리하기
-->

## `@ExceptionHandler`
`@Controller`, `@RestController`가 적용된 Bean내에서 발생하는 예외를 잡아서 하나의 메서드에서 처리해주는 기능을 한다.

```java
```

## `@ControllerAdvice`
@Controller로 선언한 모든 지점에서 발생한 에러 잡아 처리할 수 있도록 AOP를 이용한 기능이다. Controller 지점에서 에러나 예외가 발생하였을때 `@ControllerAdvice`를 선언한 클래스에서 `@ExceptionHandler`를 통해서 에러를 다루고 에러코드 및 메세지를 가공하여 Client에게 적절한 대응을 해줄 수 있다.





### API가 정상적으로 동작할때
1. 클라이언트는 데이터를 담아서 `@RequestBody`, `@RequstParam`, `@PathVariable` `Annotation`을 이용하여 API로 호출을 합니다.
2. Controller에서는 데이터를 이에 대한 처리를 수행합니다.
    1. ***데이터 처리가 정상적으로 처리된 경우***
        1. Controller 처리가 완료되면 비즈니스 로직 처리를 수행합니다.
        2. 비즈니스 로직 처리가 완료되면 Controller에서 클라이언트로 ‘성공 응답(Response)’ 데이터를 전송합니다.
3. 클라이언트와 API와의 통신이 완료되었습니다.




https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-exceptionhandler.html

https://adjh54.tistory.com/79#2.%20API%20%EB%B9%84%20%EC%A0%95%EC%83%81%20%EB%8F%99%EC%9E%91%20%EC%B2%98%EB%A6%AC%20%3A%20Exception%20%EB%B0%9C%EC%83%9D%20%EC%8B%9C-1

https://medium.com/@aedemirsen/spring-boot-global-exception-handler-842d7143cf2a