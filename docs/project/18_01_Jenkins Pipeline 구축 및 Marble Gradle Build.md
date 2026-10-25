# 18장 1교시 — Jenkins Pipeline 구축 및 Gradle Build

## 1. 학습 목표

17장에서 구축한 Jenkins + GitHub Poll SCM 환경을 이용해 Marble Spring Boot 애플리케이션의 CI Pipeline을 구성한다.

**최종 목표: GitHub에 새로운 Commit이 발생하면 Jenkins가 자동으로 소스코드를 가져와 Gradle 컴파일을 수행하도록 구축한다.**

---

## 2. Jenkins Pipeline이란?

Jenkins Pipeline은 빌드·테스트·배포 등 CI/CD 작업을 코드로 작성하고 자동 실행하는 기능이다.

### Freestyle과 Pipeline 비교

| 구분 | Freestyle | Pipeline |
|---|---|---|
| 설정 | Jenkins 웹 화면 | Jenkinsfile |
| 작업 구성 | 단순 작업에 적합 | 여러 Stage 구성 |
| 버전 관리 | 제한적 | GitHub로 관리 가능 |
| 복잡한 CI/CD | 관리가 어려워질 수 있음 | 적합 |

**Pipeline을 사용하는 이유:** CI/CD 작업을 코드로 관리하고 각 단계를 일관된 순서로 실행하기 위해서이다.

---

## 3. Marble CI Pipeline 아키텍처

```text
GitHub Private Repository
     (master)
         |
         | Poll SCM
         v
    Jenkins Pipeline
         |
         +-- Checkout
         |     GitHub 소스 가져오기
         |
         +-- Build Environment
         |     Java / Gradle 확인
         |
         +-- Build
         |     Gradle clean classes
         |
         v
     BUILD SUCCESSFUL
```

**Poll SCM과 Pipeline의 차이**

- Poll SCM: Jenkins Job을 언제 실행할지 결정
- Pipeline: Job이 시작되면 어떤 작업을 수행할지 정의

---

## 4. Jenkins 빌드 환경 준비

### JRE와 JDK 차이

| 구분 | JRE | JDK |
|---|---|---|
| Java 실행 | 가능 | 가능 |
| Java 컴파일 | 불가능 | 가능 |
| `java` | 제공 | 제공 |
| `javac` | 미제공 | 제공 |

Jenkins 실행에는 Java Runtime을 사용할 수 있지만, Spring Boot 소스코드를 컴파일하려면 JDK가 필요하다.

### JDK 21 설치

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
```

검증:

```bash
java -version
javac -version
```

**주의:** Jenkins 실행용 Java와 Marble 빌드용 Java의 버전은 다를 수 있다. `build.gradle`에서 Java 17 Toolchain을 요구한다면 JDK 17도 설치해야 한다.

---

## 5. Gradle과 Gradle Wrapper

Gradle은 Java 프로젝트의 컴파일·테스트·패키징을 자동화하는 빌드 도구이다.

Gradle Wrapper는 프로젝트에서 지정한 Gradle 버전을 실행하도록 지원한다.

### 주요 파일

```text
marble_project/
│
├── src/
├── build.gradle
├── settings.gradle
│
├── gradlew
├── gradlew.bat
│
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
│
└── Jenkinsfile.ci
```

| 파일 | 역할 |
|---|---|
| `build.gradle` | 빌드 설정 |
| `settings.gradle` | 프로젝트 설정 |
| `gradlew` | Linux Gradle 실행 스크립트 |
| `gradle-wrapper.properties` | Gradle 배포판 버전 및 URL |
| `Jenkinsfile.ci` | Jenkins CI Pipeline 정의 |

Marble의 기존 구축 기록에서는 **Gradle Wrapper 8.10.2**를 사용했다.

### 핵심 빌드 명령어

```bash
bash ./gradlew clean classes
```

- `bash`: Bash로 Wrapper 실행
- `./gradlew`: 프로젝트의 Gradle Wrapper
- `clean`: 이전 빌드 산출물 삭제
- `classes`: 메인 Java 소스 컴파일 및 리소스 처리

**`clean classes`는 컴파일 단계이며, 실행형 JAR 생성과 자동 테스트 완료를 의미하지 않는다.**

---

## 6. Jenkinsfile 핵심 문법

| 문법 | 의미 |
|---|---|
| `pipeline` | 전체 Pipeline 정의 |
| `agent any` | 사용 가능한 실행 노드 선택 |
| `stages` | 작업 단계 묶음 |
| `stage` | 개별 작업 단계 |
| `steps` | 실제 실행할 작업 |
| `sh` | Linux Shell 명령 실행 |
| `triggers` | 자동 실행 조건 |
| `post` | 성공·실패 후 처리 |

### Marble CI Jenkinsfile

아래 내용은 18장 1교시에서 사용할 핵심 예시이다.

```groovy
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Build Environment') {
            steps {
                sh '''
                    java -version
                    javac -version
                    bash ./gradlew --version
                '''
            }
        }

        stage('Build') {
            steps {
                sh '''
                    bash ./gradlew clean classes \
                      --no-daemon \
                      --max-workers=1 \
                      -Dorg.gradle.jvmargs="-Xmx512m -XX:MaxMetaspaceSize=256m"
                '''
            }
        }
    }

    post {
        success {
            echo 'Marble CI Build SUCCESS'
        }

        failure {
            echo 'Marble CI Build FAILED'
        }
    }
}
```

### 주요 설정 해석

- `skipDefaultCheckout(true)`: 기본 Workspace Checkout을 생략하고 Checkout Stage에서 직접 수행
- `timestamps()`: 실행 로그에 시간 표시
- `disableConcurrentBuilds()`: 동일 Job의 동시 실행 방지
- `pollSCM('H/5 * * * *')`: 약 5분 간격 Git 변경 검사
- `checkout scm`: Job에 설정된 Git 저장소에서 소스 가져오기
- `--no-daemon`: Gradle 데몬을 지속 사용하지 않도록 실행
- `--max-workers=1`: Gradle 병렬 작업 수 제한
- `-Xmx512m`: Gradle JVM Heap 최대 크기 설정
- `-XX:MaxMetaspaceSize=256m`: Gradle JVM Metaspace 최대 크기 설정

---

## 7. Jenkins Pipeline Job 생성

Jenkins에서 다음과 같이 설정한다.

```text
New Item
    ↓
marble-ci
    ↓
Pipeline
    ↓
Pipeline script from SCM
    ↓
SCM: Git
```

| 항목 | 설정 |
|---|---|
| Repository URL | `https://github.com/kimkiwon0307/marble_project.git` |
| Credentials | `github-marble-read` |
| Branch | `*/master` |
| Script Path | `Jenkinsfile.ci` |

Jenkinsfile을 GitHub에서 관리하면 Pipeline 변경 이력을 Git으로 추적할 수 있다.

**`Pipeline script from SCM`은 GitHub에 저장한 Jenkinsfile을 읽어서 실행하는 방식이다.**

---

## 8. CI Pipeline 실행 및 검증

### 실행 순서

```text
GitHub Push
    |
    v
Poll SCM 변경 감지
    |
    v
Checkout
    |
    v
Build Environment
    |
    v
Gradle clean classes
    |
    v
SUCCESS
```

### 검증 방법

Jenkins에서:

`marble-ci → Build History → Console Output`

정상적인 로그 예시:

```text
> Task :compileJava
> Task :processResources
> Task :classes

BUILD SUCCESSFUL

Finished: SUCCESS
```

확인해야 할 항목:

1. GitHub의 올바른 Commit을 Checkout했는가?
2. JDK와 Gradle Wrapper가 정상 실행되는가?
3. Java 컴파일이 성공했는가?
4. 새로운 Git Commit으로 Job이 자동 실행되는가?

---

## 9. 주요 장애 대응

| 장애 | 원인 후보 | 확인 방법 |
|---|---|---|
| Jenkinsfile을 찾지 못함 | Script Path·브랜치 오류 | GitHub 파일 경로 |
| `javac: command not found` | JDK 미설치 | `javac -version` |
| `gradlew` 없음 | Wrapper 누락 | `ls -la` |
| Permission denied | 실행 권한 문제 | `bash ./gradlew` |
| Java Toolchain 오류 | 필요한 JDK 버전 없음 | `build.gradle` |
| 빌드 중 메모리 부족 | EC2 자원 부족 | `free -h`, 시스템 로그 |
| 자동 실행 안 됨 | Poll SCM 설정 오류 | Trigger, Polling Log |

### 장애 분석 순서

```text
1. Jenkinsfile 로드
        ↓
2. Git Checkout
        ↓
3. Java / Gradle 확인
        ↓
4. Gradle 컴파일
        ↓
5. Console Output 분석
```

---

## 10. 최종 핵심 정리

1. Jenkins Pipeline은 CI/CD 작업을 코드로 정의하는 기능이다.
2. Jenkinsfile은 Pipeline의 실행 절차를 기록하는 파일이다.
3. Freestyle과 달리 Pipeline은 복잡한 작업을 Stage로 구분해 관리할 수 있다.
4. Gradle은 Java 프로젝트의 컴파일·테스트·패키징을 수행한다.
5. Gradle Wrapper를 사용하면 프로젝트에서 지정한 Gradle 버전을 실행할 수 있다.
6. `clean classes`는 메인 Java 코드 컴파일을 수행한다.
7. `checkout scm`은 Jenkins Job의 SCM 설정을 사용해 소스를 가져온다.
8. Poll SCM은 Git 변경을 감지하고 Pipeline 실행을 시작한다.
9. 빌드 성공과 테스트·배포 성공은 서로 다른 검증 항목이다.
10. Jenkinsfile을 GitHub에 저장하면 CI/CD 설정을 버전 관리할 수 있다.

### 최종 결론

**Marble 프로젝트는 Jenkinsfile로 Git Checkout과 Gradle 컴파일 절차를 정의하고, Poll SCM으로 새로운 GitHub Commit을 감지하여 CI Pipeline을 자동 실행하도록 구성한다.**

다음 18장 2교시에서는 **Gradle Test → Spring Boot JAR 생성 → Archive** 단계를 추가한다.

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/d4f6278d-72e0-4f65-bcb9-0dc6d7947060" />
