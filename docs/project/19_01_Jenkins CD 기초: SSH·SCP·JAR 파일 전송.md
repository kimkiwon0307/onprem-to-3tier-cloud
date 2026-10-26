# 19장 1교시 — Jenkins CD 기초: SSH·SCP·JAR 파일 전송

## 1. 학습 목표

Jenkins CI에서 생성한 `app.jar`를 WAS01/02 서버에 안전하게 전달하는 방법을 학습한다.

핵심 학습 내용:

1. CI와 CD의 차이
2. SSH와 SCP의 역할
3. Jenkins → WAS 네트워크 연결
4. SSH 공개 키·개인 키 인증
5. Linux 배포 전용 계정과 권한
6. SCP 파일 전송
7. SHA-256을 이용한 전송 검증
8. Jenkins Credentials 및 Jenkinsfile 기초
9. SSH/SCP 장애 대응

**이번 교시에서는 JAR 전송까지만 학습한다. 기존 애플리케이션 교체·재시작은 19장 2교시에서 진행한다.**

---

## 2. CI와 CD의 차이

### CI — Continuous Integration

소스코드를 가져와 빌드·테스트하고 배포 가능한 산출물을 생성하는 과정이다.

```text
GitHub
   ↓
Jenkins
   ├── Checkout
   ├── Build
   ├── Test
   ├── Package
   └── Archive
   ↓
app.jar 생성
```

### CD — Continuous Delivery / Deployment

CI에서 생성한 산출물을 배포 환경에 전달하고, 서비스에 적용·검증하는 과정이다.

```text
Jenkins CI
    ↓
app.jar
    ↓
WAS 서버로 전달
    ↓
기존 JAR 백업
    ↓
신규 JAR 교체
    ↓
서비스 재시작
    ↓
Health Check
    ↓
배포 완료 또는 Rollback
```

- **Continuous Delivery:** 언제든 배포 가능한 상태를 유지하며, 실제 운영 반영에는 승인 단계가 있을 수 있다.
- **Continuous Deployment:** 검증 단계를 통과한 변경 사항을 자동으로 운영 환경까지 반영한다.

### 핵심

**SCP는 CD 자체가 아니라, CD 과정에서 파일을 전달하는 방법 중 하나다.**

---

## 3. Marble CD 아키텍처

```text
Jenkins EC2
    │
    │ SSH 인증 (TCP 22)
    │ SCP로 app.jar 전송
    │
    ├───────────────┐
    ▼               ▼
WAS01             WAS02
10.0.21.126       10.0.22.9
    │               │
    ▼               ▼
/opt/marble-deploy/incoming/app.jar
```

- Jenkins: CI 산출물 생성 및 배포 작업 실행
- WAS01/02: Spring Boot 애플리케이션 실행
- SSH: 원격 접속과 원격 명령 실행
- SCP: JAR 파일 전달
- `incoming/`: 새 JAR를 안전하게 받는 디렉터리

**운영 중인 `/opt/marble/app.jar`를 SCP로 직접 덮어쓰지 않는다.**

---

## 4. SSH란?

SSH(Secure Shell)는 원격 서버에 암호화된 연결을 제공하는 기술이다.

### 기본 명령어

```bash
ssh ubuntu@10.0.21.126
```

| 구성 | 의미 |
|---|---|
| `ssh` | 원격 서버 접속 |
| `ubuntu` | 원격 서버 사용자 |
| `@` | 사용자와 서버 주소 구분 |
| `10.0.21.126` | WAS01 사설 IP |

### 원격 명령 실행

```bash
ssh ubuntu@10.0.21.126 'whoami'
```

서버에 접속한 뒤 `whoami` 명령을 실행하고 결과를 반환한다.

`whoami`는 현재 명령을 실행하는 사용자 계정 이름을 출력한다.

---

## 5. SCP란?

SCP(Secure Copy)는 SSH를 이용해 서버 간 파일을 안전하게 복사하는 명령어다.

### 기본 구조

```bash
scp 원본파일 사용자@서버IP:목적지경로
```

### 예시

```bash
scp test.txt ubuntu@10.0.21.126:/home/ubuntu/
```

명령어 해석:

| 부분 | 의미 |
|---|---|
| `scp` | 파일 복사 |
| `test.txt` | 전송할 파일 |
| `ubuntu` | 원격 계정 |
| `10.0.21.126` | 원격 서버 IP |
| `/home/ubuntu/` | 파일 저장 위치 |

### SSH와 SCP의 차이

| 기술 | 주요 역할 |
|---|---|
| SSH | 원격 접속·명령 실행 |
| SCP | 원격 파일 전송 |

둘 다 일반적으로 TCP 22번 포트를 사용한다.

---

## 6. Jenkins와 WAS 간 네트워크 설정

Jenkins가 WAS 서버에 접속하려면 네트워크 통신이 허용되어야 한다.

### AWS Security Group

| 서버 | 방향 | 포트 | 허용 대상 |
|---|---|---|---|
| WAS01/02 | Inbound | TCP 22 | Jenkins SG |
| Jenkins | Outbound | TCP 22 | WAS SG |

WAS SSH 포트를 인터넷 전체에 공개할 필요는 없다.

### 연결 확인 명령어

```bash
nc -zv -w 3 10.0.21.126 22
```

| 옵션 | 의미 |
|---|---|
| `nc` | TCP 연결 등을 확인하는 도구 |
| `-z` | 포트 연결 여부 확인 |
| `-v` | 자세한 출력 |
| `-w 3` | 최대 3초 대기 |
| `22` | SSH 포트 |

연결 성공은 **TCP 22 포트에 접근 가능하다는 의미**다.

SSH 사용자 인증이 성공했다는 의미는 아니다.

---

## 7. SSH 키 인증의 원리

SSH 키 인증에는 두 가지 키가 사용된다.

| 종류 | 보관 장소 | 역할 |
|---|---|---|
| 개인 키 | Jenkins Credentials 등 | SSH 인증에 사용하는 비밀키 |
| 공개 키 | WAS의 `authorized_keys` | 접속 허용 대상의 공개 키 |

### 인증 구조

```text
Jenkins
 └── 개인 키
       │
       │ SSH 인증
       ▼
WAS01 / WAS02
 └── authorized_keys
       └── 공개 키
```

### 핵심 원칙

- 개인 키는 외부에 공개하지 않는다.
- WAS에는 공개 키를 등록한다.
- 개인 키를 Jenkinsfile에 직접 작성하지 않는다.
- 원격 서버의 Host Key도 확인해야 한다.
- 운영 환경에서는 Jenkins Credentials로 비밀 키를 관리한다.

### 키 생성 예시

```bash
ssh-keygen -t ed25519 -f ~/.ssh/marble_lab
```

| 옵션 | 의미 |
|---|---|
| `ssh-keygen` | SSH 키 생성 |
| `-t ed25519` | 사용할 키 알고리즘 |
| `-f` | 키 파일 저장 경로 |

생성 파일:

```text
~/.ssh/marble_lab      ← 개인 키
~/.ssh/marble_lab.pub  ← 공개 키
```

운영용 키는 조직 보안 정책에 따라 생성·보관하고 Jenkins Credentials에 등록한다.

---

## 8. 배포 전용 계정 생성

Marble 프로젝트에서는 다음 두 계정의 역할을 구분한다.

| 계정 | 역할 |
|---|---|
| `marble` | Spring Boot 서비스 실행 |
| `marble-deploy` | Jenkins에서 파일을 전달받는 계정 |

### 계정 생성

실습용 WAS에서:

```bash
sudo adduser --disabled-password --gecos "" marble-deploy
```

### 명령어 해석

| 명령어·옵션 | 의미 |
|---|---|
| `sudo` | 관리자 권한으로 실행 |
| `adduser` | 사용자 생성 |
| `--disabled-password` | 비밀번호 인증을 사용하지 않도록 생성 |
| `--gecos ""` | 추가 사용자 정보 입력 생략 |
| `marble-deploy` | 생성할 계정 |

### JAR 수신 디렉터리 생성

```bash
sudo install -d \
  -o marble-deploy \
  -g marble-deploy \
  -m 750 \
  /opt/marble-deploy/incoming
```

| 옵션 | 의미 |
|---|---|
| `install -d` | 디렉터리 생성 |
| `-o` | 소유자 설정 |
| `-g` | 소유 그룹 설정 |
| `-m 750` | 접근 권한 설정 |

### `750` 권한 의미

```text
소유자: rwx (7)
그룹:   r-x (5)
기타:   --- (0)
```

배포 계정만 해당 디렉터리에 파일을 기록할 수 있도록 구성한다.

---

## 9. authorized_keys 등록

WAS 서버의 배포 계정에 공개 키를 등록한다.

### SSH 디렉터리 생성

```bash
sudo install -d \
  -o marble-deploy \
  -g marble-deploy \
  -m 700 \
  /home/marble-deploy/.ssh
```

### 공개 키 등록

```bash
sudo nano /home/marble-deploy/.ssh/authorized_keys
```

Jenkins용 SSH 공개 키를 한 줄로 입력한다.

### 파일 소유자 설정

```bash
sudo chown marble-deploy:marble-deploy \
  /home/marble-deploy/.ssh/authorized_keys
```

### 파일 권한 설정

```bash
sudo chmod 600 \
  /home/marble-deploy/.ssh/authorized_keys
```

- `chown`: 파일 소유자 변경
- `chmod`: 파일 접근 권한 변경
- `600`: 소유자 읽기·쓰기, 나머지 접근 불가

SSH 인증 실패 시 이 파일의 내용·소유자·권한을 확인한다.

---

## 10. SSH 인증 검증

개인 키 파일이 준비되고 WAS의 공개 키 등록이 완료된 경우:

```bash
ssh -i ~/.ssh/marble_lab \
  marble-deploy@LAB_WAS01_IP \
  'whoami'
```

실습용 WAS IP를 사용한다.

### 옵션 해석

| 부분 | 의미 |
|---|---|
| `-i` | 인증에 사용할 개인 키 |
| `marble-deploy@...` | 원격 접속 계정과 주소 |
| `'whoami'` | 원격 서버에서 실행할 명령 |

정상 결과:

```text
marble-deploy
```

처음 접속할 때는 WAS의 SSH Host Key 지문을 확인하고 신뢰할 수 있는 서버인지 검증한다.

---

## 11. SCP 파일 전송

### 전송 명령어

```bash
scp -i ~/.ssh/marble_lab \
  test.txt \
  marble-deploy@LAB_WAS01_IP:/opt/marble-deploy/incoming/
```

### 전송 순서

```text
① 개인 키로 인증
       ↓
② WAS에 SSH 연결
       ↓
③ test.txt 전송
       ↓
④ incoming/에 저장
       ↓
⑤ 파일 존재·내용 확인
```

### 전송 후 검증

WAS 서버에서:

```bash
ls -lh /opt/marble-deploy/incoming/
```

파일 내용 확인:

```bash
cat /opt/marble-deploy/incoming/test.txt
```

### 실제 JAR 전송 구조

```bash
scp -i ~/.ssh/marble_lab \
  build/libs/app.jar \
  marble-deploy@LAB_WAS01_IP:/opt/marble-deploy/incoming/app.jar
```

`build/libs/app.jar`는 현재 작업 디렉터리에 해당 파일이 있는 경우의 예시다.

**실제 운영 WAS가 아닌 LAB-WAS에서 먼저 검증한다.**

---

## 12. SHA-256으로 파일 전송 검증

SCP가 성공했다고 해서 검증을 생략하지 않는다.

Jenkins와 WAS에서 파일 체크섬을 비교한다.

### Jenkins

```bash
sha256sum build/libs/app.jar
```

### WAS

```bash
sha256sum /opt/marble-deploy/incoming/app.jar
```

두 결과의 SHA-256 값이 같다면 두 파일의 내용이 동일한지 확인할 수 있다.

### SHA-256이 필요한 이유

- 전송된 파일의 동일성 검증
- 잘못된 JAR 전달 여부 확인
- 배포할 파일의 무결성 확인

---

## 13. Jenkins Credentials란?

Jenkins가 사용하는 비밀번호, SSH 개인 키, 토큰 등의 비밀정보를 관리하는 기능이다.

### SSH 인증 정보 예시

```text
Credential Type: SSH Username with private key

Username: marble-deploy

ID: marble-lab-ssh
```

Jenkinsfile에서는 개인 키 자체를 작성하는 대신 Credential ID를 참조한다.

### 개념

```text
Jenkinsfile
    │
    │ Credential ID 참조
    ▼
Jenkins Credentials
    │
    └── SSH 개인 키
            │
            ▼
         WAS 인증
```

**비밀 키를 GitHub에 업로드하거나 Jenkinsfile에 직접 하드코딩하지 않는다.**

---

## 14. 왕초보용 Jenkinsfile 개념

### 최소 SSH 명령 실행 예시

```groovy
pipeline {
    agent any

    stages {
        stage('SSH Test') {
            steps {
                sh 'ssh marble-deploy@LAB_WAS01_IP whoami'
            }
        }
    }
}
```

이 코드는 SSH 인증 및 신뢰 설정이 이미 준비된 경우의 개념 예시다.

### 코드 해석

| 코드 | 의미 |
|---|---|
| `pipeline` | 전체 자동화 작업 |
| `agent any` | 실행할 Jenkins 노드 선택 |
| `stages` | 작업 단계 묶음 |
| `stage('SSH Test')` | SSH 검사 단계 |
| `steps` | 실제 작업 영역 |
| `sh` | Linux Shell 명령어 실행 |
| `ssh ... whoami` | WAS에서 현재 사용자 확인 |

### 핵심

Jenkinsfile은 Linux에서 수동으로 실행하던 명령어를 Pipeline Stage 안에 넣어 자동화하는 방식이다.

실제 운영에서는 Jenkins Credentials, Host Key 검증, 계정 권한 제어를 함께 적용한다.

**19장 1교시에서는 기존 `Jenkinsfile.ci`를 수정할 필요가 없다. SSH/SCP 자동화 코드는 별도 실습 예시로 이해한다.**

---

## 15. SSH·SCP 장애 대응

| 오류 | 대표 원인 | 확인 사항 |
|---|---|---|
| `Connection timed out` | 네트워크·SG·라우팅 문제 | TCP 22 연결 |
| `Connection refused` | SSH 서비스 미실행·포트 문제 | SSH 서비스 상태 |
| `Permission denied (publickey)` | 키 인증 실패 | 개인 키·공개 키·계정 |
| `Host key verification failed` | 호스트 신뢰 검증 실패 | Host Key·known_hosts |
| `scp: Permission denied` | 파일 쓰기 권한 부족 | 디렉터리 소유자·권한 |
| 파일 없음 | 전송 경로 오류 | `ls -lh` |
| 체크섬 불일치 | 파일 내용 불일치 | `sha256sum` |

### 장애 분석 순서

```text
1. Jenkins에서 WAS의 TCP 22에 연결 가능한가?
                  ↓
2. SSH 키 인증으로 로그인 가능한가?
                  ↓
3. 배포 계정에 파일 쓰기 권한이 있는가?
                  ↓
4. SCP로 파일을 정상 전송했는가?
                  ↓
5. Jenkins와 WAS의 SHA-256 값이 같은가?
```

네트워크 오류, 인증 오류, 파일 권한 오류를 구분해 원인을 좁혀야 한다.

---

## 16. SCP 외의 다른 배포 방식

| 방법 | 특징 |
|---|---|
| SCP | SSH로 파일 직접 복사 |
| rsync | SSH 기반 파일 동기화 가능 |
| S3 + SSM | S3 저장 후 SSM으로 다운로드·배포 실행 |
| AWS CodeDeploy | EC2 배포 절차와 상태 관리 |
| Nexus·Artifactory | 빌드 산출물 저장소 활용 |
| Docker Registry | 컨테이너 이미지 기반 배포 |

현재 Marble 학습에서는 원리를 이해하기 위해 Jenkins + SCP 방식을 사용한다.

이후 S3 + SSM이나 CodeDeploy 방식으로 확장할 수 있다.

---

## 17. 19장 1교시 최종 핵심

1. CI는 `app.jar`를 만들고 검증하는 과정이다.
2. CD는 산출물을 서비스에 안전하게 적용하고 검증하는 과정이다.
3. SCP는 CD에 사용할 수 있는 파일 전송 방법 중 하나다.
4. SSH는 원격 접속과 원격 명령 실행에 사용한다.
5. SSH 공개 키는 WAS에 등록하고 개인 키는 안전하게 보관한다.
6. Jenkins와 WAS 사이 TCP 22 통신을 허용해야 한다.
7. 배포 계정과 애플리케이션 실행 계정은 역할을 분리할 수 있다.
8. 새로운 JAR는 `incoming/`에 먼저 전달한다.
9. `sha256sum`으로 Jenkins와 WAS의 파일 동일성을 확인한다.
10. Jenkins Credentials로 SSH 비밀정보를 관리한다.
11. 장애 발생 시 네트워크 → 인증 → 권한 → 전송 → 체크섬 순서로 확인한다.
12. 19장 1교시에서는 서비스 재시작이나 실제 JAR 교체를 진행하지 않는다.

### 면접에서 설명할 수 있는 핵심 답변

**Q. Jenkins에서 WAS 서버로 애플리케이션을 어떻게 전달하나요?**

Jenkins CI Pipeline에서 Spring Boot 실행형 JAR를 생성한 뒤, SSH 키 인증과 SCP를 이용해 WAS 서버의 배포 전용 디렉터리로 전달하는 구조를 구성했습니다.

Jenkins Credentials로 개인 키를 관리하고, Security Group으로 Jenkins와 WAS 간 SSH 통신을 제한합니다. 전송 이후 SHA-256 체크섬을 비교하여 파일 동일성을 검증하도록 설계했습니다.

이후 배포 단계에서는 기존 JAR 백업, 신규 버전 적용, 서비스 재시작, Health Check 및 실패 시 Rollback을 수행하도록 확장할 계획입니다.

---

## 다음 수업 — 19장 2교시

**Jenkins CD: WAS01 → WAS02 순차 배포**

학습 예정 내용:

- 기존 `app.jar` 백업
- 새 JAR로 교체
- `systemctl restart marble`
- `/health` 응답 확인
- WAS01 성공 후 WAS02 배포
- 왕초보용 Jenkins 배포 Stage 작성

※ 이 문서는 19장 1교시의 강의 내용을 정리한 것으로, 실제 운영 환경에서의 SSH/SCP 성공 여부는 별도 실습 검증이 필요하다.
<img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/4aeef410-2e79-4433-8fc2-5c819779cef9" />


