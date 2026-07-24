---
title: "ENI(Elastic Network Interface)"
date: 2026-07-07 16:43:37 +0900
categories: [Knowledge]
tags: [AWS, ENI, EC2, VPC]
---
**가상의 네트워크 카드(랜카드) 를 나타내는 VPC의 논리적 구성요소**다.
EC2 인스턴스가 네트워크에 접속하게 해주는 게 이 ENI 덕분이었다.

EC2 인스턴스가 IP를 가진다는 말은, 정확히 말하면 **EC2에 붙어 있는 ENI가 IP를 가진다는 것**
- ENI는 생성될 때 **특정 서브넷에 속한다.**
  그 말은 즉슨 해당 서브넷이 속한 **AZ에 종속**된다는 뜻이다.
  즉, 생성한 ENI와 EC2의 서브넷이 다르면 붙일 수 없다.
- ENI 하나에 **고유 MAC 주소**를 가지며 이 주소는 절대 바뀌지 않는다.
![](/assets/img/eni-elastic-network-interface/image.png)
EC2는 생성하면 무조건 기본 ENI가 붙는다.
- **기본 ENI**는 EC2의 네트워크 인터페이스 중 Eth0에 붙는다.
- 기본 ENI는 **EC2와 생명주기를 같이한다**.
- 해당 EC2에서 뗄 수 없으며, 정지 상태여도 마찬가지다.
EC2 인스턴스가 삭제되면 기본 ENI도 같이 삭제된다.

단, 이때 ENI의 Delete on termination을 끄면 같이 삭제되지 않고 ENI는 남아있게 된다. 
이를 이용해서 primary ENI를 옮겨줄 수도 있다.
여기서 알 수 있는 점: ENI 없는 EC2는 존재하지 않는다.

그리고 EC2에 추가적인 부가 ENI를 붙여줄 수도 있다.

### 1. ENI 생성
서브넷, 할당할 private IP (자동 할당 또는 내가 정해서) 등을 정해서 생성한다.
### 2. private IP 붙이기
- **기본 사설 IPv4 하나**를 할당받는다. 이건 할당 해제 불가.
- **추가 사설 IPv4를 여러 개** 붙일 수 있다.

> **ENI와 장치 인덱스 번호**
> ENI가 인스턴스에 연결될 때에는 장치 인덱스 번호가 할당된다.
> 리눅스 등에서의 네트워크 인터페이스로 나타난다.
> 기본 ENI는 무조건 eth0이다.
> 기본만 있는 상태에서 IP를 추가하면 Eth1일 것이다.
> ENI 하나당 인덱스 번호 하나다.
> ENI에 IP 몇 개를 붙이든 무조건 하나의 인덱스 번호로 간다.
{: .prompt-info }

### 3. public ip 붙이기
ENI가 퍼블릭 IPv4 주소를 가지게 하는 방법은 두가지다.
1. 인스턴스 실행 시 **자동 할당되는 public IP**: 인스턴스 정지/시작할 때 옵션으로 체크해서 할당받고 종료 시 반납함.
   인스턴스 재시작 시 IP가 바뀐다.
2. **Elastic IP**를 붙이기: ENI가 없어져도 EIP는 계정에 귀속되어 지속됨.
   Elastic IP는 내 계정에 고정된 IP다. 안 바뀐다.

### 4. ENI에 보안 그룹 붙이기
EC2 인스턴스에 보안 그룹을 붙이는 것은 **정확히는 ENI에 보안 그룹을 붙이는 것**이다.
보안그룹끼리의 참조 또한 이 ENI를 중심으로 동작한다.
EC2 인스턴스에 붙은 ENI에 각각 서로 다른 보안 그룹을 적용해줄 수 있다.

> 내가 따로 ENI만 만들 수도 있고, 
> ENI가 붙어져 있던 EC2 인스턴스에서 Eth0에 해당하는 ENI만 떼서 다른 EC2 인스턴스에 갖다붙일 수도 있다.
{: .prompt-tip }

### 기타
- Allow reassociation(재연결 허용): ENI에 EIP를 붙일 때의 옵션
  현재의 elastic ip가 만약에 다른 데 붙어 있다 해도 바로 재연결 해버리는 옵션이다. 별도로 다른데서 해제해줄 필요가 없다.
![](/assets/img/eni-elastic-network-interface/pasted-image-20250901200749.png)
- 실습용으로 쓰는 저비용 인스턴스는 대부분 할당 가능한 ip주소에 제한이 있다.
  t2.micro의 경우,
  - ENI 최대 두 개
  - ENI당 최대 IPv4 두 개(public ip는 이 private ip에 매핑 가능)
  - ENI당 최대 IPv6 두 개
![](/assets/img/eni-elastic-network-interface/pasted-image-20250902104318.png)
