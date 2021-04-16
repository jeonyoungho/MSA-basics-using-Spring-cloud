# 7강-Declarative Http Client - Feign
#### 유튜브 영상: https://www.youtube.com/watch?v=SOmn6BGL884&list=PL9mhQYIlKEhdtYdxxZ6hZeb0va2Gm17A5&index=7

## Declarative Http Client - Feign
- Interface 선언을 통해 자동으로 Http Client 를 생성
- RestTemplate은 아름답진 않음. 기술적으로는 좋지만 프로그래밍적으론 좋지가 않음. 스펙도 많이 알고 있어야하고 사용 방법도 많이 알고 있어야함
- RestTemplate의 경우에는 어떤 테스트환경에서는 외부 호출하지 않고 로컬에서 띄워보고 싶은데 이런것들이 불가능함
- <b>RestTemplate 은 concreate 클래스라 테스트하기 어렵다</b>
- <b>관심사의 분리</b>
    - 서비스의 관심 - 다른 리소스, 외부 서비스 호출과 리턴값
    - 관심 X - 어떤 URL, 어떻게 파싱할 것인가
- <b>Spring Cloud 에서 Open-Feign 기반으로 Wrapping 한 것이 Spring Cloud Feign</b>

## Declarative Http Client - Spring Cloud Feign
- 인터페이스 선언 만으로 Http Client 구현물을 만들어 줌
~~~
@FeignClient(name = "db", url="http://localhost:8080/")
public interface ProductResource {
    @RequestMapping(path = "/query/{itemId}")
    String getItemDetail(@PathVariable("itemId") String itemId);
}
~~~
- @FeignClient를 Interface에 명시
- 각 API를 Spring MVC Annotation을 이용해서 정의

### How to Use
- @Autowired를 통해 DI 받아 사용
~~~
@Autowired
ProductResource productResouce;
~~~

## [실습 Step-5] Feign 클라이언트 사용하기
- Tag : step-5-feign-url
- 목적
    - Display -> Product 호출시에 Feign을 사용해서 RestTemplate을 대체해 보려고 함

#### 1. [display] build.gradle에 dependency 추가
~~~
compile('org.springframework.cloud:spring-cloud-starter-openfeign')
~~~

#### 2. [display] DisplayApplication에 @EnableFeignClients 추가
~~~
@SpringBootApplication
@EnableCircuitBreaker
@EnableEurekaClient
@EnableFeignClients
public class DisplayApplication {

    @LoadBalanced
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(DisplayApplication.class);
    }
}
~~~

#### 3. [display] Feign용 Interface 추가
- 이전 ProductRemoteService 사용하지 않고 데모를 위해 별도 정의
~~~
package com.elevenst.service;

public interface FeignProductRemoteService {
    String getProductInfo(String productId);
}
~~~

#### 4. [display] FeignProductRemoteService에 Annotation 넣기
~~~
package com.elevenst.service;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;

@FeignClient(name = "product", url = "http://localhost:8082/")
public interface FeignProductRemoteService {
    @RequestMapping(path = "/products/{productId}")
    String getProductInfo(@PathVariable("productId") String productId);
}
~~~

- Client 구현 완성. 서버 가동시
- FeignProductRemoteService용 Spring Bean 자동 생성
- FeignProductRemoteService를 주입 받아서 사용하면 됨
- <b>FeignClient도 내부적으로 ribbon을 사용해서 로드밸런싱을 하는데 유레카를 사용하려면 url속성부분을 명시 안해주면 됨</b>
- <b>@FeignClient는 사실 MSA가 아니여도 적용하기 좋음, ribbon써도 좋지만 굳이 ribbon을 쓰느니 feign을 써서 만들어도 좋음, 실제 11번가는 feign을 자주 활용함</b>

#### 5. [display] DisplayController에서 호출 부분 변경
~~~
@RestController
@RequestMapping(path = "/displays")
public class DisplayController {

    private final ProductRemoteService productRemoteService;
    private final FeignProductRemoteService feignProductRemoteService;

    public DisplayController(ProductRemoteService productRemoteService,
                             FeignProductRemoteService feignProductRemoteService) {
        this.productRemoteService = productRemoteService;
        this.feignProductRemoteService = feignProductRemoteService;
    }

    @GetMapping(path = "/{displayId}")
    public String getDisplayDetail(@PathVariable String displayId) {
        String productInfo = getProductInfo();
        return String.format("[display id = %s at %s %s ]", displayId, System.currentTimeMillis(), productInfo);
    }

    private String getProductInfo() {
        return feignProductRemoteService.getProductInfo("12345");
    }
}
~~~
- 기존 RestTemplate사용했던 서비스 대신 Feign이 자동으로 생성해준 서비스 빈을 사용

#### 6. 동작 확인
- http://localhost:8081/displays/11111

#### 7. 정리
- Feign은 Interface 선언을 통해 자동으로 HTTP Client를 만들어주는 <b>Declarative Http Client</b>
- Open-Feign 기반으로 Spring Cloud가 Wrapping한것이 오늘 실습한 Spring Cloud Feign
- <b>방금 한 실습 (Feign + URL 명시)은</b>
    - No Ribbon
    - No Eureka
    - No Hystrix


## [실습 Step-5] Feign + Hystrix,Ribbon,Eureka
- Tag : step-5-feign-eureka
- 배경
    - Feign의 또 다른 강점은 Ribbon + Eureka + Hystrix와 통합되어 있 다는 점

#### 1. [display] Feign에 Eureka + Ribbon 적용하기
~~~
@FeignClient(name = "product")
public interface FeignProductRemoteService {
    @RequestMapping(path = "/products/{productId}")
    String getProductInfo(@PathVariable("productId") String productId);
}
~~~
- @FeignClient에서 url만 제거
- 완성 !!!

#### 2. Feign의 동작
- @FeignClient에 URL 명시 시
    - 순수 Feign Client로서만 동작
- @FeignClient에 URL 명시 하지 않으면 ?
    - <b>Feign + Ribbon + Eureka 모드로 동작</b>
    - 어떤 서버 호출하나 ? @FeignClient(name='product')
    - 즉, eureka에서 product 서버 목록을 조회해서 ribbon을 통해 load-balancing하면서 HTTP 호출을 수행

#### 3. [display] application.yml : Feign + Hystrix
- Tag : step-5-feign-hystrix
- 아래의 설정이 들어가면 메소드 하나 하나가 Hystrix Command 로서 호출됨
~~~
feign:
  hystrix:
    enabled: true
~~~

#### 4. Feign에서 Fallback은 ? Hystrix 설정은 ?
- Declarative Http Client - Spring Cloud Feign
- 그럼 Hystrix Fallback은 ??
- Feign으로 정의한 Interface를 직접 구현 하고 Spring Bean으로 선언
~~~
@Component
public class ProductResourceFallback implements ProductResource {
    @Override
    public String getItemDetail(String itemId) {
        return 'default value';
    }
}
~~~
- Fallback 클래스를 @Feign선언시 명시
~~~
@FeignClient(fallback=ProductResourceFallback.class, name = "db", url="http://localhost:8080/")
public interface ProductResource {
    @RequestMapping(value = "/query/{itemId}", method=RequestMethod.GET)
    String getItemDetail(@PathVariable("itemId") String itemId);
}
~~~

#### 5. [display] Feign용 Hystrix Fallback 선언 - Spring Bean으로 정의
~~~
@Component
public class FeignProductRemoteServiceFallbackImpl implements FeignProductRemoteService {
    @Override
    public String getProductInfo(String productId) {
        return "[ this product is sold out ]";
    }
}
~~~

#### 6. [display] Feign용 Hystrix Fallback 명시
~~~
@FeignClient(name = "product", fallback = FeignProductRemoteServiceFallbackImpl.class)
public interface FeignProductRemoteService {
    @RequestMapping(path = "/products/{productId}")
    String getProductInfo(@PathVariable("productId") String productId);
}
~~~
- @FeignClient에 fallback 속성으로 클래스 명시
- 위의 단점은 에러가 나면 알수가 없음(interface를 상속한 클래스이기에 에러를 넣을 부분이 없음)

#### 7. [display] 기본 Fallback 은 에러 원인(Exception) 을 알 수 없음 - FallbackFactory 사용
~~~
package com.elevenst.service;

import feign.hystrix.FallbackFactory;
import org.springframework.stereotype.Component;

@Component
public class FeignProductRemoteServiceFallbackFactory implements FallbackFactory<FeignProductRemoteService> {

    @Override
    public FeignProductRemoteService create(Throwable cause) {
        System.out.println("t = " + cause);
        return productId -> "[ this product is sold out ]";
    }
}
~~~

#### 8. [display] Feign용 Hystrix Fallback 명시
~~~
@FeignClient(name = "product", fallbackFactory = FeignProductRemoteServiceFallbackFactory.class)
public interface FeignProductRemoteService {
    @RequestMapping(path = "/products/{productId}")
    String getProductInfo(@PathVariable("productId") String productId);
}
~~~
- @FeignClient에 fallback 속성으로 클래스 명시

#### 9. [display] Feign용 Hystrix 프로퍼티 정의하는 법
~~~
hystrix:
  command:
    productInfo:    # command key. use 'default' for global setting.
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000
      circuitBreaker:
        requestVolumeThreshold: 1   # Minimum number of request to calculate circuit breaker's health. default 20
        errorThresholdPercentage: 50 # Error percentage to open circuit. default 50
    FeignProductRemoteService#getProductInfo(String):
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000   # default 1,000ms
      circuitBreaker:
        requestVolumeThreshold: 1   # Minimum number of request to calculate circuit breaker's health. default 20
        errorThresholdPercentage: 50 # Error percentage to open circuit. default 50
~~~
- Feign 사용하는 경우 commandKey 이름 주의!
- ex) FeignProductRemoteService#getProductInfo(String)

#### 10. 정리
- Feign은 인터페이스 선언 + 설정으로 다음과 같은 것들이 가능하다
    - Http Client
    - Eureka 타겟 서버 주소 획득
    - Ribbon을 통한 Client-Side Load Balancing
    - Hystrix를 통한 메소드별 Circuit Breaker<br>

![1](https://user-images.githubusercontent.com/44339530/114987486-2f5de180-9ed0-11eb-9ffd-6c42fdfebc6e.png)<br>
- Feign + Hystrix, Ribbon, Eureka 를 사용했을 때의 장애 유형 별 동작 예
    - 1)특정 API 서버의 인스턴스가 한개 Down 된 경우
        - EUREKA - Heartbeat 송신이 중단 됨으로 일정 시간 후 목록에서 사라짐
        - RIBBON - IOException 이 발생한 경우 다른 인스턴스로 Retry
        - Hystrix - Circuit 은 오픈되지 않음(ERROR = 33%), Fallback 및 Timeout 은 동작
    - 2)특정 API 가 비정상 동작하는 경우 (지연, 에러)
        - Hystrix - 해당 API 를 호출하는 Circuit Breaker 오픈 Fallback, Timeout 도 동작<br>

#### 출처
- https://freedeveloper.tistory.com/438?category=919480