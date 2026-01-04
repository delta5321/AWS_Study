# Security Hardening with NAT Gateway & Private Subnets

**Project:** 프라이빗 네트워크 마이그레이션 (Migration to Private Network)
**Goal:** 공개망(Public Subnet)에 노출된 웹 서버들을 사설망(Private Subnet)으로 격리하고, NAT Gateway를 통해 안전한 아웃바운드 통신을 구현한다.

---

## 🏗️ 1. 아키텍처 변경 사항 (Architecture Change)

*   **Before (Phase 1):**
    *   **ALB:** Public Subnet
    *   **EC2 (ASG):** Public Subnet (보안 취약: 공인 IP 보유)
    *   **Internet Gateway (IGW):** 양방향 통신 담당
*   **After (Phase 2):**
    *   **ALB:** Public Subnet (외부 접속 담당)
    *   **EC2 (ASG):** **Private Subnet** (외부 격리: 사설 IP만 보유)
    *   **NAT Gateway:** **Public Subnet**에 위치 (내부 서버가 인터넷으로 나갈 때만 사용)

---

## ⚙️ 2. 상세 구축 가이드 (Step-by-Step)

### Step 1: 프라이빗 서브넷 (Private Subnet) 생성
서버들이 숨을 '안전 가옥'을 만듭니다.

*   **메뉴:** VPC > 서브넷 > **[서브넷 생성]**
*   **설정 상세:**
    *   **VPC:** `MyLabVPC`
    *   **서브넷 1:** `MyLabPrivateSubnet1` / AZ: `ap-northeast-2a` / CIDR: `10.0.10.0/24`
    *   **서브넷 2:** `MyLabPrivateSubnet2` / AZ: `ap-northeast-2c` / CIDR: `10.0.20.0/24`
    *   *핵심:* 기존 Public Subnet과 **같은 가용 영역(AZ)**을 쓰되, IP 대역만 다르게 설정합니다.

### Step 2: NAT 게이트웨이 (NAT Gateway) 생성
사설망 서버들이 외부(yum update 등)로 나갈 수 있는 '공용 출구'를 만듭니다.

*   **메뉴:** VPC > NAT 게이트웨이 > **[NAT 게이트웨이 생성]**
*   **설정 상세:**
    *   **이름:** `MyLab-NATGW`
    *   **서브넷:** 반드시 **Public Subnet** 중 하나를 선택해야 합니다. (★시험/실무 핵심 함정)
        *   *이유:* NAT 장비 자체는 인터넷과 연결되어야 하므로 공인망에 있어야 합니다.
    *   **연결 유형:** `퍼블릭 (Public)`
    *   **탄력적 IP (EIP):** **[탄력적 IP 할당]** 버튼 클릭.

### Step 3: 프라이빗 라우팅 테이블 (Private Route Table) 구성
사설망 서버들에게 "인터넷으로 갈 때는 NAT를 타라"고 길을 알려줍니다.

*   **메뉴:** VPC > 라우팅 테이블 > **[라우팅 테이블 생성]**
*   **설정 상세:**
    *   **이름:** `MyLab-Private-RT`
    *   **VPC:** `MyLabVPC`
*   **라우트 편집 (Routes):**
    *   **[라우트 추가]** 클릭.
    *   대상(Destination): `0.0.0.0/0` (모든 인터넷 트래픽)
    *   **타겟(Target):** `NAT Gateway` -> `MyLab-NATGW` 선택.
*   **서브넷 연결 (Subnet Associations):**
    *   **[서브넷 연결 편집]** 클릭.
    *   Step 1에서 만든 **`MyLabPrivateSubnet1`**, **`MyLabPrivateSubnet2`** 두 개만 체크하고 저장.
    *   *(Public Subnet은 절대 체크하면 안 됨)*

### Step 4: ASG 마이그레이션 (서버 이사)
오토스케일링 그룹 설정을 변경하여 서버를 사설망으로 옮깁니다.

*   **메뉴:** EC2 > Auto Scaling 그룹 > `MyLab-ASG` > **[세부 정보]** 탭
*   **네트워크 편집:**
    *   **[편집(Edit)]** 클릭.
    *   기존 Public Subnet 2개를 **제거(X)**.
    *   새로운 **Private Subnet 2개**를 선택 -> **[업데이트]**.
*   **인스턴스 새로 고침 (Instance Refresh):**
    *   상단 **[인스턴스 새로 고침]** 탭 -> **[새로 고침 시작]**.
    *   *결과:* 기존 서버(Public IP 보유)가 종료되고, 새 서버(Private IP만 보유)가 생성됨.

---

## 🚨 3. 트러블슈팅 가이드 (Troubleshooting)

Private 환경으로 전환 시 발생하는 대표적인 문제와 해결 방법입니다.

### Issue 1: 대상 그룹 상태가 `Unused` (가장 흔한 오류)
*   **증상:**
    *   ASG가 Private 인스턴스를 정상적으로 생성(`Running`)했음.
    *   하지만 대상 그룹(Target Group)에서는 상태가 `Unused`로 표시됨.
    *   로드밸런서 DNS로 접속 시 사이트가 열리지 않음.
*   **원인:** **"ALB의 가용 영역(AZ) 매핑 누락"**
    *   ASG 설정만 바꾸고, 로드밸런서가 "어느 동네(AZ)를 감시해야 하는지" 업데이트하지 않았기 때문.
    *   인스턴스는 `2c` 구역에 있는데, 로드밸런서는 `2c` 구역을 쳐다보지 않는 상태.
*   **해결 방법:**
    1.  EC2 > 로드 밸런서 > `MyLab-ALB` 선택.
    2.  중간 **[네트워크 매핑(Network mapping)]** 탭 클릭.
    3.  **[서브넷 편집]** 클릭.
    4.  인스턴스가 생성된 가용 영역(예: `ap-northeast-2c`)을 체크(V).
    5.  **서브넷 선택:** 해당 영역의 **Public Subnet**을 선택.
        *   *(주의: ALB는 항상 Public에 있어야 하므로 Private을 선택하면 안 됨)*
    6.  저장 후 `Unused` -> `Initial` -> `Healthy` 변경 확인.

### Issue 2: 대상 그룹 상태가 `Unhealthy`
*   **증상:** 상태가 `Unused`는 아닌데 `Unhealthy`(빨간색)가 뜸.
*   **원인:** **"보안 그룹(Security Group) 차단"**
    *   Private 인스턴스의 보안 그룹이 로드밸런서로부터 오는 트래픽(Port 80)을 허용하지 않음.
*   **해결 방법:**
    *   인스턴스 보안 그룹(`MyLabSG`)의 인바운드 규칙 확인.
    *   `HTTP (80)` 포트가 소스 `0.0.0.0/0` 또는 `ALB의 보안 그룹 ID`로부터 허용되어 있는지 확인.

### Issue 3: 인스턴스에 SSH 접속 불가
*   **증상:** 터미널에서 `ssh -i key.pem ec2-user@<IP>` 명령어가 먹히지 않음.
*   **원인:** **"Private Subnet의 특성"**
    *   Private 인스턴스는 공인 IP(Public IP)가 없으므로 인터넷에서 직접 접속이 불가능함. (지극히 정상)
*   **해결 방법 (참고용):**
    *   방법 A: Public Subnet에 **Bastion Host(점프 서버)**를 만들어서 경유 접속.
    *   방법 B: **AWS Systems Manager (Session Manager)**를 사용하여 브라우저 콘솔에서 접속 (권장).

---

## 🧹 4. 리소스 정리 (Clean Up - 비용 주의)

NAT Gateway는 시간당 비용이 높으므로 실습 후 즉시 삭제해야 합니다.

1.  **Auto Scaling 그룹 삭제:** 인스턴스 자동 종료.
2.  **로드 밸런서(ALB) 삭제:** 시간당 과금 방지.
3.  **NAT 게이트웨이 삭제:**
    *   삭제 후 상태가 `Deleted`로 바뀔 때까지 대기.
4.  **탄력적 IP (Elastic IP) 릴리스:**
    *   NAT 삭제 완료 후, VPC > 탄력적 IP 메뉴에서 반드시 **[릴리스(Release)]** 해야 함. (미사용 IP 과금 방지)
5.  **VPC 및 서브넷:** 필요 없다면 삭제.