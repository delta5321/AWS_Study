# AWS 환경 구축 및 보안 설정 로그 (Initial Setup)


## 목적: AWS Root 계정 보안 강화, 작업용 IAM 사용자 생성 및 CLI 로컬 환경 구축 완료

## 1. 계정 생성 및 리전 설정

### 1.1 계정 가입 및 리전 변경

- 계정 유형: Personal (학습 및 포트폴리오용)

- 리전(Region) 설정: ap-northeast-2 (Asia Pacific - Seoul)

- 이유: 물리적 거리가 가장 가까워 네트워크 지연(Latency) 최소화 및 실습 효율 증대.

---
## 2. 보안 조치 (Security Baseline)

### 2.1 Root 계정 MFA(Multi-Factor Authentication) 설정

- 위험 요소: Root 계정은 모든 권한을 가진 슈퍼 유저이므로, 비밀번호 탈취 시 방어 수단이 없음.

- 조치 내용:

    + IAM 대시보드 -> 보안 자격 증명(Security credentials) 접속.

    + MFA 디바이스(Virtual MFA device) 할당.

    + Google OTP 앱을 연동하여 2차 인증 활성화.

- 결과: 로그인 시 아이디/비번 외에 OTP 코드 입력 필수화 (계정 탈취 방지).

### 2.2 결제 알림(Budgets) 설정 - 비용 관리

- 위험 요소: 프리티어 한도 초과 또는 실수로 고사양 리소스 생성 시 요금 폭탄 위험.

- 조치 내용:

    + AWS Budgets 서비스 접속.

    + 'Zero spend budget' 템플릿 적용.

    + 월 지출이 $0.01(약 10원)를 초과할 경우 이메일 경보 발송 설정.

---
## 3. 작업용 사용자(IAM User) 생성

### 3.1 Admin 계정 생성

- 원칙: '루트 계정 사용 금지' 보안 원칙 준수. 일상적인 작업은 별도 IAM 사용자로 수행.

- 조치 내용:

    + User Name: admin-user
    + Access Type: Management Console(웹) 및 Programmatic Access(CLI) 모두 허용.
    + Permission: AdministratorAccess 정책(Policy)이 연결된 AdminGroup 생성 및 할당.

---
## 4. 로컬 CLI(Command Line Interface) 환경 구축

### 4.1 AWS CLI v2 설치

- 목적: 웹 콘솔(GUI) 의존도를 낮추고 인프라 관리 자동화(IaC) 준비.

- 설치 확인:

    + aws --version
    + 결과: aws-cli/2.32.26 Python/3.13.11 Windows/10 exe/AMD64


### 4.2 프로필 연동 (Configure)

- IAM에서 발급받은 Access Key ID와 Secret Access Key를 로컬 환경에 등록.

- aws configure
    + AWS Access Key ID: [HIDDEN]
    + AWS Secret Access Key: [HIDDEN]
    + Default region name: ap-northeast-2
    + Default output format: json


## 4.3 연결 테스트

- 정상적으로 권한이 부여되었는지 확인.

 + aws sts get-caller-identity


- [출력 결과 예시]

 + {
    "UserId": "[HIDDEN]",
    "Account": "[HIDDEN]",
    "Arn": "arn:aws:iam::[HIDDEN]"
}


---
## 5. 트러블 슈팅 (Troubleshooting)

 - 이슈: 계정 차단 (Account Blocked)

 - 현상: aws ec2 run-instances 명령어로 서버 생성 시도 시 UnauthorizedOperation 및 Blocked 에러 발생.

 - 원인 분석:
    + IAM User에게 AdministratorAccess 권한 누락 (1차 원인).
    결제 카드가 만료되어 AWS 측에서 계정을 임시 차단 (2차 원인).

 - 해결 과정:
    + Root 계정으로 로그인하여 admin-user에게 관리자 권한 부여.
    + Billing Dashboard에서 유효한 카드로 결제 수단 업데이트.
    + AWS Support Center에 계정 활성화 요청(Account Activation) 티켓 생성.