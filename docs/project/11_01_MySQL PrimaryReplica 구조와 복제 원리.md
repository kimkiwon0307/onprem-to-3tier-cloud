# 11장 1교시 — MySQL Primary/Replica 구조와 복제 원리

## 1. Primary/Replica란?

MySQL 데이터베이스를 두 대로 구성하고, Primary의 데이터 변경 사항을 Replica에 복제하는 구조이다.

- **Primary:** 원본 데이터의 읽기·쓰기 처리
- **Replica:** Primary에서 발생한 데이터 변경 사항을 복제
- **Replication:** Primary의 변경 데이터를 Replica에 전달하고 반영하는 과정

### Marble 프로젝트 구성

```text
WAS01 / WAS02
       |
       v
DB01 (Primary)
10.0.31.42:3306
       |
       | GTID 기반 비동기 복제
       v
DB02 (Replica)
10.0.32.160:3306
```

**구축 목적**
- 원본 데이터의 복제본 확보
- Primary 장애 시 복구 및 전환에 활용
- 필요 시 읽기 부하 분산에 활용
- 백업 및 운영 작업 분리 가능

※ Replica를 구성했다고 해서 읽기 부하 분산이나 자동 장애 전환이 저절로 이루어지는 것은 아니다.

## 2. MySQL 데이터 복제 동작 원리

가장 중요한 흐름은 다음과 같다.

```text
[DB01 - Primary]

INSERT / UPDATE / DELETE
          |
          v
     Binary Log
          |
          | 복제 이벤트 전송
          v
[DB02 - Replica]

     I/O Receiver
          |
          v
       Relay Log
          |
          v
      SQL Applier
          |
          v
     데이터 반영 완료
```

### 주요 구성 요소

| 구성 요소 | 역할 |
|---|---|
| Binary Log | Primary에서 발생한 데이터 변경 이벤트 기록 |
| I/O Receiver | Primary의 변경 이벤트를 Replica로 가져옴 |
| Relay Log | Replica가 수신한 변경 이벤트 저장 |
| SQL Applier | Relay Log 이벤트를 실제 데이터에 반영 |

**핵심:** Primary의 데이터를 매번 통째로 복사하는 것이 아니라, 초기 데이터 정합성을 맞춘 후 변경 이벤트를 지속적으로 복제한다.

## 3. GTID란?

GTID(Global Transaction Identifier)는 트랜잭션마다 부여되는 고유 식별자이다.

### GTID의 역할

- 각 트랜잭션을 고유하게 식별
- 이미 실행한 트랜잭션을 추적
- 복제 진행 위치 관리
- 복제 중단 후 재연결 및 복구 작업에 활용

예시:

```text
Primary: 1~105번 트랜잭션 적용
Replica: 1~102번 트랜잭션 적용

Replica 미적용: 103~105번
```

위 번호는 이해를 위한 단순화된 예시이다.

**핵심:** GTID는 데이터를 직접 전송하는 기술이 아니라, 어떤 트랜잭션이 실행되었는지 식별하고 복제 위치를 관리하는 기술이다.

## 4. 비동기 복제(Asynchronous Replication)

Marble 프로젝트에서 사용한 복제 방식이다.

### 동작 원리

1. Primary에서 데이터 변경 및 Commit
2. Primary가 Replica의 반영 완료를 기다리지 않고 성공 응답 가능
3. Replica가 Binary Log 이벤트를 수신
4. Replica가 Relay Log 이벤트를 적용

### 장단점

| 장점 | 단점 |
|---|---|
| Replica의 반영 완료를 기다리지 않아 쓰기 지연 감소 | 복제 지연 발생 가능 |
| 데이터 복제본을 유지할 수 있음 | Primary 장애 시 미복제 데이터 손실 위험 |
| 운영·읽기 작업 분리 가능 | Primary/Replica 데이터가 일시적으로 다를 수 있음 |

**핵심:** Primary에서 데이터 저장이 성공했더라도 Replica에 즉시 반영되었다는 의미는 아니다.

## 5. 복제 상태 확인 명령어

### Binary Log 활성화 확인

```sql
SHOW VARIABLES LIKE 'log_bin';
```

`ON`이면 Binary Log 활성화 상태이다.

### GTID 활성화 확인

```sql
SHOW VARIABLES LIKE 'gtid_mode';
```

`ON`이면 GTID 모드가 활성화된 상태이다.

### Replica 복제 상태 확인

DB02에서 실행한다.

```sql
SHOW REPLICA STATUS\G
```

### 주요 확인 항목

| 항목 | 의미 | 정상적인 상태의 예 |
|---|---|---|
| `Replica_IO_Running` | 변경 이벤트 수신 스레드 | Yes |
| `Replica_SQL_Running` | 변경 이벤트 적용 스레드 | Yes |
| `Seconds_Behind_Source` | 복제 지연 추정값 | 일반적으로 0 또는 낮은 값 |
| `Last_IO_Error` | 이벤트 수신 관련 오류 | 없음 |
| `Last_SQL_Error` | 이벤트 적용 관련 오류 | 없음 |
| `Auto_Position` | GTID 자동 위치 추적 사용 여부 | 사용 시 1 |

※ IO/SQL 스레드가 모두 `Yes`라고 해서 데이터가 완벽하게 일치한다고 단정할 수 없다.

※ `Seconds_Behind_Source=0`도 모든 데이터의 완전한 일치를 보장하지는 않는다.

## 6. Primary 장애 발생 시 주의사항

**Primary/Replica 복제와 자동 Failover는 서로 다른 기술이다.**

- Replication: 데이터를 다른 DB로 복제
- Failover: 장애 발생 시 다른 DB로 서비스 전환

Marble 프로젝트는 Primary/Replica 복제를 구성했지만 자동 Failover까지 구성한 것은 아니다.

Primary 장애 시에는 다음 사항을 확인해야 한다.

1. Replica에 마지막 트랜잭션까지 반영되었는지
2. 복제 지연 또는 미적용 데이터가 존재하는지
3. Replica를 새로운 Primary로 전환할 수 있는지
4. WAS의 DB 연결 대상을 어떻게 변경할지

비동기 복제에서는 Primary 장애 시 아직 복제되지 않은 데이터가 유실될 수 있다.

---

## 7. 최종 핵심 요약

1. **Primary:** 원본 데이터의 변경 처리
2. **Replica:** Primary의 변경 데이터를 복제
3. **Binary Log:** Primary의 데이터 변경 이벤트 기록
4. **Relay Log:** Replica가 전달받은 변경 이벤트 기록
5. **SQL Applier:** Relay Log의 이벤트를 실제 데이터에 반영
6. **GTID:** 트랜잭션 식별과 복제 위치 관리
7. **비동기 복제:** Replica 반영 전에 Primary Commit 성공 가능
8. **Replication ≠ Failover:** 데이터 복제와 장애 시 서버 전환은 서로 다른 기능

### 반드시 이해해야 하는 동작 흐름

**Primary 데이터 변경 → Binary Log 기록 → Replica 이벤트 수신 → Relay Log 저장 → SQL Applier 적용 → Replica 데이터 반영**

이 과정이 MySQL Primary/Replica 복제의 핵심 원리이다.

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/e0e82449-3a3e-40a4-96f7-4476d7b73dcb" />
