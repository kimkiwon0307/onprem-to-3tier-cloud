# MySQL 기초 복습 — 1~14교시

> **프로젝트:** Marble 인프라 구축 프로젝트  
> **학습 환경:** AWS EC2 / Ubuntu 24.04 / MySQL 8.0  
> **학습 범위:** MySQL Primary–Replica 복제, Binary Log, GTID, 백업·복원, 복제 상태 확인, 읽기 전용 설정, 보안 및 네트워크  
> **진행 상태:** 기초 1~14교시 학습 완료 / 15교시 장애 대응 종합 실습 보류

---

## 목차

1. [1교시 — Primary와 Replica](#1교시--primary와-replica)
2. [2교시 — Binary Log](#2교시--binary-log)
3. [3교시 — mysqldump와 초기 데이터 복원](#3교시--mysqldump와-초기-데이터-복원)
4. [4교시 — GTID와 초기 백업](#4교시--gtid와-초기-백업)
5. [5교시 — gtid_executed](#5교시--gtid_executed)
6. [6교시 — gtid_purged](#6교시--gtid_purged)
7. [7교시 — server_id와 server_uuid](#7교시--server_id와-server_uuid)
8. [8교시 — CHANGE REPLICATION SOURCE TO](#8교시--change-replication-source-to)
9. [9교시 — START REPLICA와 Relay Log](#9교시--start-replica와-relay-log)
10. [10교시 — SHOW REPLICA STATUS](#10교시--show-replica-status)
11. [11교시 — read_only와 super_read_only](#11교시--read_only와-super_read_only)
12. [12교시 — INSERT, UPDATE, DELETE 복제](#12교시--insert-update-delete-복제)
13. [13교시 — mysqldump 주요 옵션](#13교시--mysqldump-주요-옵션)
14. [14교시 — 보안 그룹, 복제 계정, 네트워크](#14교시--보안-그룹-복제-계정-네트워크)
15. [실제 Marble 프로젝트 적용 결과](#실제-marble-프로젝트-적용-결과)
16. [복제 장애 대응 요약](#복제-장애-대응-요약)
17. [핵심 면접 질문과 답변](#핵심-면접-질문과-답변)

---

# 1교시 — Primary와 Replica

## 1. 개념

MySQL 복제(Replication)는 한 MySQL 서버에서 발생한 데이터 변경 사항을 다른 MySQL 서버에 전달하고 적용하는 기술이다.

복제 환경에서는 일반적으로 두 역할을 구분한다.

### Primary

- 원본 데이터베이스 서버
- 애플리케이션의 INSERT, UPDATE, DELETE 처리
- 데이터 변경 사항을 Binary Log에 기록
- Replica가 변경 이벤트를 가져갈 수 있도록 제공

### Replica

- Primary의 변경 사항을 전달받는 서버
- Binary Log 이벤트를 받아 자신의 데이터에 적용
- 읽기 전용으로 운영할 수 있음
- Primary 장애 시 수동 또는 자동 전환 구성의 후보가 될 수 있음

## 2. Marble 프로젝트 구조

```text
Spring Boot WAS
      |
      | MySQL 접속
      v
DB01 — Primary
10.0.31.42
      |
      | MySQL Replication
      v
DB02 — Replica
10.0.32.160
```

현재 WAS는 DB01을 사용하며, DB02는 복제 서버로 구성했다.

**DB02가 존재한다고 자동 장애 전환이 구현되는 것은 아니다.**

Primary 장애 시 DB02를 사용하도록 전환하려면 별도의 Failover 절차가 필요하다.

## 3. 복제의 장점

- 데이터 복제본 확보
- 장애 복구 구조를 위한 기반
- 복제 상태 모니터링 가능
- 설계에 따라 읽기 트래픽 분산 가능

## 4. 주의사항

MySQL의 일반적인 비동기 복제는 Primary의 데이터 변경과 Replica 반영 사이에 시간차가 발생할 수 있다.

따라서 **복제는 백업을 완전히 대체하지 않는다.**

Primary에서 잘못된 DELETE가 발생하면 그 삭제도 Replica에 복제될 수 있기 때문이다.

### 핵심 정리

> Primary는 변경 이벤트를 제공하고, Replica는 변경 이벤트를 받아 적용한다.

---

# 2교시 — Binary Log

## 1. Binary Log란?

Binary Log는 MySQL에서 발생한 데이터 변경 트랜잭션 등을 기록하는 로그다.

주요 사용 목적은 다음과 같다.

- MySQL 복제
- 데이터 복구 작업 지원
- 변경 이력 기록

## 2. 복제에서의 역할

```text
DB01
  |
  | INSERT / UPDATE / DELETE
  v
Binary Log 기록
  |
  v
DB02로 변경 이벤트 전달
```

DB02는 DB01의 테이블을 매번 통째로 복사하는 것이 아니라, Binary Log에 기록된 변경 이벤트를 받아 적용한다.

## 3. 설정 확인

MySQL 접속:

```bash
sudo mysql
```

Binary Log 활성화 여부:

```sql
SHOW VARIABLES LIKE 'log_bin';
```

결과 예시:

```text
log_bin | ON
```

Binary Log 형식 확인:

```sql
SHOW VARIABLES LIKE 'binlog_format';
```

실제 프로젝트에서는 다음 설정을 확인했다.

```text
log_bin = ON
binlog_format = ROW
```

## 4. ROW 방식

Binary Log 기록 형식에는 ROW, STATEMENT, MIXED 방식이 있다.

우리는 `ROW` 방식을 사용했다.

ROW 방식은 일반적으로 SQL 문장 자체보다는 **변경된 행의 정보를 이벤트로 기록**한다.

예를 들어 다음 SQL을 실행하더라도:

```sql
UPDATE users
SET username = 'kim'
WHERE id = 1;
```

ROW 기반 복제에서는 변경된 행에 관한 이벤트가 기록되고 Replica에 적용된다.

### 핵심 정리

> Binary Log는 Primary에서 발생한 변경 사항을 Replica에 전달하기 위한 핵심 로그다.

---

# 3교시 — mysqldump와 초기 데이터 복원

## 1. mysqldump란?

`mysqldump`는 MySQL 데이터베이스의 구조와 데이터를 SQL 형식으로 내보내는 백업 프로그램이다.

MySQL 내부 SQL 명령어가 아니라 **Ubuntu 터미널에서 실행하는 프로그램**이다.

## 2. 기본 백업 명령어

```bash
mysqldump --databases marble > backup.sql
```

| 명령어 | 의미 |
|---|---|
| `mysqldump` | SQL 형식 백업 생성 |
| `--databases marble` | marble 데이터베이스 선택 |
| `>` | 출력을 파일에 저장 |
| `backup.sql` | 백업 파일 이름 |

백업 파일에는 테이블 생성 SQL과 데이터 입력 SQL 등이 저장될 수 있다.

## 3. 복원 명령어

```bash
mysql < backup.sql
```

우리 프로젝트에서는 관리자 권한으로 다음과 같이 복원했다.

```bash
sudo mysql < /home/ubuntu/marble-primary.sql
```

| 기호 | 의미 |
|---|---|
| `>` | 프로그램 출력을 파일로 저장 |
| `<` | 파일 내용을 프로그램의 입력으로 전달 |

## 4. 복제 구성 전에 백업을 복원하는 이유

DB01에는 기존 회원과 게임 데이터가 존재하지만, 새로 생성한 DB02는 빈 데이터베이스다.

이 상태에서 변경 이벤트만 받아서는 기존 데이터를 완전하게 확보할 수 없다.

따라서 다음 순서로 구성한다.

```text
1. DB01의 기존 데이터 백업
2. 백업 파일을 DB02로 전달
3. DB02에 기존 데이터 복원
4. GTID 복제 연결
5. 이후 변경 사항 적용
```

### 핵심 정리

> mysqldump는 기존 데이터를 복원하기 위한 초기 백업 도구이며, 지속적인 복제 기능 자체는 아니다.

---

# 4교시 — GTID와 초기 백업

## 1. GTID란?

GTID는 **Global Transaction Identifier**의 약자로, 복제에 사용되는 트랜잭션 고유 식별자다.

기본 형태는 다음과 같다.

```text
서버_UUID:트랜잭션_번호
```

예시:

```text
aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee:1-5
```

이는 해당 UUID에서 생성한 GTID 1번부터 5번까지를 의미한다.

단, 모든 SELECT에 GTID가 부여되는 것은 아니다. 트랜잭션의 기록과 복제 방식에 따라 달라진다.

## 2. GTID가 필요한 이유

기존 파일명과 로그 위치를 직접 관리하는 방식보다, 서버가 **이미 처리한 트랜잭션과 아직 처리하지 않은 트랜잭션을 구별하기 편리하다.**

예시:

```text
DB01 실행 GTID: 1~10
DB02 실행 GTID: 1~6
```

이때 DB02에는 7~10번 트랜잭션이 필요하다.

## 3. 초기 백업과의 관계

DB02에 기존 데이터를 백업으로 복원한 경우, 해당 데이터가 **어느 GTID 이력까지 반영한 결과인지** 알아야 한다.

그래야 이미 복원한 트랜잭션을 중복 적용하지 않고 필요한 변경 사항만 이어받을 수 있다.

### 핵심 정리

> GTID는 어떤 트랜잭션을 이미 처리했는지 구별하는 기준이다.

---

# 5교시 — gtid_executed

## 1. 개념

`gtid_executed`는 MySQL 서버가 실행했다고 기록한 GTID 집합이다.

확인 명령어:

```sql
SELECT @@GLOBAL.gtid_executed;
```

예시:

```text
DB01_UUID:1-9
```

이는 해당 UUID의 GTID 1번부터 9번까지를 실행 이력으로 보유하고 있다는 의미다.

## 2. Primary와 Replica 비교

```text
DB01 gtid_executed = DB01_UUID:1-10
DB02 gtid_executed = DB01_UUID:1-8
```

이 경우 GTID 집합만 비교하면 DB02에는 9~10번 이력이 아직 없다.

다만 실제 데이터가 정확하게 일치하는지 판단하려면 복제 상태와 데이터 자체도 확인해야 한다.

## 3. 주의사항

`gtid_executed`가 존재한다고 해서 애플리케이션 데이터가 반드시 올바르다는 뜻은 아니다.

또한 GTID 범위 하나가 SQL 명령어 하나를 뜻하지는 않는다.

한 트랜잭션 안에 여러 SQL 문장이 포함될 수도 있다.

### 핵심 정리

> `gtid_executed`는 해당 MySQL 서버가 실행했다고 기록한 트랜잭션 이력이다.

---

# 6교시 — gtid_purged

## 1. 개념

`gtid_purged`는 서버가 실행한 것으로 알고 있지만 **현재 자신의 Binary Log에 더 이상 기록으로 보유하지 않는 GTID 집합**을 나타낸다.

조회 명령어:

```sql
SELECT @@GLOBAL.gtid_purged;
```

## 2. gtid_executed와의 차이

| 항목 | 의미 |
|---|---|
| `gtid_executed` | 서버가 실행했다고 기록한 전체 GTID 이력 |
| `gtid_purged` | 실행 이력 중 현재 자신의 Binary Log에서 제공할 수 없는 GTID 집합 |

`gtid_purged`가 존재한다는 사실이 반드시 이 서버에서 과거 Binary Log 파일을 직접 삭제했다는 뜻은 아니다.

초기 SQL 백업을 복원하면서 GTID 이력을 등록한 경우에도 나타날 수 있다.

## 3. 실제 프로젝트

DB02 초기 복원 후 확인한 값:

```text
gtid_executed = DB01_UUID:1
gtid_purged   = DB01_UUID:1
```

이는 초기 백업에 해당 GTID의 처리 이력이 등록됐음을 보여 준다.

### 핵심 정리

> `gtid_executed`는 실행 이력이고, `gtid_purged`는 그중 현재 자신의 Binary Log로 제공할 수 없는 이력이다.

---

# 7교시 — server_id와 server_uuid

## 1. server_id

`server_id`는 복제 구성에서 MySQL 서버를 구분하는 숫자 식별자다.

우리 프로젝트:

```text
DB01 server_id = 1
DB02 server_id = 2
```

조회:

```sql
SHOW VARIABLES LIKE 'server_id';
```

복제에 참여하는 서버는 서로 다른 `server_id`를 사용하도록 구성한다.

## 2. server_uuid

`server_uuid`는 MySQL 서버를 구별하는 UUID다.

조회:

```sql
SHOW VARIABLES LIKE 'server_uuid';
```

실제 프로젝트:

```text
DB01:
ed995cb1-cba4-11f1-95ee-02ffe396cfed

DB02:
812eca81-cc23-11f1-ad72-0affd9d4799b
```

GTID에는 트랜잭션이 생성된 서버의 UUID가 포함된다.

## 3. 두 식별자의 차이

| 항목 | server_id | server_uuid |
|---|---|---|
| 형태 | 숫자 | UUID 문자열 |
| 역할 | 복제 서버 식별 | 서버 및 GTID 원본 식별 |
| 예시 | `1`, `2` | `ed995cb1-...` |

### 핵심 정리

> `server_id`와 `server_uuid`는 서로 다른 식별자이며, 복제 서버마다 고유하게 관리해야 한다.

---

# 8교시 — CHANGE REPLICATION SOURCE TO

## 1. 역할

`CHANGE REPLICATION SOURCE TO`는 Replica가 **어느 Primary에, 어떤 계정으로, 어떤 방식으로 연결할지** 설정하는 SQL이다.

우리 프로젝트에서 사용했던 구성 형태:

```sql
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = '10.0.31.42',
    SOURCE_PORT = 3306,
    SOURCE_USER = 'marble_repl',
    SOURCE_PASSWORD = '비밀번호',
    SOURCE_AUTO_POSITION = 1,
    SOURCE_SSL = 1;
```

**주의: 이미 복제가 구성된 운영 서버에서 이 명령어를 불필요하게 재실행하지 않는다. 비밀번호를 GitHub에 기록하지 않는다.**

## 2. 옵션별 의미

| 옵션 | 역할 |
|---|---|
| `SOURCE_HOST` | 연결 대상 Primary의 IP 또는 호스트 이름 |
| `SOURCE_PORT` | MySQL 접속 포트 |
| `SOURCE_USER` | 복제 계정 |
| `SOURCE_PASSWORD` | 계정 인증 비밀번호 |
| `SOURCE_AUTO_POSITION=1` | GTID 기반으로 필요한 이벤트 위치 결정 |
| `SOURCE_SSL=1` | TLS 암호화 복제 연결 사용 |

## 3. SOURCE_AUTO_POSITION

예시:

```text
DB01: GTID 1~10
DB02: GTID 1~6
```

`SOURCE_AUTO_POSITION=1`이면 DB02는 자신의 GTID 이력을 바탕으로 필요한 7~10번을 요청할 수 있다.

단, DB01에 필요한 Binary Log가 남아 있어야 한다.

## 4. SOURCE_SSL

`SOURCE_SSL=1`은 암호화 연결 사용을 요청하는 설정이다.

다만 암호화를 사용하는 것과 **서버 인증서를 검증해 올바른 DB01인지 확인하는 것**은 다르다.

실제 프로젝트에서는 다음 값도 확인했다.

```text
Source_SSL_Allowed: Yes
Source_SSL_Verify_Server_Cert: No
```

즉, TLS를 사용하도록 설정했지만 서버 인증서 검증 강화는 별도의 검토 사항으로 남는다.

### 핵심 정리

> `CHANGE REPLICATION SOURCE TO`는 Replica가 Primary에 연결하기 위한 정보를 등록하는 명령어다.

---

# 9교시 — START REPLICA와 Relay Log

## 1. START REPLICA

복제 설정 후 복제 작업을 시작할 때 사용한다.

```sql
START REPLICA;
```

이 명령어가 성공했다고 해서 **실제 데이터 복제까지 정상이라는 뜻은 아니다.**

반드시 상태를 확인해야 한다.

## 2. IO 스레드

Replica의 IO 스레드는 Primary에서 변경 이벤트를 받아온다.

```text
DB01 Binary Log
       |
       v
DB02 IO 스레드
```

## 3. Relay Log

Replica가 Primary로부터 수신한 변경 이벤트를 저장하는 로그다.

```text
DB01 Binary Log
       |
       v
DB02 IO 스레드
       |
       v
DB02 Relay Log
```

## 4. SQL 적용 스레드

Relay Log를 읽어 DB02의 실제 테이블에 데이터 변경을 반영한다.

```text
DB01 Binary Log
       |
       v
DB02 IO 스레드
       |
       v
DB02 Relay Log
       |
       v
DB02 SQL 적용 스레드
       |
       v
DB02 테이블 변경
```

## 5. 가장 중요한 구분

- **IO 스레드:** 이벤트를 받아옴
- **Relay Log:** 받아온 이벤트를 보관
- **SQL 스레드:** 실제 테이블에 적용

### 핵심 정리

> 이벤트 수신과 이벤트 적용은 서로 다른 단계다.

---

# 10교시 — SHOW REPLICA STATUS

## 1. 역할

Replica의 복제 상태를 확인하는 명령어다.

```sql
SHOW REPLICA STATUS\G
```

`\G`는 MySQL 명령줄 클라이언트에서 결과를 세로 형태로 출력하도록 하는 구분자다.

## 2. 주요 확인 항목

| 항목 | 의미 |
|---|---|
| `Source_Host` | 연결 대상 Primary |
| `Source_User` | 복제 계정 |
| `Replica_IO_Running` | IO 스레드 실행 상태 |
| `Replica_SQL_Running` | SQL 적용 스레드 실행 상태 |
| `Seconds_Behind_Source` | 보고된 복제 지연 |
| `Last_IO_Error` | 마지막 IO 관련 오류 |
| `Last_SQL_Error` | 마지막 SQL 적용 오류 |
| `Retrieved_Gtid_Set` | Primary로부터 수신한 GTID 집합 |
| `Executed_Gtid_Set` | Replica가 실행한 GTID 집합 |
| `Auto_Position` | GTID 자동 위치 설정 여부 |

## 3. 정상 상태 예시

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Source: 0

Last_IO_Error:
Last_SQL_Error:
```

의미:

- IO 스레드 실행 중
- SQL 스레드 실행 중
- 당시 보고된 복제 지연 0초
- 출력 당시 보고된 오류 메시지 없음

단, 이것만으로 데이터의 완전한 일치를 보장할 수는 없다.

## 4. 복제 지연 예시

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Source: 30
```

해석:

**두 스레드는 실행 중이지만 약 30초의 복제 지연이 관측되고 있다.**

스레드 실행 여부와 데이터가 최신 상태인지는 별개다.

## 5. Last_IO_Error 예시

```text
Replica_IO_Running: Connecting
Replica_SQL_Running: Yes

Last_IO_Error: Access denied for user 'marble_repl'
```

우선 점검:

1. 복제 계정
2. 비밀번호
3. 계정에 허용된 접속 호스트
4. 네트워크 연결 및 권한

## 6. Last_SQL_Error 예시

```text
Replica_IO_Running: Yes
Replica_SQL_Running: No

Last_SQL_Error: Duplicate entry '5' for key 'PRIMARY'
```

해석:

- 이벤트 수신은 가능
- 데이터 적용 중 Primary Key 중복 오류 발생
- SQL 적용 작업이 중단됨

우선 DB01과 DB02의 데이터 차이를 조사한다.

원인 분석 없이 GTID를 건너뛰거나 데이터를 삭제하면 안 된다.

### 핵심 정리

> IO 오류는 수신 단계, SQL 오류는 적용 단계부터 확인한다.

---

# 11교시 — read_only와 super_read_only

## 1. Replica를 읽기 전용으로 설정하는 이유

DB02에서 사용자가 직접 데이터를 수정하면 DB01과 DB02의 데이터가 달라질 수 있다.

이를 방지하기 위해 Replica에 읽기 전용 설정을 적용한다.

## 2. read_only

```sql
SET GLOBAL read_only = ON;
```

일반 사용자 계정 등의 직접적인 데이터 변경을 제한한다.

단, 특정 관리자 권한을 가진 계정은 제한을 우회할 수 있다.

## 3. super_read_only

```sql
SET GLOBAL super_read_only = ON;
```

`read_only`보다 더 강한 직접 쓰기 제한을 제공한다.

`super_read_only=ON`으로 설정하면 `read_only`도 ON이 된다.

## 4. 두 설정 비교

| 항목 | read_only | super_read_only |
|---|---|---|
| 일반 계정 쓰기 제한 | O | O |
| 관리자 계정의 일반적인 직접 쓰기 제한 | 일부 예외 | 더욱 강하게 제한 |
| 복제 SQL 스레드의 이벤트 적용 | 가능 | 가능 |

**읽기 전용으로 설정해도 복제 SQL 스레드는 정상적인 변경 이벤트를 적용할 수 있다.**

## 5. GLOBAL과 PERSIST

### GLOBAL

```sql
SET GLOBAL super_read_only = ON;
```

현재 실행 중인 MySQL에 설정을 적용한다.

### PERSIST

```sql
SET PERSIST super_read_only = ON;
```

현재 적용하면서 재시작 후에도 유지되도록 설정을 저장한다.

우리 프로젝트에서는 다음 설정을 적용하도록 구성했다.

```sql
SET PERSIST super_read_only = ON;
```

## 6. 읽기 전용 상태 확인

```sql
SELECT
    @@GLOBAL.read_only,
    @@GLOBAL.super_read_only;
```

결과가 다음과 같다면:

```text
read_only       = 1
super_read_only = 1
```

두 설정이 활성화된 상태다.

### 핵심 정리

> Replica를 읽기 전용으로 설정하는 이유는 직접적인 데이터 변경으로 인한 불일치를 방지하기 위해서다.

---

# 12교시 — INSERT, UPDATE, DELETE 복제

## 1. 기본 SQL

| SQL | 역할 |
|---|---|
| `INSERT` | 데이터 추가 |
| `UPDATE` | 데이터 수정 |
| `DELETE` | 데이터 삭제 |

## 2. 복제 흐름

```text
DB01에서 데이터 변경
         |
         v
DB01 Binary Log 기록
         |
         v
DB02 IO 스레드 수신
         |
         v
DB02 Relay Log 저장
         |
         v
DB02 SQL 스레드 적용
         |
         v
DB02 데이터 변경 완료
```

INSERT, UPDATE, DELETE 모두 이 흐름을 따른다.

## 3. 실습 예시

다음은 개념 복습용 SQL이다.

**운영 중인 테이블에서 그대로 재실행하지 않는다.**

### INSERT

DB01:

```sql
INSERT INTO marble.replica_test (id, message)
VALUES (1, 'INSERT replication test');
```

DB02 확인:

```sql
SELECT * FROM marble.replica_test;
```

### UPDATE

DB01:

```sql
UPDATE marble.replica_test
SET message = 'UPDATE replication success'
WHERE id = 1;
```

DB02 확인:

```sql
SELECT * FROM marble.replica_test;
```

### DELETE

DB01:

```sql
DELETE FROM marble.replica_test
WHERE id = 1;
```

DB02 확인:

```sql
SELECT * FROM marble.replica_test;
```

복제 설정 후에는 실제 데이터까지 비교해야 한다.

## 4. 장애 예시

```text
Replica_IO_Running: Yes
Replica_SQL_Running: No
```

이 경우 변경 이벤트는 받아왔더라도 SQL 적용 스레드가 중단되어 DB02의 테이블에는 아직 반영되지 않았을 수 있다.

### 핵심 정리

> 복제 스레드 상태를 확인하는 것과 실제 INSERT/UPDATE/DELETE 결과가 반영됐는지 확인하는 것은 별개다.

---

# 13교시 — mysqldump 주요 옵션

## 1. 실제 사용한 백업 명령어

우리 프로젝트에서 사용했던 옵션 구성:

```bash
mysqldump \
  --single-transaction \
  --quick \
  --routines \
  --triggers \
  --events \
  --no-tablespaces \
  --set-gtid-purged=ON \
  --databases marble \
  > marble-primary.sql
```

아래에서 옵션을 하나씩 설명한다.

## 2. --single-transaction

```bash
--single-transaction
```

### 역할

InnoDB 테이블을 백업할 때 **일관된 시점의 데이터를 읽기 위한 옵션**이다.

예를 들어 송금 과정에서:

```text
송금 전:
철수 1,000P / 영희 2,000P

송금 후:
철수 500P / 영희 2,500P
```

서로 다른 시점의 데이터를 섞어 백업하면 일관성이 깨질 수 있다.

`--single-transaction`은 InnoDB의 일관된 읽기 기능을 활용해 이를 방지한다.

### 장점

- 일반적인 데이터 변경을 장시간 차단하지 않고 백업 가능
- 일관된 시점의 데이터 확보
- 운영 중 백업에 유용

### 주의사항

- 백업 중 `ALTER TABLE`, `DROP TABLE` 등 DDL 수행 시 오류나 일관성 문제 가능
- MyISAM 등 비트랜잭션 테이블까지 같은 방식으로 보호하는 것은 아님

**암기:** `--single-transaction` = 백업 데이터의 일관성

## 3. --quick

```bash
--quick
```

### 역할

조회 결과 전체를 클라이언트 메모리에 한꺼번에 쌓지 않고 **행을 순차적으로 가져오는 옵션**이다.

### 사용하는 이유

- 대량 데이터 백업 시 메모리 부담 감소
- 결과를 순차적으로 읽어 파일에 기록

### 오해하기 쉬운 점

- 무조건 백업 시간이 빨라지는 옵션은 아님
- 자동 압축 옵션이 아님
- `--single-transaction`과 목적이 다름

**암기:** `--quick` = 백업 프로그램의 메모리 효율

참고로 MySQL 8.0 `mysqldump`의 기본 옵션 그룹에도 포함되지만, 의미를 명확히 하기 위해 명시적으로 사용했다.

## 4. --routines

```bash
--routines
```

### Routine이란?

MySQL 내부에 저장하는 SQL 기반 기능이다.

대표적으로:

- Stored Procedure(저장 프로시저)
- Stored Function(저장 함수)

예시:

```sql
CALL give_points();
```

이는 미리 만들어 둔 포인트 지급 프로시저를 호출한다는 의미다.

### 옵션 역할

**저장 프로시저와 함수의 정의를 백업에 포함한다.**

## 5. --triggers

```bash
--triggers
```

### Trigger란?

테이블에서 INSERT, UPDATE, DELETE 등이 발생할 때 자동 실행되는 데이터베이스 기능이다.

예시:

```text
회원 정보 UPDATE
       |
       v
Trigger 실행
       |
       v
변경 이력 저장
```

### 옵션 역할

**트리거 정의를 백업에 포함한다.**

MySQL 8.0의 `mysqldump`에서는 일반적으로 기본 활성화되어 있다.

## 6. --events

```bash
--events
```

### Event란?

정해진 시간이나 주기에 자동 실행되는 데이터베이스 작업이다.

예시:

```text
매일 03:00
    |
    v
Event Scheduler
    |
    v
오래된 임시 데이터 정리
```

### 옵션 역할

**예약 이벤트의 정의를 백업에 포함한다.**

이벤트의 실제 실행 여부는 MySQL의 `event_scheduler` 설정 등에도 영향을 받는다.

## 7. Routine, Trigger, Event 비교

| 개념 | 실행 시점 | 예시 |
|---|---|---|
| Procedure | 호출했을 때 | 회원 포인트 지급 |
| Trigger | 데이터 변경 시 | 회원 정보 변경 이력 |
| Event | 정해진 시간 | 새벽 임시 데이터 정리 |

이 기능들은 Spring Boot Java 코드가 아니라 MySQL 내부에 정의되는 객체다.

## 8. --no-tablespaces

```bash
--no-tablespaces
```

### Tablespace란?

MySQL InnoDB가 데이터를 저장하고 관리하는 논리적인 저장 공간이다.

### 옵션 역할

별도의 테이블스페이스 및 로그 파일 그룹 생성 정의를 덤프에서 제외한다.

**테이블의 구조와 데이터를 제외하는 옵션이 아니다.**

주요 사용 이유:

- 필요하지 않은 별도 저장 공간 정의 제외
- MySQL 8.0에서 발생할 수 있는 관련 `PROCESS` 권한 요구 방지

## 9. --set-gtid-purged=ON

```bash
--set-gtid-purged=ON
```

### 역할

백업 파일에 원본 서버의 GTID 이력을 대상 서버에 등록하기 위한 SQL 구문을 포함한다.

개념 예시:

```sql
SET @@GLOBAL.GTID_PURGED = '+DB01_UUID:1-6';
```

초기 백업에 이미 반영된 GTID가 1~6이라면, DB02가 이를 다시 복제하지 않도록 실행 이력을 등록하는 것이다.

그다음 DB01의 현재 GTID가 1~10이라면:

```text
초기 복원된 GTID: 1~6
추가 복제할 GTID: 7~10
```

### 주의사항

- 자동으로 복제를 시작하는 옵션은 아님
- GTID 이력과 복원된 데이터의 시점이 일치해야 함
- 데이터베이스 일부만 백업해도 서버 전체 GTID 이력이 포함될 수 있으므로 백업 범위에 주의해야 함
- 이미 GTID 이력이 존재하는 서버에 복원할 때는 충돌 및 중복 처리 여부 검토 필요

## 10. 옵션 전체 요약

| 옵션 | 사용하는 이유 |
|---|---|
| `--single-transaction` | 백업 시점의 데이터 일관성 |
| `--quick` | 메모리 부담 감소 |
| `--routines` | 프로시저·함수 정의 포함 |
| `--triggers` | 트리거 정의 포함 |
| `--events` | 이벤트 정의 포함 |
| `--no-tablespaces` | 별도 테이블스페이스 정의 제외 |
| `--set-gtid-purged=ON` | GTID 실행 이력 등록 |
| `--databases marble` | 백업 대상 DB 지정 |

### 핵심 정리

> 백업 데이터의 일관성, 메모리 효율, 데이터베이스 객체 포함 여부, GTID 이력 관리를 각각 다른 옵션으로 제어한다.

---

# 14교시 — 보안 그룹, 복제 계정, 네트워크

## 1. 복제 환경

| 서버 | 역할 | Private IP |
|---|---|---|
| DB01 | Primary | `10.0.31.42` |
| DB02 | Replica | `10.0.32.160` |

MySQL 기본 포트:

```text
TCP 3306
```

## 2. 연결 방향과 데이터 방향

```text
연결을 시작하는 방향:

DB02 → DB01:3306


복제 이벤트 전달 방향:

DB01 Binary Log → DB02
```

두 방향을 구분하는 것이 중요하다.

**DB02의 IO 스레드가 DB01로 연결을 시작하고, 연결을 통해 DB01의 변경 이벤트를 받아온다.**

## 3. AWS 보안 그룹

AWS 보안 그룹은 EC2 인스턴스의 네트워크 트래픽 허용 여부를 결정하는 가상 방화벽이다.

우리가 복제용으로 설정한 DB01의 인바운드 규칙:

| 항목 | 값 |
|---|---|
| 프로토콜 | TCP |
| 포트 | 3306 |
| 출발지 | `10.0.32.160/32` |

`/32`는 IPv4 주소 하나를 의미한다.

따라서 DB02의 IP만 허용하는 설정이다.

```text
10.0.32.160/32 → DB02 한 대
0.0.0.0/0      → 모든 IPv4 주소
```

보안을 위해 불필요한 전체 개방을 피한다.

## 4. 인바운드와 아웃바운드

| 구분 | 뜻 | 우리 프로젝트 |
|---|---|---|
| 인바운드 | 들어오는 연결 | DB01이 DB02의 접속 허용 |
| 아웃바운드 | 나가는 연결 | DB02가 DB01로 접속 |

AWS 보안 그룹은 Stateful 방식이므로 허용된 연결에 대한 응답 트래픽은 자동으로 허용된다.

단, 네트워크 ACL과 OS 방화벽 등은 별도로 확인할 수 있다.

## 5. 복제 전용 MySQL 계정

우리는 DB01에 다음 계정을 만들었다.

```text
'marble_repl'@'10.0.32.160'
```

계정 생성 SQL 형태:

```sql
CREATE USER 'marble_repl'@'10.0.32.160'
IDENTIFIED BY '강력한_비밀번호';
```

복제 권한:

```sql
GRANT REPLICATION SLAVE ON *.*
TO 'marble_repl'@'10.0.32.160';
```

### 왜 root를 사용하지 않았는가?

**최소 권한 원칙** 때문이다.

DB02의 복제 연결에는 관리자 권한이 아니라 복제에 필요한 권한만 부여하는 것이 적절하다.

## 6. 연결 과정

```text
① DB02 IO 스레드가 DB01:3306 연결 요청
               |
               v
② AWS 보안 그룹에서 접속 허용 여부 판단
               |
               v
③ DB01 MySQL에서 사용자 인증
               |
               v
④ 복제 계정 권한 확인
               |
               v
⑤ GTID를 기준으로 필요한 이벤트 요청
               |
               v
⑥ Binary Log 이벤트 수신
               |
               v
⑦ DB02 Relay Log에 저장
               |
               v
⑧ SQL 적용 스레드가 DB02 테이블에 반영
```

## 7. 네트워크 진단 명령어

DB02의 Ubuntu 터미널에서:

```bash
nc -zv -w 3 10.0.31.42 3306
```

옵션 설명:

| 부분 | 의미 |
|---|---|
| `nc` | TCP 연결 테스트 |
| `-z` | 포트 연결만 확인 |
| `-v` | 자세한 결과 표시 |
| `-w 3` | 최대 3초 대기 |
| `10.0.31.42` | DB01 주소 |
| `3306` | MySQL 포트 |

연결 성공 예시:

```text
Connection to 10.0.31.42 3306 port [tcp/mysql] succeeded!
```

이 결과는 TCP 연결이 가능하다는 의미다.

**MySQL 로그인 성공이나 복제 권한 정상 여부까지 증명하지는 않는다.**

## 8. 인증 장애 예시

```text
Replica_IO_Running: No

Last_IO_Error:
Access denied for user 'marble_repl'
```

확인 순서:

1. `marble_repl` 사용자 이름 확인
2. 비밀번호 확인
3. 접속 허용 호스트 확인
4. MySQL 사용자 인증 설정 확인
5. 인증 후 복제 권한 확인

네트워크 포트가 열려 있는데 인증 오류가 발생했다면 3306을 전체 개방하는 것이 해결책은 아니다.

### 핵심 정리

> 보안 그룹은 네트워크 접속을 통제하고, MySQL 계정은 사용자 인증과 권한을 통제한다.

---

# 실제 Marble 프로젝트 적용 결과

## 1. 구축 환경

| 항목 | DB01 | DB02 |
|---|---|---|
| 역할 | Primary | Replica |
| OS | Ubuntu 24.04 | Ubuntu 24.04 |
| MySQL | 8.0.46 | 8.0.46 |
| Private IP | `10.0.31.42` | `10.0.32.160` |
| Server ID | 1 | 2 |
| Binary Log | ON | ON |
| GTID Mode | ON | ON |

## 2. 사용한 MySQL 데이터베이스

```text
marble
```

초기 복원 후 확인한 테이블:

```text
SPRING_SESSION
SPRING_SESSION_ATTRIBUTES
game_event
game_player
game_property
game_room
users
```

총 7개 테이블이다.

## 3. 실제 초기 복원

DB01에서 SQL 백업을 생성했다.

파일 이름:

```text
marble-primary.sql
```

백업 파일을 DB02로 전달한 뒤 복원했다.

```bash
sudo mysql < /home/ubuntu/marble-primary.sql
```

백업 파일을 전달할 때 SHA256 해시를 비교하여 파일의 동일성도 확인했다.

## 4. 초기 GTID 결과

DB02에 백업을 복원한 후:

```text
gtid_executed:
ed995cb1-cba4-11f1-95ee-02ffe396cfed:1

gtid_purged:
ed995cb1-cba4-11f1-95ee-02ffe396cfed:1
```

## 5. 복제 연결 결과

DB02에서 `SOURCE_AUTO_POSITION=1`을 사용한 GTID 기반 복제를 구성했다.

실제 확인 결과:

```text
Source_Host: 10.0.31.42
Source_User: marble_repl
Source_Port: 3306

Replica_IO_Running: Yes
Replica_SQL_Running: Yes

Seconds_Behind_Source: 0

Last_IO_Error:
Last_SQL_Error:

Retrieved_Gtid_Set:
ed995cb1-cba4-11f1-95ee-02ffe396cfed:2-9

Executed_Gtid_Set:
ed995cb1-cba4-11f1-95ee-02ffe396cfed:1-9

Auto_Position: 1
Source_SSL_Allowed: Yes
Source_SSL_Verify_Server_Cert: No
```

## 6. 결과 해석

- DB02가 DB01을 Primary로 연결하도록 설정됨
- 복제 전용 계정 `marble_repl` 사용
- IO 스레드와 SQL 적용 스레드 실행 확인
- 확인 당시 보고된 복제 지연 0초
- 확인 당시 보고된 IO/SQL 오류 없음
- 초기 GTID 1번과 이후 복제된 GTID 2~9번의 실행 이력 확인

따라서 **DB01 → DB02 GTID 기반 MySQL 복제가 구성되고 정상 작동하는 상태를 확인했다.**

## 7. 검증 범위 및 남은 과제

확인된 사항:

- DB01과 DB02의 MySQL 설치 및 복제 설정
- DB01 초기 백업과 DB02 복원
- 복제 계정 및 보안 그룹 설정
- GTID 기반 복제 연결
- IO/SQL 스레드 정상 실행
- 초기 및 이후 GTID 이력 확인

추가로 최종 점검할 사항:

- DB02의 `read_only`, `super_read_only` 현재값 재확인
- INSERT/UPDATE/DELETE 테스트별 결과 문서화
- TLS 서버 인증서 검증 강화 검토
- 지속적인 복제 지연 모니터링
- Primary 장애 시 수동 Failover 실습

**현재 구현 상태는 비동기 MySQL 복제이며, 자동 Failover 구성이나 무손실 장애 복구를 검증한 상태는 아니다.**

---

# 복제 장애 대응 요약

## 1. 기본 점검 순서

### ① MySQL 서비스 상태

Ubuntu 터미널:

```bash
sudo systemctl status mysql
```

### ② DB01 포트 연결 확인

DB02 Ubuntu 터미널:

```bash
nc -zv -w 3 10.0.31.42 3306
```

### ③ 복제 상태 확인

DB02 MySQL:

```sql
SHOW REPLICA STATUS\G
```

### ④ 주요 상태값 분석

```text
Replica_IO_Running
Replica_SQL_Running
Seconds_Behind_Source
Last_IO_Error
Last_SQL_Error
Retrieved_Gtid_Set
Executed_Gtid_Set
```

### ⑤ GTID 이력 확인

```sql
SELECT @@GLOBAL.gtid_executed;
SELECT @@GLOBAL.gtid_purged;
```

### ⑥ 실제 데이터 반영 확인

테스트용 테이블이나 정상적인 검증 쿼리로 DB01과 DB02의 데이터 상태를 비교한다.

## 2. 장애 유형별 판단

| 현상 | 가능한 원인 | 우선 점검 |
|---|---|---|
| DB01 3306 연결 실패 | 네트워크, 보안 그룹, MySQL 서비스 | `nc`, SG, 서비스 상태 |
| `Access denied` | 계정/비밀번호/접속 호스트 | MySQL 계정 설정 |
| IO 스레드 중단 | 연결, 인증, Binary Log 문제 등 | `Last_IO_Error` |
| SQL 스레드 중단 | 데이터 충돌, 적용 실패 등 | `Last_SQL_Error` |
| IO/SQL 모두 Yes, 지연 발생 | 처리량 증가, 자원 부족 등 | 복제 지연과 자원 사용량 |
| GTID 이벤트를 가져올 수 없음 | 필요한 Binary Log 부재 등 | GTID 및 로그 보관 상태 |
| 두 DB의 데이터 불일치 | 직접 쓰기, 과거 복제 오류 등 | 실제 데이터 비교, 변경 이력 |

## 3. 피해야 할 대응

원인이 확인되지 않은 상태에서 다음 작업을 무작정 수행하지 않는다.

- DB02 데이터 삭제
- GTID 초기화
- 문제 트랜잭션 강제 건너뛰기
- `RESET MASTER` 계열 명령어 실행
- 3306 포트를 인터넷 전체에 개방
- 원본 DB01의 데이터 수정
- 정상 복제 설정 덮어쓰기

**장애 대응 원칙: 상태 확인 → 원인 분석 → 조치 계획 → 조치 → 정상 여부 재검증**

---

# 핵심 면접 질문과 답변

## Q1. MySQL Primary와 Replica의 차이는 무엇인가요?

**답변:** Primary는 애플리케이션의 데이터 변경을 처리하는 원본 서버이며, Replica는 Primary의 변경 이벤트를 받아 자신의 데이터베이스에 적용하는 서버입니다.

## Q2. MySQL Binary Log는 왜 필요한가요?

**답변:** Primary에서 발생한 데이터 변경 트랜잭션을 기록하며, Replica가 이를 받아 적용하는 데 사용합니다. 복제 외에도 데이터 복구 작업에 활용할 수 있습니다.

## Q3. GTID는 무엇인가요?

**답변:** 트랜잭션을 구별하는 고유 식별자로, Replica가 자신이 이미 처리한 트랜잭션과 아직 처리하지 않은 트랜잭션을 구분하는 데 사용합니다.

## Q4. gtid_executed와 gtid_purged의 차이는 무엇인가요?

**답변:** `gtid_executed`는 실행했다고 기록된 GTID 집합이고, `gtid_purged`는 그중 현재 자신의 Binary Log에 보관되지 않는 GTID 집합입니다.

## Q5. server_id와 server_uuid는 왜 구분하나요?

**답변:** `server_id`는 복제 구성에서 사용하는 숫자 식별자이고, `server_uuid`는 서버의 UUID이며 GTID에서 트랜잭션이 생성된 원본을 구분하는 데 사용됩니다.

## Q6. Replica의 IO 스레드와 SQL 스레드는 어떤 차이가 있나요?

**답변:** IO 스레드는 Primary에서 변경 이벤트를 받아 Relay Log에 저장하고, SQL 적용 스레드는 Relay Log에 저장된 이벤트를 실제 Replica 테이블에 적용합니다.

## Q7. Seconds_Behind_Source가 0이면 데이터가 완전히 동일한가요?

**답변:** 아닙니다. 당시 보고된 복제 지연이 0이라는 의미이며, 데이터의 완전한 일치를 보장하지는 않습니다. 스레드 상태와 실제 데이터 검증도 필요합니다.

## Q8. Replica에 super_read_only를 적용하는 이유는 무엇인가요?

**답변:** 관리자 권한을 통한 일반적인 직접 데이터 변경까지 제한하여 Primary와 Replica의 불일치 위험을 줄이기 위해서입니다. 정상적인 복제 이벤트 적용은 계속 가능합니다.

## Q9. mysqldump의 single-transaction과 quick은 무엇이 다른가요?

**답변:** `--single-transaction`은 InnoDB 데이터를 일관된 시점에서 백업하기 위한 옵션이고, `--quick`은 결과 행을 순차적으로 가져와 백업 프로그램의 메모리 부담을 줄이는 옵션입니다.

## Q10. 초기 백업에 GTID 이력을 넣는 이유는 무엇인가요?

**답변:** Replica에 기존 데이터를 복원한 후 어느 트랜잭션까지 이미 반영됐는지 알려주어, GTID 기반 복제 시 필요한 변경 이벤트만 이어받도록 하기 위해서입니다.

## Q11. AWS 보안 그룹과 MySQL 계정 인증은 무엇이 다른가요?

**답변:** 보안 그룹은 네트워크 연결 허용 여부를 판단하고, MySQL 계정은 접속 사용자의 신원과 권한을 확인합니다. 포트 연결에 성공했다고 인증까지 성공한 것은 아닙니다.

## Q12. 복제 계정에 root 대신 별도 계정을 사용하는 이유는 무엇인가요?

**답변:** 최소 권한 원칙을 적용하기 위해서입니다. 복제에 필요한 권한만 가진 전용 계정을 사용하면 불필요한 관리자 권한 노출을 줄일 수 있습니다.

## Q13. Replica_IO_Running은 Yes인데 Replica_SQL_Running이 No라면?

**답변:** 이벤트 수신은 가능하지만 적용 단계에서 문제가 발생한 상태일 수 있습니다. `Last_SQL_Error`를 확인하고 데이터 충돌이나 테이블 구조 차이 등의 원인을 조사하겠습니다.

## Q14. MySQL 복제만 구성하면 자동 Failover가 되나요?

**답변:** 아닙니다. 데이터 복제와 장애 전환은 별도 기능입니다. Primary 장애를 감지하고 Replica를 승격하고 애플리케이션 연결 대상을 변경하는 절차가 추가로 필요합니다.

---

# 최종 요약

## MySQL 복제 전체 흐름

```text
[Spring Boot 애플리케이션]
             |
             v
      [DB01 Primary]
             |
             | INSERT / UPDATE / DELETE
             v
        [Binary Log]
             |
             | GTID 포함 변경 이벤트
             |
             v
     [DB02 IO 스레드]
             |
             | 이벤트 수신
             v
        [Relay Log]
             |
             | 이벤트 전달
             v
   [DB02 SQL 적용 스레드]
             |
             | 변경 내용 적용
             v
       [DB02 Replica]
```

## 가장 중요한 7가지

1. **Primary와 Replica는 역할이 다르다.**
2. **Binary Log는 데이터 변경 이벤트의 기록이다.**
3. **GTID는 트랜잭션 처리 이력을 구분한다.**
4. **mysqldump는 기존 데이터 복원을 위한 초기 백업에 사용한다.**
5. **IO 스레드는 이벤트 수신, SQL 스레드는 이벤트 적용을 담당한다.**
6. **Replica의 읽기 전용 설정은 직접 쓰기를 제한하지만 정상적인 복제를 막지는 않는다.**
7. **장애가 발생하면 네트워크 → 인증 → 복제 수신 → 이벤트 적용 순서로 원인 범위를 좁혀야 한다.**

---

## 학습 진행 현황

- [x] 1교시 — Primary / Replica
- [x] 2교시 — Binary Log
- [x] 3교시 — mysqldump와 초기 복원
- [x] 4교시 — GTID와 초기 백업
- [x] 5교시 — gtid_executed
- [x] 6교시 — gtid_purged
- [x] 7교시 — server_id / server_uuid
- [x] 8교시 — CHANGE REPLICATION SOURCE TO
- [x] 9교시 — START REPLICA / Relay Log
- [x] 10교시 — SHOW REPLICA STATUS / 오류 분석
- [x] 11교시 — read_only / super_read_only
- [x] 12교시 — INSERT / UPDATE / DELETE 복제
- [x] 13교시 — mysqldump 주요 옵션
- [x] 14교시 — 보안 그룹 / 복제 계정 / 네트워크
- [ ] 15교시 — MySQL 복제 장애 대응 종합 실습 (보류)
- [ ] 프로젝트 12교시 — MySQL 수동 Failover (보류)
- [ ] 프로젝트 13교시 — Zabbix 모니터링 구축 (다음 진행)

---

**학습 결론**

MySQL 복제는 단순히 Primary와 Replica를 연결하는 작업이 아니라, Binary Log, GTID, 네트워크, 계정 권한, Relay Log, 복제 스레드가 서로 연결되어 동작하는 시스템이다.

이번 학습을 통해 각 구성요소의 역할과 복제 흐름, 기본적인 상태 확인 및 장애 원인 분석 방법을 이해했다.

다음에는 이 구조를 기반으로 Zabbix를 활용하여 서버 상태와 MySQL 복제 상태를 모니터링하는 방법을 학습한다.
