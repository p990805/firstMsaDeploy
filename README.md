# firstMsaDeploy

<details>
    <summary>2025-03-18</summary>

    # MSA 프로젝트 Docker 배포 및 HTTPS 설정 가이드

## 개요

이 문서는 마이크로서비스 아키텍처(MSA) 프로젝트를 Docker를 사용하여 배포하고 HTTPS를
설정하는 과정을 설명합니다. 프론트엔드와 여러 백엔드 서비스를 Docker 컨테이너로 배포하고,
Let's Encrypt 인증서를 사용하여 안전한 HTTPS 연결을 구성했습니다.

## 배포 환경

- AWS EC2 인스턴스
- Docker & Docker Compose
- SSL/TLS: Let's Encrypt 인증서
- 도메인: j12c202.p.ssafy.io

## 프로젝트 구조

```
project-root/
│
├── docker-compose.yml
├── nginx-ssl.conf
│
├── BE/
│   ├── config-service/
│   ├── eureka-service/
│   ├── gateway-service/
│   ├── user-service/
│   ├── diary-service/
│   └── lucky-service/
│
└── FE/
    ├── src/
    └── ...
```

## 배포 단계

### 1. Docker 컴포즈 구성

`docker-compose.yml` 파일을 통해 다음 서비스를 컨테이너화하여 배포:

- Config Service: 8888 포트 (마이크로서비스 설정 관리)
- Eureka Service: 8761 포트 (서비스 디스커버리)
- Gateway Service: 8080 포트 (API 게이트웨이)
- Frontend Service: 80 포트, 443 포트 (웹 인터페이스)

### 2. SSL 인증서 발급

Let's Encrypt의 무료 SSL 인증서를 발급받아 안전한 HTTPS 통신 지원:

```bash
# Certbot 설치
sudo apt-get update
sudo apt-get install certbot

# SSL 인증서 발급
sudo certbot certonly --standalone -d j12c202.p.ssafy.io
```

### 3. Nginx HTTPS 설정

HTTPS 리다이렉션 및 SSL 인증서 적용을 위한 Nginx 설정:

```nginx
# nginx-ssl.conf
server {
    listen 80;
    server_name j12c202.p.ssafy.io;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name j12c202.p.ssafy.io;

    ssl_certificate /etc/letsencrypt/live/j12c202.p.ssafy.io/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/j12c202.p.ssafy.io/privkey.pem;

    # 보안 설정
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';

    # 정적 파일 제공
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }

    # API 프록시
    location /api {
        proxy_pass http://gateway-service:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 4. Docker Compose 설정에 SSL 적용

프론트엔드 서비스에 인증서 및 Nginx 설정 마운트:

```yaml
frontend:
  image: imjuchan/frontend-service:latest
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - /etc/letsencrypt:/etc/letsencrypt:ro
    - ./nginx-ssl.conf:/etc/nginx/conf.d/default.conf
  networks:
    - frontend-network
```

## 서비스 관리 명령어

서비스 시작:

```bash
docker compose up -d
```

서비스 중지:

```bash
docker compose down
```

로그 확인:

```bash
docker compose logs -f [서비스명]
```

## 현재 상태 및 접근 방법

- 웹 인터페이스: https://j12c202.p.ssafy.io
- HTTP 요청은 자동으로 HTTPS로 리다이렉트됩니다
- 서비스 디스커버리(Eureka): http://j12c202.p.ssafy.io:8761

## 향후 계획

- Gateway Service 연결 문제 해결
- 비즈니스 마이크로서비스 추가 통합
- CI/CD 파이프라인 구축 (Jenkins)
- 모니터링 시스템 구축 (Prometheus/Grafana)
- 로그 집중화 (ELK Stack)

## 참고 사항

- SSL 인증서는 90일마다 갱신이 필요합니다
- certbot을 통한 자동 갱신이 설정되어 있습니다
</details>
