
# A과정 4장 1교시 — Spring Boot WAS 서버 구축

## 1. 실습 환경

| 서버 | IP | 역할 |
|---|---|---|
| WEB01 | 192.168.111.147 | Apache |
| WAS01 | 192.168.111.148 | Spring Boot |
| DB01 | 추후 확인 | MySQL |

- 운영체제: Ubuntu
- 로그인 계정: ubuntu
- Java: JDK 17 기준
- 애플리케이션: 기존 app.jar 사용
- WAS 포트: 8080 기준 (실제 설정 확인 필요)

---

## 2. WAS란?

WAS(Web Application Server)는 HTTP 요청을 받아 애플리케이션의 기능을 실행하는 서버이다.

### WEB과 WAS의 역할

| 구분 | 역할 |
|---|---|
| WEB (Apache) | HTTP 요청 수신, 정적 파일 제공, Reverse Proxy |
| WAS (Spring Boot) | Java 코드 실행, 비즈니스 로직 처리 |
| DB (MySQL) | 데이터 저장 및 조회 |

### 요청 흐름

```text
사용자
   ↓ HTTP :80
Apache WEB01
   ↓ Reverse Proxy
Spring Boot WAS01 :8080
   ↓
Tomcat
   ↓
Spring MVC → Controller
   ↓
Java 메서드 실행
   ↓
HTTP 응답
   ↓
Apache → 사용자
```

핵심: Apache는 요청을 전달하고, Spring Boot는 애플리케이션 기능을 실행한다.

---

## 3. Spring Boot 주요 구성요소

| 구성요소 | 역할 |
|---|---|
| JDK | Java 개발 및 실행 환경 |
| Spring Boot | Java 애플리케이션 구성 및 실행 |
| 내장 Tomcat | HTTP 연결과 요청 처리 |
| Spring MVC | URL에 따라 요청을 적절한 Controller로 전달 |
| Controller | 요청에 연결된 Java 메서드 실행 |
| Gradle | 애플리케이션 빌드 |
| JAR | 실행 가능한 애플리케이션 파일 |

Spring Boot는 내장 Tomcat을 사용할 수 있어 별도의 Tomcat 서비스를 직접 설치하지 않아도 된다.

---

## 4. Spring Boot 요청 처리 원리

예시 Controller:

```java
@RestController
public class HealthController {

    @GetMapping("/health")
    public String health() {
        return "WAS01 OK";
    }
}
```

- @RestController: HTTP 응답을 처리하는 Controller
- @GetMapping("/health"): GET /health 요청과 메서드 연결
- return: 응답 본문 반환

### 동작 과정

1. 클라이언트가 /health 요청
2. 내장 Tomcat이 HTTP 요청 수신
3. Spring MVC가 Controller 탐색
4. Controller 메서드 실행
5. HTTP 200 응답 반환

주의:
직접 만든 /health는 단순한 응답 확인 기능이며,
DB 연결이나 전체 서비스 정상 여부를 보장하지 않는다.

기존 app.jar에 /health가 실제 구현되어 있는지는 별도 확인이 필요하다.

---

## 5. Java 및 Spring Boot 실행

### Java 버전 확인

```bash
java -version
javac -version
```

- java: Java 애플리케이션 실행
- javac: Java 소스코드 컴파일
- -version: 버전 확인

### 기존 JAR 실행

```bash
java -jar app.jar
```

- java: JVM 실행
- -jar: 실행 가능한 JAR 파일 지정
- app.jar: Spring Boot 애플리케이션

주의:
app.jar가 있는 디렉터리에서 실행하거나 JAR의 전체 경로를 지정한다.

이미 실행 중인 경우 중복 실행하지 않는다.

### Gradle 빌드 명령어

```bash
./gradlew clean bootJar
```

- ./gradlew: Gradle Wrapper 실행
- clean: 이전 빌드 결과 정리
- bootJar: Spring Boot 실행형 JAR 생성

이번 실습은 기존 app.jar를 재사용하므로 새 프로젝트 생성과 빌드는 필수가 아니다.

---

## 6. Spring Boot 주요 설정

application.properties 예시:

```properties
server.port=8080
server.address=0.0.0.0
```

| 설정 | 의미 |
|---|---|
| server.port | HTTP 수신 포트 |
| server.address | 연결을 수신할 로컬 IP 주소 |

- 127.0.0.1: 해당 서버 내부 루프백 주소
- 0.0.0.0: 모든 IPv4 로컬 인터페이스에서 수신

WAS를 외부에 무조건 공개하지 않고 방화벽 등으로 접근을 제한해야 한다.

---

## 7. Spring Boot 정상 동작 검증

### ① WAS01에서 Java 프로세스 확인

```bash
ps -ef | grep '[j]ava'
```

### ② WAS01에서 TCP 8080 확인

```bash
sudo ss -lntp '( sport = :8080 )'
```

Java 프로세스가 8080 포트를 LISTEN하는지 확인한다.

### ③ WAS01에서 HTTP 응답 확인

```bash
curl -i http://127.0.0.1:8080/health
```

/health가 구현되어 있다면 HTTP 200을 기대할 수 있다.

### ④ WEB01에서 WAS01 연결 확인

WEB01에서 실행:

```bash
nc -vz -w 3 192.168.111.148 8080

curl -i http://192.168.111.148:8080/health
```

- nc: TCP 포트 연결 확인
- -v: 상세 결과
- -z: 연결 검사
- -w 3: 최대 연결 대기 3초
- curl -i: HTTP 헤더와 본문 출력

### ⑤ Apache Reverse Proxy 경유 확인

Apache가 /app/ 경로를 WAS로 전달하도록 설정된 경우:

```apache
ProxyPass "/app/" "http://192.168.111.148:8080/"
ProxyPassReverse "/app/" "http://192.168.111.148:8080/"
```

WEB01에서 실행:

```bash
curl -i -H "Host: web01.test" \
  http://127.0.0.1/app/health
```

요청 경로:

```text
/app/health
    ↓ ProxyPass
http://192.168.111.148:8080/health
```

기존 Apache 설정의 ServerName과 ProxyPass 경로가 다른 경우 실제 설정에 맞게 테스트한다.

---

## 8. 주요 장애 대응

### ① Java 실행 실패

```bash
java -version
which java
```

Java 설치 상태와 실행 경로 확인.

### ② TCP 8080 포트 충돌

```bash
sudo ss -lntp '( sport = :8080 )'
```

8080 포트를 이미 사용하는 프로세스 확인.

### ③ HTTP 404 Not Found

확인할 내용:
- 요청 URL이 실제 구현되어 있는가?
- Controller 경로가 맞는가?
- 최신 JAR가 실행되고 있는가?

404라고 Spring Boot 프로세스가 종료된 것은 아니다.

### ④ WAS 직접 요청 성공, Apache 경유 실패

WEB01에서 실행:

```bash
sudo apache2ctl configtest
sudo apache2ctl -S
apache2ctl -M | grep proxy

sudo tail -n 50 /var/log/apache2/error.log
```

확인 대상:
- VirtualHost
- ProxyPass
- mod_proxy / mod_proxy_http
- Apache Error Log

---

## 9. 최종 핵심 정리

1. WAS는 애플리케이션 기능을 실행하는 서버다.
2. Spring Boot는 내장 Tomcat으로 HTTP 요청을 처리할 수 있다.
3. Controller는 URL 요청과 Java 메서드를 연결한다.
4. java -jar로 실행형 JAR를 실행한다.
5. ss로 포트 수신 상태를 확인한다.
6. curl로 실제 HTTP 응답을 검증한다.
7. WEB01에서 WAS01 직접 연결과 Apache 경유 연결을 구분해서 확인한다.
8. 장애 발생 시 프로세스 → 포트 → 직접 HTTP → Reverse Proxy 순서로 원인을 좁힌다.

### 반드시 기억할 핵심

**Spring Boot 실행 성공 ≠ TCP 포트 수신 성공 ≠ HTTP 요청 정상 처리**

세 가지를 각각 확인해야 한다.

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/36895100-61e6-4c7f-a662-e030829778b7" />






