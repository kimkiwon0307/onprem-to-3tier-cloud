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

