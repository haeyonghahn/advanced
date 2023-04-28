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
