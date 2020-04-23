

# 중고거래 시스템 구축

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.

# Table of contents
#### 중고거래 시스템
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현](#구현)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)


# 서비스 시나리오
기능적 요구사항
1. 판매자가 상품을 등록한다.  
1. 구매자는 등록된 상품을 예약한다. 
1. 예약 후 상품 상태가 변경 된다. 
1. 예약 후 결제 정보가 생성 된다
1. 구매자가 예약한 상품을 결제한다.
1. 결제 후 상품 정보를 변경한다.
1. 구매자가 예약을 취소한다.
1. 취소 후 상품 정보를 상태를 변경한다.
1. 구매자는 예약 및 결제 상태를 중간중간 조회한다.
1. 예약 및 결제 상태가 바뀔 때 마다 카톡으로 알림을 보낸다.
1. 상태정보를 뷰에 로깅한다

비기능적 요구사항
1. 트랜잭션
    1. 예약된 상품은 상품 정보 상태를 바로 수정하고 예약하여 추가로 예약되어서는 안된다  Sync 호출 
1. 장애격리
    1. 예약 기능이 되지 않더라도 결제는 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 예약이 과중되면 사용자를 잠시동안 받지 않고 예약을 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 에약내역 및 대여상태를 my-page(프론트엔드)에서 확인할 수 있어야 한다  CQRS
    1. 예약 및 대여상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다  Event driven
    
# 서비스 실행 결과

* 상품상태:  ```01(available)``` ```02(pending)``` ```soldOut```
* 예약상태:  ```01(reserved)``` ```02(cancle)```
* 결제상태:  ```01(Unpaid)``` ```02(Paid)``` 

1. 판매자가 상품을 등록한다.  < 상품 상태  = 01 (available) >
```sh
http http://localhost:8081/product productName=Noodle productStatus=01
http http://localhost:8081/product productName=Desk productStatus=01
http http://localhost:8081/product productName=Coffee productStatus=01
```
- 결과확인 

``` sh
http http://localhost:8082/reservations
```
![image](https://user-images.githubusercontent.com/4526087/80005927-32adcf80-84ff-11ea-95bb-f6b3182d182d.png)


2. 구매자는 등록된 상품을 예약한다. < 예약 상태  = 01 (reserved) >
```sh
http http://localhost:8082/reservation productId=1 reservationStatus=01
http http://localhost:8082/reservation productId=2 reservationStatus=01
```
- 결과확인 

``` sh
http http://localhost:8082/reservations
```
![image](https://user-images.githubusercontent.com/4526087/80006871-705f2800-8500-11ea-90f1-ae09b7d545bb.png)

3. 예약 후 상품 상태가 업데이트된다.  <상품 상태 = 02 (pending) >
- 결과확인
``` sh
http http://localhost:8081/products
```
![image](https://user-images.githubusercontent.com/4526087/80007781-bc5e9c80-8501-11ea-8d8e-dc44da739832.png)


4. 예약 후 결제 정보가 생성 된다.< 결제 상태 = 01 (unpaid) >  
- 결과확인
``` sh
http http://localhost:8083/payments
```
![image](https://user-images.githubusercontent.com/4526087/80008790-2f1c4780-8503-11ea-905a-bab31f95dc62.png)

5. 구매자가 예약한 상품을 결제 한다 < 결제 상태 = 02 (paid) >
```sh
http PATCH http://localhost:8083/paid id=1 reservationId=1 productId=2 paymentStatus=02
```
- 결과확인
``` sh
http http://localhost:8083/payments
```
![image](https://user-images.githubusercontent.com/4526087/80008939-55da7e00-8503-11ea-9cfd-cbe52dc1c978.png)

6. 결제 후 상품 정보를 업데이트 한다. < 상품 상태 = soldOut >
- 결과 확인
```sh
http http://localhost:8081/products
```
![image](https://user-images.githubusercontent.com/4526087/80009089-8e7a5780-8503-11ea-986c-d8384c541462.png)

7. 구매자가 예약을 취소한다. < 예약 상태 = 02 (cancel) >
```sh
http PATCH http://localhost:8082/reservationupdate id=2 productId=1 reservationStatus=02
```
- 결과 확인
``` sh
http http://localhost:8082/reservations
```

* 취소 전

![image](https://user-images.githubusercontent.com/4526087/80009276-d0a39900-8503-11ea-975f-b3b30eb19c49.png)

* 취소 후

![image](https://user-images.githubusercontent.com/4526087/80009488-19f3e880-8504-11ea-8a98-0fd26711cab8.png)

8. 취소 후 상품 정보를 상태를 업데이트한다. < 상품 상태 =01 (available) >
- 결과 확인
``` sh
http http://localhost:8081/products
```
![image](https://user-images.githubusercontent.com/4526087/80009321-dd27f180-8503-11ea-9db8-61c43523e0fe.png)

1. 고객이 예약 및 결제 상태를 중간중간 조회한다.
``` sh
http://localhost:8084/myPages
```
![image](https://user-images.githubusercontent.com/4526087/80009842-938bd680-8504-11ea-98f9-3b64ff06c985.png)

1. 예약 및 결제 상태가 바뀔 때 마다 카톡으로 알림을 보낸다.

![image](https://user-images.githubusercontent.com/4526087/80009974-b918e000-8504-11ea-9dbc-0582920c461b.png)

1. 상태정보를 뷰에 로깅한다



# 서비스 실행 결과


# CI/CD 실행 결과

* 파이프라인

![CLOUD과제_devops (azure)](https://user-images.githubusercontent.com/4526087/80011322-ab645a00-8506-11ea-8ce7-f14f4f687fbd.png)

* POD 실행 화면

![CLOUD과제_POD실행화면 (azure)](https://user-images.githubusercontent.com/4526087/80011600-0eee8780-8507-11ea-8a80-85ea045d9490.png)


* 클라우드 內 시나리오 수행

- 상품 등록

![image](https://user-images.githubusercontent.com/4526087/80011861-6987e380-8507-11ea-9ec5-bf416a7651c8.png)

- 상품 확인

![image](https://user-images.githubusercontent.com/4526087/80011999-9a681880-8507-11ea-9381-5f0ce91b35a3.png)

- 예약
``` sh
http http://reservation:8080/reservation productId=3 reservationStatus=01
http http://reservation:8080/reservation productId=2 reservationStatus=01
```

- 예약 확인

![image](https://user-images.githubusercontent.com/4526087/80012085-bd92c800-8507-11ea-9db8-08db78f60290.png)

- 예약 후 상품 상태 변화 (02)

![image](https://user-images.githubusercontent.com/4526087/80012187-e1560e00-8507-11ea-910e-02ffd983d075.png)


-
- 예약 후 결제 생성 

![image](https://user-images.githubusercontent.com/4526087/80012227-f03cc080-8507-11ea-9158-5650943b531f.png)


- 구매
``` sh
http PATCH http://payment:8080/paid id=1 reservationId=1 productId=3 paymentStatus=02
```
![image](https://user-images.githubusercontent.com/4526087/80012321-1b271480-8508-11ea-8440-0a59ec63a97d.png)

- 구매 후 상품 변화 (soldOut)

![image](https://user-images.githubusercontent.com/4526087/80012396-3eea5a80-8508-11ea-80bc-b87a46624dee.png)

- 예약 취소 전 상태 

![image](https://user-images.githubusercontent.com/4526087/80012441-4f9ad080-8508-11ea-9004-643416ffeadd.png)

- 예약 취소 후 상태

![image](https://user-images.githubusercontent.com/4526087/80012498-5e818300-8508-11ea-981b-78652e4b2bb4.png)

- myPage 조회
``` sh 
root@httpie:/# http http://mypage:8080/myPages

```
![image](https://user-images.githubusercontent.com/4526087/80012586-7d801500-8508-11ea-93a1-e0afd4bbcf43.png)


# 분석/설계


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: http://msaez.io/#/storming/cVNuZs0oJidKR8D7ZWW4anjQhQA2/mine/83e2fd722bb1a93822e7e29d47db0227/-M5Tg016oQ03UwFRn94Q

![image](https://user-images.githubusercontent.com/4526087/80013017-2af32880-8509-11ea-98cb-99f598ab0931.png)

- Core Domain : Prdouct(상품) / Resercation (예약) / Payment (구매)
- Supporting Domain : MyPage (CQRS)
- General Domain : Notice(알림)



### 이벤트 도출
![image](https://user-images.githubusercontent.com/4526087/80012850-ebc4d780-8508-11ea-8a2f-19f4b7442921.png)


### 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/4526087/80013202-802f3a00-8509-11ea-9a70-10f216a8aa8f.png)

### 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/4526087/80013299-a1902600-8509-11ea-880d-e690ae0e6f0a.png)


- app의 Order, store 의 주문처리, 결제의 결제이력은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌

### 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/4526087/80013464-e2883a80-8509-11ea-990e-d42b22e382d3.png)


### 폴리시 부착 및 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![image](https://user-images.githubusercontent.com/4526087/80013586-12374280-850a-11ea-846b-981210bdd8c7.png)


## 헥사고날 아키텍처 다이어그램 도출
 
 ![image](https://user-images.githubusercontent.com/4526087/80013987-af927680-850a-11ea-81bd-9d42c8e13d21.png)
![image](https://user-images.githubusercontent.com/4526087/80014045-b8834800-850a-11ea-9b7c-95f8b8657690.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

중고거래 시스템은 아래의 6가지 마이크로 서비스로 구성되어 있다.

- 게이트 웨이: https://github.com/gogohs/gateway
- 예약 시스템: https://github.com/gogohs/reservation
- 결제 시스템: https://github.com/gogohs/payment
- 상품 시스템: https://github.com/gogohs/product.git
- 알림 시스템: https://github.com/gogohs/notice.git
- 조회 시스템: https://github.com/gogohs/mypage.git

모든 시스템은 Spring Boot로 구현하였고 mvn spring-boot:run 명령어로 실행할 수 있다.

## DDD 의 적용
각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 Reservation microservice). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다.

```
package market;

import javax.persistence.*;

import market.external.Product;
import market.external.ProductService;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Reservation_table")
public class Reservation {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String reservationStatus;
    private Integer productId;
    private String type;

    public String getType(){ return type;}
    public void setType(String type){this.type = type;}

    @PostPersist
    public void onPostPersist(){
        Reserved reserved = new Reserved();
        BeanUtils.copyProperties(this, reserved);
        reserved.publish();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.
        System.out.println("Type : "+ type);
        market.external.Product product = new market.external.Product();
        product.setId(this.getProductId());
        if(type.equals("Reserved")){
            product.setProductStatus("02");
        }else{
            product.setProductStatus("01");
        }
        // mappings goes here
        Application.applicationContext.getBean(market.external.ProductService.class)
            .productStatusChange(product);
    }


    @PostUpdate
    public void onPostUpdate() {
        ReservationCanceled reservationCanceled = new ReservationCanceled();
        BeanUtils.copyProperties(this, reservationCanceled);
        reservationCanceled.publish();

        Product product = new Product();

        product.setId(this.getProductId());

        product.setProductStatus("01"); // 예약됨

        ProductService bookService = Application.applicationContext.getBean(ProductService.class);
        bookService.productStatusChange(product);
    }
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getReservationStatus() {
        return reservationStatus;
    }

    public void setReservationStatus(String reservationStatus) {
        this.reservationStatus = reservationStatus;
    }
    public Integer getProductId() {
        return productId;
    }

    public void setProductId(Integer productId) {
        this.productId = productId;
    }
}
```

- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다. RDB로는 H2를 사용하였다.
```
package market;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface ReservationRepository extends PagingAndSortingRepository<Reservation, Long>{

}
```
## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 예약관리(Reservation)-> 상품관리(Product) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. (ex) 특정 상품에 대하여 예약 취소 발생 시 상품의 상태는 avaiable(01)로 변경) 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 상품 관리 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (Reservation) ProductService.java

package market.external;

@FeignClient(name="Product", url="http://Product:8080") //  Local 수행시 http://localhost:8081
public interface ProductService {

    @RequestMapping(method= RequestMethod.PATCH, path="/productupdate")
    public void productStatusChange(@RequestBody Product product);

}
```

- 예약취소 직후(@PostUpdate) 상품 상태 변경 (Available / 01 ) 하도록 처리
```
# Reservation.java.java (Entity)

    @PostUpdate
    public void onPostUpdate() {
        ReservationCanceled reservationCanceled = new ReservationCanceled();
        BeanUtils.copyProperties(this, reservationCanceled);
        reservationCanceled.publish();

        Product product = new Product();

        product.setId(this.getProductId());

        product.setProductStatus("01"); // Available

        ProductService productService = Application.applicationContext.getBean(ProductService.class);
        productService.productStatusChange(product);
    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:


```
# 상품 (Product) 서비스를 잠시 내려놓음 (ctrl+c)

#예약 취소 처리
http PATCH http://localhost:8082/reservationupdate id=2 productId=1 reservationStatus=02
   #Fail


#상품 서비스 재기동
cd product
mvn spring-boot:run

#예약 취소 처리
http PATCH http://localhost:8082/reservationupdate id=2 productId=1 reservationStatus=02   #Success

```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)




## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


예약(reservation)이 발생한 후에 결제(payment) 시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 상점 시스템의 처리를 위하여 결제주문이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 예약이력에 기록을 남긴 후에 곧바로 예약이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package market;

@Entity
@Table(name="Reservation_table")
public class Reservation {

 ...
   @PostPersist
    public void onPostPersist(){
   
        Reserved reserved = new Reserved();
        BeanUtils.copyProperties(this, reserved);
        reserved.publish();

        ...
    }

}
```
- 결제 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
#(Payment)
package package market;

...

@Service
public class PolicyHandler{

    @Autowired
    PaymentRepository paymentRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverReserved_PayListRegist(@Payload Reserved reserved){

        if(reserved.isMe()){
            Payment payment = new Payment();
            payment.setProductId(reserved.getProductId());
            payment.setPaymentStatus(reserved.getReservationStatus());
            payment.setReservationId(Integer.parseInt(reserved.getId().toString()));
            paymentRepository.save(payment);
            System.out.println("##### listener PayListRegist : " + reserved.toJson());

        }
    }

}

```
알림 시스템은 실제로 문자를 보낼 수는 없으므로, 예약/변경/취소 이벤트에 대해서 System.out.println 처리 하였다.

```
package com.example.notice;

@Service
public class KafkaListener {
    @StreamListener(Processor.INPUT)
    public void onReservationReservedEvent(@Payload Reserved Reserved) {
        if(reservationReserved.getEventType().equals("ReservationReserved")) {
            System.out.println("예약 되었습니다.");
        }
    }

    @StreamListener(Processor.INPUT)
    public void onReservationChangedEvent(@Payload ReservationCanceled  reservationCanceled) {
        if(reservationChanged.getEventType().equals("ReservationChanged")) {
            System.out.println("예약 취소 되었습니다.");
        }
    }
}

```

myPage는 상품/예약/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, mypage가 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
```
# mypage 를 잠시 내려놓음 (ctrl+c)

#예약 처리
http http://localhost:8082/reservation productId=1 reservationStatus=01  #Success
http http://localhost:8082/reservation productId=2 reservationStatus=01  #Success

#예약 상태 확인
http http://localhost:8082/reservations     # 예약 추가 된 것 확인

# mypage 기동
cd mypage
mvn spring-boot:run

#이력 확인
http://localhost:8084/myPages    # 모든 주문의 상태가 변경된 것 확인
```


# 운영

## CI/CD 설정


각 구현체들은 각자의 Git을 통해 빌드되며, Git Master에 트리거 되어 있다. pipeline build script 는 각 프로젝트 폴더 이하에 azure_pipeline.yml 에 포함되었다.

azure_pipeline.yml 참고


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 예약(Reservation)-->상품상태 변경(Product) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(Product) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (Product) Product.java (Entity)

    @PostUpdate
    public void onPostUpdate(){  //상품 상태를 변경한 후 적당한 시간 끌기

        ...
        
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

``` 
$ siege -c100 -t60S -r10 --content-type "application/json" 'http://reservation:8080/reservation POST {"productId": "2",, "reservationStatus" : "01" }'

Windows 안에서 작동하는 Ubuntu에서 siege 실행시 "[error] unable to set close control sock.c:141: Invalid argument" 이 발생하여 중간 과정은 알 수 없음.

그러나 아래와 같은 결과를 확인.

Lifting the server siege...
Transactions:                   1067 hits
Availability:                  74.91 %
Elapsed time:                  59.46 secs
Data transferred:               0.37 MB
Response time:                  5.36 secs
Transaction rate:              17.94 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                   96.13
Successful transactions:        1067
Failed transactions:             285
Longest transaction:            7.01
Shortest transaction:           0.02

```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 74.21% 가 성공.


### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 상품 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy product --min=1 --max=10 --cpu-percent=15
```
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
``` 
$ siege -c100 -t60S -r10 --content-type "application/json" 'http://reservation:8082/reservation POST {"productId": "2", "reservationStatus" : "01" }'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
pay     1         1         1            1           17s
pay     1         2         1            1           45s
pay     1         4         1            1           1m
:
```
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
```
Transactions:		        5078 hits
Availability:		       92.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02
```


## 무정지 재배포

* 모든 프로젝트의 readiness probe 및 liveness probe 설정 추가
```
readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 10
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 10
livenessProbe:
  httpGet:
     path: /actuator/health
     port: 8080
  initialDelaySeconds: 120
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 5
```

