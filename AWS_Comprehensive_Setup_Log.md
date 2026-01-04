# AWS 클라우드 인프라 구축 및 트러블슈팅 로그 (Comprehensive Guide)

#### 목적: AWS 환경 초기 보안 설정부터 VPC 네트워크 설계, EC2 인스턴스 배포까지의 전 과정을 GUI와 CLI로 수행하고, 발생한 이슈를 해결한 기록을 정리함.

## 1. 초기 보안 설정 (Security Baseline)

### 1.1 Root 계정 보안 강화

 - MFA(Multi-Factor Authentication) 설정: Root 계정 탈취 방지를 위해 Google OTP 연동.

 - 결제 알림(Budgets) 설정: 프리티어 한도 초과 및 해킹으로 인한 과금 방지를 위해 'Zero spend budget' 적용.

### 1.2 IAM 관리자 계정 생성

 - 원칙: 일상적인 운영 업무에 Root 계정 사용 금지.

 - 조치:

    * 사용자명: admin

    * 권한: AdministratorAccess 정책 부여.

    * 접근 방식: 콘솔 비밀번호(GUI) 및 Access Key(CLI) 발급.

## 2. VPC 네트워크 구축 (Infrastructure Setup)

 - 기본(Default) VPC를 사용하지 않고, 직접 설계한 격리된 네트워크 환경을 구축함.

### 2.1 아키텍처 개요

 - VPC: MyLabVPC (10.0.0.0/16)

 - Subnet: MyLabPublicSubnet (10.0.1.0/24) - 서울 리전 A존(ap-northeast-2a)

 - Gateway: MyLabIGW (인터넷 연결용)

 - Route Table: MyLabRT (Public Subnet용 커스텀 라우팅 테이블)

### 2.2 구축 과정 (GUI → CLI)

 - AWS 콘솔에서 리소스를 생성하며 구조를 파악한 뒤, AWS CLI 명령어로 재구축하여 자동화 가능성을 검증함.

- 주요 CLI 명령어:
```
# 1. VPC 생성
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=MyLabVPC}]'

# 2. 서브넷 생성 (관리용 대역 10.0.0.0/24 제외 후 1.0부터 할당)
aws ec2 create-subnet --vpc-id [VpcId] --cidr-block 10.0.1.0/24 --availability-zone ap-northeast-2a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=MyLabPublicSubnet}]'

# 3. 인터넷 게이트웨이 연결
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=MyLabIGW}]'
aws ec2 attach-internet-gateway --vpc-id [VpcId] --internet-gateway-id [IgwId]

# 4. 라우팅 테이블 설정 (0.0.0.0/0 -> IGW)
aws ec2 create-route --route-table-id [RouteTableId] --destination-cidr-block 0.0.0.0/0 --gateway-id [IgwId]
```

## 3. 보안 그룹 및 접근 제어 (Security Group)

### 3.1 보안 그룹 정책 (MyLabSG)

- Inbound (수신):

    * Protocol: SSH (TCP/22)
    * Source: My IP Only (보안 강화를 위해 0.0.0.0/0 사용 지양)

- Outbound (송신):

    * All Traffic Allow (패키지 업데이트 및 외부 통신 허용)

## 4. EC2 인스턴스 배포 (Compute)

### 4.1 인스턴스 사양

 - AMI: Amazon Linux 2023 (최신 버전 사용)
 - Type: t2.micro (프리티어)
 - Key Pair: MyCLIKey (RSA)

### 4.2 이슈 해결: 계정 차단 (Account Blocked)

 - 현상: run-instances 명령 실행 시 UnauthorizedOperation 및 Blocked 에러 발생.

 - 원인: 신규 가입 직후 리소스 생성 시도 및 결제 카드 만료로 인한 AWS 보안 시스템의 임시 차단.

 - 해결:

    * 결제 수단 업데이트.
    * Root 계정 비밀번호 변경 및 MFA 재확인. 
    * AWS Support Center에 소명 메일 발송 및 차단 해제 승인 획득.

## 5. 학습 소감 및 향후 계획

 - 성과: AWS 콘솔의 편리함과 CLI의 정확/신속함을 모두 경험하며, IaC(Infrastructure as Code)의 필요성을 체감함. 특히 네트워크(VPC)를 직접 설계하고 구축해보며 클라우드 네트워크 구조를 깊이 이해하게 됨.

 - 계획:
    * 구축된 인프라 위에서 Docker 컨테이너 실습 진행.
    * 테라폼(Terraform)을 학습하여 현재의 CLI 스크립트를 더 체계적인 코드로 전환.