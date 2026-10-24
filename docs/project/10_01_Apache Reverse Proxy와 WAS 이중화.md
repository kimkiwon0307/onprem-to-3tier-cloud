# 10장 1교시 — Apache Reverse Proxy와 WAS 이중화

## 1. Reverse Proxy란?

Reverse Proxy는 클라이언트의 요청을 대신 받아 내부 서버(WAS)에 전달하고, 처리 결과를 다시 클라이언트에게 반환하는 기능이다.

**Marble 프로젝트 요청 흐름**

```text
Client
   |
   v
Public ALB
   |
   +── WEB01 (Apache :80)
   |
   +── WEB02 (Apache :80)
              |
              v
     Reverse Proxy
     (marblecluster)
              |
       +------+------+
       |             |
       v             v
     WAS01         WAS02
10.0.21.126:8080  10.0.22.9:8080
       |             |
       +------+------+
              |
              v
           MySQL DB
```

- **ALB:** WEB01/WEB02로 요청 분산
- **Apache:** WAS01/WAS02로 요청 전달 및 부하 분산
- **Spring Boot:** 애플리케이션 로직 처리
- **MySQL:** 데이터 저장 및 조회

## 2. WEB과 WAS를 분리하는 이유

| 목적 | 설명 |
|---|---|
| 역할 분리 | WEB은 요청 전달, WAS는 애플리케이션 처리 |
| 보안 | WAS에 대한 외부 직접 접근 제한 |
| 부하 분산 | 여러 WAS에 요청 분산 |
| 확장성 | WEB과 WAS를 독립적으로 증설 |
| 장애 대응 | WEB/WAS 계층별 장애 분석 및 관리 |

※ Apache 없이 ALB가 WAS에 직접 요청을 전달하는 구조도 가능하다.

## 3. 실제 Apache 설정

설정 파일: `/etc/apache2/sites-available/marble.conf`

```apache
<VirtualHost *:80>
    ServerName marble.test

    ProxyRequests Off
    ProxyPreserveHost On

    <Proxy "balancer://marblecluster">
        BalancerMember "http://10.0.21.126:8080"
        BalancerMember "http://10.0.22.9:8080"

        ProxySet lbmethod=byrequests
    </Proxy>

    ProxyPass "/" "balancer://marblecluster/"
    ProxyPassReverse "/" "balancer://marblecluster/"
</VirtualHost>
```

위 코드는 실제 설정에서 Reverse Proxy 관련 핵심 부분만 발췌한 것이다.

### 주요 설정 설명

| 설정 | 역할 |
|---|---|
| `ProxyRequests Off` | Forward Proxy 비활성화 |
| `ProxyPreserveHost On` | 원본 Host 헤더 유지 |
| `BalancerMember` | WAS 서버 등록 |
| `lbmethod=byrequests` | 가중 요청 수 기반 부하 분산 |
| `ProxyPass` | 클라이언트 요청을 WAS로 전달 |
| `ProxyPassReverse` | 백엔드의 특정 리다이렉션 응답 헤더 조정 |

**핵심:** ALB가 WEB01을 선택하더라도 Apache 설정에 따라 WAS01 또는 WAS02로 요청이 전달될 수 있다.

## 4. Apache 부하 분산

Marble 프로젝트에서는 다음 설정을 사용한다.

```apache
ProxySet lbmethod=byrequests
```

- 요청 수와 설정된 가중치를 기준으로 부하를 분산한다.
- 가중치가 같다면 요청이 대체로 고르게 분산된다.
- 반드시 WAS01 → WAS02 순서로 번갈아 처리되는 것은 아니다.
- WAS가 장애 상태일 때 다른 WAS로 요청이 전달되는지는 장애 유형과 Balancer 설정에 영향을 받는다.

## 5. WEB → WAS 네트워크 통신

Apache는 WAS의 사설 IP와 TCP 8080 포트를 통해 통신한다.

| 항목 | 필요한 설정 |
|---|---|
| WEB Security Group | WAS TCP 8080 아웃바운드 허용 |
| WAS Security Group | WEB에서 TCP 8080 인바운드 허용 |
| Route Table | VPC 내부 Local 라우팅 |
| NACL | 요청·응답 통신 허용 |
| Spring Boot | 8080 포트 리스닝 |

같은 VPC 내부 사설망 통신에는 NAT Gateway가 필요하지 않다.

## 6. 실습 및 검증 명령어

### Apache 프록시 모듈 확인

```bash
sudo apache2ctl -M | grep -E 'proxy|lbmethod'
```

### 실제 Apache 설정 확인

```bash
sudo cat /etc/apache2/sites-available/marble.conf
```

### WEB에서 WAS01 통신 확인

```bash
curl -i http://10.0.21.126:8080/health
```

### WEB에서 WAS02 통신 확인

```bash
curl -i http://10.0.22.9:8080/health
```

※ `/health`가 실제로 구현되어 있어야 HTTP 200 응답을 기대할 수 있다.

### WAS 요청 분산 로그 확인

```bash
sudo tail -f /var/log/apache2/marble_backend.log
```

실제 Marble 설정에서는 다음 로그 포맷을 사용한다.

```apache
LogFormat "%t \"%r\" %>s backend=%{BALANCER_WORKER_NAME}e" marble_backend
```

이를 통해 Apache가 어느 WAS로 요청을 전달했는지 추적할 수 있다.

## 7. 핵심 요약

1. **Reverse Proxy:** 외부 요청을 내부 WAS로 대신 전달하는 기능
2. **ALB:** WEB01/WEB02 사이에서 요청 분산
3. **Apache Balancer:** WAS01/WAS02 사이에서 요청 분산
4. **BalancerMember:** 부하 분산 대상 WAS 등록
5. **byrequests:** 가중 요청 수 기반 분산 방식
6. **ProxyPass:** WEB에서 WAS로 요청 전달
7. **보안 그룹:** WEB → WAS TCP 8080 통신 허용
8. **로그:** `marble_backend.log`에서 실제 WAS 분산 결과 확인

**최종 핵심:** Marble 프로젝트는 ALB에서 WEB 서버를 선택하고, Apache Reverse Proxy에서 WAS 서버를 선택하는 2단계 부하 분산 구조이다.

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/a56f34f6-225f-4f52-93fc-29a287e645de" />
