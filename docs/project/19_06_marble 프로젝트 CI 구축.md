# Marble 프로젝트 — Jenkins CI 구축 (1교시)

## 1. 학습 목표

이번 교시에서는 Jenkins를 이용해 GitHub의 Spring Boot 프로젝트를 자동으로 빌드하고 테스트한 뒤, 실행형 JAR 파일을 생성하고 보관하는 CI 환경을 학습한다.

**최종 목표**

```text
개발자 PC
    |
    | git push
    v
GitHub Repository
    |
    | Poll SCM (변경 사항 확인)
    v
Jenkins
    |
    ├── 1. Checkout : 소스코드 가져오기
    ├── 2. Build    : Java 컴파일
    ├── 3. Test     : 자동 테스트
    ├── 4. Package  : app.jar 생성
    └── 5. Archive  : JAR 파일 보관
```

> CI는 소스코드를 가져와 빌드·테스트하고, 배포 가능한 결과물을 생성하는 과정이다. WAS01/02에 실제 배포하는 CD 과정은 2교시에서 다룬다.

---

## 2. CI/CD란?

### CI (Continuous Integration)

지속적 통합을 의미한다.

개발자가 GitHub에 코드를 Push하면 Jenkins가 소스코드를 가져와 자동으로 빌드하고 테스트한다.

**CI가 필요한 이유**

- 코드 수정 후 반복적인 수동 빌드 작업 감소
- 컴파일 오류 조기 발견
- 자동 테스트를 통한 문제 확인
- 일관된 빌드 과정 유지
- 배포할 JAR 파일 생성

### CD (Continuous Delivery / Deployment)

CI에서 생성한 결과물을 실제 서버에 전달하는 후속 과정이다.

Marble 프로젝트에서는 Jenkins가 WAS01/02에 JAR 파일을 전송하고 서비스를 재시작하는 자동 배포를 구성했다.

| CI | CD |
|---|---|
| GitHub Checkout | WAS에 JAR 전송 |
| Gradle Build | 기존 JAR 백업 |
| Gradle Test | 신규 JAR 교체 |
| JAR 생성 | 서비스 재시작 |
| JAR 보관 | Health Check |

---

## 3. 실습 환경

| 항목 | 구성 |
|---|---|
| CI 서버 | AWS EC2 |
| 운영체제 | Ubuntu 24.04 |
| CI 도구 | Jenkins |
| Java | JDK 21 |
| Build Tool | Gradle Wrapper 8.10.2 |
| 애플리케이션 | Spring Boot Marble |
| 형상관리 | GitHub |
| Repository | `kimkiwon0307/marble_project` |
| Branch | `master` |
| Pipeline 파일 | `Jenkinsfile.ci` |
| 빌드 결과물 | `build/libs/app.jar` |
| 변경 감지 | Poll SCM |

---

# PART 1. Jenkins 환경 확인

## 4. Jenkins 서버 상태 확인

**실행 위치: Jenkins EC2**

### 4-1. Jenkins 서비스 확인

```bash
# Jenkins 서비스의 상세 상태 확인
sudo systemctl status jenkins

# Jenkins가 실행 중인지 간단하게 확인
sudo systemctl is-active jenkins
```

정상 결과:

```text
active
```

**사용 이유**

Jenkins가 실행 중이어야 GitHub 소스코드를 가져오고 Gradle 작업을 수행할 수 있다.

### 4-2. Java 확인

```bash
# Java 실행 버전 확인
java -version

# Java 컴파일러 확인
javac -version
```

**사용 이유**

Jenkins는 Java 기반 애플리케이션이며, Spring Boot 프로젝트의 컴파일에도 적절한 JDK가 필요하다.

### 4-3. Git 확인

```bash
# Git 설치 여부와 버전 확인
git --version
```

**사용 이유**

GitHub Repository에서 Marble 소스코드를 가져오기 위해 필요하다.

---

# PART 2. GitHub 연결

## 5. GitHub Repository 설정

Marble 프로젝트 Repository:

```text
https://github.com/kimkiwon0307/marble_project.git
```

브랜치:

```text
master
```

Private Repository를 사용하므로 Jenkins가 해당 Repository에 접근할 수 있도록 인증 정보를 등록했다.

### Jenkins Credentials

Jenkins 웹 화면에서 다음 메뉴로 이동한다.

```text
Manage Jenkins
    ↓
Credentials
    ↓
System
    ↓
Global credentials
    ↓
Add Credentials
```

설정:

| 항목 | 값 |
|---|---|
| Kind | Username with password |
| Username | GitHub 계정명 |
| Password | GitHub Personal Access Token |
| Credential ID | `github-marble-read` |

**PAT(Personal Access Token)를 사용하는 이유**

Jenkins가 비공개 GitHub Repository의 소스코드를 읽을 수 있도록 인증하기 위해서다.

PAT는 GitHub에 업로드하거나 Jenkinsfile에 직접 작성하지 않는다.

---

## 6. Jenkins Pipeline Job 생성

Jenkins 웹 화면에서 다음과 같이 진행한다.

```text
Dashboard
    ↓
New Item
    ↓
Pipeline 선택
```

### Pipeline 설정

| 항목 | 설정 |
|---|---|
| Definition | Pipeline script from SCM |
| SCM | Git |
| Repository URL | GitHub Repository URL |
| Credentials | `github-marble-read` |
| Branch | `*/master` |
| Script Path | `Jenkinsfile.ci` |

### Pipeline script from SCM이란?

Jenkins 화면에 모든 코드를 직접 작성하지 않고 GitHub Repository에 저장된 Jenkinsfile을 가져와 실행하는 방식이다.

**선택 이유**

- Jenkins Pipeline 코드를 GitHub에서 관리
- 변경 이력 확인 가능
- Jenkins 설정을 코드로 재사용 가능
- CI/CD 구성 내용을 포트폴리오에 기록 가능

---

# PART 3. Gradle 프로젝트 준비

## 7. Gradle Wrapper란?

Gradle은 Java 프로젝트를 빌드하는 도구다.

Gradle Wrapper는 프로젝트에서 지정한 Gradle 버전을 실행하도록 도와준다.

Marble 프로젝트에서는 다음 파일을 사용한다.

```text
marble_project/
├── src/
├── build.gradle
├── settings.gradle
├── gradlew
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
└── Jenkinsfile.ci
```

### 버전 확인

**실행 위치: Marble 프로젝트 디렉터리**

```bash
# Gradle Wrapper 버전 확인
bash ./gradlew --version

# Wrapper 설정 파일 확인
cat gradle/wrapper/gradle-wrapper.properties
```

### 왜 `bash ./gradlew`인가?

- `bash`: Bash로 스크립트 실행
- `./gradlew`: 현재 프로젝트의 Gradle Wrapper 실행

`gradlew`에 실행 권한이 없어도 Bash를 통해 실행할 수 있다.

---

## 8. JAR 파일명 설정

**파일: `build.gradle`**

```bash
# Gradle 설정 파일 직접 수정
nano build.gradle
```

기존 설정을 확인한 다음 필요한 경우 추가한다.

```groovy
// Spring Boot 실행형 JAR 파일명을 app.jar로 지정
tasks.named('bootJar') {
    archiveFileName = 'app.jar'
}
```

**설정 이유**

기본적으로 생성되는 JAR 이름에는 프로젝트명과 버전이 포함될 수 있다.

배포 스크립트에서 일정한 파일명을 사용하도록 `app.jar`로 고정했다.

---

# PART 4. Jenkins CI Pipeline

## 9. Jenkinsfile 기본 구조

Jenkinsfile은 Jenkins가 실행할 작업 순서를 정의하는 파일이다.

기본 구조:

```groovy
pipeline {

    // Jenkins 실행 노드 선택
    agent any

    stages {

        // 작업 단계
        stage('Build') {
            steps {

                // Linux 명령어 실행
                sh 'echo Build'
            }
        }
    }
}
```

| 문법 | 의미 |
|---|---|
| `pipeline` | 전체 Pipeline 정의 |
| `agent any` | 실행 가능한 Jenkins 노드 사용 |
| `stages` | 여러 작업 단계 묶음 |
| `stage` | 개별 작업 단위 |
| `steps` | 실제 수행할 작업 |
| `sh` | Linux Shell 명령어 실행 |

---

## 10. Poll SCM 설정

```groovy
triggers {
    // GitHub 변경 사항을 약 5분 간격으로 확인
    pollSCM('H/5 * * * *')
}
```

### 동작 원리

```text
개발자 GitHub Push
        ↓
Jenkins Poll SCM
        ↓
GitHub 변경 사항 확인
        ↓
새 Commit 감지
        ↓
Pipeline 실행
```

**Poll SCM이 필요한 이유**

GitHub Webhook을 사용하지 않아도 Jenkins가 주기적으로 Repository의 변경 사항을 확인할 수 있다.

### 특징

- `H/5 * * * *`: 약 5분 간격으로 변경 확인
- `H`: Jenkins가 검사 시점을 분산하는 기호
- 변경 사항을 감지하면 Pipeline 실행
- Push 직후 즉시 실행되는 방식은 아님

**Poll SCM은 5분마다 무조건 빌드하는 기능이 아니라, 변경 사항을 확인하는 기능이다.**

---

## 11. Checkout 단계

```groovy
stage('Checkout') {
    steps {

        // Jenkins Job에 설정된 GitHub 소스코드 가져오기
        checkout scm
    }
}
```

### `checkout scm`이란?

Jenkins Job에 설정된 Repository와 브랜치의 소스코드를 Jenkins Workspace로 가져오는 명령어다.

**왜 필요한가?**

빌드와 테스트를 실행하려면 Java 소스코드와 `build.gradle` 등이 Jenkins 작업 디렉터리에 있어야 하기 때문이다.

---

## 12. Build 단계

```groovy
stage('Build') {
    steps {

        // 기존 빌드 결과 정리 후 Java 컴파일
        sh 'bash ./gradlew clean classes --no-daemon --max-workers=1'
    }
}
```

### 실행 명령어

```bash
bash ./gradlew clean classes --no-daemon --max-workers=1
```

### 명령어 해석

| 명령어/옵션 | 역할 |
|---|---|
| `bash` | Bash로 실행 |
| `./gradlew` | Gradle Wrapper |
| `clean` | 기존 빌드 결과 삭제 |
| `classes` | Java 컴파일 및 메인 리소스 처리 |
| `--no-daemon` | 기존 Gradle Daemon 재사용 방지 |
| `--max-workers=1` | 병렬 작업 Worker 수 제한 |

### Build 단계가 필요한 이유

작성한 Java 소스코드가 정상적으로 컴파일되는지 확인하기 위해서다.

```text
Java Source (.java)
        ↓
Gradle Compile
        ↓
Java Bytecode (.class)
```

컴파일 성공은 소스코드가 빌드 가능하다는 뜻이며, 모든 기능이 정상 동작한다는 의미는 아니다.

---

## 13. Test 단계

```groovy
stage('Test') {
    steps {

        // 자동 테스트 실행
        sh 'bash ./gradlew test --no-daemon --max-workers=1'

        // 생성된 JUnit XML 결과를 Jenkins에 등록
        junit testResults: 'build/test-results/test/TEST-*.xml',
              allowEmptyResults: false
    }
}
```

### 사용한 명령어

```bash
# Gradle 테스트 실행
bash ./gradlew test --no-daemon --max-workers=1
```

### 왜 테스트를 실행하나?

코드가 컴파일에 성공하더라도 실제 기능에 문제가 있을 수 있기 때문이다.

JUnit 테스트 등을 통해 주요 기능이 기대한 대로 동작하는지 확인한다.

### 테스트 결과 경로

```text
build/reports/tests/test/
build/test-results/test/
```

### `junit` 설정

```groovy
junit testResults: 'build/test-results/test/TEST-*.xml',
      allowEmptyResults: false
```

- `testResults`: Jenkins가 읽을 테스트 결과 파일
- `allowEmptyResults: false`: 지정한 테스트 결과가 없으면 실패 처리

**주의:** 테스트 작업이 `NO-SOURCE`로 표시된다면 실제 실행한 테스트가 없는 상태일 수 있으므로 확인해야 한다.

---

## 14. Package 단계

```groovy
stage('Package') {
    steps {

        // Spring Boot 실행형 JAR 생성
        sh 'bash ./gradlew bootJar --no-daemon --max-workers=1'

        // app.jar가 존재하고 비어 있지 않은지 확인
        sh 'test -s build/libs/app.jar'
    }
}
```

### JAR 생성

```bash
bash ./gradlew bootJar --no-daemon --max-workers=1
```

### `bootJar`란?

Spring Boot 애플리케이션을 실행 가능한 JAR 파일로 생성하는 Gradle 작업이다.

Marble은 Spring Boot 내장 Tomcat 구조이므로 별도의 Tomcat 서버에 WAR를 배포하지 않고 실행형 JAR를 사용한다.

```text
Java 클래스
     +
애플리케이션 라이브러리
     +
Spring Boot 실행 구조
     ↓
   app.jar
```

### 생성 결과 확인

```bash
# JAR 생성 확인
ls -lh build/libs/

# 파일이 존재하고 크기가 0보다 큰지 검사
test -s build/libs/app.jar
```

`test -s`는 파일이 존재하고 내용이 있는지 검사한다.

파일이 없거나 비어 있다면 Jenkins가 실패를 감지할 수 있다.

---

## 15. Archive 단계

```groovy
stage('Archive') {
    steps {

        // 생성된 JAR 파일을 Jenkins 빌드 산출물로 보관
        archiveArtifacts(
            artifacts: 'build/libs/app.jar',
            fingerprint: true
        )
    }
}
```

### `archiveArtifacts`란?

Jenkins에서 빌드한 결과물을 해당 빌드 기록에 보관하는 기능이다.

### 왜 필요한가?

- 생성된 JAR 파일을 확인할 수 있음
- Jenkins 빌드별 산출물 추적 가능
- 빌드 결과물을 다운로드할 수 있음
- 배포에 사용한 파일 확인에 도움

### `fingerprint: true`

빌드 산출물의 지문 정보를 Jenkins에 기록하는 옵션이다.

아카이브된 JAR는 Jenkins의 보관 정책에 따라 나중에 삭제될 수 있다.

---

# PART 5. CI Jenkinsfile 전체 코드

## 16. 1교시 CI 통합 예제

아래 코드는 **1교시에서 학습한 CI 부분만 분리한 복습용 Jenkinsfile**이다.

실제로 배포 성공한 `Jenkinsfile.ci`에는 이 코드 뒤에 WAS01/02 CD Stage도 포함되어 있다.

**현재 운영 중인 통합 Jenkinsfile을 아래 CI 전용 코드로 덮어쓰지 않는다.**

```groovy
pipeline {

    // Jenkins 실행 노드 지정
    agent any

    options {

        // 기본 Checkout을 생략하고 명시적으로 수행
        skipDefaultCheckout(true)

        // 동시에 여러 빌드 실행 방지
        disableConcurrentBuilds()

        // Console Output에 시간 표시
        timestamps()
    }

    // GitHub 변경 사항을 약 5분 간격으로 확인
    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {

        // 1. GitHub 소스코드 가져오기
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // 2. Java 컴파일
        stage('Build') {
            steps {
                sh 'bash ./gradlew clean classes --no-daemon --max-workers=1'
            }
        }

        // 3. 테스트 실행
        stage('Test') {
            steps {
                sh 'bash ./gradlew test --no-daemon --max-workers=1'

                junit testResults: 'build/test-results/test/TEST-*.xml',
                      allowEmptyResults: false
            }
        }

        // 4. 실행형 JAR 생성
        stage('Package') {
            steps {
                sh 'bash ./gradlew bootJar --no-daemon --max-workers=1'

                // JAR 생성 확인
                sh 'test -s build/libs/app.jar'
            }
        }

        // 5. 생성된 JAR 보관
        stage('Archive') {
            steps {
                archiveArtifacts(
                    artifacts: 'build/libs/app.jar',
                    fingerprint: true
                )
            }
        }
    }

    post {

        // Pipeline 성공
        success {
            echo 'Marble CI SUCCESS'
        }

        // Pipeline 실패
        failure {
            echo 'Marble CI FAILED'
        }
    }
}
```

---

# PART 6. CI 검증

## 17. Jenkins 결과 확인

Jenkins 웹 화면:

```text
Dashboard
    ↓
Marble Pipeline Job
    ↓
Build History
    ↓
Console Output
```

확인할 단계:

```text
Checkout
    ↓
Build
    ↓
Test
    ↓
Package
    ↓
Archive
```

### 실제 실습 로그

```text
BUILD SUCCESSFUL in 13s

4 actionable tasks:
2 executed, 2 up-to-date

[Pipeline] sh
+ test -s build/libs/app.jar

[Pipeline] archiveArtifacts
Archiving artifacts
```

### 결과 해석

- `BUILD SUCCESSFUL`: Gradle 작업 성공
- `2 executed`: 실제로 실행된 Gradle 작업 2개
- `2 up-to-date`: 기존 결과가 최신이어서 재실행이 불필요했던 작업 2개
- `test -s build/libs/app.jar`: JAR 파일 존재 및 크기 검사 성공
- `Archiving artifacts`: Jenkins의 빌드 산출물 보관 작업 실행

**실제 실습에서 CI 빌드와 JAR 생성·Archive가 성공했다.**

단, Poll SCM이 자동 감지해서 실행된 것인지는 별도의 트리거 검증 결과로 확인해야 한다.

---

# PART 7. CI 장애 대응

## 18. 주요 오류와 점검 방법

| 장애 | 원인 후보 | 점검 |
|---|---|---|
| Checkout 실패 | GitHub PAT 또는 URL 오류 | Jenkins Credentials |
| 브랜치 오류 | `master` 설정 불일치 | Branch Specifier |
| Gradle 실행 실패 | Wrapper/JDK 문제 | `java -version`, `gradlew` |
| Java 컴파일 실패 | 코드 또는 라이브러리 오류 | Jenkins Console Output |
| 테스트 실패 | 테스트 로직 오류 | JUnit 결과 |
| `app.jar`가 없음 | `bootJar` 또는 파일명 설정 | `build/libs/` |
| Archive 실패 | JAR 파일 경로 불일치 | `archiveArtifacts` 설정 |

### Jenkins에서 로그를 볼 때 중요한 순서

```text
1. 실패한 Stage 찾기
        ↓
2. 처음 발생한 오류 메시지 찾기
        ↓
3. 오류가 발생한 명령어 확인
        ↓
4. 원인 수정
        ↓
5. 다시 실행하여 검증
```

마지막의 `Pipeline FAILED` 메시지만 보지 말고 처음 실패한 작업을 찾아야 한다.

---

# PART 8. 1교시 핵심 정리

## 19. 반드시 이해할 핵심 개념

| 개념 | 설명 |
|---|---|
| Jenkins | CI/CD 작업을 실행하는 자동화 서버 |
| GitHub | 소스코드 저장소 |
| PAT | Private Repository 접근 인증 |
| Poll SCM | 저장소 변경 여부 확인 |
| Jenkinsfile | 자동화 작업 순서 정의 |
| Checkout | GitHub 소스코드 가져오기 |
| Gradle Wrapper | 프로젝트에 지정된 Gradle 버전 실행 |
| `clean classes` | 기존 결과 정리 및 컴파일 |
| `test` | 자동 테스트 |
| `bootJar` | Spring Boot 실행형 JAR 생성 |
| `test -s` | JAR 파일 존재 및 크기 검사 |
| `archiveArtifacts` | Jenkins에 빌드 결과물 보관 |

## 20. 기술 선택 이유

### 왜 Jenkins를 선택했는가?

Marble 프로젝트의 빌드와 테스트를 자동화하고, 이후 SSH/SCP 기반 WAS 배포까지 하나의 Pipeline으로 연결하기 위해서다.

### 왜 Gradle Wrapper를 사용하는가?

프로젝트별 Gradle 버전을 일관되게 유지하기 위해서다.

### 왜 Poll SCM을 선택했는가?

Jenkins에 외부 Webhook 수신 주소를 별도로 구성하지 않고도 GitHub 변경 사항을 감지하도록 하기 위해서다.

### 왜 JAR로 패키징하는가?

Marble 애플리케이션이 Spring Boot 내장 Tomcat을 사용하는 실행형 JAR 구조이기 때문이다.

### 왜 Archive를 사용하는가?

빌드한 결과물을 Jenkins의 빌드 기록과 연결해 확인하고 추적하기 위해서다.

---

## 21. 1교시 복습 문제

1. CI와 CD의 차이는 무엇인가?
2. Jenkins가 GitHub Private Repository에 접근하려면 무엇이 필요한가?
3. Poll SCM과 Webhook의 차이는 무엇인가?
4. `checkout scm`이 하는 일은 무엇인가?
5. Gradle Wrapper를 사용하는 이유는 무엇인가?
6. `clean classes`는 어떤 작업을 수행하는가?
7. `test`가 필요한 이유는 무엇인가?
8. `bootJar`와 `classes`는 무엇이 다른가?
9. `test -s build/libs/app.jar`는 왜 사용하는가?
10. `archiveArtifacts`는 어떤 역할인가?

---

## 22. 최종 한 줄 요약

**Marble 프로젝트의 CI는 Jenkins가 GitHub의 변경 사항을 감지하고, Gradle을 통해 Java 컴파일·자동 테스트·실행형 JAR 생성까지 수행한 후 결과물을 Jenkins에 보관하도록 구성한 자동화 과정이다.**

### 다음 교시

**2교시 — Jenkins CD 구축**

```text
Jenkins
    ↓
SSH Credentials
    ↓
SCP (JAR 파일 전송)
    ↓
WAS01 배포
    ├── 기존 JAR 백업
    ├── 신규 JAR 교체
    ├── systemd 재시작
    └── Health Check
    ↓
WAS02 배포
    ├── 기존 JAR 백업
    ├── 신규 JAR 교체
    ├── systemd 재시작
    └── Health Check
    ↓
Jenkins Pipeline SUCCESS
```

**2교시에서는 실제로 성공한 WAS01/02 배포를 기준으로 SSH 키, Jenkins Credentials, SCP, Bash 배포 스크립트, systemd, Health Check와 실제 해결한 장애 사례까지 학습한다.**
