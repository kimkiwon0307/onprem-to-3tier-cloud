# 12장 1교시 — On-Premise → AWS 마이그레이션 구축 과정

## 1. 마이그레이션(Migration)이란?

기존 환경에서 운영하던 시스템과 데이터를 새로운 환경으로 이전하는 과정이다.

Marble 프로젝트에서는 VMware Ubuntu 환경에서 구축한 Apache–Spring Boot–MySQL 3-Tier 서비스를 AWS EC2 기반으로 이전했다.

### 기존 환경과 AWS 환경 비교

| 구분 | VMware On-Premise | AWS |
|---|---|---|
| WEB | Apache | Apache WEB01/02 |
| WAS | Spring Boot | Spring Boot WAS01/02 |
| DB | MySQL | MySQL Primary/Replica |
| 네트워크 | VMware 가상 네트워크 | VPC / Subnet / Route Table |
| 부하 분산 | 별도 AWS ALB 없음 | ALB + Apache Balancer |
| 접근 제어 | Linux 방화벽 | Security Group / NACL |
| 관리 접속 | SSH | Bastion 경유 SSH |

**핵심:** 단순히 VM 이미지를 복사한 것이 아니라 AWS에서 서비스를 재구축하고 이중화 및 네트워크 구조를 개선했다.

---

## 2. Marble AWS 전체 아키텍처

```text
                    Internet
                       |
                       v
                 Public ALB
                       |
             +---------+---------+
             |                   |
             v                   v
           WEB01               WEB02
         Apache :80          Apache :80
      10.0.11.0/24        10.0.12.0/24
             |                   |
             +---------+---------+
                       |
                       v
              Apache Balancer
             (marblecluster)
                       |
             +---------+---------+
             |                   |
             v                   v
           WAS01               WAS02
     10.0.21.126:8080    10.0.22.9:8080
             |                   |
             +---------+---------+
                       |
                       v
                 DB01 Primary
               10.0.31.42:3306
                       |
                       | GTID 비동기 복제
                       v
                 DB02 Replica
              10.0.32.160:3306
```

- **ALB:** WEB01/WEB02 요청 분산
- **Apache:** WAS01/WAS02 요청 분산
- **Spring Boot:** 비즈니스 로직 처리
- **MySQL:** 데이터 저장 및 복제
- **Bastion:** Private EC2 관리 접속
- **Zabbix:** 서버 및 서비스 모니터링

## 3. 마이그레이션 구축 순서

### 1단계 — 기존 On-Premise 환경 조사

AWS로 이전하기 전에 기존 WEB/WAS/DB의 버전과 설정을 확인한다.

**WEB 서버**

```bash
sudo systemctl status apache2
sudo apache2ctl -S
```

**WAS 서버**

```bash
java -version
sudo systemctl status marble
```

**DB 서버**

```sql
SELECT VERSION();
SHOW DATABASES;
```

확인 사항:
- OS 및 애플리케이션 버전
- 서버별 IP와 사용 포트
- Apache Reverse Proxy 설정
- Spring Boot DB 연결 설정
- MySQL 데이터 및 계정
- 백업 및 복구 방법

**이유:** 기존 환경과 AWS 환경의 버전·설정 불일치로 발생하는 장애를 예방하기 위해서이다.

### 2단계 — AWS VPC 및 Subnet 구성

Marble VPC:

```text
10.0.0.0/16
```

실제 8개 서브넷:

| 용도 | CIDR |
|---|---|
| Public-A | 10.0.1.0/24 |
| Public-C | 10.0.2.0/24 |
| Private WEB-A | 10.0.11.0/24 |
| Private WEB-C | 10.0.12.0/24 |
| Private WAS-A | 10.0.21.0/24 |
| Private WAS-C | 10.0.22.0/24 |
| Private DB-A | 10.0.31.0/24 |
| Private DB-C | 10.0.32.0/24 |

**구성 이유**
- WEB/WAS/DB 네트워크 계층 분리
- 외부 직접 접근 제한
- 서로 다른 AZ를 활용한 가용성 확보 기반
- 계층별 보안 및 라우팅 관리

※ WEB/WAS/DB는 Private Subnet에 배치하고 ALB는 Public Subnet에 배치한다.

### 3단계 — IGW / NAT Gateway / Route Table

**Public Route Table**

```text
10.0.0.0/16 → local
0.0.0.0/0   → Internet Gateway
```

**NAT Gateway를 사용하는 Private Route Table 예시**

```text
10.0.0.0/16 → local
0.0.0.0/0   → NAT Gateway
```

| 구성 요소 | 역할 |
|---|---|
| Internet Gateway | VPC와 인터넷 간 통신 경로 |
| NAT Gateway | Private 서버의 외부 방향 연결 지원 |
| Route Table | 목적지에 따른 네트워크 경로 결정 |
| local | 같은 VPC 내부의 사설 IP 통신 |

**중요:** WEB → WAS → DB 통신은 VPC 내부 `local` 경로를 사용하므로 NAT Gateway가 필요하지 않다.

### 4단계 — Security Group 구성

요청 경로:

```text
Internet → ALB:80 → WEB:80 → WAS:8080 → DB:3306
```

| SG | 인바운드 허용 |
|---|---|
| ALB-SG | 사용자 → HTTP 80 또는 실제 HTTPS 포트 |
| WEB-SG | ALB-SG → TCP 80 |
| WAS-SG | WEB-SG → TCP 8080 |
| DB-SG | WAS-SG → TCP 3306 |
| DB 복제 규칙 | Replica → Primary TCP 3306 |
| SSH 관리 규칙 | Bastion-SG → TCP 22 |

Security Group은 Stateful 방식이므로 허용된 연결에 대한 응답 트래픽은 자동으로 허용된다.

**핵심:** 각 서버는 필요한 계층에서 오는 트래픽만 허용하도록 구성한다.

### 5단계 — EC2 및 Bastion 구성

AWS EC2 콘솔에서 다음 항목을 설정한다.

1. Ubuntu AMI 선택
2. Instance Type 선택
3. Key Pair 지정
4. Marble VPC 및 Subnet 선택
5. Security Group 연결
6. EBS 용량 지정
7. EC2 생성 및 SSH 연결 확인

관리 접속:

```text
관리자 PC
    |
    v
Bastion (Public)
    |
    v
WEB / WAS / DB (Private)
```

**이유:** WEB/WAS/DB에 Public IP를 직접 부여하지 않고 관리하기 위함이다.

### 6단계 — DB → WAS → WEB 구성

의존성을 고려하여 데이터베이스부터 구성하고 애플리케이션 및 웹서버를 연결한다.

**① MySQL DB 구성**

```text
기존 VMware MySQL
       |
       v
데이터 백업
       |
       v
AWS DB01 복원
       |
       v
데이터 정합성 검증
       |
       v
DB02 Primary/Replica 복제 구성
```

- 기존 DB 데이터 이전
- MySQL 계정·권한 확인
- GTID 복제 구성
- 데이터 동기화 확인

**② Spring Boot WAS 구성**

Java 버전 확인:

```bash
java -version
```

서비스 상태 확인:

```bash
sudo systemctl status marble
```

AWS DB01 연결 예시:

```properties
spring.datasource.url=jdbc:mysql://10.0.31.42:3306/DB_NAME
```

※ 실제 DB 이름과 계정 정보는 기존 Marble 설정을 기준으로 확인한다.

**③ Apache WEB 구성**

실제 Marble Reverse Proxy 설정:

```apache
<Proxy "balancer://marblecluster">
    BalancerMember "http://10.0.21.126:8080"
    BalancerMember "http://10.0.22.9:8080"

    ProxySet lbmethod=byrequests
</Proxy>

ProxyPass "/" "balancer://marblecluster/"
ProxyPassReverse "/" "balancer://marblecluster/"
```

설정 문법 확인:

```bash
sudo apache2ctl configtest
```

**핵심:** VMware에서 사용하던 Proxy 및 DB 연결 IP를 AWS 사설 IP로 변경해야 한다.

### 7단계 — ALB 및 Target Group 구성

AWS 콘솔에서 다음 순서로 구성한다.

1. WEB Target Group 생성
2. WEB01/WEB02 등록
3. Health Check Path 및 Success Code 설정
4. Public ALB 생성
5. 서로 다른 AZ의 Public Subnet 두 개 선택
6. Listener와 Target Group 연결
7. Target Healthy 상태 확인
8. ALB DNS를 통한 서비스 응답 검증

```text
Client
   |
   v
Public ALB
   |
   +── WEB01 (Healthy)
   |
   +── WEB02 (Healthy)
```

**주의:** ALB Target이 Healthy라고 해서 WAS 및 DB의 모든 기능까지 정상이라는 의미는 아니다.

### 8단계 — 서비스 전환(Cutover)

AWS 환경을 구축한 뒤 기존 환경에서 새로운 환경으로 서비스 요청을 전환하는 과정이다.

일반적인 순서:

1. 기존 서비스 상태 확인
2. AWS WEB/WAS/DB 사전 검증
3. 기존 DB의 쓰기 통제 및 최종 데이터 동기화
4. 필요 시 DNS 등 사용자 접속 경로 변경
5. 데이터 조회·저장·로그인·세션 검증
6. 안정화 후 기존 환경 정리

**Rollback:** AWS 전환 후 문제가 발생하면 기존 환경으로 되돌리는 과정이다.

단, 새 환경에서 데이터 변경이 발생했다면 DB 정합성을 확인하지 않고 단순히 접속 주소만 되돌려서는 안 된다.

※ Cutover와 Rollback은 일반적인 마이그레이션 절차이며, Marble에서 실서비스 DNS 전환까지 수행했다는 의미는 아니다.

---

## 4. 마이그레이션 장애 대응

| 장애 | 확인 항목 |
|---|---|
| ALB 접속 실패 | Listener, Target Group, Health Check |
| WEB → WAS 실패 | Apache ProxyPass, SG, TCP 8080 |
| WAS → DB 실패 | JDBC URL, SG, TCP 3306, DB 권한 |
| Spring Boot 실행 실패 | Java 버전, systemd, 애플리케이션 로그 |
| 데이터 누락 | DB 백업·복원, 최종 변경분 |
| 로그인/세션 오류 | Spring Session JDBC, MySQL 세션 데이터 |
| 외부 패키지 설치 실패 | NAT Gateway, Route Table, 외부 연결 |

장애 원인은 다음 순서로 좁힌다.

```text
ALB
 |
 v
WEB (Apache)
 |
 v
WAS (Spring Boot)
 |
 v
DB (MySQL)
```

각 계층에서 서비스 상태, 네트워크 통신, 로그를 확인한다.

---

## 5. 최종 핵심 정리

### 마이그레이션 전체 구축 순서

```text
1. 기존 VMware 인프라 조사
               |
               v
2. AWS VPC / Subnet 설계
               |
               v
3. IGW / NAT / Route Table
               |
               v
4. Security Group 구성
               |
               v
5. EC2 / Bastion 생성
               |
               v
6. MySQL 데이터 이전 및 복제
               |
               v
7. Spring Boot WAS 구성
               |
               v
8. Apache WEB 및 Reverse Proxy
               |
               v
9. ALB / Target Group 연결
               |
               v
10. 통합 검증 및 서비스 전환
```

### 반드시 기억할 핵심 5가지

1. **Migration:** 서버·애플리케이션·데이터·네트워크·설정을 새로운 환경으로 이전
2. **네트워크 분리:** Public ALB, Private WEB/WAS/DB 구조
3. **설정 변경:** ProxyPass, JDBC URL 등 기존 IP 의존 설정 수정
4. **데이터 정합성:** 이전 중 발생한 추가 변경 데이터까지 확인
5. **Cutover/Rollback:** 서비스 전환과 장애 발생 시 복구 절차 준비

### 프로젝트 설명

> VMware Ubuntu 환경의 Apache–Spring Boot–MySQL 3-Tier 서비스를 AWS EC2 기반으로 이전하면서 VPC 네트워크 분리, ALB를 통한 WEB 이중화, Apache를 통한 WAS 이중화, MySQL Primary/Replica 복제를 적용하였다.

**핵심 결론:** 마이그레이션은 단순한 서버 이전이 아니라 기존 서비스의 기능과 데이터를 유지하면서 새로운 인프라 환경에 맞게 네트워크, 보안, 가용성 및 운영 구조를 재구성하는 작업이다.


<img width="935" height="1683" alt="image" src="https://github.com/user-attachments/assets/3105b9f6-79cf-4302-a95e-f9d6388f34f4" />


