# Coniguration Server, Tracing, Monitoring, 그리고 남은 이야기..
#### 유튜브 영상(25분23초 부터): https://www.youtube.com/watch?v=6g1wH97BiuQ&list=PL9mhQYIlKEhdtYdxxZ6hZeb0va2Gm17A5&index=8

## Spring Cloud Config

![1](https://user-images.githubusercontent.com/44339530/115006939-6807b600-9ee4-11eb-8cc9-4f8cadc78333.jpeg)<br>
- 중앙화 된 설정 서버
    - 설정이 크게 어렵지 않음
    - @EnableConfigServer 한 줄추가하고 정해진 git에 있는 url을 정해주기만 하면 됨
    - 구글 검색을 통해 추가적으로 찾아볼 것
- 예를 들어, 서버가 30대가 배포되어있고 스레드풀 코어가 30개로 되있는 40개로 바꾸려고 한다고 가정했을때
    - 만약에 중앙화된 Config 서버가 없다면 특정 서버의 스레드풀 설정을 바꾸고 그거를 각자 서버에 배포해서 모든 서버를 재기동해야함
    - 만약에 중앙화된 Config 서버가 있다면 서버들은 실행하는 시점에 config서버를 통해서 property(스레드풀 설정, Log레벨 설정 같은 것들)를 읽어오기 때문에 서버들을 재기동할 필요 없이 refresh 쫙 이벤트만 날리면 전체 서버에 반영이 되게 됨
    - 하지만 이상적이지만은 않은게 실제 spring-cloud-config를 써서 refresh를 하더라도 해당하는 @RefreshScope 어노테이션이 붙은 bean이 완전히 재생성이 되는데 아직은 11번가에서 검증은 하지 못한 상황
    - 11번가에서 만약 스레드가 30개에서 40개로 바꾼다하면 중앙화된 config서버 하나만 바꾸고 재기동만 하면 알아서 config에서 읽어와서 재배포하게 됨

## Spring Boot Admin, Turbine
![2](https://user-images.githubusercontent.com/44339530/115008258-de58e800-9ee5-11eb-9774-9126049781cc.png)<br>

## Zipkin, Spring Cloud Sleuth
<img width="633" alt="3" src="https://user-images.githubusercontent.com/44339530/115008306-ed3f9a80-9ee5-11eb-8196-08c076e78ce1.png"><br>

- Distributed Tracing, Twitter
- MSA환경에서 서비스가 많기 때문에 그것들을 전체적으로 한 번 Tracing해서 정보를 보는 것이 중요함
- rc라는걸 호출해서 rc api가 DB조회하는데 얼마나 걸렸고, display api를 호출했을때 시간이 얼마나 걸렸는지 같은 것을 분산 추적이라 함
- Twitter에서 제공해주는 Zipkin이라는 tool이 굉장히 좋음
- 실제 MSA를 구현한다면 꼭 적용할 것

## 세미나에서 다루지 못한 얘기들.. FAQ
- 코드 리뷰
- TDD
- Database 분리 방법
- 분산 트랜잭션: DB에 데이터를 생성하고 수정하고 삭제할 때 전체적인 트랜잭션을 보장할 수가 없기에 11번가는 트랜잭션하지 않음, 보상 트랜잭션이나 이벤트 드리븐으로 그쪽에 위임해서 전체적인 flow에서 그 트랜잭션을 처리하는 식으로 하고 있음
- Event Driven Architecture
- ...

#### 출처
- https://freedeveloper.tistory.com/440?category=919480

