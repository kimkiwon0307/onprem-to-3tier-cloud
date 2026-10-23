
# A과정 7장 1교시 — AWS Route Table 및 Internet Gateway

## 1. Route Table이란?

Route Table은 패킷의 목적지 IP 주소를 확인하고, 어디로 전달할지 결정하는 경로표이다.

### Route Table 구성

| 항목 | 의미 |
|---|---|
| Destination | 패킷의 목적지 IP 주소 범위 |
| Target | 패킷을 전달할 다음 경로 |
| Status | 해당 경로의 사용 가능 상태 |

### Public Route Table 예시

```text
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         igw-xxxxxxxx
```

- `10.0.0.0/16 → local`: VPC 내부 통신
- `0.0.0.0/0 → IGW`: 인터넷 목적지로 향하는 기본 경로

**0.0.0.0/0은 모든 접근을 허용한다는 뜻이 아니라 기본 목적지 경로를 의미한다.**

---

## 2. Route Table의 경로 선택 원리

AWS는 목적지 IP 주소와 가장 구체적으로 일치하는 경로를 선택한다.

이를 Longest Prefix Match라고 한다.

### 예시

목적지: 10.0.11.25

```text
10.0.0.0/16 → local
0.0.0.0/0   → IGW
```

두 경로 중 더 구체적인 `10.0.0.0/16 → local`을 사용한다.

목적지가 8.8.8.8이면 `0.0.0.0/0 → IGW`를 사용한다.

**VPC 내부 통신은 기본 인터넷 경로보다 local 경로가 우선된다.**

---

## 3. Internet Gateway(IGW)란?

IGW는 VPC가 인터넷과 통신할 수 있도록 연결하는 AWS 관리형 게이트웨이다.

### 인터넷 연결에 필요한 조건

1. IGW가 VPC에 연결되어 있어야 한다.
2. Subnet에 적용되는 Route Table에 IGW 경로가 있어야 한다.
3. EC2의 직접 IPv4 인터넷 통신에는 공인 IPv4 주소가 필요하다.
4. Security Group과 NACL에서 필요한 통신을 허용해야 한다.

IGW를 생성하고 VPC에 연결하는 것만으로 모든 Subnet이 인터넷에 연결되지는 않는다.

---

## 4. Public Subnet과 Private Subnet

### Public Subnet

IGW로 향하는 직접 라우팅 경로가 있는 Subnet이다.

```text
10.0.0.0/16 → local
0.0.0.0/0   → IGW
```

### Private Subnet

IGW로 향하는 직접 라우팅 경로가 없는 Subnet이다.

인터넷으로 나가는 통신이 필요한 경우 NAT Gateway 등의 경로를 사용할 수 있다.

```text
10.0.0.0/16 → local
0.0.0.0/0   → NAT Gateway
```

NAT Gateway는 7장 2교시에서 자세히 학습한다.

---

## 5. 실제 Marble 프로젝트 적용

### 네트워크 구성

- VPC: 10.0.0.0/16
- Public Subnet: 2개
- Private Subnet: 6개
- AZ: 2개

### 서비스 요청 흐름

```text
인터넷 사용자
      ↓
Internet Gateway
      ↓
Public Subnet
ALB
      ↓ VPC 내부 통신(local)
Private WEB01 / WEB02
      ↓
Private WAS01 / WAS02
      ↓
Private DB Primary / Replica
```

### WEB 서버를 Private Subnet에 배치한 이유

- WEB EC2의 직접적인 인터넷 노출 최소화
- ALB를 통해 외부 HTTP/HTTPS 요청 수신
- Security Group으로 ALB → WEB 접근 제한
- WEB·WAS·DB 계층별 네트워크 구조 유지

**ALB와 WEB이 같은 VPC에 있으므로 local 경로를 통해 내부 통신할 수 있다.**

ALB에서 Private WEB으로 요청을 전달하는 데 NAT Gateway는 필요하지 않다.

단, Target Group, Security Group, NACL, 서비스 상태가 정상이어야 한다.

---

## 6. Subnet Association

Subnet Association은 어떤 Route Table이 Subnet에 적용되는지 지정하는 연결 관계이다.

### 핵심 사항

1. Route Table을 생성했다고 모든 Subnet에 자동 적용되는 것은 아니다.
2. Subnet에 별도 연결이 없으면 Main Route Table이 적용된다.
3. 여러 Subnet이 하나의 Route Table을 공유할 수 있다.
4. Public Subnet에 실제로 IGW 경로가 적용되는지 확인해야 한다.

### Marble 프로젝트 확인 대상

- Public-a: 10.0.1.0/24
- Public-c: 10.0.2.0/24

두 Subnet 모두 IGW 기본 경로가 적용되는지 확인한다.

---

## 7. AWS 콘솔 검증

### IGW 확인

AWS 콘솔 → VPC → Internet Gateways

확인:
- IGW ID
- State: Attached
- 올바른 VPC ID

### Route Table 확인

AWS 콘솔 → VPC → Route Tables

확인:
- Routes 탭: Destination, Target, Status
- Subnet associations: 실제 적용된 Subnet
- Public Subnet의 IGW 경로
- Private Subnet의 IGW 직접 경로 유무

기존 프로젝트의 실제 Route Table 구성은 콘솔에서 검증한다.

---

## 8. 주요 장애 대응

### ① ALB 외부 접속 실패

확인 순서:

1. ALB가 internet-facing인지 확인
2. ALB가 올바른 Public Subnet에 있는지 확인
3. Public Route Table의 IGW 경로 확인
4. IGW가 VPC에 Attached 상태인지 확인
5. ALB Security Group 및 Listener 확인

### ② ALB 접속은 되지만 WEB이 Unhealthy

확인:
- Target Group 등록 상태
- Target Port 및 Health Check
- ALB → WEB Security Group
- NACL
- Apache 실행 상태 및 HTTP 응답

**WEB이 Unhealthy라고 무조건 IGW 또는 Route Table 문제는 아니다.**

### ③ IGW 경로가 있는데 인터넷 접속 실패

확인:
- IGW의 VPC 연결 상태
- Subnet과 Route Table 연결
- 직접 인터넷 통신에 필요한 공인 IPv4
- Security Group 및 NACL
- DNS 및 대상 서비스 상태

---

## 9. 반드시 기억할 핵심

1. Route Table은 패킷의 이동 경로를 결정한다.
2. Destination은 목적지, Target은 다음 경로이다.
3. local은 VPC 내부 통신을 위한 경로이다.
4. 0.0.0.0/0은 기본 경로를 뜻한다.
5. AWS는 더 구체적으로 일치하는 경로를 우선 선택한다.
6. IGW는 VPC의 인터넷 연결을 가능하게 한다.
7. Public Subnet에는 IGW 직접 경로가 필요하다.
8. Subnet Association이 올바르게 적용되어야 한다.
9. ALB는 Public Subnet, WEB은 Private Subnet에 배치할 수 있다.
10. 같은 VPC의 ALB → WEB 통신에는 일반적으로 local 경로를 사용한다.

### 최종 암기 문장

**Route Table = 패킷을 어디로 보낼 것인가?**

**IGW = VPC와 인터넷을 연결하는 게이트웨이**

**local = VPC 내부 통신 경로**

**0.0.0.0/0 = 더 구체적인 경로가 없을 때 사용하는 기본 경로**

**Subnet Association = 어떤 Subnet에 Route Table을 적용하는가?**


<img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/cc804938-526e-43ee-932d-dacb48881249" />








