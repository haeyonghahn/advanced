# advanced
## 목차
* **[예제 만들기](#예제-만들기)**
  * **[프로젝트 생성](#프로젝트-생성)**
  * **[예제 프로젝트 만들기 - VO](#예제-프로젝트-만들기---vo)**
  * **[로그 추적기 - 요구사항 분석](#로그-추적기---요구사항-분석)**
  * **[로그 추적기 V1 - 프로토타입 개발](#로그-추적기-v1---프로토타입-개발)**
  * **[로그 추적기 V1 - 적용](#로그-추적기-v1---적용)**
  * **[로그 추적기 V2 - 파라미터로 동기화 개발](#로그-추적기-v2---파라미터로-동기화-개발)**
  * **[로그 추적기 V2 - 적용](#로그-추적기-v2---적용)**
  * **[정리](#정리)**
* **[쓰레드 로컬 - ThreadLocal](#쓰레드-로컬---threadlocal)**
  * **[필드 동기화 - 개발](#필드-동기화---개발)**
  * **[필드 동기화 - 적용](#필드-동기화---적용)**
  * **[필드 동기화 - 동시성 문제](#필드-동기화---동시성-문제)**
  * **[동시성 문제 - 예제 코드](#동시성-문제---예제-코드)**
  * **[ThreadLocal - 소개](#threadlocal---소개)**
  * **[ThreadLocal - 예제 코드](#threadlocal---예제-코드)**
  * **[쓰레드 로컬 동기화 - 개발](#쓰레드-로컬-동기화---개발)**
  * **[쓰레드 로컬 동기화 - 적용](#쓰레드-로컬-동기화---적용)**
* **[템플릿 메서드 패턴과 콜백 패턴](템플릿-메서드-패턴과-콜백-패턴)**
  * **[템플릿 메서드 패턴 - 시작](#템플릿-메서드-패턴---시작)**
  * **[템플릿 메서드 패턴 - 예제2](#템플릿-메서드-패턴---예제2)**
  * **[템플릿 메서드 패턴 - 예제3](#템플릿-메서드-패턴---예제3)**
  * **[템플릿 메서드 패턴 - 적용1](#템플릿-메서드-패턴---적용1)**
  * **[템플릿 메서드 패턴 - 적용2](#템플릿-메서드-패턴---적용2)**
  * **[템플릿 메서드 패턴 - 정의](#템플릿-메서드-패턴---정의)**
  * **[전략 패턴 - 시작](#전략-패턴---시작)**
  * **[전략 패턴 - 예제1](#전략-패턴---예제1)**
  * **[전략 패턴 - 예제2](#전략-패턴---예제2)**
  * **[전략 패턴 - 예제3](#전략-패턴---예제3)**
  * **[템플릿 콜백 패턴 - 시작](#템플릿-콜백-패턴---시작)**
  * **[템플릿 콜백 패턴 - 적용](#템플릿-콜백-패턴---적용)**

## 예제 만들기
### 프로젝트 생성
__build.gradle__   
```gradle
plugins {
    id 'org.springframework.boot' version '2.7.1'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

group 'hello'
version '1.0-SNAPSHOT'
sourceCompatibility = '11'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

test {
    useJUnitPlatform()
}
```
- 동작 확인
  - 기본 메인 클래스 실행(`AdvancedApplication`)
  - http://localhost:8080 호출해서 Whitelabel Error Page가 나오면 정상 동작
  
### 예제 프로젝트 만들기 - VO
상품을 주문하는 프로세스로 가정하고, 일반적인 웹 애플리케이션에서 Controller -> Service -> Repository로 이어지는 흐름을 최대한 단순하게 만들어보자.   

__OrderRepositoryV0__
```java
@Repository
@RequiredArgsConstructor
public class OrderRepositoryV0 {

    public void save(String itemId) {
        //저장 로직
        if(itemId.equals("ex")) {
            throw new IllegalStateException("예외 발생!");
        }
        sleep(1000);
    }

    private void sleep(int millis) {
        try {
            Thread.sleep(millis);
        } catch(InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
- `@Repository` : 컴포넌트 스캔의 대상이 된다. 따라서 스프링 빈으로 자동 등록된다.
- `sleep(1000)` : 리포지토리는 상품을 저장하는데 약 1초 정도 걸리는 것으로 가정하기 위해 1초 지연을 주었다. (1000ms)
- 예외가 발생하는 상황도 확인하기 위해 파라미터 `itemId` 의 값이 `ex` 로 넘어오면 `IllegalStateException` 예외가 발생하도록 했다.

__OrderServiceV0__   
```java
@Service
@RequiredArgsConstructor
public class OrderServiceV0 {

    private final OrderRepositoryV0 orderRepository;

    public void orderItem(String itemId) {
        orderRepository.save(itemId);
    }
}
```
- `@Service` : 컴포넌트 스캔의 대상이 된다.
- 실무에서는 복잡한 비즈니스 로직이 서비스 계층에 포함되지만, 예제에서는 단순함을 위해서 리포지토리에 저장을 호출하는 코드만 있다.

__OrderControllerV0__   
```java
@RestController
@RequiredArgsConstructor
public class OrderControllerV0 {

    private final OrderServiceV0 orderService;

    @GetMapping("/v0/request")
    public String request(String itemId) {
        orderService.orderItem(itemId);
        return "ok";
    }
}
```
- `@RestController` : 컴포넌트 스캔과 스프링 Rest 컨트롤러로 인식된다.
- `/v0/request` 메서드는 HTTP 파라미터로 itemId 를 받을 수 있다.
- 실행: http://localhost:8080/v0/request?itemId=hello
- 결과: ok

### 로그 추적기 - 요구사항 분석
__요구사항__    
- 모든 PUBLIC 메서드의 호출과 응답 정보를 로그로 출력
- 애플리케이션의 흐름을 변경하면 안됨
  - 로그를 남긴다고 해서 비즈니스 로직의 동작에 영향을 주면 안됨
- 메서드 호출에 걸린 시간
- 정상 흐름과 예외 흐름 구분
  - 예외 발생시 예외 정보가 남아야 함
- 메서드 호출의 깊이 표현
- HTTP 요청을 구분
  - HTTP 요청 단위로 특정 ID를 남겨서 어떤 HTTP 요청에서 시작된 것인지 명확하게 구분이 가능해야 함
  - 트랜잭션 ID (DB 트랜잭션X), 여기서는 하나의 HTTP 요청이 시작해서 끝날 때까지를 하나의 트랜잭션이라 함

__예시__   
```
정상 요청
[796bccd9] OrderController.request()
[796bccd9] |-->OrderService.orderItem()
[796bccd9] | |-->OrderRepository.save()
[796bccd9] | |<--OrderRepository.save() time=1004ms
[796bccd9] |<--OrderService.orderItem() time=1014ms
[796bccd9] OrderController.request() time=1016ms
예외 발생
[b7119f27] OrderController.request()
[b7119f27] |-->OrderService.orderItem()
[b7119f27] | |-->OrderRepository.save()
[b7119f27] | |<X-OrderRepository.save() time=0ms 
ex=java.lang.IllegalStateException: 예외 발생!
[b7119f27] |<X-OrderService.orderItem() time=10ms 
ex=java.lang.IllegalStateException: 예외 발생!
[b7119f27] OrderController.request() time=11ms 
ex=java.lang.IllegalStateException: 예외 발생!
```

### 로그 추적기 V1 - 프로토타입 개발
애플리케이션의 모든 로직에 직접 로그를 남겨도 되지만, 그것보다는 더 효율적인 개발 방법이 필요하다.    
특히 트랜잭션ID와 깊이를 표현하는 방법은 기존 정보를 이어 받아야 하기 때문에 단순히 로그만 남긴다고 해결할 수 있는 것은 아니다.   
요구사항에 맞추어 애플리케이션에 효과적으로 로그를 남기기 위한 로그 추적기를 개발해보자.   

먼저 프로토타입 버전을 개발해보자. 아마 코드를 모두 작성하고 테스트 코드까지 실행해보아야 어떤 것을 하는지 감이 올 것이다.

### 로그 추적기 V1 - 적용
애플리케이션에 개발한 로그 추적기를 적용해보자.    
기존 `v0` 패키지에 코드를 직접 작성해도 되지만, 기존 코드를 유지하고, 비교하기 위해서 `v1` 패키지를 새로 만들고 기존 코드를 복사하자.   

__정상 실행__   
http://localhost:8080/v1/request?itemId=hello   
![image](https://user-images.githubusercontent.com/31242766/234506656-08fa9651-e2d1-424e-98eb-fe13a03f71e3.png)

__예외 실행__   
http://localhost:8080/v1/request?itemId=ex   
![image](https://user-images.githubusercontent.com/31242766/234507539-77499975-6385-456f-afb5-ff6d480e21a8.png)


> 참고 : 아직 level 관련 기능을 개발하지 않았다. 따라서 level 값은 항상 0이다. 그리고 트랜잭션ID 값도
다르다. 이 부분은 아직 개발하지 않았다.

### 로그 추적기 V2 - 파라미터로 동기화 개발
트랜잭션ID와 메서드 호출의 깊이를 표현하는 하는 가장 단순한 방법은 첫 로그에서 사용한 `트랜잭션ID` 와 `level` 을 다음 로그에 넘겨주면 된다.
현재 로그의 상태 정보인 `트랜잭션ID` 와 `level` 은 `TraceId` 에 포함되어 있다. 따라서 `TraceId` 를 다음 로그에 넘겨주면 된다. 이 기능을 추가한 `HelloTraceV2` 를 개발해보자.

### 로그 추적기 V2 - 적용
메서드 호출의 깊이를 표현하고, HTTP 요청도 구분해보자.    
이렇게 하려면 처음 로그를 남기는 `OrderController.request()` 에서 로그를 남길 때 어떤 깊이와 어떤
트랜잭션 ID를 사용했는지 다음 차례인 `OrderService.orderItem()` 에서 로그를 남기는 시점에 알아야한다.   
결국 현재 로그의 상태 정보인 `트랜잭션ID` 와 `level` 이 다음으로 전달되어야 한다.

이 정보는 `TraceStatus.traceId` 에 담겨있다. 따라서 `traceId` 를 컨트롤러에서 서비스를 호출할 때 넘겨주면 된다.   
![image](https://user-images.githubusercontent.com/31242766/234511247-e9833e4d-d06e-4839-be5a-5486b2bce2bd.png)

`traceId` 를 넘기도록 V2 전체 코드를 수정하자.

__정상 실행__   
http://localhost:8080/v2/request?itemId=hello    
![image](https://user-images.githubusercontent.com/31242766/234515536-0956a47b-c877-48c3-a42b-693241229ae9.png)

__예외 실행__    
http://localhost:8080/v2/request?itemId=ex    
![image](https://user-images.githubusercontent.com/31242766/234515618-cbdf536f-23fd-4e57-9c4e-194dda13a9ab.png)

실행 로그를 보면 같은 HTTP 요청에 대해서 트랜잭션ID 가 유지되고, level 도 잘 표현되는 것을 확인할 수 있다.

### 정리
__요구 사항__   
- 모든 PUBLIC 메서드의 호출과 응답 정보를 로그로 출력
- 애플리케이션의 흐름을 변경하면 안됨
  - 로그를 남긴다고 해서 비즈니스 로직의 동작에 영향을 주면 안됨
- 메서드 호출에 걸린 시간
- 정상 흐름과 예외 흐름 구분
  - 예외 발생시 예외 정보가 남아야 함
- 메서드 호출의 깊이 표현
- HTTP 요청을 구분
  - HTTP 요청 단위로 특정 ID를 남겨서 어떤 HTTP 요청에서 시작된 것인지 명확하게 구분이 가능해야 함
  - 트랜잭션 ID (DB 트랜잭션X)

__남은 문제__   
- HTTP 요청을 구분하고 깊이를 표현하기 위해서 `TraceId` 동기화가 필요하다.
- `TraceId` 의 동기화를 위해서 관련 메서드의 모든 파라미터를 수정해야 한다.
  - 만약 인터페이스가 있다면 인터페이스까지 모두 고쳐야 하는 상황이다.
- 로그를 처음 시작할 때는 `begin()` 을 호출하고, 처음이 아닐때는 `beginSync()` 를 호출해야 한다.
  - 만약에 컨트롤러를 통해서 서비스를 호출하는 것이 아니라, 다른 곳에서 서비스를 처음으로 호출하는
  상황이라면 파리미터로 넘길 `TraceId` 가 없다.
  
HTTP 요청을 구분하고 깊이를 표현하기 위해서 `TraceId` 를 파라미터로 넘기는 것 말고 다른 대안은 없을까?

## 쓰레드 로컬 - ThreadLocal
### 필드 동기화 - 개발
앞서 로그 추적기를 만들면서 다음 로그를 출력할 때 `트랜잭션ID` 와 `level` 을 동기화 하는 문제가 있었다. 이 문제를 해결하기 위해 `TraceId` 를 파라미터로 넘기도록 구현했다. 
이렇게 해서 동기화는 성공했지만, 로그를 출력하는 모든 메서드에 `TraceId` 파라미터를 추가해야 하는 문제가 발생했다.   
TraceId 를 파라미터로 넘기지 않고 이 문제를 해결할 수 있는 방법은 없을까?

이런 문제를 해결할 목적으로 새로운 로그 추적기를 만들어보자.   
향후 다양한 구현제로 변경할 수 있도록 `LogTrace` 인터페이스를 먼저 만들고, 구현해보자.   

```
[c80f5dbb] OrderController.request() //syncTraceId(): 최초 호출 level=0
[c80f5dbb] |-->OrderService.orderItem() //syncTraceId(): 직전 로그 있음 level=1 증가
[c80f5dbb] | |-->OrderRepository.save() //syncTraceId(): 직전 로그 있음 level=2 증가
[c80f5dbb] | |<--OrderRepository.save() time=1005ms //releaseTraceId(): level=2->1 감소
[c80f5dbb] |<--OrderService.orderItem() time=1014ms //releaseTraceId(): level=1->0 감소
[c80f5dbb] OrderController.request() time=1017ms //releaseTraceId(): level==0, traceId 제거
```

실행 결과를 보면 `트랜잭션ID` 도 동일하게 나오고, `level` 을 통한 깊이도 잘 표현된다.   
`FieldLogTrace.traceIdHolder` 필드를 사용해서 TraceId 가 잘 동기화 되는 것을 확인할 수 있다.   
이제 불필요하게 `TraceId` 를 파라미터로 전달하지 않아도 되고, 애플리케이션의 메서드 파라미터도 변경하지 않아도 된다.

### 필드 동기화 - 적용
지금까지 만든 `FieldLogTrace` 를 애플리케이션에 적용해보자.   

__LogTrace 스프링 빈 등록__   
`FieldLogTrace` 를 수동으로 스프링 빈으로 등록하자. 수동으로 등록하면 향후 구현체를 편리하게 변경할 수 있다는 장점이 있다.

__LogTraceConfig__   
```java
package hello.advanced;
import hello.advanced.trace.logtrace.FieldLogTrace;
import hello.advanced.trace.logtrace.LogTrace;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class LogTraceConfig {
  @Bean
  public LogTrace logTrace() {
    return new FieldLogTrace();
  }
}
```

### 필드 동기화 - 동시성 문제
잘 만든 로그 추적기를 실제 서비스에 배포했다 가정해보자.   
테스트 할 때는 문제가 없는 것 처럼 보인다. 사실 직전에 만든 `FieldLogTrace` 는 심각한 동시성 문제를 가지고 있다.    
동시성 문제를 확인하려면 다음과 같이 동시에 여러번 호출해보면 된다.

__동시성 문제 확인__    
다음 로직을 1초 안에 2번 실행해보자.   
http://localhost:8080/v3/request?itemId=hello   
http://localhost:8080/v3/request?itemId=hello   

__실제 결과__   
```
[nio-8080-exec-3] [aaaaaaaa] OrderController.request()
[nio-8080-exec-3] [aaaaaaaa] |-->OrderService.orderItem()
[nio-8080-exec-3] [aaaaaaaa] | |-->OrderRepository.save()
[nio-8080-exec-4] [aaaaaaaa] | | |-->OrderController.request()
[nio-8080-exec-4] [aaaaaaaa] | | | |-->OrderService.orderItem()
[nio-8080-exec-4] [aaaaaaaa] | | | | |-->OrderRepository.save()
[nio-8080-exec-3] [aaaaaaaa] | |<--OrderRepository.save() time=1005ms
[nio-8080-exec-3] [aaaaaaaa] |<--OrderService.orderItem() time=1005ms
[nio-8080-exec-3] [aaaaaaaa] OrderController.request() time=1005ms
[nio-8080-exec-4] [aaaaaaaa] | | | | |<--OrderRepository.save() time=1005ms
[nio-8080-exec-4] [aaaaaaaa] | | | |<--OrderService.orderItem() time=1005ms
[nio-8080-exec-4] [aaaaaaaa] | | |<--OrderController.request() time=1005ms
```
__기대하는 결과__   
```
[nio-8080-exec-3] [52808e46] OrderController.request()
[nio-8080-exec-3] [52808e46] |-->OrderService.orderItem()
[nio-8080-exec-3] [52808e46] | |-->OrderRepository.save()
[nio-8080-exec-4] [4568423c] OrderController.request()
[nio-8080-exec-4] [4568423c] |-->OrderService.orderItem()
[nio-8080-exec-4] [4568423c] | |-->OrderRepository.save()
[nio-8080-exec-3] [52808e46] | |<--OrderRepository.save() time=1001ms
[nio-8080-exec-3] [52808e46] |<--OrderService.orderItem() time=1001ms
[nio-8080-exec-3] [52808e46] OrderController.request() time=1003ms
[nio-8080-exec-4] [4568423c] | |<--OrderRepository.save() time=1000ms
[nio-8080-exec-4] [4568423c] |<--OrderService.orderItem() time=1001ms
[nio-8080-exec-4] [4568423c] OrderController.request() time=1001ms
```
기대한 것과 전혀 다른 문제가 발생한다. `트랜잭션ID` 도 동일하고, `level` 도 뭔가 많이 꼬인 것 같다.    
분명히 테스트 코드로 작성할 때는 문제가 없었는데, 무엇이 문제일까?

__동시성 문제__   
`FieldLogTrace` 는 싱글톤으로 등록된 스프링 빈이다. 이 객체의 인스턴스가 애플리케이션에 딱 1 존재한다는 뜻이다.    
이렇게 하나만 있는 인스턴스의 FieldLogTrace.traceIdHolder 필드를 여러 쓰레드가 동시에 접근하기 때문에 문제가 발생한다.    
실무에서 한번 나타나면 개발자를 가장 괴롭히는 문제도 바로 이러한 동시성 문제이다.

### 동시성 문제 - 예제 코드
동시성 문제가 어떻게 발생하는지 단순화해서 알아보자.   
테스트에서도 lombok을 사용하기 위해 다음 코드를 추가하자.   
`build.gradle`   
```gradle
dependencies {
  ...
  //테스트에서 lombok 사용
  testCompileOnly 'org.projectlombok:lombok'
  testAnnotationProcessor 'org.projectlombok:lombok'
}
```
이렇게 해야 테스트 코드에서 `@Slfj4` 같은 애노테이션이 작동한다.

__FieldService__    
주의: 테스트 코드(src/test)에 위치한다.   
```java
package hello.advanced.trace.threadlocal.code;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class FieldService {   

    private String nameStore;

    public String logic(String name) {
        log.info("저장 name={} -> nameStore={}", name, nameStore);
        nameStore = name;
        sleep(1000);
        log.info("조회 nameStore={}", nameStore);
        return nameStore;
    }

    private void sleep(int millis) {
        try {
            Thread.sleep(millis);
        } catch(InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
파라미터로 넘어온 `name` 을 필드인 `nameStore` 에 저장한다. 그리고 1초간 쉰 다음 필드에 저장된 `nameStore` 를 반환한다.

__FieldServiceTest__   
```java
package hello.advanced.trace.threadlocal;

import hello.advanced.trace.threadlocal.code.FieldService;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;

@Slf4j
public class FieldServiceTest {

    private FieldService fieldService = new FieldService();

    @Test
    void field() {
        log.info("main start");
        Runnable userA = () -> {
            fieldService.logic("userA");
        };

        Runnable userB = () -> {
            fieldService.logic("userB");
        };

        Thread threadA = new Thread(userA);
        threadA.setName("thread-A");
        Thread threadB = new Thread(userB);
        threadB.setName("thread-B");

        threadA.start();    //A실행
        sleep(2000);        //동시성 문제 발생X
//      sleep(100);         //동시성 문제 발생O
        threadB.start();    //B실행

        sleep(3000);        //메인 쓰레드 종료 대기
        log.info("main exit");
    }

    private void sleep(int millis) {
        try {
            Thread.sleep(millis);
        } catch(InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
__순서대로 실행__   
`sleep(2000)` 을 설정해서 `thread-A` 의 실행이 끝나고 나서 `thread-B` 가 실행되도록 해보자. 참고로 `FieldService.logic()` 메서드는 내부에` sleep(1000)` 으로 1초의 지연이 있다. 따라서 1초 이후에 호출하면 순서대로 실행할 수 있다. 여기서는 넉넉하게 2초 (2000ms)를 설정했다.
```java
sleep(2000); //동시성 문제 발생X
//sleep(100); //동시성 문제 발생O
```

__실행 결과__   
```
[Test worker] main start
[Thread-A] 저장 name=userA -> nameStore=null
[Thread-B] 저장 name=userB -> nameStore=userA
[Thread-A] 조회 nameStore=userB
[Thread-B] 조회 nameStore=userB
[Test worker] main exit
```
실행 결과를 보자. 저장하는 부분은 문제가 없다. 문제는 조회하는 부분에서 발생한다.

__동시성 문제__    
결과적으로 `Thread-A` 입장에서는 저장한 데이터와 조회한 데이터가 다른 문제가 발생한다. 이처럼 여러 쓰레드가 동시에 같은 인스턴스의 필드 값을 변경하면서 발생하는 문제를 동시성 문제라 한다. 
이런 동시성 문제는 여러 쓰레드가 같은 인스턴스의 필드에 접근해야 하기 때문에 트래픽이 적은 상황에서는 확률상 잘 나타나지 않고, 트래픽이 점점 많아질 수 록 자주 발생한다.
특히 스프링 빈 처럼 싱글톤 객체의 필드를 변경하며 사용할 때 이러한 동시성 문제를 조심해야 한다.

> 참고    
> 이런 동시성 문제는 지역 변수에서는 발생하지 않는다. 지역 변수는 쓰레드마다 각각 다른 메모리 영역이 할당된다.   
> 동시성 문제가 발생하는 곳은 같은 인스턴스의 필드(주로 싱글톤에서 자주 발생), 또는 static 같은 공용 필드에 접근할 때 발생한다.   
> 동시성 문제는 값을 읽기만 하면 발생하지 않는다. 어디선가 값을 변경하기 때문에 발생한다.   

### ThreadLocal - 소개
쓰레드 로컬은 해당 쓰레드만 접근할 수 있는 특별한 저장소를 말한다. 쉽게 이야기해서 물건 보관 창구를 떠올리면 된다.    
여러 사람이 같은 물건 보관 창구를 사용하더라도 창구 직원은 사용자를 인식해서 사용자별로 확실하게 물건을 구분해준다.    
사용자A, 사용자B 모두 창구 직원을 통해서 물건을 보관하고, 꺼내지만 창구 지원이 사용자에 따라 보관한 물건을 구분해주는 것이다.   

__일반적인 변수 필드__   
여러 쓰레드가 같은 인스턴스의 필드에 접근하면 처음 쓰레드가 보관한 데이터가 사라질 수 있다.   
![image](https://user-images.githubusercontent.com/31242766/235358117-89510ef3-1ab3-4877-b53d-719d5be8abca.png)   
`thread-A` 가 `userA` 라는 값을 저장하고    
![image](https://user-images.githubusercontent.com/31242766/235358131-3365c125-5a97-428c-af67-f7dd13c033a2.png)   
`thread-B` 가 `userB` 라는 값을 저장하면 직전에 thread-A 가 저장한 userA 값은 사라진다.   

__쓰레드 로컬__   
쓰레드 로컬을 사용하면 각 쓰레드마다 별도의 내부 저장소를 제공한다. 따라서 같은 인스턴스의 쓰레드 로컬 필드에 접근해도 문제 없다.   
![image](https://user-images.githubusercontent.com/31242766/235358161-82779ace-aeff-4c9b-a58f-8dfb08e41ef9.png)    
`thread-A` 가 `userA` 라는 값을 저장하면 쓰레드 로컬은 thread-A 전용 보관소에 데이터를 안전하게 보관한다.   
![image](https://user-images.githubusercontent.com/31242766/235358174-9994c856-2024-417b-9798-be30352a6168.png)   
`thread-B` 가 `userB` 라는 값을 저장하면 쓰레드 로컬은 thread-B 전용 보관소에 데이터를 안전하게 보관한다.   
![image](https://user-images.githubusercontent.com/31242766/235358195-d2f31993-62eb-47e7-a8b4-d62b13dd3209.png)   
쓰레드 로컬을 통해서 데이터를 조회할 때도 `thread-A` 가 조회하면 쓰레드 로컬은 `thread-A` 전용 보관소에서 `userA` 데이터를 반환해준다.    
물론 `thread-B` 가 조회하면 `thread-B` 전용 보관소에서 `userB` 데이터를 반환해준다.   

자바는 언어차원에서 쓰레드 로컬을 지원하기 위한 `java.lang.ThreadLocal` 클래스를 제공한다.

### ThreadLocal - 예제 코드
__ThreadLocalService__   
주의: 테스트 코드(src/test)에 위치한다.
```java
package hello.advanced.trace.threadlocal.code;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class ThreadLocalService {

    private ThreadLocal<String> nameStore = new ThreadLocal<>();

    public String logic(String name) {
        log.info("저장 name={} -> nameStore={}", name, nameStore.get());
        nameStore.set(name);
        sleep(1000);
        log.info("조회 nameStore={}",nameStore.get());
        return nameStore.get();
    }

    private void sleep(int millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
기존에 있던 `FieldService` 와 거의 같은 코드인데, `nameStore` 필드가 일반 `String` 타입에서
`ThreadLocal` 을 사용하도록 변경되었다.

> 주의
> 해당 쓰레드가 쓰레드 로컬을 모두 사용하고 나면 `ThreadLocal.remove()` 를 호출해서 쓰레드 로컬에
> 저장된 값을 제거해주어야 한다. 제거하는 구체적인 예제는 조금 뒤에 설명하겠다.

__ThreadLocalServiceTest__   
```java
package hello.advanced.trace.threadlocal;

import hello.advanced.trace.threadlocal.code.ThreadLocalService;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;

@Slf4j
class ThreadLocalServiceTest {

    private ThreadLocalService service = new ThreadLocalService();

    @Test
    void threadLocal() {
        log.info("main start");
        Runnable userA = () -> {
            service.logic("userA");
        };
        Runnable userB = () -> {
            service.logic("userB");
        };
        Thread threadA = new Thread(userA);
        threadA.setName("thread-A");
        Thread threadB = new Thread(userB);
        threadB.setName("thread-B");
        threadA.start();
        sleep(100);
        threadB.start();
        sleep(2000);
        log.info("main exit");
    }

    private void sleep(int millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
__실행 결과__   
```
[Test worker] main start
[Thread-A] 저장 name=userA -> nameStore=null
[Thread-B] 저장 name=userB -> nameStore=null
[Thread-A] 조회 nameStore=userA
[Thread-B] 조회 nameStore=userB
[Test worker] main exit
```
쓰레드 로컬 덕분에 쓰레드 마다 각각 별도의 데이터 저장소를 가지게 되었다. 결과적으로 동시성 문제도 해결되었다.

### 쓰레드 로컬 동기화 - 개발
`FieldLogTrace` 에서 발생했던 동시성 문제를 `ThreadLocal` 로 해결해보자.   
`TraceId traceIdHolder` 필드를 쓰레드 로컬을 사용하도록 `ThreadLocal<TraceId> traceIdHolder` 로 변경하면 된다.

필드 대신에 쓰레드 로컬을 사용해서 데이터를 동기화하는 `ThreadLocalLogTrace` 를 새로 만들자.   

__ThreadLocalLogTrace__   
```java
package hello.advanced.trace.logtrace;

import hello.advanced.trace.TraceId;
import hello.advanced.trace.TraceStatus;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class ThreadLocalLogTrace implements LogTrace {

    private static final String START_PREFIX = "-->";
    private static final String COMPLETE_PREFIX = "<--";
    private static final String EX_PREFIX = "<X-";

    private ThreadLocal<TraceId> traceIdHolder = new ThreadLocal<>();

    @Override
    public TraceStatus begin(String message) {
        syncTraceId();
        TraceId traceId = traceIdHolder.get();
        Long startTimeMs = System.currentTimeMillis();
        log.info("[{}] {}{}", traceId.getId(), addSpace(START_PREFIX,
                traceId.getLevel()), message);
        return new TraceStatus(traceId, startTimeMs, message);
    }

    @Override
    public void end(TraceStatus status) {
        complete(status, null);
    }

    @Override
    public void exception(TraceStatus status, Exception e) {
        complete(status, e);
    }

    private void complete(TraceStatus status, Exception e) {
        Long stopTimeMs = System.currentTimeMillis();
        long resultTimeMs = stopTimeMs - status.getStartTimeMs();
        TraceId traceId = status.getTraceId();
        if (e == null) {
            log.info("[{}] {}{} time={}ms", traceId.getId(),
                    addSpace(COMPLETE_PREFIX, traceId.getLevel()), status.getMessage(),
                    resultTimeMs);
        } else {
            log.info("[{}] {}{} time={}ms ex={}", traceId.getId(),
                    addSpace(EX_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs,
                    e.toString());
        }
        releaseTraceId();
    }

    private void syncTraceId() {
        TraceId traceId = traceIdHolder.get();
        if (traceId == null) {
            traceIdHolder.set(new TraceId());
        } else {
            traceIdHolder.set(traceId.createNextId());
        }
    }

    private void releaseTraceId() {
        TraceId traceId = traceIdHolder.get();
        if (traceId.isFirstLevel()) {
            traceIdHolder.remove();//destroy
        } else {
            traceIdHolder.set(traceId.createPreviousId());
        }
    }

    private static String addSpace(String prefix, int level) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < level; i++) {
            sb.append( (i == level - 1) ? "|" + prefix : "| ");
        }
        return sb.toString();
    }
}
```
`traceIdHolder` 가 필드에서 `ThreadLocal` 로 변경되었다. 따라서 값을 저장할 때는 `set(..)` 을 사용하고, 값을 조회할 때는 `get()` 을 사용한다.   

__ThreadLocal.remove()__   
추가로 쓰레드 로컬을 모두 사용하고 나면 꼭 `ThreadLocal.remove()` 를 호출해서 쓰레드 로컬에 저장된 값을 제거해주어야 한다.   
쉽게 이야기해서 다음의 마지막 로그를 출력하고 나면 쓰레드 로컬의 값을 제거해야 한다.
```
[3f902f0b] hello1
[3f902f0b] |-->hello2
[3f902f0b] |<--hello2 time=2ms
[3f902f0b] hello1 time=6ms //end() -> releaseTraceId() -> level==0, ThreadLocal.remove() 호출
```
여기서는 `releaseTraceId()` 를 통해 `level` 이 점점 낮아져서 2 1 0이 되면 로그를 처음 호출한 부분으로 돌아온 것이다. 따라서 이 경우 연관된 로그 출력이 끝난 것이다. 이제 더 이상 TraceId 값을
추적하지 않아도 된다. 그래서 `traceId.isFirstLevel() ( level==0 )`인 경우 `ThreadLocal.remove()` 를 호출해서 쓰레드 로컬에 저장된 값을 제거해준다.

### 쓰레드 로컬 동기화 - 적용
```java
package hello.advanced;

import hello.advanced.trace.logtrace.LogTrace;
import hello.advanced.trace.logtrace.ThreadLocalLogTrace;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class LogTraceConfig {

    @Bean
    public LogTrace logTrace() {
//        return new FieldLogTrace();
        return new ThreadLocalLogTrace();
    }
}
```
동시성 문제가 있는 `FieldLogTrace` 대신에 문제를 해결한 `ThreadLocalLogTrace` 를 스프링 빈으로 등록하자.

## 템플릿 메서드 패턴과 콜백 패턴
### 템플릿 메서드 패턴 - 시작
__로그 추적기 도입 전 - V0 코드__   
```java
//OrderControllerV0 코드
@GetMapping("/v0/request")
public String request(String itemId) {
 orderService.orderItem(itemId);
 return "ok";
}
//OrderServiceV0 코드
public void orderItem(String itemId) {
 orderRepository.save(itemId);
}
```
__로그 추적기 도입 후 - V3 코드__   
```java
//OrderControllerV3 코드
@GetMapping("/v3/request")
public String request(String itemId) {
  TraceStatus status = null;
  try {
    status = trace.begin("OrderController.request()");
    orderService.orderItem(itemId); //핵심 기능
    trace.end(status);
  } catch (Exception e) {
    trace.exception(status, e);
    throw e;
  }
  return "ok";
}
//OrderServiceV3 코드
public void orderItem(String itemId) {
  TraceStatus status = null;
  try {
    status = trace.begin("OrderService.orderItem()");
    orderRepository.save(itemId); //핵심 기능
    trace.end(status);
  } catch (Exception e) {
    trace.exception(status, e);
    throw e;
  }
}
```
__핵심 기능 vs 부가 기능__    
- `핵심 기능`은 해당 객체가 제공하는 고유의 기능이다. 예를 들어서 `orderService` 의 핵심 기능은 주문
로직이다. 메서드 단위로 보면 `orderService.orderItem()` 의 핵심 기능은 주문 데이터를 저장하기 위해
리포지토리를 호출하는 `orderRepository.save(itemId)` 코드가 핵심 기능이다.
- `부가 기능`은 핵심 기능을 보조하기 위해 제공되는 기능이다. 예를 들어서 로그 추적 로직, 트랜잭션 기능이
있다. 이러한 부가 기능은 단독으로 사용되지는 않고, 핵심 기능과 함께 사용된다. 예를 들어서 로그 추적
기능은 어떤 핵심 기능이 호출되었는지 로그를 남기기 위해 사용한다. 그러니까 핵심 기능을 보조하기 위해
존재한다.

V0는 핵심 기능만 있지만, 로그 추적기를 추가한 V3코드는 핵심 기능과 부가 기능이 함께 섞여있다.
V3를 보면 로그 추적기의 도입으로 핵심 기능 코드보다 부가 기능을 처리하기 위한 코드가 더 많아졌다.
소위 배보다 배꼽이 큰 상황이다. 만약 클래스가 수백 개라면 어떻게 하겠는가?

이 문제를 좀 더 효율적으로 처리할 수 있는 방법이 있을까?    
V3 코드를 유심히 잘 살펴보면 다음과 같이 동일한 패턴이 있다.
```java
TraceStatus status = null;
try {
  status = trace.begin("message");
  //핵심 기능 호출
  trace.end(status);
} catch (Exception e) {
  trace.exception(status, e);
  throw e;
}
```
`Controller`, `Service`, `Repository` 의 코드를 잘 보면, 로그 추적기를 사용하는 구조는 모두 동일하다.
중간에 핵심 기능을 사용하는 코드만 다를 뿐이다.
부가 기능과 관련된 코드가 중복이니 중복을 별도의 메서드로 뽑아내면 될 것 같다. 그런데, `try ~ catch`
는 물론이고, 핵심 기능 부분이 중간에 있어서 단순하게 메서드로 추출하는 것은 어렵다.

__변하는 것과 변하지 않는 것을 분리__   
좋은 설계는 변하는 것과 변하지 않는 것을 분리하는 것이다.
여기서 핵심 기능 부분은 변하고, 로그 추적기를 사용하는 부분은 변하지 않는 부분이다.
이 둘을 분리해서 모듈화해야 한다.

`템플릿 메서드 패턴(Template Method Pattern)`은 이런 문제를 해결하는 디자인 패턴이다.

### 템플릿 메서드 패턴 - 예제2
__템플릿 메서드 패턴 구조 그림__   
![image](https://user-images.githubusercontent.com/31242766/235698074-c26d0545-70bd-441e-9a3c-2fc0ace4aff3.png)

### 템플릿 메서드 패턴 - 예제3
__익명 내부 클래스 사용하기__   
템플릿 메서드 패턴은 `SubClassLogic1`, `SubClassLogic2` 처럼 클래스를 계속 만들어야 하는 단점이
있다. 익명 내부 클래스를 사용하면 이런 단점을 보완할 수 있다.
익명 내부 클래스를 사용하면 객체 인스턴스를 생성하면서 동시에 생성할 클래스를 상속 받은 자식
클래스를 정의할 수 있다. 이 클래스는 `SubClassLogic1` 처럼 직접 지정하는 이름이 없고 클래스 내부에
선언되는 클래스여서 익명 내부 클래스라 한다.

__TemplateMethodTest - templateMethodV2() 추가__   
```java
/**
 * 템플릿 메서드 패턴, 익명 내부 클래스 사용
 */
@Test
void templateMethodV2() {
    AbstractTemplate template1 = new AbstractTemplate() {
        @Override
        protected void call() {
            log.info("비즈니스 로직1 실행");
        }
    };
    log.info("클래스 이름1={}", template1.getClass());
    template1.execute();

    AbstractTemplate template2 = new AbstractTemplate() {
        @Override
        protected void call() {
            log.info("비즈니스 로직2 실행");
        }
    };
    log.info("클래스 이름2={}", template2.getClass());
    template2.execute();
}
```

__실행 결과__   
```
클래스 이름1 class hello.advanced.trace.template.TemplateMethodTest$1
비즈니스 로직1 실행
resultTime=3
클래스 이름2 class hello.advanced.trace.template.TemplateMethodTest$2
비즈니스 로직2 실행
resultTime=0
```
실행 결과를 보면 자바가 임의로 만들어주는 익명 내부 클래스 이름은 `TemplateMethodTest$1`,
`TemplateMethodTest$2` 인 것을 확인할 수 있다.   
![image](https://user-images.githubusercontent.com/31242766/235964661-f2172030-a336-4d81-adff-2b58d34adfe0.png)

### 템플릿 메서드 패턴 - 적용1
__AbstractTemplate__   
```java
package hello.advanced.template;

import hello.advanced.trace.TraceStatus;
import hello.advanced.trace.logtrace.LogTrace;

public abstract class AbstractTemplate<T> {

    private final LogTrace trace;

    public AbstractTemplate(LogTrace trace) {
        this.trace = trace;
    }

    public T execute(String message) {
        TraceStatus status = null;
        try {
            status = trace.begin(message);

            //로직 호출
            T result = call();

            trace.end(status);
            return result;
        } catch(Exception e) {
            trace.exception(status, e);
            throw e;
        }
    }

    protected abstract T call();
}
```
- `AbstractTemplate` 은 템플릿 메서드 패턴에서 부모 클래스이고, 템플릿 역할을 한다.
- `<T>` 제네릭을 사용했다. 반환 타입을 정의한다.
- 객체를 생성할 때 내부에서 사용할 `LogTrace trace` 를 전달 받는다.
- 로그에 출력할 `message` 를 외부에서 파라미터로 전달받는다.
- 템플릿 코드 중간에 `call()` 메서드를 통해서 변하는 부분을 처리한다.
- `abstract T call()` 은 변하는 부분을 처리하는 메서드이다. 이 부분은 상속으로 구현해야 한다.

### 템플릿 메서드 패턴 - 적용2
템플릿 메서드 패턴 덕분에 변하는 코드와 변하지 않는 코드를 명확하게 분리했다. 로그를 출력하는 템플릿 역할을 하는 변하지 않는 코드는 모두 `AbstractTemplate` 에 담아두고, 변하는 코드는 자식 클래스를 만들어서 분리했다.

지금까지 작성한 코드를 비교해보자.
```java
//OrderServiceV0 코드
public void orderItem(String itemId) {
  orderRepository.save(itemId);
}

//OrderServiceV3 코드
public void orderItem(String itemId) {
  TraceStatus status = null;
  try {
    status = trace.begin("OrderService.orderItem()");
    orderRepository.save(itemId); //핵심 기능
    trace.end(status);
  } catch (Exception e) {
    trace.exception(status, e);
    throw e;
  }
}

//OrderServiceV4 코드
AbstractTemplate<Void> template = new AbstractTemplate<>(trace) {
  @Override
  protected Void call() {
    orderRepository.save(itemId);
    return null;
  }
};
template.execute("OrderService.orderItem()");
```
- `OrderServiceV0` : 핵심 기능만 있다.
- `OrderServiceV3` : 핵심 기능과 부가 기능이 함께 섞여 있다.
- `OrderServiceV4` : 핵심 기능과 템플릿을 호출하는 코드가 섞여 있다.

V4는 템플릿 메서드 패턴을 사용한 덕분에 핵심 기능에 좀 더 집중할 수 있게 되었다.

__좋은 설계란?__    
좋은 설계라는 것은 무엇일까? 수 많은 멋진 정의가 있겠지만, 진정한 좋은 설계는 바로 변경이 일어날 때
자연스럽게 드러난다.
지금까지 로그를 남기는 부분을 모아서 하나로 모듈화하고, 비즈니스 로직 부분을 분리했다. 여기서 만약
로그를 남기는 로직을 변경해야 한다고 생각해보자. 그래서 `AbstractTemplate` 코드를 변경해야 한다
가정해보자. 단순히 `AbstractTemplate` 코드만 변경하면 된다.
템플릿이 없는 `V3` 상태에서 로그를 남기는 로직을 변경해야 한다고 생각해보자. 이 경우 모든 클래스를 다
찾아서 고쳐야 한다. 클래스가 수백 개라면 생각만해도 끔찍하다.

__단일 책임 원칙(SRP)__   
`V4` 는 단순히 템플릿 메서드 패턴을 적용해서 소스코드 몇줄을 줄인 것이 전부가 아니다.
로그를 남기는 부분에 단일 책임 원칙(SRP)을 지킨 것이다. 변경 지점을 하나로 모아서 변경에 쉽게 대처할
수 있는 구조를 만든 것이다.

### 템플릿 메서드 패턴 - 정의

> 템플릿 메서드 디자인 패턴의 목적은 다음과 같습니다.   
> "작업에서 알고리즘의 골격을 정의하고 일부 단계를 하위 클래스로 연기합니다. 템플릿 메서드를 사용하면
> 하위 클래스가 알고리즘의 구조를 변경하지 않고도 알고리즘의 특정 단계를 재정의할 수 있습니다."

__GOF 템플릿 메서드 패턴 정의__    
![image](https://github.com/haeyonghahn/advanced/assets/31242766/6de7356f-5ae7-48f9-8bc1-7af009a07e88)

풀어서 설명하면 다음과 같다.    
부모 클래스에 알고리즘의 골격인 템플릿을 정의하고, 일부 변경되는 로직은 자식 클래스에 정의하는 것이다.    
이렇게 하면 자식 클래스가 알고리즘의 전체 구조를 변경하지 않고, 특정 부분만 재정의할 수 있다.    
결국 상속과 오버라이딩을 통한 다형성으로 문제를 해결하는 것이다.    

__하지만__    
템플릿 메서드 패턴은 상속을 사용한다. 따라서 상속에서 오는 단점들을 그대로 안고간다. 특히 자식 클래스가 부모 클래스와 컴파일 시점에 강하게 결합되는 문제가 있다. 이것은 의존관계에 대한 문제이다. 자식 클래스 입장에서는 부모 클래스의 기능을 전혀 사용하지 않는다. 이번 장에서 지금까지 작성했던 코드를 떠올려보자. 자식 클래스를 작성할 때 부모 클래스의 기능을 사용한 것이 있었던가?

그럼에도 불구하고 템플릿 메서드 패턴을 위해 자식 클래스는 부모 클래스를 상속 받고 있다.    

상속을 받는다는 것은 특정 부모 클래스를 의존하고 있다는 것이다. 자식 클래스의 `extends` 다음에 바로 부모 클래스가 코드상에 지정되어 있다. 따라서 부모 클래스의 기능을 사용하든 사용하지 않든 간에 부모 클래스를 강하게 의존하게 된다. 여기서 강하게 의존한다는 뜻은 자식 클래스의 코드에 부모 클래스의 코드가 명확하게 적혀 있다는 뜻이다. UML에서 상속을 받으면 삼각형 화살표가 `자식 -> 부모` 를 향하고 있는 것은 이런 의존관계를 반영하는 것이다.   

자식 클래스 입장에서는 부모 클래스의 기능을 전혀 사용하지 않는데, 부모 클래스를 알아야한다. 이것은 좋은 설계가 아니다. 그리고 이런 잘못된 의존관계 때문에 부모 클래스를 수정하면, 자식 클래스에도 영향을 줄 수 있다.

추가로 템플릿 메서드 패턴은 상속 구조를 사용하기 때문에, 별도의 클래스나 익명 내부 클래스를 만들어야 하는 부분도 복잡하다.   
지금까지 설명한 이런 부분들을 더 깔끔하게 개선하려면 어떻게 해야할까? 템플릿 메서드 패턴과 비슷한 역할을 하면서 상속의 단점을 제거할 수 있는 디자인 패턴이 바로 `전략 패턴 (Strategy Pattern)`이다.

### 전략 패턴 - 시작
전략 패턴의 이해를 돕기 위해 템플릿 메서드 패턴에서 만들었던 동일한 예제를 사용해보자.     
__ContextV1Test__   
```java
package hello.advanced.trace.strategy;

import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;

@Slf4j
public class ContextV1Test {

    @Test
    void strategyV0() {
        logic1();
        logic2();
    }

    private void logic1() {
        long startTime = System.currentTimeMillis();
        //비즈니스 로직 실행
        log.info("비즈니스 로직1 실행");
        //비즈니스 로직 종료
        long endTime = System.currentTimeMillis();
        long resultTime = endTime - startTime;
        log.info("resultTime={}", resultTime);
    }

    private void logic2() {
        long startTime = System.currentTimeMillis();
        //비즈니스 로직 실행
        log.info("비즈니스 로직2 실행");
        //비즈니스 로직 종료
        long endTime = System.currentTimeMillis();
        long resultTime = endTime - startTime;
        log.info("resultTime={}", resultTime);
    }
}
```

### 전략 패턴 - 예제1
탬플릿 메서드 패턴은 부모 클래스에 변하지 않는 템플릿을 두고, 변하는 부분을 자식 클래스에 두어서 상속을 사용해서 문제를 해결했다. 전략 패턴은 변하지 않는 부분을 `Context` 라는 곳에 두고, 변하는 부분을 `Strategy` 라는 인터페이스를 만들고 해당 인터페이스를 구현하도록 해서 문제를 해결한다. 상속이 아니라 위임으로 문제를 해결하는 것이다. 전략 패턴에서 `Context` 는 변하지 않는 템플릿 역할을 하고, `Strategy` 는 변하는 알고리즘 역할을 한다.

GOF 디자인 패턴에서 정의한 전략 패턴의 의도는 다음과 같다.   
> 알고리즘 제품군을 정의하고 각각을 캡슐화하여 상호 교환 가능하게 만들자. 전략을 사용하면 알고리즘을   
> 사용하는 클라이언트와 독립적으로 알고리즘을 변경할 수 있다.

![image](https://github.com/haeyonghahn/advanced/assets/31242766/688f2acf-a151-42b9-8697-108e2d3f0f6b)

__Strategy 인터페이스__   
주의: 테스트 코드(src/test)에 위치한다.
```java 
package hello.advanced.trace.strategy.code.strategy;

public interface Strategy {
 void call();
}
```
이 인터페이스는 변하는 알고리즘 역할을 한다.

__StrategyLogic1__   
주의: 테스트 코드(src/test)에 위치한다.   
```java
package hello.advanced.trace.strategy.code.strategy;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class StrategyLogic1 implements Strategy {
  @Override
  public void call() {
    log.info("비즈니스 로직1 실행");
  }
}
```
변하는 알고리즘은 `Strategy` 인터페이스를 구현하면 된다. 여기서는 비즈니스 로직1을 구현했다.

__StrategyLogic2__    
주의: 테스트 코드(src/test)에 위치한다.
```java
package hello.advanced.trace.strategy.code.strategy;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class StrategyLogic2 implements Strategy {
  @Override
  public void call() {
    log.info("비즈니스 로직2 실행");
  }
}
```
비즈니스 로직2를 구현했다.

__ContextV1__   
주의: 테스트 코드(src/test)에 위치한다.   
```java
package hello.advanced.trace.strategy.code.strategy;

import lombok.extern.slf4j.Slf4j;
/**
 * 필드에 전략을 보관하는 방식
 */
@Slf4j
public class ContextV1 {
  private Strategy strategy;
  
  public ContextV1(Strategy strategy) {
    this.strategy = strategy;
  }
  public void execute() {
    long startTime = System.currentTimeMillis();
    //비즈니스 로직 실행
    strategy.call(); //위임
    //비즈니스 로직 종료
    long endTime = System.currentTimeMillis();
    long resultTime = endTime - startTime;
    log.info("resultTime={}", resultTime);
  }
}
```
`ContextV1`은 변하지 않는 로직을 가지고 있는 템플릿 역할을 하는 코드이다. 전략 패턴에서는 이것을 컨텍스트(문맥)이라 한다. 쉽계 이야기해서 컨텍스트(문맥)은 크게 변하지 않지만, 그 문맥 속에서 `strategy`를 통해 일부 전략이 변경된다 생각하면 된다.

`Context`는 내부에 `Strategy strategy` 필드를 가지고 있다. 이 필드에 변하는 부분인 `Strategy`의 구현체를 주입하면 된다. 전략 패턴의 핵심은 `Context`는 `Strategy` 인터페이스만 의존한다는 점이다. 덕분에 `Strategy`의 구현체를 변경하거나 새로 만들어도 `Context` 코드에는 영향을 주지 않는다.

어디서 많이 본 코드 같지 않은가? 스프링에서 의존관계 주입에서 사용하는 방식이 바로 전략 패턴이다.

__ContextV1Test - 추가__   
```java
/**
 * 전략 패턴 적용
 */
@Test
void strategyV1() {
  Strategy strategyLogic1 = new StrategyLogic1();
  ContextV1 context1 = new ContextV1(strategyLogic1);
  context1.execute();
  
  Strategy strategyLogic2 = new StrategyLogic2();
  ContextV1 context2 = new ContextV1(strategyLogic2);
  context2.execute();
}
```
전략 패턴을 사용해보자.   
코드를 보면 의존관계 주입을 통해 `ContextV1` 에 `Strategy` 의 구현체인 `strategyLogic1` 를 주입하는
것을 확인할 수 있다. 이렇게해서 `Context` 안에 원하는 전략을 주입한다. 이렇게 원하는 모양으로 조립을
완료하고 난 다음에 `context1.execute()` 를 호출해서 `context` 를 실행한다.

__전략 패턴 실행 그림__    
![image](https://github.com/haeyonghahn/advanced/assets/31242766/18c3f42c-6e43-4d30-9f12-38605feb820b)   
1. `Context`에 원하는 `Strategy` 구현체를 주입한다.
2. 클라이언트는 `context`를 실행한다.
3. `context`는 `context`로직을 시작한다.
4. `context`로직 중간에 `strategy.call()`을 호출해서 주입 받은 `strategy`로직을 실행한다.
5. `context`는 나머지 로직을 실행한다.

### 전략 패턴 - 예제2
전략 패턴도 익명 내부 클래스를 사용할 수 있다.   
__ContextV1Test - 추가__   
```java
/**
 * 전략 패턴 익명 내부 클래스1
 */
@Test
void strategyV2() {
    Strategy strategyLogic1 = new Strategy() {
        @Override
        public void call() {
            log.info("비즈니스 로직1 실행");
        }
    };
    log.info("strategyLogic1={}", strategyLogic1.getClass());
    ContextV1 context1 = new ContextV1(strategyLogic1);
    context1.execute();

    Strategy strategyLogic2 = new Strategy() {
        @Override
        public void call() {
            log.info("비즈니스 로직2 실행");
        }
    };
    log.info("strategyLogic2={}", strategyLogic2.getClass());
    ContextV1 context2 = new ContextV1(strategyLogic2);
    context2.execute();
}
```
__실행 결과__   
```
ContextV1Test - strategyLogic1=class
hello.advanced.trace.strategy.ContextV1Test$1
ContextV1Test - 비즈니스 로직1 실행
ContextV1 - resultTime=0
ContextV1Test - strategyLogic2=class
hello.advanced.trace.strategy.ContextV1Test$2
ContextV1Test - 비즈니스 로직2 실행
ContextV1 - resultTime=0
```
실행 결과를 보면 `ContextV1Test$1`, `ContextV1Test$2`와 같이 익명 내부 클래스가 생성된 것을 확인할 수 있다.

__ContextV1Test - 추가__   
```java
@Test
 void strategyV3() {
     ContextV1 context1 = new ContextV1(new Strategy() {
         @Override
         public void call() {
             log.info("비즈니스 로직1 실행");
         }
     });
     context1.execute();

     ContextV1 context2 = new ContextV1(new Strategy() {
         @Override
         public void call() {
             log.info("비즈니스 로직2 실행");
         }
     });
     context2.execute();
 }
```
익명 내부 클래스를 변수에 담아두지 말고, 생성하면서 바로 `ContextV1`에 전달해도 된다.

__ContextV1Test - 추가__   
```java
@Test
void strategyV4() {
    ContextV1 context1 = new ContextV1(() -> log.info("비즈니스 로직1 실행"));
    context1.execute();

    ContextV1 context2 = new ContextV1(() -> log.info("비즈니스 로직2 실행"));
    context2.execute();
}
```
익명 내부 클래스를 자바8부터 제공하는 람다로 변경할 수 있다. 람다로 변경하려면 인터페이스에 메서드가 1개만 있으면 되는데, 여기에서 제공하는 `Strategy`인터페이스는 메서드가 1개만 있으므로 람다로 사용할 수 있다.

__정리__   
변하지 않는 부분을 `Context` 에 두고 변하는 부분을 `Strategy` 를 구현해서 만든다. 그리고 `Context` 의 내부 필드에 `Strategy` 를 주입해서 사용했다.

__선 조립, 후 실행__   
여기서 이야기하고 싶은 부분은 `Context`의 내부 필드에 `Strategy`를 두고 사용하는 부분이다. 이 방식은 `Context`와 `Strategy`를 실행 전에 원하는 모양으로 조립해두고, 그 다음에 `Context`를 실행하는 선 조립, 후 실행 방식에서 유용하다. 

`Context`와 `Strategy`를 한번 조립하고 나면 이후로는 `Context`를 실행하기만 하면 된다. 우리가 스프링으로 애플리케이션을 개발할 때 애플리케이션 로딩 시점에 의존관계 주입을 통해 필요한 의존관계를 모두 맺어두고 난 다음에 실제 요청을 처리하는 것과 같은 원리이다.

이 방식의 단점은 `Context`와 `Strategy`를 조립한 이후에는 전략을 변경하기가 번거롭다는 점이다. 물론 `Context`에 `setter`를 제공해서 `Strategy`를 넘겨 받아 변경하면 되지만, `Context`를 싱글톤으로 사용할 때는 동시성 이슈 등 고려할 점이 많다. 그래서 전략을 실시간으로 변경해야 하면 차라리 이전에 개발한 테스트 코드처럼 `Context`를 하나 더 생성하고 그곳에 다른 `Strategy`를 주입하는 것이 더 나은 선택일 수 있다.

이렇게 먼저 조립하고 사용하는 방식보다 더 유연하게 전략 패턴을 사용하는 방법은 없을까?

### 전략 패턴 - 예제3
이번에는 전략 패턴을 조금 다르게 사용해보자. 이전에는 `Context`의 필드에 `Strategy`를 주입해서 사용했다. 이번에는 전략을 실행할 때 직접 파라미터로 전달해서 사용해보자.

__ContextV2__   
주의: 테스트 코드(src/test)에 위치한다.   
```java
package hello.advanced.trace.strategy.code.strategy;

import lombok.extern.slf4j.Slf4j;

/**
 * 전략을 파라미터로 전달 받는 방식
 */
@Slf4j
public class ContextV2 {

    public void execute(Strategy strategy) {
        long startTime = System.currentTimeMillis();
        //비즈니스 로직 실행
        strategy.call();    //위임
        //비즈니스 로직 종료
        long endTime = System.currentTimeMillis();
        long resultTime = endTime - startTime;
        log.info("resultTime={}", resultTime);
    }
}
```
`ContextV2`는 전략을 필드로 가지지 않는다. 대신에 전략을 `execute(...)`가 호출될 때 마다 항상 파라미터로 전달 받는다.

__ContextV2Test__   
```java
package hello.advanced.trace.strategy.code.strategy;

import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;

@Slf4j
public class ContextV2Test {

    /**
     * 전략 패턴 적용
     */
    @Test
    void strategyV1() {
        ContextV2 context = new ContextV2();
        context.execute(new StrategyLogic1());
        context.execute(new StrategyLogic2());
    }

    @Test
    void strategyV2() {
        ContextV2 context = new ContextV2();
        context.execute(new Strategy() {
            @Override
            public void call() {
                log.info("비즈니스 로직1 실행");
            }
        });
        context.execute(new Strategy() {
            @Override
            public void call() {
                log.info("비즈니스 로직2 실행");
            }
        });
    }

    @Test
    void strategyV3() {
        ContextV2 context = new ContextV2();
        context.execute(() -> log.info("비즈니스 로직1 실행"));
        context.execute(() -> log.info("비즈니스 로직2 실행"));
    }
}
```
`Context`와 `Strategy`를 '선 조립 후 실행'하는 방식이 아니라 `Context`를 실행할 때 마다 전략을 인수로 전달한다. 클라이언트는 `Context`를 실행하는 시점에 원하는 `Strategy`를 전달할 수 있다. 따라서 이전 방식과 비교해서 원하는 전략을 더욱 유연하게 변경할 수 있다.    
테스트 코드를 보면 하나의 `Context`만 생성한다. 그리고 하나의 `Context`에 실행 시점에 여러 전략을 인수로 전달해서 유연하게 실행하는 것을 확인할 수 있다.

__전략 패턴 파라미터 실행 그림__    
![image](https://github.com/haeyonghahn/advanced/assets/31242766/fdc740bd-6162-476a-b0d2-45236dfecc8a)   
1. 클라이언트는 `Context`를 실행하면서 인수로 `Strategy`를 전달한다.
2. `Context`는 `execute()` 로직을 실행한다.
3. `Context`는 파라미터로 넘어온 `strategy.call()` 로직을 실행한다.
4. `Context`의 `execute()` 로직이 종료된다.

__정리__   
- `ContextV1`은 필드에 `Strategy`를 저장하는 방식으로 전략 패턴을 구사했다.
  - 선 조립, 후 실행 방법에 적합하다.
  - `Context`를 실행하는 시점에는 이미 조립이 끝났기 때문에 전략을 신경쓰지 않고 단순히 실행만 하면 된다.
- `ContextV2`는 파라미터에 `Strategy`를 전달받는 방식으로 전략 패턴을 구사했다.
  - 실행할 때 마다 전략을 유연하게 변경할 수 있다.
  - 단점 역시 실행할 때 마다 전략을 게속 지정해주어야 한다는 점이다.

__템플릿__   
지금 우리가 해결하고 싶은 문제는 변하는 부분과 변하지 않는 부분을 분리하는 것이다. 변하지 않는 부분은 템플릿이라고 하고, 그 템플릿 안에서 변하는 부분에 약간 다른 코드 조각을 넘겨서 실행하는 것이 목적이다. `ContextV1`, `ContextV2` 두 가지 방식 다 문제를 해결할 수 있지만, 어떤 방식이 조금 더 나아 보이는가? 지금 우리가 원하는 것은 애플리케이션 의존 관계를 설정하는 것 처럼 선 조립, 후 실행이 아니다. 단순히 코드를 실행할 때 변하지 않는 템플릿이 있고, 그 템플릿 안에서 원하는 부분만 살짝 다른 코드를 실행하고 싶을 뿐이다. 따라서 우리가 고민하는 문제는 실행 시점에 유연하게 실행 코드 조각을 전달하는 `ContextV2`가 더 적합하다.

### 템플릿 콜백 패턴 - 시작
`ContextV2`는 변하지 않는 템플릿 역햘을 한다. 그리고 변하는 부분은 파라미터로 넘어온 `Strategy`의 코드를 실행해서 처리한다. 이렇게 다른 코드의 인수로서 넘겨주는 실행 가능한 코드를 콜백(callback)이라 한다.

> 콜백 정의
> 프로그래밍에서 콜백(callback) 또는 콜에프터 함수(call-after function)는 다른 코드의 인수로서 넘겨주는 실행 가능한 코드를 말한다. 콜백을 넘겨받는 코드는 
> 이 콜백을 필요에 따라 즉시 실행할 수도 있고, 아니면 나중에 실행할 수도 있다.

쉽계 이야기해서 `callback`은 코드가 호출(`call`)은 되는데 코드를 넘겨준 곳의 뒤(`back`)에서 실행된다는 뜻이다.
- `ContextV2`예제에서 콜백은 `Strategy`이다.
- 여기에서는 클라이언트에서 직접 `Strategy`를 실행하는 것이 아니라, 클라이언트가 `ContextV2.execute(..)`를 실행할 때 `Strategy`를 넘겨주고,
`ContextV2` 뒤에서 `Strategy`가 실행된다.

__자바 언어에서 콜백__   
- 자바 언어에서 실행 가능한 코드를 인수로 넘기려면 객체가 필요하다. 자바8부터 람다를 사용할 수 있다.
- 자바 8 이전에는 보통 하나의 메소드를 가진 인터페이스를 구현하고, 주로 익명 내부 클래스를 사용했다.
- 최근에는 주로 람다를 사용한다.

__템플릿 콜백 패턴__   
- 스프링에서는 `ContextV2`와 같은 방식의 전략 패턴을 템플릿 콜백 패턴이라 한다. 전략 패턴에서 `Context`가 템플릿 역할을 하고, `Strategy`부분이 콜백으로 넘어온다 생각하면 된다.
- 참고로 템플릿 콜백 패턴은 GOF 패턴은 아니고, 스프링 내부에서 이런 방식을 자주 사용하기 때문에, 스프링 안에서만 이렇게 부른다. 전략 패턴에서 템플릿과 콜백 부분이 강조된 패턴으로 생각하면 된다.
- 스프링에서는 `JdbcTemplate`, `RestTemplate`, `TransactionTemplate`, `RedisTemplate`처럼 다양한 템플릿 콜백 패턴이 사용된다. 스프링에서 이름에 `XxxTemplate`가 있다면 템플릿 콜백 패턴으로 만들어져 있다 생각하면 된다.   

![image](https://github.com/haeyonghahn/advanced/assets/31242766/ac78315f-ae7f-490d-9d14-e430f748b8f2)

### 템플릿 콜백 패턴 - 예제
템플릿 콜백 패턴을 구현해보자. `ContextV2`와 내용이 같고 이름만 다르므로 크게 어려움은 없을 것이다.
- `Context` -> `Template`
- `Strategy` -> `Callback`

__Callback - 인터페이스__    
주의: 테스트 코드(src/test)에 위치한다.    
```java
package hello.advanced.trace.strategy.code.template;

public interface Callback {
    void call();
}
```
콜백 로직을 전달할 인터페이스이다.

__TimeLogTemplate__   
주의: 테스트 코드(src/test)에 위치한다.   
```java
package hello.advanced.trace.strategy.code.template;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class TimeLogTemplate {

    public void execute(Callback callback) {
        long startTime = System.currentTimeMillis();
        //비즈니스 로직 실행
        callback.call();
        //비즈니스 로직 종료
        long endTime = System.currentTimeMillis();
        long resultTime = endTime - startTime;
        log.info("resultTime={}", resultTime);
    }
}
```

__TemplateCallbackTest__   
```java
package hello.advanced.trace.strategy;

import hello.advanced.trace.strategy.code.template.Callback;
import hello.advanced.trace.strategy.code.template.TimeLogTemplate;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;

@Slf4j
public class TemplateCallbackTest {

    /**
     * 템플릿 콜백 패턴 - 익명 내부 클래스
     */
    @Test
    void callbackV1() {
        TimeLogTemplate template = new TimeLogTemplate();

        template.execute(new Callback() {
            @Override
            public void call() {
                log.info("비즈니스 로직1 실행");
            }
        });

        template.execute(new Callback() {
            @Override
            public void call() {
                log.info("비즈니스 로직2 실행");
            }
        });
    }

    @Test
    void callbackV2() {
        TimeLogTemplate template = new TimeLogTemplate();
        template.execute(() -> log.info("비즈니스 로직1 실행"));
        template.execute(() -> log.info("비즈니스 로직2 실행"));
    }
}
```
별도의 클래스를 만들어서 전달해도 되지만, 콜백을 사용할 경우 익명 내부 클래스나 람다를 사용하는 것이 편리하다.   
물론 여러곳에서 함께 사용되는 경우 재사용을 위해 콜백을 별도의 클래스로 만들어도 된다.   
















