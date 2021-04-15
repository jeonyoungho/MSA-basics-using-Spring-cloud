# 4강-Circuit Breaker-Hystrix

## Netflix OSS
- 50개 이상의 사내 프로젝트를 오픈소스로 공개
- 플랫폼(AWS) 안의 여러 컴포넌트와 자동화 도구를 사용하면서 파악한 패턴과 해결 방법을 블로그, 오픈소스로 공개<br>
<img width="536" alt="스크린샷 2021-04-15 오후 8 09 19" src="https://user-images.githubusercontent.com/44339530/114860027-78595b80-9e26-11eb-8c3b-b5456dc4ddde.png"><br>

## Spring Cloud
- 교집합이 spring-cloud-netflix<br>
<img width="477" alt="스크린샷 2021-04-15 오후 8 10 15" src="https://user-images.githubusercontent.com/44339530/114860151-99ba4780-9e26-11eb-9293-7c53346aa908.png"><br>

#### Spring boot와 Netflix OSS의 네 가지 주요한 교집합 부분
- 1)Hystrix
- 2)Eureka
- 3)Ribbon
- 4)Zuul

#### 모놀리틱 구조에서의 의존성 호출
<img width="548" alt="스크린샷 2021-04-15 오후 8 12 44" src="https://user-images.githubusercontent.com/44339530/114860428-f1f14980-9e26-11eb-9d5f-91c2c866cf48.png"><br>

- 모놀리틱에서는 의존성을 호출할 때 메서드를 호출하는데 이 부분은 100%신뢰할 수 있음(controller가 service호출하고 service가 dao호출하는 등)
- MSA 환경에서는 위와 같은 부분을 신뢰할 수 없음<br>

#### Failure as a First Class Citizen(실패를 1급 시민으로 생각해야한다는 유명한 명언)
- 분산 시스템, 특히 클라우드 환경에선 실패(Failure) 는 일반적인 표준이다
- 모놀리틱엔 없던(별로 신경 안썼던..) 장애 유형
- 넷플릭스는 한 서비스의 가동율(uptime) 최대 99.99%라 주장한다.
    - 99.99^30 = 99.7% uptime
    - 10 억 요청 중 0.3% 실패 = 300만 요청이 실패
    - 모든 서비스들이 이상적인 uptime 을 갖고 있어도 매 달마다 2시간 이상의 downtime 이 발생<br>
<img width="546" alt="스크린샷 2021-04-15 오후 8 14 40" src="https://user-images.githubusercontent.com/44339530/114860643-37157b80-9e27-11eb-8d8a-68786f20f258.png"><br>

## Hystrix
- Latency Tolerance and Fault Tolerance for Distributed Systems(분산 시스템에서의 지연 감내 그리고 실패 감내)<br>
<img width="506" alt="스크린샷 2021-04-15 오후 8 20 14" src="https://user-images.githubusercontent.com/44339530/114861279-fec26d00-9e27-11eb-8b1d-10d747465e43.png"><br>

- 위의 그림과 같이 네 개의 서버가 배포되었고 톰캣 옵션을 설정하지 않아서 서블릿 스레드 값은 200이라 가정해보자. 그리고 RestTemplate의 timeout 을 디폴트값인 무한대라 가정해보자.(timeout이 무한대라면 한 번의 요청이 끝나기전에 해당 스레드가 계속 붙잡고 있음)<br>

<img width="522" alt="스크린샷 2021-04-15 오후 8 25 42" src="https://user-images.githubusercontent.com/44339530/114861835-c1aaaa80-9e28-11eb-92f3-0ab9d11f97f3.png"><br>

- 만약 한 서버가 다운되면 해당 서비스를 호출하는 서비스는 무한정 대기하고 timeout을 기다리며 결론적으로 네트워크 장애가 발생하여 서비스가 뻗을 것이다.
- 위와 같은 서비스는 가용성이 높은 서비스가 아니다.

### Hystrix 적용하기

~~~
@HystrixCommand // 해당 어노테이션으로 인해 restTemplate을 인터럽트해서 대신 실행한다.
public String anyMethodWithExternalDependency() {
    URI uri = URI.create("http://172.32.1.22:8090/recommended);
    String result = this.restTemplate.getForObject(uri, String.class);
    return result;
}
~~~

- <b>위의 메소드를 호출할 때 벌어지는 일</b>
    - 이 메소드 호출을 'intercept'하여 '대신' 실행
    - 실행된 결과의 성공 / 실패 (Exception) 여부를 기록하고 '통계'를 낸다
    - 실행 결과 '통계'에 따라 Circuit Open 여부를 판단하고 필요한 '조치'를 취한다.

- Circuit open??
    - Circuit이 오픈된 Method는 (주어진 시간동안) 호출이 '제한'되며, '즉시' 에러를 반환한다.
    - 특정 외부 시스템에서 계속 에러를 발생시키면 해당 외부 시스템을 호출하지 않는 것이다.
    - 모든 요청이 실제 그 다음 단계로 넘어가지 않고 미리 리턴을 하는 것이다.
- why?
    - 특정 메소드에서의 지연(주로 외부 연동에서의 지연)이 시스템 전체의 Resource를(Thread, Memory등)를 모두 소모하여 시스템 전체의 장애를 유발한다.
    - 특정 외부 시스템에서 계속 에러를 발생 시킨다면, 지속적인 호출이 에러 상황을 더욱 악화시킨다.
- So!
    - 장애를 유발하는 (외부)시스템에 대한 연동을 조기에 차단(Fail Fast) 시킴으로서 나와 시스템을 보호한다.

- 기본 설정
    - 10초동안 20개 이상의 호출이 발생 했을 때 50% 이상의 호출에서 에러가 발생하면 Circuit Open
    - 즉, 10초 동안 @HystrixCommand 어노테이션이 붙은 메소드들에 대해 몇개 가 성공했고 몇개가 실패했는지에 대해 통계를 낸다.
    - 20개의 이상의 호출이 발생 했을 때의 시점에 통계된 데이터를 훑어봐서 50%이상이 에러가 발생했다면 circuit을 open한다.

- <b>Circuit이 오픈된 경우의 에러 처리는 ? - Fallback</b>
    - Fallback method는 Circuit이 오픈된 경우 <b>혹은 모든 Exception이 발생한 경우</b> 대신 호출될 Method. 장애 발생시 Exception 대신 응답할 Default 구현을 넣는다
    - Exception에 대한 처리를 할 필요 없이 Fallback을 통해 간단하게 사용할 수 있음
    - 실제 예제 코드는 아래와 같음

~~~
@HystrixCommand(commandKey = 'ExtDep1', fallbackMethod='recommendFallback')
public String anyMethodWithExternalDependency1() {
    URI uri = URI.create("http://172.32.1.22:8090/recommend");
    String result = this.restTemplate.getForObject(uri, String.class);
    return result;
}
public String recommendFallback() {
    return "No recommend available";
}
~~~

- 오랫동안 응답이 없는 메소드에 대한 처리 방법은? - Timeout
~~~
@HystrixCommand(commandKey = 'ExtDep1', fallbackMethod='recommendFallback',
    commandProperties = {
        @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="500")
    }) // 0.5초 동안 메소드가 성공하지 않으면 timeout exception이 발생하여 fallbackmethod가 실행되어 리턴됨
public String anyMethodWithExternalDependency1() {
    URI uri = URI.create("http://172.32.1.22:8090/recommend");
    String result = this.restTemplate.getForObject(uri, String.class);
    return result;
}
public String recommendFallback() {
    return "No recommend available";
}
~~~

- <b>설정하지 않으면 default 1,000ms(1초 이상 걸릴 가능성이 꽤 있기에 유념할 것)</b>
- 설정 시간동안 메소드가 끝나지 않으면(return/ exception) Hystrix는 메소드를 실제 실행중인 Thread에 interrupt()를 호출하고, 자신은 즉시 HystrixException을 발생시킨다.
- 물론 이 경우에도 Fallback을 있다면 Fallback을 수행<br>


#### Circuit Breaker의 Flow chart
<img width="556" alt="스크린샷 2021-04-15 오후 8 48 07" src="https://user-images.githubusercontent.com/44339530/114864375-e3f1f780-9e2b-11eb-8156-61c4718ad693.png"><br>

- 동기든 비동기 요청이 들어오면(1) Hystrixcommand로 받는데 현장 상태가 circuit이 open된 상태(3: short circuit)라면 getFallback()메소드를 호출하여 바로 리턴하게 됨
- 만약 circuit이 open된 상태가 아니더라도 스레드 풀 크기나 세마포어 부분이 꽉찼다면 동일하게 getFallback()메소드를 호출하여 바로 리턴하게 됨
- 위의 상황들이 아니라면 run(5)이 실행되는 이 과정속에서 timeout이 발생하면 fallback실행(5a)
- run이 실행 후 404나 exception이 발생한다면 fallback이 실행되고 성공한다면 정상적으로 리턴된다. 위와 같은 결과는 모두 통계로 수집되어 현재 circuit이 open되어야하는지 판단이 내려진다.

## [실습 Step-2] Hystrix 사용하기
- Tag : step-2-hystrix-basic
    - git reset --hard HEAD
    - git checkout step-2-hystrix-basic
- 배경
    - Display 서비스는 외부 Server인 Product API와 연동되어있음
    - Product API에 장애가 나더라도 Display의 다른 서비스는 이상없이 동작하였으면 합니다
    - Product API에 응답 오류인경우, Default값을 넣어 주고 싶습니다
- 결정 내용
    - Display -> Product 연동 구간에 Circuit Breaker를 적용 !
- 실습 과정
~~~
1. [display] build.gradle에 hystrix dependency추가
compile('org.springframework.cloud:spring-cloud-starter-netflix-hystrix')
2. [display] DisplayApplication에 @EnableCircuitBreaker추가
@EnableCircuitBreaker
3. [display] ProductRemoteServiceImp에 @HystrixCommand추가
@HystrixCommand
~~~

#### 출처
- https://www.youtube.com/watch?v=iHHuYGdG_Yk&list=PL9mhQYIlKEhdtYdxxZ6hZeb0va2Gm17A5&index=4
- https://freedeveloper.tistory.com/435?category=919480