
# A과정 5장 1교시 — MySQL 기본 운영 및 계정·권한

## 1. 실습 환경

| 서버 | IP | 역할 | 포트 |
|---|---|---|---|
| WEB01 | 192.168.111.147 | Apache | 80 |
| WAS01 | 192.168.111.148 | Spring Boot | 8080 |
| DB01 | 192.168.111.149 | MySQL | 3306 |

### 요청 흐름

사용자 → Apache → Spring Boot → MySQL → Spring Boot → Apache → 사용자

- WEB → WAS: HTTP 통신
- WAS → DB: MySQL 프로토콜(TCP 3306)

---

## 2. MySQL 핵심 개념

- MySQL: SQL을 처리하고 데이터를 저장·조회하는 DBMS
- Database: 관련 테이블을 관리하는 논리적 공간
- Table: 데이터를 행과 열로 저장하는 구조
- Column: 데이터 항목
- Row: 데이터 한 건
- InnoDB: 데이터 저장, 트랜잭션 처리 등을 담당하는 스토리지 엔진

### MySQL 처리 과정

1. 클라이언트 연결
2. 사용자 인증 및 권한 확인
3. SQL 분석 및 실행
4. 데이터 조회·수정
5. 결과 반환

---

## 3. MySQL 서비스 및 포트 확인

DB01에서 실행한다.

```bash
# 서비스 상태
sudo systemctl status mysql

# 자동 시작 여부
systemctl is-enabled mysql

# TCP 3306 수신 확인
sudo ss -lntp '( sport = :3306 )'

# 최근 서비스 로그
sudo journalctl -u mysql -n 50 --no-pager
```

중요: 서비스가 active여도 WAS에서 접속할 수 있다는 의미는 아니다.

---

## 4. bind-address

MySQL이 어떤 로컬 IP에서 TCP 연결을 받을지 지정한다.

| 설정 | 의미 |
|---|---|
| 127.0.0.1 | DB 서버 내부에서만 수신 |
| 192.168.111.149 | DB01의 해당 IP에서 수신 |
| 0.0.0.0 | 모든 IPv4 로컬 인터페이스에서 수신 |

### 실제 설정 확인

MySQL 접속:

```bash
sudo mysql
```

설정 조회:

```sql
SHOW VARIABLES LIKE 'bind_address';
SHOW VARIABLES LIKE 'port';
```

무조건 0.0.0.0으로 변경하지 않고 필요한 주소와 접근 범위를 확인한다.

---

## 5. MySQL 사용자와 권한

MySQL 계정은 Linux 계정과 별개이다.

### 계정 구조

```sql
'사용자'@'접속호스트'
```

예시:

```sql
'app_db'@'192.168.111.148'
```

WAS01에서 접속할 수 있도록 지정한 MySQL 계정의 예시다.

### 계정과 권한 확인

```sql
-- 현재 접속 계정
SELECT USER(), CURRENT_USER();

-- 데이터베이스 목록
SHOW DATABASES;

-- 현재 계정 권한
SHOW GRANTS;

-- 계정 목록 (조회 권한 필요)
SELECT user, host FROM mysql.user;
```

### 계정 생성 및 권한 부여 예시

아래 SQL은 학습용이며 기존 프로젝트에 그대로 실행하지 않는다.

```sql
CREATE USER 'app_db'@'192.168.111.148'
IDENTIFIED BY '강력한_비밀번호';

GRANT SELECT, INSERT, UPDATE, DELETE
ON marble.*
TO 'app_db'@'192.168.111.148';
```

| 권한 | 의미 |
|---|---|
| SELECT | 조회 |
| INSERT | 삽입 |
| UPDATE | 수정 |
| DELETE | 삭제 |

원칙: 필요한 권한만 부여한다. (최소 권한 원칙)

---

## 6. WAS01 → DB01 연결 검증

### ① TCP 연결 확인 — WAS01

```bash
nc -vz -w 3 192.168.111.149 3306
```

- -v: 상세 결과
- -z: 연결 검사
- -w 3: 최대 대기 시간 3초

### ② MySQL 로그인 — WAS01

```bash
mysql -h 192.168.111.149 -P 3306 -u 실제_DB계정 -p
```

- -h: 대상 DB IP
- -P: 대상 DB 포트
- -u: MySQL 사용자
- -p: 비밀번호 입력

### ③ SQL 실행

```sql
SELECT 1;
```

### 반드시 구분

TCP 연결 성공 ≠ DB 인증 성공 ≠ SQL 실행 성공

각 단계별로 검증해야 한다.

---

## 7. MySQL 연결 장애 대응

| 증상 | 우선 확인 |
|---|---|
| Connection refused | MySQL 실행·수신 주소·포트 |
| Connection timed out | 방화벽·네트워크 |
| Access denied (1045) | 계정·비밀번호·접속 호스트 |
| Unknown database (1049) | 데이터베이스 이름 |
| SQL 실행 실패 | SQL·테이블·계정 권한 |

### 장애 분석 순서

1. DB01 MySQL 서비스 확인
2. DB01 TCP 3306 LISTEN 확인
3. bind-address 및 방화벽 확인
4. WAS01 → DB01 TCP 연결 검사
5. MySQL 로그인 및 인증 확인
6. SELECT 1 실행
7. Spring Boot의 실제 DB 사용 기능 검증

---

## 8. 최종 핵심 정리

1. MySQL은 SQL 처리와 데이터 저장·조회를 담당한다.
2. bind-address는 MySQL의 수신 IP를 결정한다.
3. 'user'@'host'는 MySQL 계정의 접속 호스트 조건을 구분한다.
4. GRANT는 계정의 DB 작업 권한을 부여한다.
5. Linux 계정과 MySQL 계정은 별개이다.
6. nc는 TCP 연결, mysql은 인증·SQL 실행을 검증한다.
7. 장애 발생 시 서비스 → 포트 → 네트워크 → 인증 → SQL 순서로 원인을 좁힌다.

### 반드시 기억할 3가지

**bind-address = 어느 IP에서 연결을 받을까?**

**'user'@'host' = 어떤 계정이 어느 호스트에서 접속할까?**

**GRANT = 접속한 계정이 어떤 작업을 할 수 있을까?**
<img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/fab000f9-d4e0-4908-a541-7b495065df91" />
