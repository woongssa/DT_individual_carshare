# DT_6th_team5_carShare (자동차 공유 서비스)

5팀 자동차 공유 서비스 CNA개발 실습을 위한 프로젝트

# 구현 Repository
 1. 접수관리 : https://github.com/YoungDukGe1000Won/carShareOrder.git
 1. 결제관리 : https://github.com/YoungDukGe1000Won/carSharePayment.git
 1. 배송관리 : https://github.com/YoungDukGe1000Won/carShareDelivery.git
 1. 고객페이지 : https://github.com/YoungDukGe1000Won/carShareStatusview.git
 1. 게이트웨이 : https://github.com/byounghoonmoon/carShareGateway.git


# Table of contents

- [서비스 시나리오](#서비스-시나리오)
  - [시나리오 테스트결과](#시나리오-테스트결과)
- [분석/설계](#분석설계)
- [구현](#구현)
  - [DDD 의 적용](#ddd-의-적용)
  - [Gateway 적용](#Gateway-적용)
  - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
  - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
  - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
- [운영](#운영)
  - [CI/CD 설정](#cicd설정)
  - [서킷 브레이킹 / 장애격리](#서킷-브레이킹-/-장애격리)
  - [오토스케일 아웃](#오토스케일-아웃)
  - [무정지 재배포](#무정지-재배포)
  

# 서비스 시나리오

## 기능적 요구사항
1. 고객이 공유차를 선택하여 렌탈한다.
1. 고객이 결제하여 접수한다.
1. 업체가 공유차를 고객위치로 가져다놓는다.
1. 고객이 렌탈을 취소할 수 있다.
1. 렌탈이 취소되면 배송이 취소된다.
1. 고객이 자신의 렌탈 정보를 조회한다.

## 비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 접수가 성립되지 않아야 한다(Sync 호출)
1. 장애격리
    1. 배송관리 기능이 수행되지 않더라도 접수는 정상적으로 처리 가능하다(Async(event-driven), Eventual Consistency)
    1. 접수시스템이 과중되면 사용자를 잠시동안 받지 않고 결게를 잠시후에 하도록 유도한다(Circuit breaker, fallback)
1. 성능
    1. 고객이 본인의 렌탈 상태 및 이력을 접수시스템에서 확인할 수 있어야 한다(CQRS)


# 분석/설계

## AS-IS 조직 (Horizontally-Aligned)
![79684144-2a893200-826a-11ea-9a01-79927d3a0107](https://user-images.githubusercontent.com/42608068/96371393-84789f00-119c-11eb-80d9-ffbcab38ff84.png)


## TO-BE 조직 (Vertically-Aligned)
![제목 없음23](https://user-images.githubusercontent.com/42608068/96586404-edd2ec00-131b-11eb-9996-eb312be000b5.png)



## 이벤트스토밍
* MSAEz 로 모델링한 이벤트스토밍 결과:  
![image](https://user-images.githubusercontent.com/42608068/96539757-c9085580-12d6-11eb-8721-8bb7e0601d53.png)

### 이벤트 도출 
![제목 없음1](https://user-images.githubusercontent.com/42608068/96541160-60bb7300-12da-11eb-8eda-4beb637fa24f.png)

### 부적격 이벤트 탈락
![제목 없음2](https://user-images.githubusercontent.com/42608068/96541195-6fa22580-12da-11eb-94c0-9efb0947e5aa.png)

### 액터, 커맨드 부착하여 읽기 좋게
![제목 없음3](https://user-images.githubusercontent.com/42608068/96541203-77fa6080-12da-11eb-8a8a-50a018a72961.png)

### 어그리게잇으로 묶기
![제목 없음3](https://user-images.githubusercontent.com/42608068/96402579-423d7500-1211-11eb-9784-c1f7e1c4b2fb.png)

### 바운디드 컨텍스트로 묶기
![제목 없음4](https://user-images.githubusercontent.com/42608068/96541221-86487c80-12da-11eb-96c9-705fd7957d01.png)

### 도메인분리
![제목 없음5](https://user-images.githubusercontent.com/42608068/96541235-919ba800-12da-11eb-8c49-84655f2ca88e.png)

### 폴리시 부착 
![제목 없음6](https://user-images.githubusercontent.com/42608068/96541251-99f3e300-12da-11eb-99f9-8a9027a7b855.png)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)
![제목 없음7](https://user-images.githubusercontent.com/42608068/96541279-a11af100-12da-11eb-9d0d-3cf209f7216b.png)

### 완성된 모형
![제목 없음11](https://user-images.githubusercontent.com/42608068/96543764-0b826000-12e0-11eb-9296-112459a6027b.png)

### 완성본에 대한 기능적 요구사항을 커버하는지 검증
![제목 없음12](https://user-images.githubusercontent.com/42608068/96543922-72077e00-12e0-11eb-91bf-ae6aaf5e8fbb.png)
    
    - 고객이 공유차를 선택하여 렌탈한다 (ok)
    - 고객이 결제하여 접수한다 (ok)
    - 업체가 공유차를 고객위치로 가져다놓는다 (ok)

![제목 없음13](https://user-images.githubusercontent.com/42608068/96543936-79c72280-12e0-11eb-98a2-0c67478f6926.png)

    - 고객이 주문을 취소할 수 있다 (ok)
    - 렌탈이 취소되면 배송이 취소된다 (ok)

![제목 없음14](https://user-images.githubusercontent.com/42608068/96543997-9cf1d200-12e0-11eb-9a71-9aa743f7de44.png)
   
    - 고객이 자신의 렌탈 정보를 조회한다 (ok)
    
### 완성본에 대한 비기능적 요구사항을 커버하는지 검증
![제목없음22](https://user-images.githubusercontent.com/42608068/96582783-c2013780-1316-11eb-8bfc-dba64c7af837.png)

    1. 트랜잭션
    - 고객의 주문에 따라 결제가 진행된다(결제가 정상적으로 완료되지 않으면 주문이 되지 않는다) > Sync
    - 고객의 결제 완료에 따라 배송이 진행된다 > Async
    2. 장애격리
    - 배송 서비스에 장애가 발생하더라도 주문 및 결제는 정상적으로 처리 가능하다 > Async(event driven)
    - 서킷 브레이킹 프레임워크 > istio-injection + DestinationRule
    3. 성능
    - 고객은 본인의 상태 정보를 확인할 수 있다 > CQRS

## 헥사고날 아키텍처 다이어그램 도출
![제목없음21](https://user-images.githubusercontent.com/42608068/96549943-260e0680-12eb-11eb-8119-394cb324883d.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐

   
# 구현

## 시나리오 테스트결과
| 기능 | 이벤트 Payload |
|---|:---:|
| 1.고객이 공유차 렌탈을 접수한다.</br>2. 결제가 정상적으로 완료되면 접수가 진행된다. (Sync)</br>3.접수가 완료되면 배송이 시작된다. (Async)</br>3.배송이 시작되면 접수정보의 상태를 변경한다. (Async)|![제목없음21](https://user-images.githubusercontent.com/42608068/96580615-831db280-1313-11eb-930e-be9b41b256ad.png)|
| 4.고객이 공유차 렌탈을 취소한다.</br>5. 배송 취소가 정상적으로 완료되면 결제 취소가 진행된다. (Sync)</br>6.결제 취소도 정상적으로 이어지면 접수가 최종적으로 취소된다. (Async)|![제목없음21](https://user-images.githubusercontent.com/42608068/96580830-d7289700-1313-11eb-985b-aa1036db3e57.png)|
| 7.고객이 접수 상태를 조회한다.|![제목없음21](https://user-images.githubusercontent.com/42608068/96581350-a5640000-1314-11eb-8336-0474e2d1716b.png)|

## DDD 의 적용
분석/설계 단계에서 도출된 MSA는 총 5개로 아래와 같다.
* customerpage 는 CQRS 를 위한 서비스

| MSA | 기능 | port | 조회 API | Gateway 사용시 |
|---|:---:|:---:|---|---|
| order | 접수 관리 | 8081 | http://localhost:8081/orders |http://carshareorder:8080/orders |
| delivery | 배송 관리 | 8082 | http://localhost:8082/deliveries | http://carsharedelivery:8080/deliveries |
| customerpage | 상태 조회 | 8083 | http://localhost:8083/customerpages | http://carsharestatusview:8080/customerpages |
| payment | 결제 관리 | 8084 | http://localhost:8084/payments | http://carsharepayment:8080/payments |

## Gateway 적용

```
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://carshareorder:8080
          predicates:
            - Path=/orders/** 
        - id: delivery
          uri: http://carsharedelivery:8080
          predicates:
            - Path=/deliveries/**,/cancellations/**
        - id: statusview
          uri: http://carsharestatusview:8080
          predicates:
            - Path= /customerpages/**
        - id: payment
          uri: http://carsharepayment:8080
          predicates:
            - Path=/payments/**,/paymentCancellations/**
```


## 폴리글랏 퍼시스턴스

CQRS 를 위한 Mypage 서비스만 DB를 구분하여 적용함. 인메모리 DB인 hsqldb 사용.

```
pom.xml 에 적용
<!-- 
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
 -->
		<dependency>
		    <groupId>org.hsqldb</groupId>
		    <artifactId>hsqldb</artifactId>
		    <version>2.4.0</version>
		    <scope>runtime</scope>
		</dependency>
```


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 접수(order)->결제(payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.
- FeignClient 서비스 구현

```
# PaymentService.java

@FeignClient(name="payment", contextId ="payment", url="${api.payment.url}", fallback = PaymentServiceFallback.class)
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void pay(@RequestBody Payment payment);

}
```
- 접수요청을 받은 직후(@PostPersist) 결제를 요청하도록 처리
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.publishAfterCommit();

        carshare.external.Payment payment = new carshare.external.Payment();
        payment.setOrderId(this.getId());
        payment.setProductId(this.getProductId());
        payment.setQty(this.getQty());
        payment.setStatus("OrderApproved");
        OrderApplication.applicationContext.getBean(carshare.external.PaymentService.class)
            .pay(payment);
    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 서비스가 장애가 나면 접수요청 못받는다는 것을 확인


```
#결제(payment) 서비스를 잠시 내려놓음 (ctrl+c)

#접수요청 처리
http localhost:8081/orders productId=1001 qty=1 status="order"   #Fail
http localhost:8081/orders productId=1002 qty=3 status="order"   #Fail

#결제 서비스 재기동
cd carsharepayment
mvn spring-boot:run

#접수요청 처리 성공
http localhost:8081/orders productId=1001 qty=1 status="order"   #Success
http localhost:8081/orders productId=1002 qty=3 status="order"   #Success
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, Fallback 처리는 운영단계에서 설명한다.)


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

결제가 이루어진 후에 배송 서비스로 이를 알려주는 행위는 동기식이 아니라 비동기식으로 처리하여 배송 시스템의 처리를 위해 결제가 블로킹되지 않도록 처리한다.
 
- 이를 위하여 결제이력 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package carshare;

@Entity
@Table(name="Payment_table")
public class Payment {

 ...
    @PostPersist
    public void onPostPersist(){
        Paid paid = new Paid();
        BeanUtils.copyProperties(this, paid);
        paid.publishAfterCommit();    
    }
}
```
- 배송 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다

```
package carshare;

...

@Service
public class PolicyHandler{

   @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaid_Ship(@Payload Paid paid){

        if(paid.isMe()){
            Delivery delivery = new Delivery();
            delivery.setOrderId(paid.getOrderId());
            delivery.setPaymentId(paid.getId());
            delivery.setStatus("Shipped");

            deliveryRepository.save(delivery) ;
        }
    }

}

```

배송 서비스는 접수/결제 서비스와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 배송 서비스가 유지보수로 인해 잠시 내려간 상태라도 접수신청을 받는데 문제가 없다.

```
#배송(delivery) 서비스를 잠시 내려놓음 (ctrl+c)

#접수요청 처리
http localhost:8081/orders productId=1003 qty=2 status="order"   #Success
http localhost:8081/orders productId=1004 qty=4 status="order"   #Success

#접수상태 확인
http localhost:8081/orders     # 주문상태 안바뀜 확인

#배송 서비스 기동
cd carsharedelivery
mvn spring-boot:run

#접수상태 확인
http localhost:8081/orders     # 접수상태가 "shipped(배송됨)"으로 확인
```
