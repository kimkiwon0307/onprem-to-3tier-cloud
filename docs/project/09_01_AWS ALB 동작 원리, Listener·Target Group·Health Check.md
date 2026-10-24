
# A과정 9장 1교시 — AWS ALB 핵심 정리

## 1. ALB란?

ALB(Application Load Balancer)는 HTTP/HTTPS 요청을 여러 대상 서버에 분산하는 AWS 관리형 로드밸런서이다.

OSI 7계층의 HTTP 요청 정보를 활용한다.

### ALB를 사용하는 이유

1. WEB 서버 한 대에 요청이 집중되는 것을 방지
2. WEB01/WEB02로 사용자 요청 분산
3. Health Check를 통한 정상 대상 선택
4. 서버 장애 발생 시 정상 대상에 요청 전달
5. URL 경로·호스트명에 따른 라우팅 지원

**ALB는 요청 분산과 정상 대상 선택을 담당한다.**

---

## 2. Marble 프로젝트 ALB 구조

### 네트워크 구성

- VPC: 10.0.0.0/16
- Public Subnet: 10.0.1.0/24, 10.0.2.0/24
- Private WEB: 10.0.11.0/24, 10.0.12.0/24

### 요청 흐름

```text
인터넷 사용자
      ↓
Public ALB
      ↓
Listener
      ↓
WEB Target Group
      ↓
Private WEB01 / WEB02 (Apache)
      ↓
Private WAS01 / WAS02 (Spring Boot)
      ↓
Private MySQL
```

### 설계 이유

- ALB는 Public Subnet에서 인터넷 요청 수신
- WEB01/WEB02는 Private Subnet에 배치
- ALB가 WEB으로 HTTP 요청 전달
- Apache는 필요한 요청을 WAS에 전달
- WEB EC2의 직접적인 인터넷 노출 최소화

**ALB → WEB 로드밸런싱과 WEB → WAS Reverse Proxy는 서로 다른 역할이다.**

---

## 3. ALB의 핵심 구성 요소

| 구성 요소 | 역할 |
|---|---|
| Listener | 사용자 요청 수신 |
| Listener Rule | 요청 처리 방식 결정 |
| Target Group | 요청을 전달할 서버 관리 |
| Registered Target | 등록된 실제 대상 서버 |
| Health Check | 대상 서버 정상 여부 검사 |

### 요청 처리 과정

```text
사용자 요청
    ↓
Listener (HTTP 80 / HTTPS 443)
    ↓
Listener Rule 평가
    ↓
Target Group 선택
    ↓
Healthy 대상 선택
    ↓
WEB01 또는 WEB02
```

※ Health Check는 사용자 요청이 지나가는 단계가 아니라 대상 서버를 주기적으로 검사하는 기능이다.

---

## 4. Listener란?

Listener는 ALB가 사용할 프로토콜과 포트를 정의하고 요청을 처리한다.

### 설정 예시

```text
Protocol: HTTP
Port: 80
Default Action: Forward
Target Group: WEB-TG
```

### 주요 설정

- Protocol: HTTP 또는 HTTPS
- Port: 사용자 요청을 받을 포트
- Default Action: 기본 처리 방식
- Listener Rule: 조건에 따른 요청 처리

HTTPS 443으로 요청을 받더라도 ALB에서 TLS를 종료한 뒤 WEB 서버에는 HTTP 80으로 전달할 수 있다.

실제 Marble 프로젝트의 Listener 포트와 HTTPS 사용 여부는 AWS 콘솔에서 확인한다.

---

## 5. Target Group이란?

Target Group은 ALB가 요청을 전달할 대상 서버들을 관리하는 그룹이다.

### Marble 프로젝트 예시

```text
Target Group: WEB-TG
Protocol: HTTP
Port: 80

Registered Targets:
- WEB01
- WEB02
```

### Target Type

| 유형 | 설명 |
|---|---|
| Instance | EC2 인스턴스 ID로 등록 |
| IP | 대상 IP 주소로 등록 |

### 핵심

1. WEB01/WEB02를 Target Group에 등록한다.
2. Listener가 해당 Target Group으로 요청을 전달해야 한다.
3. Health Check를 통해 대상 상태를 확인한다.

**Target Group을 생성하는 것만으로 로드밸런싱이 완성되는 것은 아니다.**

---

## 6. Health Check란?

Health Check는 ALB가 등록된 서버에 주기적으로 요청을 보내 정상 여부를 판단하는 기능이다.

### 설정 예시

```text
Protocol: HTTP
Port: traffic-port
Path: /
Success Codes: 200
```

### 주요 설정

| 설정 | 의미 |
|---|---|
| Protocol | 검사 프로토콜 |
| Port | 검사 대상 포트 |
| Path | 검사할 URL |
| Success Codes | 정상으로 인정할 응답 코드 |
| Interval | 검사 간격 |
| Timeout | 응답 제한 시간 |
| Healthy Threshold | 정상 판단에 필요한 연속 성공 횟수 |
| Unhealthy Threshold | 비정상 판단에 필요한 연속 실패 횟수 |

### Health Check 주의사항

- EC2가 Running이어도 Health Check는 실패할 수 있다.
- Apache가 실행 중이어도 HTTP 403·404 등을 반환할 수 있다.
- 검사 경로가 실제 WEB 서버에서 정상 응답해야 한다.
- WAS의 /health 경로가 존재해도 WEB에서 같은 경로가 동작한다고 보장할 수 없다.

**Health Check는 서버 실행 여부가 아니라 설정된 조건에 대한 실제 응답으로 판단한다.**

---

## 7. Healthy / Unhealthy 동작

### WEB01 Healthy / WEB02 Unhealthy

```text
ALB
 ├── WEB01: Healthy   → 요청 전달
 └── WEB02: Unhealthy → 일반적으로 요청 제외
```

### WEB01 / WEB02 모두 Unhealthy

ALB는 모든 등록 대상이 Unhealthy인 상황에서 Fail-open 동작으로 비정상 대상에도 요청을 전달할 수 있다.

### 반드시 구분할 내용

- ALB: 정상 대상에 요청 분산
- Auto Scaling 등 별도 구성: 서버 추가·교체

**ALB만 구성했다고 장애 서버를 자동 재생성하는 것은 아니다.**

---

## 8. AWS 콘솔 검증

### ALB 확인

AWS 콘솔 → EC2 → Load Balancers

확인:
- Type: Application
- Scheme: Internet-facing
- State: Active
- VPC 및 Public Subnet
- Availability Zones
- Security Group
- DNS Name

### Listener 확인

EC2 → Load Balancers → Listeners and rules

확인:
- HTTP/HTTPS 프로토콜
- Listener 포트
- Default Action
- 연결된 Target Group

### Target Group 확인

EC2 → Target Groups

확인:
- Target Type
- Protocol / Port
- WEB01/WEB02 등록 상태
- Healthy / Unhealthy 상태
- Health Check Path / Success Codes

※ 실제 설정을 변경하지 않고 기존 Marble 프로젝트의 구성을 조회한다.

---

## 9. Linux에서 WEB 응답 검증

### Apache 서비스 확인

```bash
sudo systemctl status apache2
```

### TCP 80 수신 상태 확인

```bash
sudo ss -lntp | grep ':80'
```

### HTTP 응답 확인

```bash
curl -i --max-time 5 http://127.0.0.1/
```

- `-i`: 응답 헤더 포함
- `--max-time 5`: 최대 실행 시간 5초
- `127.0.0.1`: 현재 서버 자신

### 주의사항

WEB 내부에서 HTTP 200을 받았더라도 ALB → WEB 통신까지 검증된 것은 아니다.

최종적으로 Target Group의 Health Check 상태도 확인해야 한다.

---

## 10. 주요 장애 대응

### ① Target Group에 WEB 서버가 없음

확인:
- WEB01/WEB02 등록 여부
- Target Group 선택
- Listener 연결 상태

### ② WEB02만 Unhealthy

확인:
1. WEB02 EC2 상태
2. WEB02 Security Group
3. Target Group 포트
4. Health Check Path
5. Apache 실행 상태
6. Apache 로그 및 HTTP 응답

### ③ WEB01/WEB02 모두 Unhealthy

확인:
1. 공통 Security Group 규칙
2. Target Group 포트
3. Health Check URL
4. Success Codes
5. Apache 공통 설정
6. 실제 HTTP 응답

**Unhealthy의 원인이 반드시 EC2 장애인 것은 아니다.**

---

## 11. 반드시 기억할 핵심

1. ALB는 HTTP/HTTPS 요청을 여러 서버에 분산한다.
2. Listener는 요청을 받는다.
3. Listener Rule은 요청 처리 방식을 결정한다.
4. Target Group은 요청을 전달할 서버들을 관리한다.
5. Health Check는 대상 서버의 정상 여부를 검사한다.
6. EC2 Running과 Target Healthy는 서로 다르다.
7. Healthy 대상이 있으면 일반적으로 Unhealthy 대상은 요청 전달에서 제외한다.
8. 모든 대상이 Unhealthy이면 ALB가 Fail-open할 수 있다.
9. ALB는 장애 EC2를 자동으로 재생성하지 않는다.
10. Marble 프로젝트는 Public ALB → Private WEB01/WEB02 구조이다.

### 최종 암기 문장

**ALB = HTTP/HTTPS 요청 분산**

**Listener = 요청을 받는 입구**

**Target Group = 요청을 전달할 서버 목록**

**Health Check = 대상 서버 정상 여부 검사**

**Healthy ≠ EC2 Running**

**ALB → WEB = 로드밸런싱**

**WEB → WAS = Reverse Proxy**
