# meetingroom


# 서비스 시나리오

1. 비품 관리를 위한 서비스를 미리 생성한다.
2. 회의실 등록을 하면 비품이 등록된다.
3. 관리자에 의해 등록된 비품은 수정될 수 있다.
4. 회의실을 제거하면 비품 항목도 모두 삭제한다.


비기능적 요구사항
1. 트랜잭션
  - 회의실 삭제 시 비품서비스가 없다면 삭제하지 못하게 한다. 생성단계부터 오류가 있는 것으로 확인 필요 (Sync 호출) 
2. 장애격리
  - 비품 서비스가 수행되지 않더라도 회의실 생성은 365일 24시간 생성되어야 회의 진행하는데 문제가 없다. (Async 호출)
3. 성능
  - 비품 현황에 대해 별도록 확인할 수 있어야 한다. (CQRS)


# 체크포인트
https://workflowy.com/s/assessment/qJn45fBdVZn4atl3


# 모형
![모형](https://user-images.githubusercontent.com/78134049/109769512-a8191f00-7c3d-11eb-88bb-334660ee98be.png)

# 기능적/비기능적 요구사항에 대한 검증
![모형2](https://user-images.githubusercontent.com/78134049/109780714-9ab66180-7c4a-11eb-8437-438f08662859.png)

1. 비품 등록 서비스를 생성한다. (1)
2. 관리자가 회의실을 생성 및 등록한다.(2)
   - 생성하면 비품이 자동 등록된다. (2 -> 4)
3. 관리자가 회의실을 삭제 한다.(3) 
   - 삭제하면 비품 항목을 전부 삭제한다. (3 -> 5)
4. Stock 메뉴에서 회의실에 대한 예약 정보를 조회한다.(6)

# 헥사고날 아키텍쳐 다이어그램 도출 (Polyglot)
![polyglot](https://user-images.githubusercontent.com/78134049/109812524-82a50900-7c6f-11eb-86e2-f0e2ae8d1ce2.png)

# 구현
도출해낸 헥사고날 아키텍처에 맞게, 로컬에서 SpringBoot를 이용해 Maven 빌드 하였다. 각각의 포트넘버는 8081 ~ 8086, 8088 이다.

```linux
cd conference
mvn spring-boot:run

cd gateway
mvn spring-boot:run

cd reserve
mvn spring-boot:run

cd schedule
mvn spring-boot:run

cd supplies
mvn spring-boot:run

cd stock
mvn spring-boot:run
```

# DDD의 적용
supplies 서비스의 supplies.java
```java
package meetingroom;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Supplies_table")
public class Supplies {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id; 
    private int phone;
    private int pc;
    private int beam;

    @PrePersist
    public void onPrePersist(){
    }


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public int getPhone() {
        return phone;
    }

    public void setPhone(int phone) {
        this.phone = phone;
    }
    public int getPc() {
        return pc;
    }

    public void setPc(int pc) {
        this.pc = pc;
    }
    public int getBeam() {
        return beam;
    }

    public void setBeam(int beam) {
        this.beam = beam;
    }
}
```
supplies 서비스의 PolicyHandler.java
```java
package meetingroom;

import meetingroom.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

import java.util.Optional;

@Service
public class PolicyHandler{
    @Autowired
    SuppliesRepository suppliesRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverAdded_Use(@Payload Added added){

        if(added.isMe()){
            long roomid = added.getId();
            Optional<Supplies> supplies = suppliesRepository.findById(roomid);
            System.out.println("##### listener  : " + added.toJson());
            if (supplies.isPresent()){
                // default 값으로 각 비품을 1개씩 자동 할당함
                supplies.get().setBeam(1);
                supplies.get().setPc(1);
                supplies.get().setPhone(1);
                suppliesRepository.save(supplies.get());
            }
        }
    }

}
```

- 적용 후 REST API의 테스트를 통해 정상적으로 작동함을 알 수 있었다.
 < 스샷 >
- 비품 서비스 생성 후 결과
  < 스샷 >
- 회의실 생성 후 결과
   < 스샷 >

# Gateway 적용
API Gateway를 통해 마이크로 서비스들의 진입점을 하나로 진행하였다.
 - Conference 서비스의 경우, 다른 서비스들이 h2 저장소를 이용한 것과는 다르게 hsql을 이용하였다.
 - 이 작업을 통해 서비스들이 각각 다른 데이터베이스를 사용하더라도 전체적인 기능엔 문제가 없음을, 즉 Polyglot Persistence를 충족하였다.
```yml
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: conference
          uri: http://localhost:8081
          predicates:
            - Path=/conferences/** 
        - id: reserve
          uri: http://localhost:8082
          predicates:
            - Path=/reserves/** 
        - id: room
          uri: http://localhost:8083
          predicates:
            - Path=/rooms/** 
        - id: schedule
          uri: http://localhost:8084
          predicates:
            - Path= /reserveTables/**
        - id: room
          uri: http://localhost:8085
          predicates:
            - Path=/stockTables/** 
        - id: room
          uri: http://localhost:8086
          predicates:
            - Path=/suppliess/** 
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true


---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: conference
          uri: http://conference:8080
          predicates:
            - Path=/conferences/** 
        - id: reserve
          uri: http://reserve:8080
          predicates:
            - Path=/reserves/** 
        - id: room
          uri: http://room:8080
          predicates:
            - Path=/rooms/** 
        - id: schedule
          uri: http://schedule:8080
          predicates:
            - Path= /reserveTables/**
        - id: stock
          uri: http://stock:8080
          predicates:
            - Path= /stockTables/**
        - id: supplies
          uri: http://supplies:8080
          predicates:
            - Path= /suppliess/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

server:
  port: 8080

```


# Polyglot Persistence
 - stock 서비스의 경우, 다른 서비스들이 h2 저장소를 이용한 것과는 다르게 hsql을 이용하였다.
 - 이 작업을 통해 서비스들이 각각 다른 데이터베이스를 사용하더라도 전체적인 기능엔 문제가 없음을, 즉 Polyglot Persistence를 충족하였다.
```xml
		<!-- <dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency> -->
		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<version>2.4.1</version>
		</dependency>
```

# 동기식 호출(Req/Res 방식)과 Fallback 처리
 - room 서비스의 external/SuppliesService.java 내에 supplies 서비스가 존재하는지 확인하는 Service 대행 인터페이스(Proxy)를 FeignClient를 이용하여 구현하였다.
```java
@FeignClient(name="supplies", url="${api.supplies.url}")
public interface SuppliesService {

    @RequestMapping(method= RequestMethod.GET, path="/supplies/return")
    public String s_return(@RequestBody Supplies supplies);
}
```
 
 - room 서비스의 room.java 내에 supplies 서비스 존재 확인 후 삭제할지, 않을지 결정.(@PrePersist)
```java
    @PrePersist
    public void onPrePersist(){
//        Searched searched = new Searched();
//        BeanUtils.copyProperties(this, searched);
//        searched.publishAfterCommit();

        meetingroom.external.Supplies supplies = new meetingroom.external.Supplies();
        // mappings goes here
        supplies.setId(id);
        String result = RoomApplication.applicationContext.getBean(meetingroom.external.SuppliesService.class).s_return(supplies);

        if(result.equals("valid")){
            System.out.println("Success!");
        }
        else{
            /// room 번호가 유효하지 않을 때 강제로 예외 발생
                System.out.println("FAIL!! InCorrect Room Nubmer or Not exist");
                Exception ex = new Exception();
                ex.notify();
        }

    }
```
 
 - 동기식 호출에서는 호출 시간에 따른 커플링이 발생하여, Supplies 시스템에 장애가 나면 회의실을 삭제할 수 없다.
   -> room 서비스에서 회의실 삭제 시 에러 발생
   < 결과 스샷 >

# 비동기식 호출 (Pub/Sub 방식)
 - room 서비스 내 Room.java에서 아래와 같이 서비스 Pub 구현
```java
@Entity
@Table(name="Room_table")
public class Room {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String status;
    private Integer floor;

    @PostPersist
    public void onPostPersist(){
        Added added = new Added();
        BeanUtils.copyProperties(this, added);
        added.publishAfterCommit();
    }
```
 
 - supplies 서비스 내 PolicyHandler.java 에서 아래와 같이 Sub 구현
```java
@Service
public class PolicyHandler{
    @Autowired
    SuppliesRepository suppliesRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverAdded_Use(@Payload Added added){

        if(added.isMe()){
            long roomid = added.getId();
            Optional<Supplies> supplies = suppliesRepository.findById(roomid);
            System.out.println("##### listener  : " + added.toJson());
            if (supplies.isPresent()){
                // default 값으로 각 비품을 1개씩 자동 할당함
                supplies.get().setBeam(1);
                supplies.get().setPc(1);
                supplies.get().setPhone(1);
                suppliesRepository.save(supplies.get());
            }
        }
    }

}
```

- 비동기 호출은 다른 서비스 하나가 비정상이어도 해당 메세지를 다른 메시지 큐에서 보관하고 있기에, 서비스가 다시 정상으로 돌아오게 되면 그 메시지를 처리하게 된다.
  -> supplies 서비스를 내려도 room 생성에는 문제 없이 동작한다.
   < 스샷 >
   
# CQRS
viewer인 stock 서비스를 별도로 구현하여 아래와 같이 view를 출력한다.
  < 스샷 >
  
# 운영
# CI/CD 설정
  - git에서 소스 가져오기
  - Build 하기
  - Dockerlizing, ACR(Azure Container Registry에 Docker Image Push하기
  - ACR에서 이미지 가져와서 Kubernetes에서 Deploy하기
  - Kubectl Deploy 결과 확인
  - Kubernetes에서 서비스 생성하기 (Docker 생성이기에 Port는 8080이며, Gateway는 LoadBalancer로 생성)
  - Kubectl Expose 결과 확인
 
# 오토스케일 아웃

# ConfigMap 적용

# 동기식 호출 / 서킷 브레이킹 / 장애격리

# 무정지 재배포

# Self-healing (Liveness Probe)
