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
