
# 비대면 개인과외 스토어앱

본 시스템은 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현 전단계를 커버하도록 구성한 시스템입니다.

# Table of contents

- [비대면 개인과외 스토어앱](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)

# 서비스 시나리오

기능적 요구사항
1. 개인과외 구매고객은 비대면 개인과외를 선택하여 구매를 요청한다.
1. 구매요청 되면 결제시스템에서 결제가 완료된다.
1. 송금시스템에서 판매고객 계좌에 수수료를 제외하고 돈이 입금된다. 
1. 개인과외 판매고객은 비대면 개인과외를 등록하여 판매를 요청한다.
1. 판매승인이 요청되면 승인시스템에서 관리자가 조회한 후 승인한다.
1. 판매요청이 승인되면 강의시스템에 개인과외가 등록된다.
1. 개인과외 구매/판매완료 고객은 강의시스템에서 실시간 화상강의를 진행한다.
1. 개인과외 구매고객은 구매를 취소할 수 있다.
1. 구매가 취소되면 결제가 취소된다.
1. 개인과외 구매고객이 판매대상 개인과외를 중간 중간 조회한다.
1. 개인과외 판매고객이 자신이 등록한 개인과외 상태를 중간 중간 조회한다.
1. 승인시스템 관리자가 판매 승인요청 대상을 중간 중간 조회한다.

비기능적 요구사항
1. 장애격리
    1. 과외승인 기능이 수행되지 않더라도 구매/판메 요청은 365일 24시간 받을 수 있어야 한다. Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 승인시스템 관리자가 결제를 잠시후에 하도록 유도한다. 고객에게는 Pending상태로 보여준다. Circuit breaker, fallback
1. 성능
    1. 구매고객이 구매요청 최종상태를 스토어시스템(프론트엔드)에서 확인할 수 있어야 한다. CQRS
    1. 판매고객이 구매요청 최종상태를 스토어시스템(프론트엔드)에서 확인할 수 있어야 한다. CQRS
    1. 승인관리자가 숙소요청상태를 승인시스템(프론트엔드)에서 확인할 수 있어야 한다. CQRS

# 분석/설계

### 이벤트 도출
![es-01](https://user-images.githubusercontent.com/63624005/81627098-63b35d00-9438-11ea-9fcd-6908cd746957.jpeg)


### 부적격 이벤트 탈락
![es-02](https://user-images.githubusercontent.com/63624005/81627175-91000b00-9438-11ea-80cd-45cf02f9e909.jpg)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
      . :숙소 재고가능 확인됨", "숙소 재고불가능 확인됨" 제외 : UI 이벤트 (업무적인 의미의 이벤트 아님)


### 액터, 커맨드, 폴리시 부착
![es-03](https://user-images.githubusercontent.com/63624005/81627702-bb9e9380-9439-11ea-8748-a64ba9e2a677.jpeg)


### 어그리게잇으로 묶기
![es-04](https://user-images.githubusercontent.com/63624005/81630276-1fc45600-9440-11ea-82a9-1a9732a10b44.jpg)


### 바운디드 컨텍스트로 묶기
![es-05](https://user-images.githubusercontent.com/63624005/81630293-2ce14500-9440-11ea-9072-5126f89d21c8.jpg)

![ee-05-1](https://user-images.githubusercontent.com/63624005/81630449-96f9ea00-9440-11ea-8631-678dc09c9dec.jpeg)

    - 도메인 서열 분리
      . Core Domain : 예약 
      . Supporting Domain : 숙소관리
      . General Domain : 결제


### 폴리시의 이동과 컨텍스트 매핑
![es-06](https://user-images.githubusercontent.com/63624005/81630868-a2014a00-9441-11ea-94a5-83f64c2f41b1.jpeg)


### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증
![es-06-0](https://user-images.githubusercontent.com/63624005/81761831-f241e000-9505-11ea-9266-e2d396bab028.jpg)

![es-06-old](https://user-images.githubusercontent.com/63624005/81771800-ef9fb480-951e-11ea-856c-f0eca4414cdc.jpg)

    - 고객이 숙소를 선택하여 예약한다. (ok)
    - 예약이 발생하면 숙소관리시스템에서 관리자가 예약을 확정한다. (ok)
    - 예약이 확정되면 결제시스템에서 결제가 완료된다. --> 결제 실패시?
    - 예약이 불가하면 숙소관리시스템에서 예약을 반려한다. (ok)
    
    - 고객이 예약을 취소할 수 있다. (ok)
    - 예약이 취소되면 결제가 취소된다. (ok)
    - 고객이 숙소 예약상태를 중간중간 조회한다. (ok)
    - 관리자가 숙소 요청상태를 중간중간 조회한다. (ok)


### 수정 모형    
    - 모든 요구사항을 커버함.

![es-06-1](https://user-images.githubusercontent.com/63624005/81762482-aee87100-9507-11ea-8a6c-c58679dab0f1.jpg)
    
    
    - 빠른 고객응답 보다는 서비스의 안정성을 더욱 중시하는 비즈니스적인 이유로 수정하였다. (Pub/Sub)
    
<img width="1102" alt="es-07" src="https://user-images.githubusercontent.com/63624005/81761227-5cf21c00-9504-11ea-9d43-2b6679d85b12.png">


### 비기능 요구사항에 대한 검증
![es-08](https://user-images.githubusercontent.com/63624005/81761245-68ddde00-9504-11ea-8a48-35891592d982.jpg)

    - 트랜잭션 (1)
      . 고객이 예약한 건에 대하여 관리자가 예약가능 여부를 확인한 후에 결제를 진행한다.
    - 장애격리 (2)
      . 숙소관리 기능이 수행되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다. 
        [Async (event-driven), Eventual Consistency]
      . 결제시스템이 과중되면 관리자가 결제를 잠시후에 하도록 유도한다. 고객에게는 Pending상태로 보여준다. 
    - 성능 (3)
      . 고객이 숙소에 대한 최종 예약상태를 예약시스템(프론트엔드)에서 확인할 수 있어야 한다. [CQRS]
      . 관리자가 숙소요청상태를 숙소관리시스템(프론트엔드)에서 확인할 수 있어야 한다. [CQRS]


## 헥사고날 아키텍처 다이어그램 도출
- 1차
![es-09](https://user-images.githubusercontent.com/63624005/81773899-56739c80-9524-11ea-97dc-cb1b968c2046.jpg)

    - 빠른 고객응답 보다는 서비스의 안정성을 더욱 중시하는 비즈니스적인 이유로 수정하였다. (Pub/Sub)

- 최종
![es-09-1](https://user-images.githubusercontent.com/63624005/81773921-5e334100-9524-11ea-855a-a72b69b77503.jpg)


# 구현

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다. (각자의 포트넘버는 8081 ~ 808n 이다)

```
# reservationservice //port number: 8081
cd reservation
mvn spring-boot:run

# managementservice //port number: 8082
cd management
mvn spring-boot:run

# paymentservice //port number: 8083
cd payment
mvn spring-boot:run
```

## DDD 의 적용

각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity로 선언하였다(예시는 reservationsystem 마이크로 서비스).  
  이때 가능한 현업에서 사용하는 언어(유비쿼터스 랭귀지)를 그대로 사용하여 모델링시 영문화 하였다.

```
package roomreservation;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Reservation_table")
public class Reservation {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String userId;
    private String reserveId;
    private String status;

    @PostPersist
    public void onPostPersist(){
        Reserved reserved = new Reserved();
        BeanUtils.copyProperties(this, reserved);
        reserved.publish();
    }
    @PostRemove
    public void onPostRemove(){
        ReservationCanceled reservationCanceled = new ReservationCanceled();
        BeanUtils.copyProperties(this, reservationCanceled);
        reservationCanceled.publish();

    }

    public Long getId() {
        return id;
    }
    public void setId(Long id) {
        this.id = id;
    }
    public String getUserId() {
        return userId;
    }
    public void setUserId(String userId) {
        this.userId = userId;
    }
    public String getReserveId() {
        return reserveId;
    }
    public void setReserveId(String reserveId) {
        this.reserveId = reserveId;
    }
    public String getStatus() {
        return status;
    }
    public void setStatus(String status) {
        this.status = status;
    }

}
```

Entity Pattern과 Repository Pattern을 적용하여 JPA를 통하여 다양한 데이터소스 유형(RDB)에 대한 별도의 처리가 없도록, 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST의 RestRepository를 적용하였다.

```
package roomreservation;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface ReservationRepository extends PagingAndSortingRepository<Reservation, Long>{
```


## 적용 후 REST API 의 테스트
- reservation 서비스에서 예약요청 
```  
  http localhost:8081/reservations reserveId=”reserve1” userId=”user1” status=”reserve”
```   

![dv-01](https://user-images.githubusercontent.com/63624005/81763734-df7dda00-950a-11ea-9793-34abab44c077.png)


- management 서비스 확인

![dv-02](https://user-images.githubusercontent.com/63624005/81763750-e9074200-950a-11ea-8d9a-f533be2ffbde.png)
  

- managementList 서비스에서 reserveId 저장 확인
```  
http localhost:8082/managementLists/1
``` 

![dv-03](https://user-images.githubusercontent.com/63624005/81763766-f15f7d00-950a-11ea-9ea3-d138ee246485.png)


- management 서비스의 승인처리
```  
http localhost:8082/managements reserveId=”reserve1”
``` 

![dv-04](https://user-images.githubusercontent.com/63624005/81763782-f9b7b800-950a-11ea-94d2-b6c9d96e9c59.png)


- payment 서비스 확인

![dv-05](https://user-images.githubusercontent.com/63624005/81763795-03412000-950b-11ea-8597-a3c0713cd5fd.png)
  

- kafka 수신 확인
 
![dv-06](https://user-images.githubusercontent.com/63624005/81763810-0b995b00-950b-11ea-99fa-13e089a3060b.png)
  


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

모든 시스템은 장애 상태나 유지보수 상태일 때, 잠시 시스템이 내려가도 작동에 문제가 없게 하기 위해 동기식이 아닌 비동기식으로 처리한다. 그 중 고객의 예약이 이루어진 후에 Management시스템을 이를 알려주는 행위를 예를 들었다.

- reservation 시스템에 고객의 예약을 받았다는 기록을 남긴 후에 예약이 되었다는 도메인 이벤트를 카프카로 송출한다.(Publish)

```
@PostPersist
    public void onPostPersist() {
        Reserved reserved = new Reserved();
        BeanUtils.copyProperties(this, reserved);
        reserved.publish();
    }
```

- management 시스템에서는 예약완료 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:
```
public class PolicyHandler{
...

@StreamListener(KafkaProcessor.INPUT)
    public void wheneverReserved_ReserveConfirm(@Payload Reserved reserved){

        if(reserved.isMe()){
            System.out.println("##### listener ReserveConfirm : " + reserved.toJson());
        }
    }
```

management와 시스템은 reservation, payment 시스템과 완전히 분리되어 있으며, 이벤트 수신에 따라 처리되기 때문에 시스템이 유지보수로 인해 잠시 내려간 예약을 받아 managementList에 저장하는데에 아무 문제 없다.


- management 시스템을 잠시 내려 놓음

![dv-11](https://user-images.githubusercontent.com/63624005/81765091-eeb25700-950d-11ea-92f6-436329052752.png)


- 예약 처리
```
http POST localhost:8081/reservations reserveId=”reserve2” userId=”user1”   #Success
http POST localhost:8081/reservations reserveId=”reserve3” userId=”user1”   #Success
```

- 예약상태 확인
```
http localhost:8081/reservations     
``` 

![dv-12](https://user-images.githubusercontent.com/63624005/81765107-f70a9200-950d-11ea-8e3b-029d9b1bb629.png)


- 예약 완료 상태까지 Event 진행 확인

![dv-13](https://user-images.githubusercontent.com/63624005/81765130-012c9080-950e-11ea-84d3-9d3a4f6136ba.png)


- management 시스템 재기동 후 management 시스템에 Update 되었는지 확인(CQRS)
  고객이 숙소에 예약 신청한 내역을 managementList view에서 확인할 수 있다.

![dv-14](https://user-images.githubusercontent.com/63624005/81765144-0984cb80-950e-11ea-9c98-dd84597c3825.png)

![dv-15](https://user-images.githubusercontent.com/63624005/81765184-1b666e80-950e-11ea-8722-60464240fe71.png)

![dv-16](https://user-images.githubusercontent.com/63624005/81765197-228d7c80-950e-11ea-8ff6-835dbc760c26.png)


## API 게이트웨이

 Clous 환경에서는Clous 환경에서는 //서비스명:8080 에서 Gateway API가 작동해야함 
 application.yml 파일에 profile별 gateway 설정

- Gateway 설정 파일 
 GATEWAY 

![dv-17](https://user-images.githubusercontent.com/63624005/81765217-2caf7b00-950e-11ea-8d8b-935347e5dfc8.png)



# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, pipeline build script 는 각 프로젝트 폴더 이하에 azure-pipelines.yml 에 포함되었다.

*devops를 활용하여 pipeline을 구성하였고, CI CD 자동화를 구현하였다.

![image](https://user-images.githubusercontent.com/63624035/81771610-71431280-951e-11ea-91e9-8498a62e636e.png)

* 아래와 같이 pod 가 정상적으로 올라간 것을 확인하였다.
![image](https://user-images.githubusercontent.com/63624035/81761872-156c8f80-9506-11ea-8c23-55f8d347a2c8.png)

* 아래와 같이 쿠버네티스에 모두 서비스로 등록된 것을 확인할 수 있다.
![image](https://user-images.githubusercontent.com/63624035/81762154-de4aae00-9506-11ea-99b1-1b6068e34547.png)

### 오토스케일 아웃
시스템을 안정되게 운영할 수 있도록 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

- 안정성이 중요한 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy payment --min=1 --max=10 --cpu-percent=15
```
- 워크로드를 동시사용자 10명으로 20초 동안 걸어준다.
![시즈적용_10](https://user-images.githubusercontent.com/63624014/81764751-3edce980-950d-11ea-806e-d8f51a26c46d.PNG)

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다. 동시 사용자 10명으로 한 경우에는 시스템에 변화가 없다.
![시즈적용_10_3](https://user-images.githubusercontent.com/63624014/81764796-574d0400-950d-11ea-88d4-56428f5be633.PNG)

- siege 의 로그를 보명 10명까지는 성능에 문제가 없어보인다. 
![시즈적용_10_2](https://user-images.githubusercontent.com/63624014/81764775-4c926f00-950d-11ea-93b8-ae86f7bb4cf5.PNG)



- 워크로드를 동시사용자 100명으로 20초 동안 걸어준다.
![시즈적용_100_1](https://user-images.githubusercontent.com/63624014/81766034-42be3b00-9510-11ea-8682-dab260440772.PNG)

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다. 시스템이 중간에 멈추는 경우가 발생한다.
![시즈적용_100_2](https://user-images.githubusercontent.com/63624014/81766051-4d78d000-9510-11ea-82c2-6a830718042c.PNG)

- siege 의 로그를 보명 100일 때는 95%정도의 서비스 Available을 유지하고, SLA 수준에 따라 오토스케일 아웃을 지속적으로 조정한다.
![시즈적용_100_3](https://user-images.githubusercontent.com/63624014/81766079-59649200-9510-11ea-96c5-dc233ed007b6.PNG)


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.

![image](https://user-images.githubusercontent.com/63624035/81765032-ccb8d480-950d-11ea-9ca8-ec492af06c01.png)

![image](https://user-images.githubusercontent.com/63624035/81764389-6da69000-950c-11ea-98d6-114141561d3d.png)

- 새버전으로의 배포 시작

![image](https://user-images.githubusercontent.com/63624035/81765398-ac3d4a00-950e-11ea-8e3b-a01e66031559.png)

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:		        3078 hits
Availability:		       70.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```
배포기간중 Availability 가 평소 100%에서 70% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

![image](https://user-images.githubusercontent.com/63624035/81766634-78afef00-9511-11ea-8573-23a287118556.png)


- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:		        3078 hits
Availability:		       100 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


