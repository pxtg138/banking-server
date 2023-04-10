# 뱅킹서버
친구신청이 완료된 사용자끼리 계좌이체를 할 수 있는 토이프로젝트

## 구현목표

- 대규모 트래픽을 감당할 수 있도록 확장에 열려있도록 설계한다.
- 유지보수가 용이하도록 객체지향적으로 설계한다.
- 다른 개발자와 협업을 할 수 있다고 가정하고 코드를 가독성있게 작성한다.
- 특정 기술을 사용할 때 합리적인 이유가 뒷받침되어야한다.

## 인프라 구조
<img width="1398" alt="image" src="https://user-images.githubusercontent.com/35839248/230807987-860090af-f33d-466e-8dfb-ba785c7c6363.png">

## ERD
![image](https://user-images.githubusercontent.com/35839248/230808121-0c1e0111-3d94-4294-bae6-04a03e41500f.png)

## 설계시 고려한 핵심기술

### Scale Out을 고려한 설계

이 프로젝트는 대규모 트래픽을 감당한다고 가정하고 설계했습니다. 단일 서버로 트래픽을 감당하기 어려울 때 비용 및 성능향상의 폭이 Scale Up보다 나은 Scale Out 방식으로 서버를 확장하고자합니다. 이 때 발생할 수 있는 세션의 정합성 문제를 해결하기 위해 Redis를 세션 저장소로 이용했습니다. 또한 프로젝트 디렉토리 구조를 회원정보, 친구정보, 계좌정보, 알람 4가지의 관심사로 분류하여 추후에 MSA로 리펙토링을 진행할 때 부담이 적게 설계하였습니다.

### 조회쿼리 성능을 고려한 적절한 DB INDEX 설정

DB의 성능에 많은 영향을 끼치는 것은 조회쿼리입니다. 테이블의 필드들은 B트리 형태로 저장이 되었으며 이 때 key는 index 혹은 PK를 기준으로 합니다. PK는 별도로 index로 설정하지 않아도 B트리가 이에 맞게 생성이 되며 조회쿼리가 자주 발생하는 값들을 기준으로 index를 설정했습니다. 

### 자바코드가 테이블에 의존하지 않도록 설계

MyBatis와 같이 쿼리에 의존한 자바코드 작성은 유지보수를 어렵게 하고 개발자를 피로하게 만듭니다. 이를 위해 JPA를 도입하여 쿼리로 부터 자바코드를 독립시키려고 노력했습니다. 또 엔티티 필드에 인스턴스 변수를 선언할 때 이전에는 DB 테이블의 외래키값을 저장했지만 최대한 객체를 저장하고 메소드의 파라미터로도 객체를 넘기도록 설계했습니다. 

### 신뢰있는 프로그램 개발을 위한 단위테스트 작성

전체 프로그램이 동작하기 위해서는 개발을 진행하면서 단위테스트를 진행해야합니다. 이를 위해서 비즈니스 요구사항과 관련된 부분을 중점적으로 단위테스트를 작성했습니다. 단위테스트는 외부의 의존도를 최소화 해야하기 위해 Mockito를 이용해 Mock과 Stub을 구현했습니다.

### 다른 개발자와의 협업을 위한 CI 구축

여러사람과 개발을 한다고 가정하고 git을 적극적으로 활용했습니다. 이슈기반으로 프로젝트를 진행하였고 이슈가 해결될 때마다 develop 브랜치로 PR을 올렸습니다. 내가 작성한 코드가 전체 프로그램과 잘 융화될 수 있는지 확인하고 싶었고 이를 자동화시켜 Github Actions를 이용해 CI를 구축했습니다. CI에서는 빌드와 테스트를 진행하고 성공여부를 알려줍니다. 

### Redis를 이용한 분산락 처리

계좌이체 기능의 동시성 문제를 해결하기 위해 Redis를 이용했습니다. Redis는 싱글스레드로 구성되어 있으며 한번에 하나의 요청만 처리하도록 보장합니다. 락을 획득하기 위해서 Redis에 특정 (key, value)가 존재하는지 확인하고 없으면 (key, value)를 생성합니다. 락을 해제할 때는 (key, value) 값을 삭제합니다. 

계좌이체 메소드를 synchronized 처리를 할 수 있지만 이렇게 되면 하나의 스레드만 해당 메소드를 사용할 수 있으므로 성능 저하가 발생합니다. 또 다중 서버 환경에서는 동시성 이슈가 발생할 수 있습니다. 

MySQL의 pessimistic Lock을 이용하여 exclusive Lock을 구현할 수 있지만 데드락의 위험이 있고 MySQL로 향하는 트래픽의 대기시간이 길어질 수 있습니다.

### MySql 비관적락을 이용한 정합성 보장

계좌이체의 동시성 문제를 좀 더 확실하게 처리하기 위해 비관적락을 적용했습니다. 이전에 Redis를 이용하여 분산락을 처리하면 Redis에 오류가 발생했을 때 계좌이체와 같은 중요한 로직에도 문제가 생길 수 있기 때문에 이체 관련 데이터를 직접 저장하는 MySql의 비관적락을 이용합니다.

### Kafka를 이용하여 알림서버에 데이터 전송

이전에 계좌이체로직을 try-catch 문으로 묶었고 finally 블록에 알림전송 기능을 포함시켰습니다. 하지만 계좌이체 로직은 트랜잭션 처리가 되어 있고 만약에 알림서버에 장애가 발생하면 RunTimeException이 발생해 롤백됩니다. 알림서버에 장애가 발생해도 계좌 이체로직은 정상적으로 동작할 수 있도록 Kafka를 통해 알림서버에 알림정보를 전송합니다. Kafka는 고가용성을 보장하고 빠른 처리가 가능하기 때문에 Kafka에 장애가 발생해도 복구가 되면 정상적으로 Consumer에게 토픽을 전송할 수 있고 알림서버에 장애가 발생해도 Producer는 정상적으로 동작할 수 있습니다.
