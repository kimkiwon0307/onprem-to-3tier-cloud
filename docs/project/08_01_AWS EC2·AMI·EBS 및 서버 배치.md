
# A과정 8장 1교시 — EC2·AMI·EBS 핵심 정리

## 1. EC2란?

EC2(Elastic Compute Cloud)는 AWS에서 가상서버를 생성하고 운영하는 서비스이다.

VMware 가상머신과 비슷하지만, AWS 클라우드의 컴퓨팅 자원을 사용한다.

### EC2의 주요 구성 요소

| 구성 요소 | 역할 |
|---|---|
| EC2 | 가상서버 실행 |
| Instance Type | CPU·메모리 사양 결정 |
| AMI | 서버 생성에 사용하는 이미지 |
| EBS | 운영체제·파일·데이터 저장 |
| VPC | 서버가 속한 네트워크 |
| Subnet | IP 주소 범위 및 AZ 결정 |
| Security Group | 네트워크 접근 제어 |
| Key Pair | SSH 공개키 인증에 활용 |

**EC2는 서버를 실행하고, AMI는 서버 생성의 기준이 되며, EBS는 데이터를 저장한다.**

---

## 2. AMI란?

AMI(Amazon Machine Image)는 EC2를 생성할 때 사용하는 서버 이미지이다.

### AMI를 사용하는 이유

- 동일한 서버 환경을 반복해서 구축하기 위해
- 서버 생성 시간을 줄이기 위해
- WEB01과 유사한 구성의 WEB02를 만들기 위해
- 장애 시 서버를 재생성하는 기반으로 활용하기 위해

### Marble 프로젝트 적용

```text
WEB01 (Ubuntu + Apache + 설정)
          ↓
      Custom AMI
          ↓
        WEB02
```

### AMI 사용 시 주의사항

1. AMI 생성 이후 WEB01의 설정 변경은 WEB02에 자동 반영되지 않는다.
2. 복제 후 호스트명·IP 관련 설정을 확인해야 한다.
3. Security Group·Subnet 등 새 EC2의 환경을 별도로 확인한다.
4. AMI 복제가 자동 로드밸런싱이나 데이터 동기화를 의미하지는 않는다.

**AMI = 서버 복제 기반이지, 자동 이중화 서비스가 아니다.**

---

## 3. EBS란?

EBS(Elastic Block Store)는 EC2에서 사용하는 블록 스토리지이다.

운영체제, 프로그램, 설정 파일, 로그 및 데이터를 저장한다.

```text
EC2
 └── EBS
      ├── Ubuntu
      ├── Apache / Spring Boot / MySQL
      ├── 설정 파일
      └── 로그 및 데이터
```

### EC2 상태에 따른 EBS

| EC2 작업 | EBS 데이터 |
|---|---|
| Reboot | 유지 |
| Stop | 유지 |
| Start | 기존 EBS 사용 |
| Terminate | Delete on termination 설정에 따라 삭제 가능 |

**EC2를 중지하는 것과 종료(Terminate)하는 것은 다르다.**

### EBS Snapshot

Snapshot은 EBS 볼륨의 특정 시점 데이터를 보관하는 백업 수단이다.

- AMI: EC2 생성용 이미지
- Snapshot: EBS 데이터의 특정 시점 백업

주의: Snapshot만으로 실행 중인 MySQL 데이터의 트랜잭션 정합성이 항상 보장되는 것은 아니다.

---

## 4. Marble 프로젝트의 EC2 배치

### VPC

10.0.0.0/16

### 서버 배치

| 구분 | AZ-a | AZ-c |
|---|---|---|
| Public Subnet | 10.0.1.0/24 | 10.0.2.0/24 |
| Private WEB | 10.0.11.0/24 | 10.0.12.0/24 |
| Private WAS | 10.0.21.0/24 | 10.0.22.0/24 |
| Private DB | 10.0.31.0/24 | 10.0.32.0/24 |

### 서비스 요청 흐름

```text
인터넷 사용자
      ↓
Public ALB
      ↓
Private WEB01 / WEB02
      ↓
Private WAS01 / WAS02
      ↓
Private MySQL Primary / Replica
```

### Private Subnet에 배치한 이유

- WEB: 외부 요청을 ALB를 통해 수신
- WAS: WEB에서 전달한 애플리케이션 요청 처리
- DB: 필요한 내부 서버에서만 접근하도록 제한
- Bastion: 관리자가 Private 서버에 접근하는 경유지 역할

**WEB·WAS·DB 서버는 직접 인터넷에 노출하지 않고 필요한 경로로만 접근하도록 설계했다.**

---

## 5. AWS 콘솔 검증

### EC2 확인

AWS 콘솔 → EC2 → Instances

확인 항목:
- Instance ID / State
- Instance Type
- AMI ID
- VPC / Subnet / AZ
- Private IP / Public IP
- Security Group

### AMI 확인

AWS 콘솔 → EC2 → AMIs

확인 항목:
- AMI ID 및 이름
- State: Available
- 생성 이미지의 용도

### EBS 확인

AWS 콘솔 → EC2 → Volumes

확인 항목:
- Volume ID
- Volume Type / Size
- 연결된 EC2
- Availability Zone
- Encryption
- Delete on termination

---

## 6. Linux 서버 확인 명령어

```bash
# CPU 정보
nproc
lscpu

# 메모리 확인
free -h

# 디스크 및 파일시스템 확인
lsblk
df -h

# 호스트명과 네트워크 정보
hostname
ip -4 addr
ip route
```

### 주요 명령어

| 명령어 | 역할 |
|---|---|
| `nproc` | 사용 가능한 CPU 처리 단위 수 확인 |
| `free -h` | 메모리 사용량 확인 |
| `lsblk` | 디스크·파티션 구조 확인 |
| `df -h` | 파일시스템 용량 및 사용률 확인 |
| `ip -4 addr` | IPv4 주소 확인 |
| `ip route` | Linux 운영체제 라우팅 확인 |

---

## 7. 주요 장애 대응

### ① EC2는 Running인데 서비스 접속 실패

확인:
1. EC2 Status Checks
2. Subnet / Security Group
3. Apache·Spring Boot 서비스 상태
4. 애플리케이션 로그
5. ALB Target Group Health Check

**EC2 Running은 애플리케이션 정상 작동을 보장하지 않는다.**

### ② AMI로 생성한 WEB02가 정상 동작하지 않음

확인:
- AMI 생성 이후 변경된 설정
- 호스트명 및 IP 관련 설정
- Security Group 및 Subnet
- Apache 실행 상태
- ALB Target Group 등록 상태

### ③ EBS 용량 부족

증상:
- 로그 기록 실패
- 애플리케이션 오류
- MySQL 데이터 기록 실패
- 패키지 설치 실패

확인:
1. `df -h`로 파일시스템 사용률 확인
2. `lsblk`로 디스크 구조 확인
3. AWS 콘솔에서 EBS 용량 확인
4. 로그 증가 및 데이터 저장량 조사
5. 필요 시 EBS와 파일시스템 확장 검토

---

## 8. 반드시 기억할 핵심

1. EC2는 AWS 가상서버이다.
2. Instance Type은 CPU·메모리 등 서버 사양을 결정한다.
3. AMI는 EC2를 생성하는 이미지이다.
4. Custom AMI를 이용해 유사한 서버를 생성할 수 있다.
5. AMI 복제는 자동 이중화나 데이터 동기화가 아니다.
6. EBS는 EC2의 영속 블록 스토리지이다.
7. EC2 Terminate 시 EBS 삭제 여부를 반드시 확인해야 한다.
8. Snapshot은 EBS 데이터의 특정 시점 백업이다.
9. EC2 Running과 애플리케이션 정상 상태는 다르다.
10. Marble 프로젝트는 WEB·WAS·DB를 모두 Private Subnet에 배치했다.

### 최종 암기 문장

**EC2 = 서버를 실행하는 컴퓨팅 자원**

**Instance Type = 서버의 CPU·메모리 사양**

**AMI = 서버를 생성하기 위한 이미지**

**EBS = 서버의 디스크**

**Snapshot = EBS 데이터 백업**

**Private Subnet = 서버의 직접 인터넷 노출을 줄이는 네트워크 설계**
