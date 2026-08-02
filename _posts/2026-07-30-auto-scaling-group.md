---
title: "Auto Scaling Group"
date: 2026-07-30 02:39:00 +0900
categories: [Knowledge]
tags: [AWS, ASG, EC2]
---
![](/assets/img/auto-scaling-group/attachment.jpeg)
- 언제 쓰나?
  - **스케일링**: EC2 인스턴스를 scale-out(늘리기) 또는 scale-in(줄이기)
  - 여러 EC2 인스턴스를 자동으로 **멀티 AZ 고가용성 구성** 해야할 때
  - 특정 EC2 인스턴스가 **최소 몇 개에서 최대 몇 개 실행됨을 보장**해야 할 때
- **로드밸런서(ELB)에 ASG를 연결**하는 것이 꽤 흔한 구성이다.
  → 이럴 경우, 자동으로 새 인스턴스는 로드밸런서의 대상이 된다.
- EC2 인스턴스가 **종료되었을 경우 다시 시작**시키는 기능을 제공한다.
- **ASG 자체로는 과금되지 않으며**, ASG 하위에서 실행되는 EC2의 비용만 내면 된다.
### 고가용성
- **ASG는 멀티 AZ 수준의 고가용성 구성이 가능하다.**
  - ASG를 생성할 때 AZ가 다른 **여러 Subnet을 지정**할 수 있다.
  - **기본적으로** 인스턴스들이 서로 다른 AZ에 속한 subnet에 **분산 배치**된다.
  - 특정 Availability Zone에 장애가 발생하여 해당 구역에서 **새 인스턴스를 생성하지 못하는 상황일 경우**
    - 세 옵션 중 하나에 따라, ASG는 건강한 **다른 Availability Zone에 새로운 인스턴스를 생성**하여 전체 서비스 용량을 유지한다.
    ![](/assets/img/auto-scaling-group/image-6.png)

### ASG 생성 시 정해야 할 것들
- **Launch Template**
  - 다음의 요소가 포함된다.
  ![](/assets/img/auto-scaling-group/image-3.png)
- **최소 용량, 목표 용량, 최대 용량**을 미리 정한다.
  - **Minimum / Maximum Capacity**: 최소, 최대 인스턴스 수
  - **Desired Capacity**: 초기 인스턴스 수 = 목표 인스턴스 수
  - ASG는 최소 용량과 최대 용량 사이에서 인스턴스 수를 늘리거나 줄일 수 있다.
  ![](/assets/img/auto-scaling-group/image.png)
- **스케일링 정책**: 자세한 내용 후술
- 교체 시 동작(종료 후 시작, 시작후 healthy 후 종료 등..)
### ASG의 Health Check
![](/assets/img/auto-scaling-group/image-7.png)
- **EC2 Health check**: 가장 기본적인 Health Check로 항상 활성화되어 있다.
  - OS 수준에서 인스턴스가 멀쩡한가를 확인하는 정도다.
- **ELB Health Check**: ELB랑 연결한다면 ELB의 헬스체크를 통해 최종적으로 healthy 여부를 판단한다.
  - ELB는 target port와 path를 지정하여 헬스 체크가 가능하므로, 이를 통해 애플리케이션의 정상 여부 또한 확인 가능하다.
- VPC Lattice, EBS health check 또한 지원한다. 
- ASG는 Health Check의 결과가 unhealthy인 경우, EC2를 terminate 하고 새로 시작한다.
### ELB와의 연계
- ELB와 ASG를 연결하면, ELB는 ASG에 속한 모든 인스턴스에 트래픽을 분산시킬 수 있다.
![](/assets/img/auto-scaling-group/image-8.png)
- **Health Check 관련**
  - **ELB의 health check**가 ASG 안 EC2에게 가게 된다.
    - ELB는 unhealty라면 Target group 내 대상으로 전달하지 않는다.
  - ASG의 Health check의 결과로 ELB를 지정할 수 있다. 이를 통해서 **ELB와 ASG의 헬스 체크의 양 장점을 모두 취할 수 있다.**
    1. ELB의 특정 포트와 경로로 HTTP 요청을 통한 헬스 체크로 **애플리케이션 수준에서 괜찮은지 검사**한다.
    2. ASG의 **terminate 처리**를 통해 인스턴스의 애플리케이션이 런타임 에러가 발생해 죽은 경우 기존 인스턴스를 terminate 하고 새 인스턴스를 만든다.
- ASG에서 scale out / scale in 시
  - scale out하면 새로 추가된 EC2가 ALB의 target group에 연결된다. ELB의 health check 후 healthy함이 확인되면 요청을 전달한다.
  - scale in 하면 target group에서 제거해 신규 트래픽 유입 차단 → deregistration delay 시간 동안 기다림(기존에 처리 중이던 요청이 있을 수 있기 때문) →  ASG의 Terminate
  ![](/assets/img/auto-scaling-group/image-2.png)
- **고가용성** 관점: 로드밸런서 + ASG 구성에서, **로드밸런서 또한 멀티 AZ** 구성이 가능하므로(2개 AZ를 지정해야 하므로 반필수적), 시스템 전체가 멀티 AZ급 고가용성이 확보된다.

### ★ ASG의 스케일링 정책 ★
**1. 동적 스케일링 (Dynamic Scaling): CloudWatch Alarms와 연계하여 동작**
* **대상 추적 정책(Target Tracking Scaling)**
  * 특정 메트릭(예: CPU 사용률)을 정하고, 목표값(예: 40%)을 설정하면, scale in/out 하여 메트릭을 목표값 근처로 유지하는 방식
  * Desired Capacity가 조정되며, ASG는 이에 맞게 인스턴스를 늘리거나 줄이게 된다.
  * ASG가 자동으로 Scale Out용 Alarm, Scale In용 Alarm 2개를 자동으로 생성하고 관리한다.
* **Step/simple Scaling**
  ![](/assets/img/auto-scaling-group/image-4.png)
  * ASG에 **Step Scaling 정책을 생성**한다.
    * e.g. 이름: scale-out-policy, 조정 방식: ChangeInCapacity, 조정량: +2 (정책 실행 시 Desired Capacity 2만큼 증가)
  * **CloudWatch alarm을 직접 설정**하여, 특정 metric이 조건을 만족하면 Step Scaling 정책의 내용이 실행되도록 구성한다.
    * e.g. AWS/EC2 의 CPUUtilization 메트릭의 평균이 5분 동안 70% 이상이면 Alarm 트리거
  * Alarm이 울리면 몇 대를 추가하거나 제거할 것인가를 설정하면 알아서 해 준다.
  * **Step Scaling**의 경우, **단계적으로 동작을 설정**한다는 것이 차이점이다.
    * 예시
      * CPU 사용률이 60~70%이면 인스턴스 1대 추가 
      * CPU 사용률이 70~80%이면 인스턴스 2대 추가
      * CPU 사용률이 80% 이상이면 인스턴스 4대 추가

**2. 예약 스케일링 (Scheduled Scaling)**
- **일회성 예약 또는 크론식**으로 단일 이벤트나 반복적인 스케일링을 예약할 수 있다.
- **예약하는 동작: 최소, 최대, 목표 인스턴스 수 중 하나 이상의 변경**
- 중요한 점: 변경한 변수는 **알아서 원래 값으로 돌아가지 않으며, 별도로 축소를 위한 예약 스케일링을 만들어야 한다.**
* 예시
  * 매일 오후 5시에 식당 예약 애플리케이션의 트래픽이 몰릴 것으로 예상, Min=10, Max=20, Desired=15로 설정
  * 매월 1일에 결산 작업 예약

**3. 예측 스케일링 (Predictive Scaling)**
* 과거 부하를 분석해서 미래 부하를 예측 후, 그 예측치를 기반으로 현재 최소로 필요한 인스턴스 수를 예측, 현재 필요 용량을 증가시키는 방식
  * **내가 스케일링할 용량을 정하지 않고**, AWS가 계산하는 **과거 패턴에 따른 동적 예측치**에 따라 스케일링된다.
* **예측에 사용할 지표**를 정한 후, 해당 **지표의 목표 값**, **얼마나 미리 스케일링**할 것인지, **최대 용량 설정값을 무시하고 증가**시킬 것인지 정한다.
* 예측만 할지(`ForecastOnly`), 실제 조정까지 할지(`ForecastAndScale`) 을 정한다.
  * 처음 적용 시엔 `ForecastOnly`로 예측 정확도를 검토한 후 `ForcastAndScale` 로 전환하는 전략이 가능하다.
* 알려진 사용 패턴에 따라 트래픽이 늘어나기 전에 **미리 스케일링**하고 싶을 때 사용한다.
  * 또한 고정된 값으로 늘리는 경우가 아닌 **패턴에 따라 매주 다른 처리량이 필요할 때** 유용하다.

### ★ 동적 스케일링의 대표 기준 metric ★
일반적으로 사용하는 메트릭들은 다음 4가지다.
* **CPU 사용률 (CPUUtilization)**
  * 모든 인스턴스의 평균 CPU 사용률
* **타깃당 요청 수 (RequestCountPerTarget)**
  * 보통 ALB가 요청을 세서 CloudWatch에게 메트릭으로 전송, ASG 스케일링 정책이 그 메트릭을 사용하는 구조
  * 실제 메트릭 `ALBRequestCountPerTarget`: 인스턴스 한 대당 평균 요청 수
  * 예시: EC2 인스턴스가 타깃당 1000개의 요청에서 최적으로 작동한다면, 이를 스케일링 목표로 설정
  * 계산 방법: ASG에 3개 EC2가 있고, 각각 평균 3개의 미해결 요청이 있다면 → 타깃당 요청 수는 3
* **네트워크 사용량 (Average Network In/Out)**
  * 업로드/다운로드가 많은 애플리케이션
  * 네트워크가 병목이 될 것으로 예상되는 경우
* **사용자 정의 메트릭 (Custom CloudWatch Metric)**
  * CloudWatch에 직접 설정한 애플리케이션별 고유 메트릭
  * 고유 메트릭의 목푯값을 정하며, 이를 기반으로 스케일링 정책이 동작한다.

### SQS와의 연계
![](/assets/img/auto-scaling-group/image-9.png)
- 위 사진의 아키텍처에서, Web Tier ASG → Application Tier ASG로 트래픽이 흐르는 상황
  * 주문 이행 서비스 같은 경우엔
    * 결제 API의 응답 대기, 데이터베이스 쿼리 등이 있을 것
    * 작업을 쌓아두고 나서 처리해야 하는 상황
    * 주문 데이터가 손실되어서는 안 된다
* 이런 상황에서, ASG만 쓰고 ALBRequestCountPerTarget 지표로 ASG의 스케일링 정책을 구성할 수도 있다.
  * 하지만 주문 데이터가 손실되면 안 되는 상황이라면?
    * ASG가 스케일링하는 과정에서 시간이 걸려 처리하지 못하고 손실되는 주문이 생길 수 있다.
  * 아예 최소 EC2 개수 자체를 보수적으로 잡는 방법도 있을 것이다.
* 정석적인 해결책은 **SQS를 두고 주문을 쌓아둔 다음 ASG 내의 EC2는 SQS에서 가져가는 방식**이다.
  * SQS는 갑자기 늘어나는 트래픽을 감당하는 임시 버퍼의 역할도 하면서, 추가로 손실된 주문이 없게끔 보장하는 역할을 겸한다.
  * EC2가 SQS 메시지를 받더라도 메시지는 즉시 삭제되지 않고 **Visibility Timeout 동안 다른 소비자에게만 보이지 않게** 되며, 작업을 끝내고 DeleteMessage를 호출해야 실제로 삭제된다.
  * 인스턴스가 처리 도중 종료되어 메시지를 삭제하지 못하면 Visibility Timeout이 지난 뒤 메시지가 다시 큐에 나타나 다른 인스턴스가 재처리하게 된다.
  * 참고로 여러 번의 재시도(maxReceiveCount = 5 라면 5번) 끝에도 정상적으로 삭제하지 못하는 경우, **DLQ(Dead Letter Queue)**로 이동시킨다. 따라서 주문 메세지가 잘못된 경우에 계속 큐에 쌓여서 추가적으로 처리량을 낭비하는 걸 막고, 별도의 처리를 할 수 있다.
* SQS를 두었기 때문에, **SQS 큐의 상태를 나타내는 CloudWatch Metrics 지표까지 스케일링의 선택지로 활용 가능**하다.
  * 대표적으로 AWS가 권장하는 건 백로그 기반 지표다.
    * 백로그: 아직 처리되지 못하고 쌓여있는 작업량
  * 총 백로그가 아닌 **인스턴스당 백로그**를 주로 사용, **대기 중인 메세지 수(= 큐 길이)/ 현재 실행 중인 EC2 인스턴스 수** 로 계산한다.
  * 실제 지표는 다음과 같다.
    * `ApproximateNumberOfMessagesVisible`: 아직 아무 워커도 가져가지 않아 **SQS에서 대기 중인 메시지 수**
    * `GroupInServiceInstances`: 해당 ASG에서 실제 서비스 중인 EC2 수
    * 이 두 지표를 토대로 Metric Math 옵션으로 나눠서 계산하면 된다.
  * 별개 지표를 만드는 방법도 있으나, Metric Math가 권장된다.
* SQS를 두었기 때문에, **기존 ELB는 이제 필요없다**.
  * SQS에서 **각각의 EC2가 능동적으로 poll하는 구조**가 된다.
  * SQS →  ALB 구조는 불가능하다. 
### ASG Lifecycle Hook (EC2 Auto Scaling 수명 주기 후크)
* EC2의 기본적인 인스턴스 상태는 다음과 같음 Pending → Inservice → Terminating → Terminated
* 여기서 Lifecycle Hook을 활성화하면 다음과 같이
  * 시작할 때: Pending → **Pending:wait** → Pending:proceed → Inservice
  * 삭제할 때: Inservice → Terminating → **Terminating:wait** → Terminating:proceed → Terminated
  ![](/assets/img/auto-scaling-group/image-5.png)
  * 여기서 Pending:wait 하고 Terminating:wait 에 **내가 원하는 동작**을 집어넣을 수 있다.
    * 기본적으로 이벤트는 **EventBridge로 전달되며, EventBridge에서 Lambda, SNS, SQS 등을 추가로 연결** 가능하다.
    * Lambda에서 EC2에 접근하고 싶은 경우, SSM Run Command를 통해 가능하다.
  * 작업이 완료되고, 내가 **직접 CONTINUE 신호를 보내야 다음 단계가 진행**된다.
  * 사용 사례
    * 시작 Lifecycle Hook
      * 애플리케이션 패키지 설치
      * 대용량 모델·데이터 다운로드
      * 설정 파일과 인증서 구성
      * 보안 에이전트·모니터링 에이전트 설치
      * 캐시 또는 JVM 워밍업
      * 서비스 디스커버리에 등록
      * 준비 상태를 실제로 확인한 뒤 로드 밸런서 투입
    * 종료 Lifecycle Hook
      * CloudWatch Logs 외부의 로컬 로그 업로드
      * 처리 중인 백그라운드 작업 완료 또는 체크포인트 저장
      * 서비스 디스커버리에서 인스턴스 제거
      * EBS 스냅샷 생성
      * 라이선스 반환
      * 외부 모니터링 시스템에서 등록 해제
* **User Data는 왜 안 쓰고..?**
  * ASG는 일반적으로 User Data가 모두 실행되었는지 기다리지 않는다. 
  * ELB랑 같이 사용한다면 ELB Health Check을 통해 애플리케이션의 시작을 거의 관찰 가능하므로 큰 문제가 없긴 할 것이다.
  * 하지만 ALB가 없는 경우에 바로 ASG Health Check 결과가 Healthy인 EC2에게 트래픽을 보내면 문제가 생길 수 있다.
  * 그래서 오히려 **Lifecycle Hook과 User Data를 같이 사용**할 수도 있다.
    * 이 경우 **Lifecycle Hook의 진행을 User Data가 트리거**하는 방식이다.
    * Lifecycle Hook의 Pending:Wait 상태에서 **User Data의 마지막에 CONTINUE 신호를 보내는 것을 포함**한다.
    * 이를 통해 User Data가 모두 실행되면 다음 단계로 넘어가게 하는 구성이 가능하다.

### EC2 Auto Scaling Warm Pool
* 초기화한 EC2를 ASG 옆에 미리 대기시킨다.
* **초기화 시간을 단축시켜 빠르게 스케일 아웃하는 게 목적**이다.
* 주로 Hibernate 한 인스턴스를 대기시키나, 실행 중인 인스턴스 또한 대기시킬 수 있다.


### 스케일링 한 후 쿨다운 동안은 절대 추가 스케일하지 않음 (쿨다운)
* 스케일링 작업(인스턴스 추가/제거) 후 **기본 5분(300초) 쿨다운**이 발생
* 쿨다운 동안 ASG는 추가 인스턴스를 시작하거나 종료하지 않음
* 왜?
  * 메트릭이 안정화되도록 대기
  * 새로운 인스턴스가 적용되어 메트릭이 어떻게 변하는지 지켜봄
* **스케일링 작업 발생 시 로직**:
  * 쿨다운 중인가? → 스케일링 작업 무시
  * 쿨다운 아닌가? → 스케일링 작업 진행

### 추가 팁
* **준비된 AMI 사용**해서 준비시간 줄이기
* 상세 모니터링을 활성화: **ASG가 1분마다 메트릭에 접근**할 수 있도록 하여 빠르게 변하는 지표에 반응하게끔 설정 가능하다.
