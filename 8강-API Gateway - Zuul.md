# 8강-API Gateway - Zuul
#### 유튜브 영상: https://www.youtube.com/watch?v=6g1wH97BiuQ&list=PL9mhQYIlKEhdtYdxxZ6hZeb0va2Gm17A5&index=8

## API Gateway
- 클라이언트와 백엔드 서버 사이의 출입문(front door)
- 라우팅(라우팅, 필터링, API변환, 클라이언트 어댑터 API, 서비스 프록시)
- <b>횡단 관심사 cross-service concerns</b>
    - 보안. 인증(authentication) , 인가(authorization)
    - 일정량 이상의 요청 제한(rate limiting)
    - 계측(metering)<br>
![2](https://user-images.githubusercontent.com/44339530/114993368-b57d2680-9ed6-11eb-875f-c0a3c15b52da.png)<br>

- 위의 그림에 물음표 세개는 Hystrix, Eureka, Ribbon
    - Netflix의 API Gateway인 zuul은 위의 세개를 내장하고 있어 굉장히 강력함

## Netflix Zuul
- 마이크로 프록시
- 50개 이상의 AWS ELB 의 앞단에 위치해 3개의 AWS 리전에 걸쳐 하루 백억 이상의 요청을 처리(2015 년 기준)<br>
![3](https://user-images.githubusercontent.com/44339530/114994634-f164bb80-9ed7-11eb-9e29-293d9d0507b6.png)<br>
![4](https://user-images.githubusercontent.com/44339530/114994723-07727c00-9ed8-11eb-906f-66e0324379ee.png)<br>
- Netflix에서는 multi region으로 하나의 region에 장애가 났을 때 zuul을 통해서 다른 region으로 proxy해주도록 하고 있음

## API Gateway - Zuul
#### 1. Zuul의 모든 API 요청은 HystrixCommand로 구성되어 전달된다
![5](https://user-images.githubusercontent.com/44339530/114994986-54eee900-9ed8-11eb-9dc8-82e6d9475e34.png)<br>
- 각 API 경로 (서버군) 별로 Circuit Breaker 생성
- <b>하나의 서버군이 장애를 일으켜도 다른 서버군의 서비스에는 영향이 없다</b>
- CircuitBreaker / ThreadPool의 다양한 속성을 통해 서비스 별 속성에 맞는 설정 가능
- <b>zuul은 하나만 뒀을 때 장애가 발생하면 서비스가 아예 죽기 때문에 보통 이중화를 하거나 앞단에 ELB, L4를 붙임(배민)</b>

#### 2. API를 전달할 서버의 목록을 갖고 Zuul에 내장된 Ribbon을 통해 Load-Balancing을 수행한다
![6](https://user-images.githubusercontent.com/44339530/114996821-2e31b200-9eda-11eb-986e-6b0eab9f7326.png)<br>
- 주어진 서버 목록들을 Round-Robin으로 호출
- Coding을 통해 Load Balancing 방식 Customize 가능
- 예를 들어, 실제 product에 있는 인스턴스 리스트에 대한 로드 밸런싱은 ribbon을 통해 수행됨

#### 3. zuul에 내장된 Eureka Client를 사용하여 주어진 URL의 호출을 전달할 '서버 리스트'를 찾는다
![7](https://user-images.githubusercontent.com/44339530/114997433-ca5bb900-9eda-11eb-864f-07ac39b56447.png)<br>
- Zuul에는 Eureka Client가 내장
- 각 URL에 Mapping된 서비스 명을 찾아서 Eureka Server를 통해 목록을 조회 한다
- 조회된 서버 목록을 Ribbon 클라이언트에게 전달한다

#### 4. Eureka + Ribbon에 의해서 결정된 Server 주소로 HTTP 요청
![8](https://user-images.githubusercontent.com/44339530/114997713-1c044380-9edb-11eb-8793-b7f802e21708.png)<br>
- Apache Http Client가 기본 사용
- OKHttp Client 사용 가능(설정 필요)

#### 5. 선택된 첫 서버로의 호출이 실패할 경우 Ribbon에 의해서 자동으로 Retry 수행
![9](https://user-images.githubusercontent.com/44339530/114997898-51a92c80-9edb-11eb-8de9-455c03181580.png)<br>
- Retry 수행 조건
    - Http Client에서 Exception 발생 (IOException)
    - 설정된 HTTP 응답코드 반환 (ex 503)
- 11번가에서도 위와 같은 방식으로 사용 중

#### 6. 정리하자면 zuul의 모든 호출은...
![10](https://user-images.githubusercontent.com/44339530/114998128-98972200-9edb-11eb-8733-108c9def857f.png)<br>
- HystrixCommand로 실행되므로 Circuit Breaker를 통해
    - 장애시 Fail Fast 및 Fallback 수행 가능
- Ribbon + Eureka 조합을 통해
    - 현재 서비스가 가능한 서버의 목록을 자동으로 수신
- Ribbon의 Retry 기능을 통해
    - 동일한 종류의 서버들로의 자동 재시도가 가능<br>
![11](https://user-images.githubusercontent.com/44339530/114998407-e4e26200-9edb-11eb-8c78-e5f428481d30.png)<br>

## [실습 Step-6] Spring Cloud Zuul
- Tag : step-6-zuul-baseline

#### 1. [준비작업] Checkout
- 시간 절약을 위해 미리 준비된 모듈 Checkout(“zuul”)
~~~
git reset HEAD --hard
git checkout tags/step-6-zuul-baseline -b my-step-6
~~~

#### 2. [zuul] Zuul 과 Eureka 디펜던시 추가 (build.gradle)
~~~
compile('org.springframework.cloud:spring-cloud-starter-netflix- zuul')
compile('org.springframework.cloud:spring-cloud-starter-netflix-eureka-client')
compile('org.springframework.retry:spring-retry:1.2.2.RELEASE')
~~~
- application.yml : Enable Zuul, Eureka
- Main Class :
    - @EnableZuulProxy
    - @EnableDiscoveryClient

- Tag : step-6-zuul-proxy-with-eureka

#### 3. [zuul] application.yml 를 다음과 같이 설정
~~~
spring:
  application:
    name: zuul

server:
  port: 8765

zuul:
  routes:
    product:
      path: /products/**
      serviceId: product
      stripPrefix: false
    display:
      path: /displays/**
      serviceId: display
      stripPrefix: false

eureka:
  instance:
    non-secure-port: ${server.port}
    prefer-ip-address: true
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
~~~

#### 4. 테스트
- http://localhost:8765/product/products/11111
- http://localhost:8765/display/displays/11111
    - <b>zuul로 display라는 serviceId를 앞에 붙여서 요청을 하면 display밑에 있는 **두개 값이 그대로 해당하는 유레카에 등록된 product로 proxy가 됨</b>
    - <b>요청 그대로 product로 요청하게 되고 거기에 대한 response가 오면 zuul을 통해서 사용자에게 전달하게 됨</b>

#### 5. [zuul] application.yml 를 다음과 같이 설정
- Tag : step-6-zuul-isolation-thread-pool
~~~
zuul:
  routes:
    product:
      path: /products/**
      serviceId: product
      stripPrefix: false
    display:
      path: /displays/**
      serviceId: display
      stripPrefix: false
  ribbon-isolation-strategy: thread
  thread-pool:
    use-separate-thread-pools: true
    thread-pool-key-prefix: zuul-
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 1000
    product:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 10000
  threadpool:
    zuul-product:
      coreSize: 30
      maximumSize: 100
      allowMaximumSizeToDivergeFromCoreSize: true
    zuul-display:
      coreSize: 30
      maximumSize: 100
      allowMaximumSizeToDivergeFromCoreSize: true
product:
  ribbon:
    MaxAutoRetriesNextServer: 1
    ReadTimeout: 3000
    ConnectTimeout: 1000
    MaxTotalConnections: 300
    MaxConnectionsPerHost: 100

display:
  ribbon:
    MaxAutoRetriesNextServer: 1
    ReadTimeout: 3000
    ConnectTimeout: 1000
    MaxTotalConnections: 300
    MaxConnectionsPerHost: 100
~~~

## Spring Cloud Zuul - Isolation
![12](https://user-images.githubusercontent.com/44339530/115002776-15c49600-9ee0-11eb-8cc5-4f8ba26f2fda.png)<br>
- spring-cloud-zuul 의 <b>기본 Isolation 은 SEMAPHORE</b> (Netflix Zuul 은 threadpool)
    - netflix-zuul에는 spring-cloud-config가 없기때문에 spring-cloud-zuul은 netflix-zuul을 많이 바꿨음
    - <b>spring-cloud를 생태계에서 spring-cloud-config가 엄청 중요함(나중에 꼭 찾아볼 것)</b>

- 세마포어를 쓰게 되면 서블릿 스레드가 작업(hystricx command에 있는 메소드)을 하기전에 그냥 count만 하는 것임, 서블릿 스레드가 200개라도 세마포어가 5라면 토탈 5개만 실행을 할 수 있게 됨
- <b>세마포어는 network timeout 을 격리시켜주지 못하기에 11번가는 스레드풀을로 바꿨음</b>
- 사용할 떄 스레드풀을 선택할지 세마포어를 선택할지 결정해야함
    - 세마포어가 성능상에선 더 좋음(아무래도 스레드를 생성하는 비용이 없어지기에), 하지만 timeout을 하지못하기에 유의해야함<br>

![13](https://user-images.githubusercontent.com/44339530/115004467-ca12ec00-9ee1-11eb-8c5a-1284eb94d995.png)<br>
- spring-cloud-zuul 의 Isolation 을 THREAD 로 변경
    - 전체가 하나의 ribbon커맨드가 되서 ribbon커맨드라는 하나의 thread-pool만 생성됨
- zuul.ribbon-isolation-strategy : thread
- 모든 HystrixCommand 가 하나의 쓰레드풀에!!<br>

![14](https://user-images.githubusercontent.com/44339530/115004819-30980a00-9ee2-11eb-865e-502d66e5ae6e.png)<br>
- 유레카 등록된 서비스 별로 THREAD 생성
    - zuul.thread-pool.useSeparateThreadPools : true
    - zuul.thread-pool.threadPoolKeyPrefix : zuulgw
- 위의 설정을 통해 각 서비스 별 스레드풀 격리 성공
- 적용 예시<br>
<img width="591" alt="스크린샷 2021-04-16 오후 6 38 39" src="https://user-images.githubusercontent.com/44339530/115005578-f7ac6500-9ee2-11eb-9d79-cc5849e3df1c.png"><br>
<img width="565" alt="스크린샷 2021-04-16 오후 6 39 09" src="https://user-images.githubusercontent.com/44339530/115005646-08f57180-9ee3-11eb-9ada-5d8977ff77b0.png"><br>

#### zuul이 내부적으로 동작하는 과정
![15](https://user-images.githubusercontent.com/44339530/115005138-879ddf00-9ee2-11eb-88f6-d913f039539a.jpeg)<br>
- 실제 11번가 팀에서 운영하고 있는 zuul이 내부적인 설정을 크게 바꾸지 않았는데 뛰어난 성능을 보이며 지금까지 큰 문제는 없었음
- zuul에는 공통의 관심사 Log같은 것들이 있는데 11번가는 모든 요청에 대한 Log를 Hive에 넣어서 로그를 분석하고 있음
    - 어떠한 request의 에러 발생율 같은 것들을 비교 분석함
    - 에러들을 정형화된 에러로 에러 필터를 통해서 사용자들에게 내보내주고 있기도함

#### 출처
- https://freedeveloper.tistory.com/439?category=919480