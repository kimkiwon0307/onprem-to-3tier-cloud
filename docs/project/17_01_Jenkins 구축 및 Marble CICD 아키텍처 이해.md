# 17장 1교시 — Jenkins 구축 및 CI/CD 기초

## 1. Jenkins란?

Jenkins는 애플리케이션의 빌드, 테스트, 배포 과정을 자동화하는 CI/CD 도구이다.

### 도입 이유

수동 배포에서는 개발자가 직접 다음 작업을 수행해야 한다.

```text
소스코드 수정
    ↓
Gradle Build
    ↓
JAR 파일 생성
    ↓
WAS 서버에 파일 전송
    ↓
Spring Boot 재시작
    ↓
정상 동작 확인
```

Jenkins를 사용하면 위 과정을 자동화하여 반복 작업과 배포 실수를 줄일 수 있다.

### CI와 CD

| 구분 | 의미 | Marble 적용 |
|---|---|---|
| CI | 지속적 통합 | GitHub → Build → Test |
| CD | 지속적 전달·배포 | JAR 배포 → 검증 → 롤백 |

**핵심:** Jenkins는 정해진 절차에 따라 빌드·테스트·배포 작업을 자동으로 실행하도록 관리한다.

---

## 2. Marble CI/CD 전체 아키텍처

```text
개발자
   |
   | Git Push
   v
GitHub
   |
   | Webhook
   v
Jenkins
   |
   +-- 1. Git Checkout
   |
   +-- 2. Gradle Build
   |
   +-- 3. Test
   |
   +-- 4. JAR 생성
   |
   +-- 5. WAS 배포
   |       |
   |       +-- WAS01
   |       +-- WAS02
   |
   +-- 6. Health Check
   |
   +-- 7. 실패 시 Rollback
```

실제 Marble WAS:

- WAS01: `10.0.21.126:8080`
- WAS02: `10.0.22.9:8080`

**Jenkins는 WEB/WAS 서버가 아니라 애플리케이션의 빌드·테스트·배포를 관리하는 서버이다.**

---

## 3. Jenkins 주요 구성요소

| 구성요소 | 역할 |
|---|---|
| Controller | Jenkins 작업 관리 및 실행 제어 |
| Agent | 실제 빌드·테스트 작업 수행 |
| Job | Jenkins가 실행할 작업 |
| Pipeline | CI/CD 작업을 단계별 코드로 정의 |
| Workspace | 소스코드와 빌드 파일 작업 공간 |
| Plugin | GitHub, Gradle 등 기능 확장 |
| Credentials | 인증 토큰, SSH 키 등 비밀정보 관리 |

### Controller와 Agent

```text
     Jenkins Controller
        (작업 관리)
              |
        +-----+-----+
        |           |
        v           v
      Agent01     Agent02
     (Build)      (Deploy)
```

- Controller: 작업을 관리하고 실행을 조정
- Agent: 할당된 작업을 실제로 실행

실무에서는 보안과 자원 분리를 위해 Controller와 빌드 Agent를 분리하는 구성이 권장된다.

---

## 4. Jenkins 설치 (Ubuntu 24.04)

### ① Java 21 설치

```bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
```

Java 버전 확인:

```bash
java -version
```

Jenkins 실행용 Java와 Marble 애플리케이션 빌드용 Java는 서로 다를 수 있다.

### ② Jenkins 공식 저장소 등록

서명 키 다운로드:

```bash
sudo mkdir -p /etc/apt/keyrings

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
```

저장소 파일 생성:

```bash
cd /etc/apt/sources.list.d/
sudo nano jenkins.list
```

파일 내용:

```text
deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/
```

### ③ Jenkins 설치

```bash
sudo apt update
sudo apt install -y jenkins
```

### ④ Jenkins 실행 상태 확인

```bash
sudo systemctl status jenkins
```

정상 상태:

```text
Active: active (running)
```

부팅 시 자동 시작 여부:

```bash
sudo systemctl is-enabled jenkins
```

### ⑤ 포트 확인

```bash
sudo ss -lntp | grep ':8080 '
```

Jenkins의 기본 HTTP 포트는 `8080`이다.

로컬 HTTP 응답 확인:

```bash
curl -I http://127.0.0.1:8080/
```

### ⑥ Jenkins 로그 확인

```bash
sudo journalctl -u jenkins -n 50 --no-pager
```

**핵심:** 서비스 실행 상태, TCP 8080 LISTEN, HTTP 응답을 각각 확인해야 한다.

---

## 5. Jenkins 웹 초기 설정

접속 주소:

```text
http://JENKINS_VM_IP:8080
```

실습 환경에서는 SSH 터널을 이용해 외부 포트 공개 없이 접속할 수도 있다.

초기 관리자 비밀번호 조회:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

초기 설정 순서:

1. 관리자 잠금 해제
2. Suggested Plugins 설치
3. 관리자 계정 생성
4. Jenkins URL 설정
5. 대시보드 접속 확인

**주의:** 초기 관리자 비밀번호와 인증 정보는 GitHub에 공개하지 않는다.

---

## 6. Jenkins 주요 디렉터리

기본 Jenkins 데이터 저장 경로:

```text
/var/lib/jenkins
```

| 경로 | 역할 |
|---|---|
| `/var/lib/jenkins` | Jenkins 설정 및 데이터 |
| `/var/lib/jenkins/jobs` | Job 정보 |
| `/var/lib/jenkins/workspace` | Job 작업 공간 |
| `/var/lib/jenkins/plugins` | 설치된 Plugin |
| `/var/lib/jenkins/secrets` | 보안 관련 데이터 |

### Workspace란?

Jenkins가 소스코드를 가져오고 빌드 작업을 실행하는 디렉터리이다.

예시:

```text
/var/lib/jenkins/workspace/marble-environment-check
```

단, 실제 Workspace 위치는 실행 노드와 Jenkins 설정에 따라 달라질 수 있다.

---

## 7. 첫 번째 Jenkins Job 생성

### Freestyle Job

Jenkins 웹 화면에서 다음 순서로 생성한다.

```text
New Item
    ↓
marble-environment-check
    ↓
Freestyle project
    ↓
Build Steps
    ↓
Execute shell
```

Shell 명령어:

```bash
echo "=== Marble Jenkins 확인 ==="

whoami
pwd
java -version
echo "$WORKSPACE"
date
```

### 명령어 의미

| 명령어 | 설명 |
|---|---|
| `whoami` | Job 실행 계정 확인 |
| `pwd` | 현재 작업 디렉터리 |
| `java -version` | Java 버전 확인 |
| `$WORKSPACE` | Jenkins 작업 공간 경로 |
| `date` | 실행 날짜 및 시간 |

### 실행 검증

`Build Now → Console Output`

정상 실행 시 마지막에 다음 메시지가 표시된다.

```text
Finished: SUCCESS
```

**핵심:** Jenkins는 Job을 실행 노드에 할당하고, Workspace에서 명령어를 실행한 뒤 결과와 로그를 기록한다.

---

## 8. Jenkins 장애 대응

| 장애 | 원인 후보 | 확인 방법 |
|---|---|---|
| Jenkins 실행 실패 | Java 버전·설정 오류 | `systemctl`, `journalctl` |
| 웹 접속 실패 | 포트·방화벽·네트워크 | `ss`, `curl`, IP 확인 |
| 8080 포트 충돌 | 다른 프로세스 점유 | `ss -lntp` |
| Job 실행 실패 | 명령어·권한·환경 문제 | Console Output |
| Plugin 설치 실패 | 네트워크·저장소·호환성 | Jenkins 로그 |

### 기본 장애 분석 순서

```text
1. Jenkins 서비스 상태
          ↓
2. Java 버전 확인
          ↓
3. TCP 8080 포트 확인
          ↓
4. HTTP 응답 확인
          ↓
5. Jenkins 로그 분석
          ↓
6. Job Console Output 확인
```

서비스가 실행 중이라는 사실만으로 웹 접속과 Job 실행까지 정상이라고 판단하면 안 된다.

---

## 9. Jenkins 보안 핵심

- Jenkins 관리자 페이지를 인터넷에 무분별하게 공개하지 않는다.
- GitHub Token과 SSH 키는 Jenkins Credentials로 관리한다.
- Jenkins에 불필요한 root 권한을 부여하지 않는다.
- Plugin을 관리하고 보안 업데이트를 적용한다.
- Jenkins Controller와 빌드 Agent 분리를 고려한다.
- 배포 권한은 필요한 서버와 명령어로 제한한다.

---

## 10. 최종 핵심 정리

1. **Jenkins:** 빌드·테스트·배포 작업을 자동화하는 도구
2. **CI:** 소스코드 변경을 통합하고 자동 검증
3. **CD:** 검증된 결과물을 전달·배포
4. **Controller:** Jenkins 작업 관리
5. **Agent:** 실제 빌드 및 배포 작업 실행
6. **Job:** Jenkins가 수행할 작업
7. **Pipeline:** CI/CD 절차를 코드로 정의
8. **Workspace:** 빌드 작업 디렉터리
9. **Credentials:** 인증 정보 안전하게 관리
10. **Console Output:** Job 실행 결과와 오류 확인

### Marble 프로젝트에서 Jenkins를 도입한 이유

수동으로 진행하던 Spring Boot 빌드, JAR 파일 전달, WAS01/02 배포, 정상 동작 확인을 자동화하기 위해서이다.

### 최종 결론

**Jenkins는 GitHub의 소스코드 변경을 시작으로 빌드·테스트·배포·검증 과정을 자동화하기 위한 CI/CD 서버이다. Marble 프로젝트에서는 WAS01/02에 일관된 버전을 배포하고 배포 실패 시 롤백할 수 있는 파이프라인 구축을 목표로 한다.**

<img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/8ca1df3e-661c-4cfd-9704-eb059adffe0a" />
