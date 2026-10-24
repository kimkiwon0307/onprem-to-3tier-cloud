# 13장 1교시 — HTTP 세션과 WAS 이중화, Sticky Session, HTTP 403

## 1. HTTP, Cookie, Session

### HTTP Stateless

HTTP는 기본적으로 Stateless(무상태) 프로토콜이다.

서버는 HTTP 요청 자체만으로 이전에 요청한 사용자인지 기억하지 못한다.

### Cookie

브라우저에 저장되는 데이터로, HTTP 요청 시 서버에 전달할 수 있다.

대표적으로 세션 식별자(Session ID)를 전달하는 데 사용된다.

### Session

여러 HTTP 요청에 걸쳐 사용자의 로그인 상태 등을 유지하는 방식이다.

**동작 순서**

```text
1. 사용자 로그인 요청
          ↓
2. 서버에서 세션 생성
          ↓
3. 브라우저에 세션 ID 쿠키 발급
          ↓
4. 브라우저가 다음 요청에 쿠키 전달
          ↓
5. 서버가 세션 ID로 사용자 상태 조회
```

**핵심:** Cookie는 세션 식별자를 전달하고, Session은 사용자 상태를 관리하는 데 사용된다.

---

## 2. WAS 이중화에서 세션 문제가 발생하는 이유

세션이 각 WAS의 메모리에 따로 저장되는 경우:

```text
사용자 A (Session ID: abc123)
             |
             v
       Apache Balancer
             |
       +-----+-----+
       |           |
       v           v
     WAS01       WAS02
   abc123 있음  abc123 없음
       |           |
       v           v
   로그인 유지   세션 조회 실패
```

사용자가 WAS01에서 로그인했지만 다음 요청이 WAS02로 전달되면, WAS02는 WAS01의 메모리 세션을 알지 못할 수 있다.

이로 인해 로그인 상태 불일치, 인증 실패, CSRF 오류 등이 발생할 수 있다.

**핵심:** WAS를 이중화할 때는 부하 분산뿐 아니라 세션 공유 방법도 고려해야 한다.

---

## 3. Sticky Session

Sticky Session은 동일 사용자의 요청을 가능한 한 같은 서버로 전달하도록 하는 기능이다.

### ALB Sticky Session

```text
사용자 A
    |
    v
   ALB
    |
    v
  WEB01 (고정)
```

ALB에서 설정하며, 사용자를 WEB01 또는 WEB02에 고정한다.

### Apache Sticky Session

```text
사용자 A
    |
    v
   ALB
    |
    v
 WEB01 (Apache)
    |
    v
 WAS01 (고정)
```

WEB 서버의 Apache에서 설정하며, 사용자를 WAS01 또는 WAS02에 고정한다.

| 구분 | 설정 위치 | 고정 대상 |
|---|---|---|
| ALB Sticky Session | AWS ALB Target Group | WEB01/02 |
| Apache Sticky Session | WEB Apache | WAS01/02 |

**중요:** ALB에서 WEB01을 고정해도 WEB01의 Apache는 WAS01/WAS02로 요청을 계속 분산할 수 있다.

### Round Robin과 Sticky Session

- Round Robin: 요청을 여러 서버에 순서대로 분산하는 알고리즘
- Sticky Session: 사용자의 후속 요청을 선택된 서버로 유지하는 기능

두 기능은 함께 사용할 수 있다.

**예시**

```text
Sticky Session OFF
사용자 A: WEB01 → WEB02 → WEB01

Sticky Session ON
사용자 A: WEB01 → WEB01 → WEB01
```

단, Sticky Session을 사용해도 대상 서버 장애나 쿠키 만료 등으로 다른 서버가 선택될 수 있다.

### Sticky Session의 한계

- 선택된 WAS가 장애로 중단되면 메모리 세션이 사라질 수 있음
- 사용자별 트래픽이 불균형해질 수 있음
- 다른 WAS에 세션을 공유하는 기능은 아님

---

## 4. Spring Session JDBC

Spring Session JDBC는 세션을 WAS 메모리 대신 공통 관계형 DB에 저장하여 여러 WAS가 공유할 수 있도록 하는 기술이다.

### Marble 실제 구조

```text
             사용자
               |
               v
              ALB
               |
        +------+------+
        |             |
        v             v
      WEB01         WEB02
        |             |
        +------+------+
               |
        Apache Balancer
               |
        +------+------+
        |             |
        v             v
      WAS01         WAS02
        |             |
        +------+------+
               |
               v
        MySQL DB01
      (공통 세션 저장)
               |
      +-------------------+
      | SPRING_SESSION    |
      | SPRING_SESSION_   |
      | ATTRIBUTES        |
      +-------------------+
```

### 동작 원리

1. 사용자가 WAS01을 통해 로그인한다.
2. 세션 정보가 공통 MySQL에 저장된다.
3. 브라우저는 세션 ID 쿠키를 보관한다.
4. 다음 요청이 WAS02로 전달된다.
5. WAS02가 MySQL에서 동일한 세션을 조회한다.
6. 사용자의 로그인 상태가 유지된다.

### MySQL 세션 테이블

| 테이블 | 역할 |
|---|---|
| `SPRING_SESSION` | 세션 ID, 생성 시간, 만료 시간 등 |
| `SPRING_SESSION_ATTRIBUTES` | 세션에 저장된 속성 데이터 |

**핵심:** WAS01과 WAS02가 같은 MySQL 세션 저장소에 접근하기 때문에 요청이 다른 WAS로 전달되어도 동일한 세션을 사용할 수 있다.

---

## 5. Marble 프로젝트에 Sticky Session이 필요한가?

**세션 유지를 목적으로 한다면 필수는 아니다.**

Marble은 Spring Session JDBC를 사용하므로 다음과 같은 구조가 가능하다.

```text
첫 번째 요청
사용자 → ALB → WEB01 → WAS01
                            |
                            v
                        MySQL Session

두 번째 요청
사용자 → ALB → WEB02 → WAS02
                            |
                            v
                        MySQL Session
```

WAS01/02가 공통 세션을 조회하기 때문에 ALB와 Apache에서 사용자를 특정 서버에 고정하지 않아도 된다.

다만 다음 조건을 확인해야 한다.

- WAS01/02가 동일한 세션 저장소 사용
- 브라우저 세션 쿠키 정상 전달
- Spring Security 인증 및 CSRF 검증 정상
- WEB/WAS에 세션 외 별도의 사용자별 로컬 상태가 없음

**결론:** Spring Session JDBC가 올바르게 구성된 Marble에서는 세션 유지를 위해 ALB 및 Apache Sticky Session을 반드시 사용할 필요가 없다.

---

## 6. CSRF와 HTTP 403

### CSRF란?

Cross-Site Request Forgery(사이트 간 요청 위조)의 약자다.

로그인된 사용자의 브라우저를 악용하여 사용자가 의도하지 않은 요청을 보내게 하는 공격이다.

### CSRF Token

CSRF 공격을 방어하기 위한 검증 값이다.

Spring Security에서는 일반적으로 POST 등 상태 변경 요청에 CSRF 보호가 적용된다.

```text
1. GET /signup
       |
       v
서버에서 CSRF Token 제공
       |
       v
2. POST /signup
   CSRF Token 제출
       |
       v
3. Spring Security 검증
       |
       +-- 일치 → 요청 처리
       |
       +-- 불일치 → HTTP 403 가능
```

### WAS 이중화와 HTTP 403

```text
GET /signup
    |
    v
  WAS01
    |
    v
세션에 연결된 CSRF Token 제공

POST /signup
    |
    v
  WAS02
    |
    v
WAS01의 메모리 세션을 조회하지 못함
    |
    v
CSRF Token 검증 실패 가능
    |
    v
HTTP 403
```

기본적인 세션 기반 CSRF 저장 방식에서는 WAS 간 세션 불일치가 CSRF 검증 실패로 이어질 수 있다.

Spring Session JDBC로 공통 세션을 사용하면 이러한 원인을 해결할 수 있다.

단, HTTP 403은 권한 부족, CSRF Token 누락, 쿠키 오류, Spring Security 설정 오류 등으로도 발생하므로 로그를 확인해야 한다.

**주의:** CSRF 보호를 단순히 비활성화하는 것은 근본적인 해결책이 아니다.

---

## 7. Marble 실제 Apache 설정

```apache
<Proxy "balancer://marblecluster">
    BalancerMember "http://10.0.21.126:8080"
    BalancerMember "http://10.0.22.9:8080"

    ProxySet lbmethod=byrequests
</Proxy>
```

- `BalancerMember`: WAS 부하 분산 대상 등록
- `lbmethod=byrequests`: 요청 수를 기준으로 분산하는 방식
- `stickysession`: 공유받은 설정에는 명시되어 있지 않음

`byrequests`는 요청 수 기반 분산 방식이며 항상 단순 Round Robin과 동일하다고 볼 수는 없다.

### 현재 설정 확인 명령어

WEB 서버:

```bash
sudo grep -RniE \
'BalancerMember|lbmethod|stickysession|ProxyPass' \
/etc/apache2/sites-enabled/ \
/etc/apache2/conf-enabled/
```

WAS 분산 로그:

```bash
sudo tail -n 20 /var/log/apache2/marble_backend.log
```

DB01 MySQL:

```sql
SHOW TABLES LIKE 'SPRING_SESSION%';
```

```sql
SELECT COUNT(*) FROM SPRING_SESSION;
```

모두 조회 목적의 명령어이며 기존 설정을 변경하지 않는다.

---

## 8. 최종 핵심 정리

| 개념 | 반드시 기억할 내용 |
|---|---|
| HTTP Stateless | HTTP는 이전 요청의 상태를 자동으로 기억하지 않음 |
| Cookie | 브라우저에서 세션 ID 등을 저장·전송 |
| Session | 사용자 로그인 상태 등을 유지 |
| WAS 세션 불일치 | WAS별 메모리 세션이 공유되지 않아 발생 |
| ALB Sticky | 사용자를 특정 WEB에 고정 |
| Apache Sticky | 사용자를 특정 WAS에 고정 |
| Round Robin | 요청을 여러 서버로 순차 분산 |
| Spring Session JDBC | MySQL을 이용한 WAS 간 세션 공유 |
| CSRF | 사이트 간 요청 위조 공격 |
| HTTP 403 | CSRF 검증 실패, 권한 부족 등으로 발생 가능 |

### 가장 중요한 3가지

**① WAS 이중화 시 세션 문제**

WAS01에서 생성한 세션을 WAS02가 조회하지 못하면 로그인·인증 관련 장애가 발생할 수 있다.

**② Sticky Session과 Spring Session JDBC의 차이**

Sticky Session은 사용자를 같은 서버로 보내는 방식이고, Spring Session JDBC는 여러 서버가 세션을 공유하는 방식이다.

**③ Marble에서 선택한 방식**

Marble은 MySQL 기반 Spring Session JDBC를 이용하므로, 세션 유지 목적으로 ALB와 Apache의 Sticky Session을 반드시 설정할 필요는 없다.

### 최종 결론

**WAS 이중화에서는 단순히 요청을 분산하는 것뿐 아니라 여러 WAS가 동일한 사용자 세션을 사용할 수 있는지 검증해야 한다. Marble에서는 Spring Session JDBC를 통해 공통 MySQL에 세션을 저장하여 이 문제를 해결하는 구조를 사용한다.**

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/6a15bf2b-cb08-47a7-bbc2-2e4f07db1001" />

