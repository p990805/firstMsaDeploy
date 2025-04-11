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

<details>
    <summary>2025-03-19</summary>

# 1. 설정 초기화

```docker
# 시스템 패키지 업데이트
sudo apt update
sudo apt upgrade -y
```

# 2. 기본 유틸리티 설정

```docker
# 기본 유틸리티 설치
sudo apt install -y git curl wget vim htop
```

# 3. UFW 허용번호 확인

```powershell

# UFW 상태 확인
sudo ufw status numbered

#대충이런식으로나옴
ubuntu@ip-172-26-14-197:~$ sudo ufw status numbered
Status: active
     To                         Action      From
     --                         ------      ----
[ 1] 22                         ALLOW IN    Anywhere
[ 2] 80                         ALLOW IN    Anywhere
[ 3] 44                         ALLOW IN    Anywhere
[ 4] 8989                       ALLOW IN    Anywhere
[ 5] 22 (v6)                    ALLOW IN    Anywhere (v6)
[ 6] 80 (v6)                    ALLOW IN    Anywhere (v6)
[ 7] 44 (v6)                    ALLOW IN    Anywhere (v6)
[ 8] 8989 (v6)                  ALLOW IN    Anywhere (v6)
ubuntu@ip-172-26-14-197:~$ ls -la ~/ | grep .ssh
drwx------ 2 ubuntu ubuntu 4096 Mar 18 05:48 .ssh
drwx------ 2 ubuntu ubuntu 4096 Mar 18 05:48 .ssh_bak
ubuntu@ip-172-26-14-197:~$
```

# 4. Docker 설치

```powershell
# 필요한 패키지 설치
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Docker 공식 GPG 키 추가
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Docker 저장소 추가
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# 패키지 정보 업데이트
sudo apt update

# Docker 설치
sudo apt install -y docker-ce

# Docker 서비스 상태 확인
sudo systemctl status docker

# 현재 사용자를 docker 그룹에 추가 (sudo 없이 docker 명령어 실행 가능)
sudo usermod -aG docker $USER
```

# 5. Docker Compose 설치

```powershell
# Docker Compose 최신 버전 설치
sudo curl -L "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 실행 권한 부여
sudo chmod +x /usr/local/bin/docker-compose

# 버전 확인
docker-compose --version

# 로그아웃 후 다시 로그인하거나, 다음 명령어로 현재 세션에 그룹 변경 적용
newgrp docker
```

# 6. 도커와 도커 컴포즈 설치 확인

```powershell
docker-compose --version
# Hello World 컨테이너 실행 테스트
docker run hello-world
```

# 7. 각 프로젝트 마다 도커파일 생성

- 아래와 방법으로 할거면 spring은 mvn clean package 로 빌드 먼저해야함

```powershell
#스프링 예시
FROM openjdk:21
WORKDIR /app
COPY target/*.jar config-service.jar
EXPOSE 8888
ENTRYPOINT ["java", "-jar", "config-service.jar"]
```

```powershell
# 프론트 예시
FROM node:18-alpine AS build
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY ./nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

# 8. 프론트 프로젝트 dockerfile이랑 같은 위치에 Nginx.conf 파일 생성

```powershell
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }

    # 백엔드 API 요청 프록시
    location /api {
        proxy_pass http://gateway-service:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

# 9. 프로젝트 제일 루트 디렉토리에 Docker compose 파일 구성

- docker-compose.yml

```powershell
version: '3'

services:
  config-service:
    image: imjuchan/config-service:latest
    ports:
      - "8888:8888"
    networks:
      - backend-network

  eureka-service:
    image: imjuchan/eureka-service:latest
    ports:
      - "8761:8761"
    depends_on:
      - config-service
    networks:
      - backend-network
    environment:
      - SPRING_CLOUD_CONFIG_URI=http://config-service:8888

  gateway-service:
    image: imjuchan/gateway-service:latest
    ports:
      - "8080:8080"
    depends_on:
      - eureka-service
    networks:
      - backend-network
      - frontend-network
    environment:
      - SPRING_CONFIG_IMPORT=configserver:http://config-service:8888
      - SPRING_CLOUD_CONFIG_URI=http://config-service:8888
      - SPRING_CLOUD_CONFIG_FAIL_FAST=false
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-service:8761/eureka/

  frontend:
    image: imjuchan/frontend-service:latest
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - gateway-service
    networks:
      - frontend-network
    volumes:
      -  /etc/letsencrypt:/etc/letsencrypt:ro
      - ./nginx-ssl.conf:/etc/nginx/conf.d/default.conf
  # 나머지 서비스들 추가 (user-service, diary-service, lucky-service 등)

networks:
  backend-network:
    driver: bridge
  frontend-network:
    driver: bridge

```

# 10. ec2 들어와서 테스트

```powershell

docker-compose up -d
docker-compose ps
docker-compose logs -f [서비스명]

docker-compose logs -f config-service
docker-compose logs -f eureka-service
docker-compose logs -f gateway-service
docker-compose logs -f frontend
```

# 11. ec2에서 docker-compose.yml 생성

```powershell

# docker-compose.yml 파일 생성 및 편집
nano docker-compose.yml
```

# 12. ec2에서 docker-compose.yml 파일 작성

- 위에서 작성했떤 docker-compose.yml 파일 그대로 가져와서 써도 됌
- ctrl+o 저장
- 엔터
- ctrl+x 나가기

# 13. ec2에서 확인

```powershell
docker compose up -d
docker compose ps
docker compose logs -f
```

- 여기까지 하면 http 배포는 끝

# 14. https 설정

- Let’s Encrypt 로 무료 SSL 인증서 발급

```powershell
sudo apt-get update
sudo apt-get install certbot python3-certbot-nginx
sudo certbot --nginx -d j12c202.p.ssafy.io
```

# 15. nginx 설정

- docker-compose.yml 파일과 같은 위치

```powershell
# EC2 서버에서 nginx-ssl.conf 파일 생성
cat > nginx-ssl.conf << EOF
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

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://gateway-service:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```

- 사실 이거 끝나고 docker-compose.yml 에 추가해야할 내용있는데 귀찮아서 위에 올릴때 한꺼번에 올림

```powershell
frontend:
  image: imjuchan/frontend-service:latest
  ports:
    - "80:80"
    - "443:443"  # HTTPS 포트 추가
  depends_on:
    - gateway-service
  networks:
    - frontend-network
  volumes:
    - /etc/letsencrypt:/etc/letsencrypt:ro  # 인증서 디렉토리 마운트
    - ./nginx-ssl.conf:/etc/nginx/conf.d/default.conf  # Nginx 설정 파일 마운트
```

# 16. 도커 적용

```powershell
# 기존 컨테이너 중지
docker compose down

# 새 설정으로 컨테이너 시작
docker compose up -d
```

# 17.certbot 설치

```powershell
sudo apt-get update
sudo apt-get install certbot

docker compose down

certbot --version
sudo certbot certonly --standalone -d j12c202.p.ssafy.io

docker compose up -d
```

- 인증서 발급 과정에서 일시적으로 80번 포트를 사용함으로 잠시 도커를 꺼놈
- 그리고 다시 키기

# 18. ec2에 젠킨스 설치

```powershell
# Jenkins 저장소 키 가져오기
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Jenkins 저장소 추가
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# 패키지 목록 업데이트
sudo apt-get update

# Jenkins 설치 전 Java 설치 (Jenkins는 Java 기반)
sudo apt-get install -y openjdk-17-jdk

# Jenkins 설치
sudo apt-get install -y jenkins

# Jenkins 서비스 시작
sudo systemctl start jenkins

# Jenkins 서비스 상태 확인
sudo systemctl status jenkins
```

# 19. 젠킨스 설정

```powershell
관리자 비밀번호 확인해야함
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

- 이때 젠킨스는 8080 쓰기 때문에 아까 ufw에서 8080 허용되고 있는지 확인

```powershell
sudo ufw status numbered
sudo ufw allow 8080
sudo ufw status numbered

```

# 20. 젠킨스 가입

- 가입되면 http://j12c202.p.ssafy.io:8080에 들어가서 가입하기

# 21. 플러그인 설치

- 관리 들어가서 플러그인 관리 들어가기
-

GitLab
GitLab API
Docker
Docer pipeline

이거 선택 하기

- 설치 후 재시작
</details>

<details>
  <summary>2025-04-11</summary>

# 1. 젠킨스?? 빌드 트리거 설정?

- 젠킨스 API_TOKEN

```docker
curl -X POST "http://j12c202.p.ssafy.io:8080/job/j12c202-pipeline/build" \
     --user USERNAME:YOUR_API_TOKEN
```

```docker

curl -X POST "http://j12c202.p.ssafy.io:8080/job/j12c202-pipeline/build" --user
```

- 201이나 302 뜨면 잘된거

# 2. Docker-compose-monitoring.yml

```docker
version: '3'
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    restart: unless-stopped
    networks:
      - monitoring-network
      - backend-network
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: unless-stopped
    networks:
      - monitoring-network
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8082:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    restart: unless-stopped
    networks:
      - monitoring-network
  loki:
    image: grafana/loki:2.8.0
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yaml
      - ./loki/tmp:/tmp/loki
      - ./loki/wal:/wal
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped
    networks:
      - monitoring-network
  promtail:
    image: grafana/promtail:2.8.0
    container_name: promtail
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/config.yml
      - /var/log:/var/log
    command: -config.file=/etc/promtail/config.yml
    restart: unless-stopped
    networks:
      - monitoring-network
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123  # 실제 환경에서는 보안을 위해 변경하세요
      - GF_USERS_ALLOW_SIGN_UP=false
    restart: unless-stopped
    networks:
      - monitoring-network
      - backend-network  # 기존 백엔드 네트워크와 연결
      - frontend-network  # 기존 프론트엔드 네트워크와 연결
volumes:
  prometheus_data:
  grafana_data:
networks:
  monitoring-network:
    driver: bridge
  backend-network:
    external: true
    name: ubuntu_backend-network  # 실제 Docker 네트워크 이름으로 수정
  frontend-network:
    external: true
    name: ubuntu_frontend-network  # 실제 Docker 네트워크 이름으로 수정

```

# 3. /home/ubuntu/ 루트로 모니터링 폴더 만들기

```docker
mkdir -p /home/ubuntu/prometheus
mkdir -p /home/ubuntu/loki
mkdir -p /home/ubuntu/promtail
mkdir -p /home/ubuntu/grafana/provisioning/datasources
mkdir -p /home/ubuntu/grafana/provisioning/dashboards
```

# 4. prometheus 경로에서 prometheus.yml 생성

```docker
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'spring-boot-apps'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 5s
    eureka_sd_configs:
      - server: http://eureka-service:8761/eureka
    relabel_configs:
      - source_labels: [__meta_eureka_app_name]
        action: keep
        regex: 'CONFIG-SERVICE|EUREKA-SERVICE|GATEWAY-SERVICE'
      - source_labels: [__meta_eureka_app_instance_id]
        target_label: instance_id

```

# 5. loki경로에서 loki-config.yml 생성

```docker
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 5m
  wal:
    enabled: true
    dir: "/wal"

schema_config:
  configs:
    - from: 2020-05-15
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /tmp/loki/index
    cache_location: /tmp/loki/cache
    shared_store: filesystem
  filesystem:
    directory: /tmp/loki/chunks

compactor:
  working_directory: /tmp/loki/compactor

```

# 6. Promtail 경로에서 promtail-config.yml 생성

```docker
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log
  - job_name: containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: containerlogs
          __path__: /var/lib/docker/containers/*/*log

```

# 7. Grafana 데이터 소스 설정

- ./grafana/provisioning/datasources/datasources.yml

```docker
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    version: 1
    editable: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    version: 1
    editable: true

```

# 8. Grafana 대시보드 설정

```docker
apiVersion: 1

providers:
  - name: 'Default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    editable: true
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards

```

# 9. 파일 권한 주기

```docker
chmod -R 755 /home/ubuntu/prometheus /home/ubuntu/loki /home/ubuntu/promtail /home/ubuntu/grafana
```

# 10. 백엔드 application.properties에서 추가 설정

- config-service에서 몰려있는 곳에 추가 근데 나는 이렇게 추가함

```docker
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
  endpoint:
    prometheus:
      enabled: true
    health:
      show-details: always
```

```docker
management:
  endpoints:
    web:
      exposure:
        include: ${MANAGEMENT_ENDPOINTS_INCLUDE}
  metrics:
    export:
      prometheus:
        enabled: ${PROMETHEUS_ENABLED}
  endpoint:
    prometheus:
      enabled: ${PROMETHEUS_ENDPOINT_ENABLED:true}
    health:
      show-details: ${HEALTH_DETAILS}

```

# 11. UFW 방화벽 설정해주기

```docker
# Prometheus UI 접근 허용
sudo ufw allow 9090/tcp

# Node Exporter 접근 허용
sudo ufw allow 9100/tcp

# cAdvisor 접근 허용
sudo ufw allow 8082/tcp

# Loki 접근 허용
sudo ufw allow 3100/tcp

# Grafana UI 접근 허용
sudo ufw allow 3000/tcp
```

# 12. pom.xml 에 프로메테우스 의존성 추가

- 모든 서비스 다 넣긴했음

```docker
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

# 13. 모니터링 스택 실행하기

```docker
docker-compose -f docker-compose-monitoring.yml up -d
docker ps | grep 'prometheus\|grafana\|loki\|promtail\|cadvisor\|node-exporter'

```

**Loki가 오류가 난다면?**

```docker
# Loki 임시 디렉토리 생성
mkdir -p /home/ubuntu/loki/tmp/loki/index
mkdir -p /home/ubuntu/loki/tmp/loki/cache
mkdir -p /home/ubuntu/loki/tmp/loki/chunks
mkdir -p /home/ubuntu/loki/tmp/loki/compactor

# 권한 설정
chmod -R 777 /home/ubuntu/loki/tmp

# WAL 디렉토리 생성
mkdir -p /home/ubuntu/loki/wal

# 모든 사용자에게 쓰기 권한 부여
chmod -R 777 /home/ubuntu/loki/wal

# 디렉토리 권한 확인
ls -la /home/ubuntu/loki/tmp

# 권한 수정
sudo chmod -R 777 /home/ubuntu/loki/tmp

sudo chown -R loki:loki /tmp/loki
sudo chmod -R 755 /tmp/loki
```

</details>
