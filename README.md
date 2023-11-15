# coupon-issuance
동시성 이슈 해결 관련 학습 - 쿠폰 발급

#### 요구사항 정의  
선착순 100명에게 할인쿠폰을 제공하는 이벤트를 진행한다.  

이 이벤트는 아래와 같은 조건을 만족해야 한다.  
- 선착순 100명에게만 지급되어야한다.
- 101개 이상이 지급되면 안된다.
- 순간적으로 몰리는 트래픽을 버틸 수 있어야 한다.

#### 문제점 및 해결방안
`동시성을 고려하지 않으면 동시에 여러 스레드에서 접근 했을 때, 레이스 컨디션이 발생하여 정상적인 쿠폰 발행이 불가능하다.`

싱글 스레드로 작업하게 된다면 레이스 컨디션은 발생하지 않게 되지만, 성능이 저하된다.  
가장 먼저 발급 시도한 유저의 쿠폰이 발급 완료 될때까지 다음 유저는 기다려야 하기 때문이다.  

synchronized를 사용하는 방식은 서버가 여러대가 된다면 레이스 컨디션이 동일하게 발생한다.  
또, lock을 걸게 된다면 쿠폰 갯수를 가져올때 부터 쿠폰 발급이 완료 될때까지 기다리는 시간이 길어지게 된다.  

#### `redis`  
redis는 싱글 스레드 기반으로 동작하며, incr 명령어는 key 값을 1씩 증가시키며 성능적으로도 이점이 있다.  

`문제점`
mysql에서 1분에 100개의 insert만 가능하다고 가정해보면,  

10:00 10000개의 쿠폰 생성 요청  
10:01 주문 생성 요청  
10:02 회원 가입 요청  

이러한 경우, 주문 요청과 회원가입 요청은 100분 이후에 가능하게 되며 일반적으로 타임아웃이 발생하게 된다.  
짧은 시간에 많은 요청이 들어오게되면 db 리소스를 많이 사용하게 되며, 서비스 지연 및 오류발생의 원인이 된다.  
<img width="1141" alt="쿠폰" src="https://github.com/hong9Lee/coupon-issuance/assets/94272140/92710e52-004e-4973-8e7b-c88acc925849">

#### `kafka`  
분산 이벤트 스트리밍 플랫폼  
이벤트 스트리밍이란 소스에서 목적지까지 이벤트를 실시간으로 스트리밍 하는 것  

<img width="759" alt="스크린샷 2023-11-07 오후 8 51 47" src="https://github.com/hong9Lee/coupon-issuance/assets/94272140/4a7e3293-9421-451a-9291-6fb55aaeff24">

`topic 발행`  
docker exec -t kafka kafka-console-producer.sh --topic testTopic --broker-list 0.0.0.0:9092  

`topic producer`  
docker exec -it kafka kafka-console-producer.sh --topic testTopic --broker-list 0.0.0.0:9092  

`topic consumer`  
docker exec -it kafka kafka-console-consumer.sh --topic testTopic --bootstrap-server localhost:9092


#### `한명당 하나의 쿠폰 발급` 
1. 유니크 키를 부여해준다.  
사용자는 해당 쿠폰 이외에 다른 쿠폰도 가지고 있을 수 있으므로 실용적인 방법이 아니다.

2. 범위 락을 잡는다.  
락의 범위가 크기 때문에 기다리는 동안 성능이 좋지 않을 수 있으며, 아래 이미지와 같이 발급이 끝나지 않은 상태에서 또 요청을 보낸다면 중복 발급 될 수 있다.
<img width="1187" alt="스크린샷 2023-11-15 오후 8 54 40" src="https://github.com/hong9Lee/coupon-issuance/assets/94272140/4a90b9c0-d106-444f-a5ed-9588731f224f">

3.  Set  
Set 자료구조를 사용하여 1인당 1개의 쿠폰만 발급 가능하도록 할 수 있다.  










