
# A과정 6장 1교시 — AWS VPC 및 CIDR 설계

## 1. VPC란?

VPC(Virtual Private Cloud)는 AWS에서 사용하는 논리적으로 격리된 가상 네트워크이다.

### VPC를 사용하는 이유

- 서버에 사용할 사설 IP 주소 범위 지정
- WEB/WAS/DB 네트워크 분리
- 라우팅 및 보안 정책 관리
- 여러 가용 영역(AZ)에 서버 배치

### VMware와 비교

| 구분 | VMware | AWS |
|---|---|---|
| 네트워크 | VMnet 가상 네트워크 | VPC |
| 서버 | 가상머신(VM) | EC2 |
| IP 관리 | 가상 네트워크 및 OS 설정 | VPC·Subnet·ENI |
| 통신 제어 | 가상 네트워크·방화벽 | Route Table·SG·NACL |

기존 VMware: 192.168.111.0/24

AWS 설계: 10.0.0.0/16

---

## 2. CIDR란?

CIDR는 IP 주소의 네트워크 범위를 표현하는 방식이다.

예시:

10.0.0.0/16

- 10.0.0.0: 네트워크 주소
- /16: 앞의 16비트가 네트워크 부분
- 나머지 16비트: 호스트 주소 부분

### CIDR 비교

| CIDR | 서브넷 마스크 | 전체 IP 수 |
|---|---|---:|
| /16 | 255.255.0.0 | 65,536 |
| /24 | 255.255.255.0 | 256 |

계산 공식:

IP 주소 수 = 2^(32 - CIDR 접두사)

### AWS Subnet의 예약 IP

AWS는 서브넷마다 첫 4개와 마지막 1개의 IPv4 주소를 예약한다.

예시: 10.0.1.0/24

- 전체: 256개
- AWS 예약: 5개
- EC2 등에 할당 가능: 251개

---

## 3. VPC 주소 설계

이번 프로젝트의 VPC:

10.0.0.0/16

### /16을 선택한 이유

1. 여러 Subnet을 생성하기 위한 주소 공간 확보
2. WEB/WAS/DB 분리 및 서버 증설 고려
3. 기존 VMware 대역과 IP 중복 방지

주의: VPN 등으로 다른 네트워크와 연결할 계획이 있다면 CIDR 중복 여부를 반드시 확인해야 한다.

---

## 4. 프로젝트 Subnet 설계

### 전체 구조

```text
VPC 10.0.0.0/16
│
├── AZ-a
│   ├── Public-WEB-a    10.0.1.0/24
│   ├── Private-WAS-a   10.0.11.0/24
│   └── Private-DB-a    10.0.21.0/24
│
└── AZ-c
    ├── Public-WEB-c    10.0.2.0/24
    ├── Private-WAS-c   10.0.12.0/24
    └── Private-DB-c    10.0.22.0/24
```

### 핵심 원칙

- VPC는 리전 단위 리소스다.
- Subnet은 하나의 AZ에 속한다.
- Subnet CIDR은 VPC CIDR 범위에 포함되어야 한다.
- 같은 VPC의 Subnet CIDR은 서로 겹칠 수 없다.
- Public/Private 여부는 이름이나 IP 주소가 아니라 라우팅 구성으로 결정된다.

실제 Subnet 생성은 6장 2교시에서 진행한다.

---

## 5. AWS 콘솔에서 VPC 생성

### 생성 순서

1. AWS 콘솔 접속
2. 리전: 서울(ap-northeast-2)
3. VPC → Your VPCs → Create VPC
4. Resources to create: VPC only
5. Name tag: marble-vpc
6. IPv4 CIDR: 10.0.0.0/16
7. IPv6 CIDR: No IPv6 CIDR block
8. Tenancy: Default
9. Create VPC

### VPC only를 선택하는 이유

VPC만 먼저 생성하고 Subnet, Route Table, Gateway 등을 단계별로 학습하기 위해서다.

VPC 자체에는 추가 사용 요금이 없지만 NAT Gateway, EC2, 공인 IPv4 등은 별도 비용이 발생할 수 있다.

---

## 6. VPC 생성 결과 확인

AWS 콘솔에서 확인할 항목:

| 항목 | 예상 값 |
|---|---|
| VPC Name | marble-vpc |
| IPv4 CIDR | 10.0.0.0/16 |
| State | Available |
| Region | ap-northeast-2 |
| VPC ID | vpc-... |

### AWS CLI 조회

```bash
aws ec2 describe-vpcs \
  --region ap-northeast-2 \
  --filters "Name=tag:Name,Values=marble-vpc" \
  --query 'Vpcs[].{VpcId:VpcId,CIDR:CidrBlock,State:State}' \
  --output table
```

옵션:

- --region: 조회 리전
- --filters: 조회 조건
- --query: 출력할 항목 선택
- --output table: 표 형식 출력

---

## 7. VPC 생성 시 기본 제공 리소스

VPC 생성 시 기본적으로 다음 리소스가 제공된다.

- Main Route Table
- Default Security Group
- Default Network ACL
- DHCP Option Set 연결

### Main Route Table 기본 경로

| Destination | Target |
|---|---|
| 10.0.0.0/16 | local |

`local`은 VPC 내부 CIDR로 향하는 트래픽의 라우팅 경로를 의미한다.

중요:

**VPC 생성만으로 인터넷 연결이 되는 것은 아니다.**

Internet Gateway, NAT Gateway 및 외부 라우팅은 7장에서 구성한다.

---

## 8. 주요 설정 오류

| 문제 | 원인 |
|---|---|
| VPC CIDR 생성 실패 | 허용되지 않는 CIDR 크기 또는 잘못된 주소 |
| Subnet 생성 실패 | VPC 주소 범위를 벗어남 |
| Subnet CIDR 충돌 | 기존 Subnet과 주소 범위 중복 |
| 생성한 VPC가 안 보임 | 다른 AWS 리전에서 조회 |
| VPC 생성 후 인터넷 불가 | IGW·Route Table 등 연결 구성 미완료 |

### CIDR 충돌 예시

```text
VPC:      10.0.0.0/16

Subnet A: 10.0.1.0/24
Subnet B: 10.0.1.0/25
```

Subnet A와 B는 주소 범위가 겹치므로 같은 VPC 안에서 각각 생성할 수 없다.

---

## 9. 최종 핵심 정리

1. VPC는 AWS의 논리적으로 격리된 가상 네트워크다.
2. CIDR는 IP 주소의 네트워크 범위를 나타낸다.
3. /16은 65,536개, /24는 256개의 IPv4 주소를 포함한다.
4. AWS Subnet은 IPv4 주소 5개를 예약한다.
5. VPC는 리전 단위, Subnet은 AZ 단위다.
6. Subnet은 VPC 범위 안에서 서로 겹치지 않게 설계한다.
7. Public/Private는 라우팅 설정으로 결정된다.
8. VPC만 생성해도 인터넷에 연결되는 것은 아니다.

### 반드시 기억할 3가지

**VPC = AWS 네트워크의 전체 주소 공간**

**CIDR = IP 주소의 범위와 규모**

**Subnet = VPC 주소 공간을 나누어 특정 AZ에 배치한 네트워크**

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/5b1f2485-c495-4364-a0d6-3d8a599273a5" />

