# 🚀 On-Premise to 3-Tier Cloud Migration Project

> **레거시 온프레미스 단일 서버 구축부터 고가용성 3-Tier 클라우드 아키텍처 전환까지의 End-to-End 프로젝트**

---

## 📌 프로젝트 개요
본 프로젝트는 기존 온프레미스 환경에서 단일 서버를 직접 구축 및 운영해 본 후, 이를 기반으로 현대적인 클라우드 환경의 3-Tier(Web-WAS-DB) 아키텍처로 안전하게 전환(마이그레이션)하는 과정을 기록합니다.

---

## 🗺️ 전체 로드맵 (Architecture Roadmap)
* **[Phase 1]** 온프레미스 환경 기반 단일 서버 구축 👈 *현재 진행 단계*
* **[Phase 2]** 인프라 분석 및 3-Tier 아키텍처 설계
* **[Phase 3]** 클라우드 마이그레이션 및 고가용성(HA) 환경 고도화

---

## 🏢 [Phase 1] 온프레미스 단일 서버 구축

### 📝 프로젝트 목적
* 기존 단일 서버 구조를 기반으로 Linux 서버 운영 체제 및 네트워크 기본 환경을 이해하고 구축합니다.

### 🛠️ 사용 기술 및 환경
가상화 툴: VirtualBox 또는 VMware (무료)

OS: Ubuntu Server 또는 Rocky Linux (가상 머신 내에 설치)

Web Server: Apache (80 포트)

WAS: Tomcat (스프링 부트 구동, 8080 포트)

DB: MySQL (3306 포트)

1. 필수 빌드 도구 및 패키지 설치 : 먼저 소스 코드를 컴파일하기 위해 필요한 기본적인 빌드 도구(gcc, make 등)을 설치한다.
   1.1 sudo apt update
   1.2 sudo apt install -y build-essential libssl-dev grep libpcre3-dev libexpat1-dev
       
2. 소스 코드 다운로드 및 압축 해제 : 아파치와 필수 라이브러리들을 /usr/local/src 디렉토리에 다운로드하고 압축을 푼다.
   2.1 작업 디렉토리 이동 : cd /usr/local/src
   2.2 아파치 및 라이브러리 다운로드
     2.2.1 : sudo wget https://dlcdn.apache.org/httpd/httpd-2.4.67.tar.gz
     2.2.2 : sudo wget https://dlcdn.apache.org/apr/apr-1.7.6.tar.gz
     2.2.3 : sudo wget https://dlcdn.apache.org/apr/apr-util-1.6.3.tar.gz
   2.3 압축 해제
     2.3.1 : sudo tar -xvf httpd-2.4.67.tar.gz
     2.3.2 : sudo tar -xvf apr-1.7.6.tar.gz
     2.3.3 : sudo tar -xvf apr-util-1.6.3.tar.gz
   2.4 APR 및 APR-Util을 아파치 내부 srclib 안으로 이동
     2.4.1 sudo mv apr-1.7.5 httpd-2.4.67/srclib/apr
     2.4.2 sudo mv apr-util-1.6.3 httpd-2.4.67/srclib/apr-util
   2.5 환경 설정 (Configure) 및 빌드/인스톨 : 아파치 소스 디렉토리로 이동하여 컴파일 옵션을 지정하고 최종 설치를 진행한다.
     2.5.1 : cd /usr/local/src/httpd-2.4.67
     2.5.2 : 꼬임 방지를 위해 이전 빌드 내역 초기화 : sudo make clean
     2.5.3 : 환경 설정 수행 : sudo ./configure --prefix=/usr/local/apache2 --with-included-apr --with-pcre --enable-modules=most --enable-mods-shared=all --enable-ssl
     2.5.4 : 컴파일 및 설치 진행 : sudo make, sudo make install
   2.6 아파치 설정 수정 및 실행 : 서버 식별 경고를 방지하기 위해 ServerName 설정을 추가한 후 아파치를 구동한다.
     2.6.1 설정 파일 수정 (ServerName 주석 해제 또는 추가) : # sudo nano /usr/local/apache2/conf/httpd.conf 진입 후 ServerName localhost:80 수정
     2.6.2 아파치 서버 시작 : sudo /usr/local/apache2/bin/apachectl start
     2.6.3 구동 확인 : ps -ef | grep httpd

  3. Troubleshooting 정리
     이슈1 : configure: error: Did not find working script at pcre-config
     원인 : 아파치를 컴파일할 때 필요한 정규표현식 라이브러리인 PCRE가 시스템에 없거나, 수동 설치된 경로를 아파치 빌드 스크립트가 인식하지 못함
     해결방법 : 수동 컴파일 대신 우분투 패키지 관리자(APT)를 통해 안정적인 개발용 PCRE 패키지를 설치한 후 설정을 다시 함
     sudo apt install -y libpcre3-dev
# 이후 아파치 configure 옵션에서 --with-pcre 경로 지정을 단순화하여 시스템 기본 경로를 타도록 유도

    이슈2 : xml/apr_xml.c:35:10: fatal error: expat.h: No such file or directory
    원인: 아파치 부속 라이브러리인 apr-util 컴파일 단계에서 XML 파싱을 담당하는 Expat 라이브러리의 헤더 파일(expat.h)을 찾지 못해 빌드가 중단되었습니다.
    해결 방법: 우분투 공식 저장소에서 Expat 개발용 패키지를 추가로 설치하고, 기존 빌드 실패 찌꺼기를 지운 뒤 재컴파일을 수행했습니다.
    sudo apt install -y libexpat1-dev
sudo make clean
sudo ./configure ...
sudo make && sudo make install

    이슈3 : Notification: AH00558: httpd: Could not reliably determine the server's fully qualified domain name...
    원인: 아파치 구동 시 명시적인 도메인 네임(ServerName) 설정이 부재하여 시스템이 임시로 루프백 IP(127.0.1.1)를 할당하며 보낸 단순 경고(Warning) 메시지입니다. (서버 동작에는 지장 없음)

해결 방법: /usr/local/apache2/conf/httpd.conf 파일을 열어 주석 처리되어 있던 ServerName 부분을 찾아 ServerName localhost:80으로 명시해 준 뒤 서비스를 재시작(graceful)하여 해결했습니다.


sudo /usr/local/apache2/bin/apachectl configtest   # Syntax OK 확인
sudo /usr/local/apache2/bin/apachectl restart

