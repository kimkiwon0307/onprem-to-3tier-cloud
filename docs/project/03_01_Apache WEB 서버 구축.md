
# A과정 3장 1교시 — Apache 개념 및 WEB 서버 구조

## 1. Apache HTTP Server란?

Apache는 클라이언트의 HTTP 요청을 수신하고 응답하는 웹 서버 소프트웨어이다.

### 주요 역할
- HTTP 요청 및 응답 처리
- HTML, CSS, 이미지 등 정적 콘텐츠 제공
- Reverse Proxy를 통한 WAS 요청 전달
- 여러 WAS 서버로 요청 분산(Load Balancing)
- Access Log 및 Error Log 기록

## 2. WEB / WAS / DB 역할

| 구분 | 기술 | 역할 | 포트 |
|---|---|---|---|
| WEB | Apache | HTTP 요청 수신 및 전달 | 80 |
| WAS | Spring Boot / Tomcat | 애플리케이션 로직 처리 | 8080 |
| DB | MySQL | 데이터 저장 및 조회 | 3306 |

### 프로젝트 요청 흐름

사용자 → Apache(WEB) → Spring Boot(WAS) → MySQL(DB)

응답은 반대 방향으로 전달된다.

- WEB01: 192.168.111.141
- WAS01: 192.168.111.142
- DB01: 192.168.111.143

### WEB과 WAS를 분리하는 이유
1. 서버별 역할과 자원 분리
2. 계층별 장애 원인 분석 용이
3. WAS 확장 및 요청 분산 가능
4. WEB 계층에서 요청 전달과 접근 제어 관리

단점: 서버 수 증가에 따른 관리 비용과 네트워크 복잡성 증가

## 3. 정적 콘텐츠와 동적 콘텐츠

- 정적 콘텐츠: HTML, CSS, 이미지 등 파일을 그대로 제공
- 동적 콘텐츠: 로그인, 회원정보 조회 등 애플리케이션 로직으로 응답 생성

Apache는 정적 콘텐츠를 제공할 수 있고,
Spring Boot는 동적 요청을 처리한다.

## 4. HTTP 기본 개념

HTTP는 클라이언트와 서버가 요청과 응답을 주고받는 프로토콜이다.

### 주요 HTTP 메서드
- GET: 조회
- POST: 데이터 제출 및 처리
- PUT: 리소스 생성 또는 전체 교체
- PATCH: 일부 변경
- DELETE: 삭제

### 주요 HTTP 상태 코드
- 200: 성공
- 301 / 302: 리다이렉션
- 400: 잘못된 요청
- 401: 인증 필요 또는 실패
- 403: 접근 거부
- 404: 리소스 없음
- 500: 서버 내부 오류
- 502: 게이트웨이가 잘못된 업스트림 응답 수신
- 503: 서비스 이용 불가
- 504: 업스트림 응답 시간 초과

## 5. Reverse Proxy

Reverse Proxy는 클라이언트의 요청을 대신 받아
백엔드 서버에 전달하는 기능이다.

### Apache 설정 예시

```apache
ProxyPass / http://192.168.111.142:8080/
ProxyPassReverse / http://192.168.111.142:8080/
```

- ProxyPass: 클라이언트 요청을 WAS에 전달
- ProxyPassReverse: WAS의 리다이렉션 관련 응답 헤더 URL 보정

### Reverse Proxy와 Load Balancing 차이

- Reverse Proxy: 요청을 백엔드로 대신 전달
- Load Balancing: 요청을 여러 백엔드 서버에 분산

Reverse Proxy를 사용한다고 반드시 Load Balancing을 하는 것은 아니다.

## 6. 핵심 Linux 명령어

```bash
hostname
# 현재 서버 호스트명 확인

ip -br addr
# IP 주소 및 인터페이스 확인

sudo ss -lntp '( sport = :80 )'
# TCP 80 포트 LISTEN 확인

dpkg -l apache2
# Apache 패키지 설치 여부 확인

curl -i --max-time 5 http://127.0.0.1/
# 로컬 HTTP 응답 확인

nc -vz 192.168.111.142 8080
# WAS TCP 8080 연결 확인
```

### 명령어별 검증 범위
- ping: ICMP 통신 확인
- nc: TCP 포트 연결 확인
- curl: HTTP 응답 확인
- ss: 서버의 포트 수신 상태 확인

주의: TCP 연결 성공이 애플리케이션 정상 동작을 보장하지는 않는다.

## 7. 확인문제 정답 (5/5)

1. Apache의 역할 → HTTP 요청 수신 및 WAS 전달
2. 동적 콘텐츠 → 로그인 정보를 확인하고 응답 생성
3. Reverse Proxy 요청 전달 설정 → ProxyPass
4. 요청 흐름 → 사용자 → Apache → WAS → Apache → 사용자
5. nc로 확인 가능한 것 → 해당 TCP 포트 연결 가능 여부

## 핵심 요약

1. Apache는 WEB 계층에서 HTTP 요청을 처리한다.
2. Spring Boot는 WAS 계층에서 비즈니스 로직을 처리한다.
3. MySQL은 데이터를 저장하고 조회한다.
4. Reverse Proxy는 클라이언트 요청을 WAS로 전달한다.
5. Load Balancing은 여러 WAS에 요청을 분산한다.
6. TCP 연결과 HTTP 애플리케이션 정상 동작은 별도로 검증한다.
