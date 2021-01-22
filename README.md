# 호텔 예약

> Winter School 2팀 Workspace입니다.



## 시나리오

호텔 예약 시스템에서 요구하는 기능/비기능 요구사항은 다음과 같습니다. 사용자가 예약과 함께 결제를 진행하고 나면 객실 관리자가 객실을 배정하는 시스템입니다. 이 과정에 대해서 고객은 진행 상황을 확인할 수 있고, 카카오톡으로 알림을 받을 수 있습니다. 

#### 기능적 요구사항

1. 고객이 원하는 객실을 선택 하여 예약한다.
2. 고객이 결제 한다.
3. 예약이 신청 되면 예약 신청 내역이 호텔에 전달 된다.
4. 호텔이 확인 하여 예약을 확정 한다.
5. 고객이 예약 신청을 취소할 수 있다.
6. 예약이 취소 되면 호텔 예약이 취소 된다.
7. 고객이 예약 진행 상황을 중간 중간 조회 한다.
8. 예약 상태가 바뀔 때 마다 카카오톡으로 알림을 보낸다.
9. 고객이 예약 취소를 하면 예약 정보는 삭제되나, 객실 관리팀에서는 이력 관리를 위해 취소 장부를 별도 저장한다.

#### 비 기능적 요구사항

1. 트랜잭션
   - 결제가 되지 않은 예약건은 아예 호텔 예약 신청이 되지 않아야 한다. `Sync 호출`

2. 장애격리
   - 객실관리 기능이 수행 되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다. `Pub/Sub`
   - 결제 시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도 한다.
     (장애처리)

3. 성능
   - 고객이 예약 확인 상태를 마이페이지에서 확인할 수 있어야 한다. `CQRS`
   - 예약 상태가 바뀔 때 마다 카톡 등으로 알림을 줄 수 있어야 한다.
   
# 분석/설계

## Event Storming

#### 초기 버전

1. 잘못된 이벤트 제거 : "호텔선택됨", "룸타입선택됨" 등 UI 이벤트 제거 
pp의 Order, store 의 주문처리, 결제의 결제이력은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌
2. order의 주문, reservation의 예약과 취소, payment의 결제 이력, 고객관리의 카카오톡 알림 등은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌(바운디드 컨텍스트)
3. 도메인 서열 분리 
   - Core Domain: Order, Reservation(없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.7% 목표, 배포주기는 1주일 1회 미만)
   - Supporting Domain: customermanagement(경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율)
   - General Domain: payment(결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높을 것으로 판단. 향후 External로 전환 필요)

![screenshot-miro com-2020 12 18-09_30_35](https://user-images.githubusercontent.com/76149887/102559719-e6f60900-4113-11eb-96c6-54829db72270.png)

#### 2020.12.21

- 1차 완성본

![screenshot-miro com-2020 12 21-13_50_26](https://user-images.githubusercontent.com/76149887/102837515-453a2900-443f-11eb-8293-e7f46bc94732.png)


#### 2020.12.22

- 예약 취소 건에 대한 추가 의견으로, 예약 취소는 이력 정보만 남기고 실제 예약 정보는 삭제하는 것으로 합의
- 예약 취소 시, 예약 취소에 대한 이력만을 저장하기 때문에 이 이력의 저장에 대한 동기 호출로 결제 취소를 하지 않고, 비동기 호출로 전환.

![screenshot-miro com-2020 12 22-10_19_41](https://user-images.githubusercontent.com/76149887/102837538-51be8180-443f-11eb-9fbc-00bec667f053.png)

#### 2021.01.22

- 오더 속성 변경(호텔 서비스인 것을 좀 더 명확하게 드러내려고 호텔아이디와 룸타입으로 변경)
- 호텔이 확인 하여 예약을 확정한다는 기능 요구사항을 표현 하기 위해 액터(호텔리어), 커맨드(컨펌) 추가. 뷰는 기존의 리저베이션 디테일 위치 이동.  

<img width="643" alt="miro_20210122" src="https://user-images.githubusercontent.com/58290368/105454042-9ea6a980-5cc4-11eb-95bd-439831fc6575.png">

### 기능 요구사항을 커버하는지 검증
1. 고객이 원하는 객실을 선택 하여 예약한다.(O)
2. 고객이 결제 한다.(O)
3. 예약이 신청 되면 예약 신청 내역이 호텔에 전달 된다.(O)
4. 호텔이 확인 하여 예약을 확정 한다.(O)
5. 고객이 예약 신청을 취소할 수 있다.(O)
6. 예약이 취소 되면 호텔 예약이 취소 된다.(O)
7. 고객이 예약 진행 상황을 중간 중간 조회 한다.(O)
8. 예약 상태가 바뀔 때 마다 카카오톡으로 알림을 보낸다.(O)
9. 고객이 예약 취소를 하면 예약 정보는 삭제되나, 객실 관리팀에서는 이력 관리를 위해 취소 장부를 별도 저장한다.(O)

### 비기능 요구사항을 커버하는지 검증
1. 트랜잭션 
   - 결제가 되지 않은 예약건은 아예 호텔 예약 신청이 되지 않아야 한다. `Sync 호출`(O)
   - Request-Response 방식 처리

2. 장애격리
   - 객실관리 기능이 수행 되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다.(O)
   - Eventual Consistency 방식으로 트랜잭션 처리(Pub/Sub)

## 헥사고날 아키텍처 다이어그램 도출
- Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
- 호출관계에서 Pub/Sub 과 Req/Resp 를 구분함
- 서브 도메인과 바운디드 컨텍스트의 분리: 각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
<img width="949" alt="hda_20210122" src="https://user-images.githubusercontent.com/58290368/105463644-b33e6e00-5cd3-11eb-98fd-358f8c626c32.png">
