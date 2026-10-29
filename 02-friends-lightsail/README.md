# friends — Lightsail 서버 구축과 운영

[← 전체 기록](../README.md) · 기간: 2026-07 ~ 2026-08

## 환경

| 항목 | 구성 |
|---|---|
| 서버 | AWS Lightsail · Ubuntu LTS |
| 실행 방식 | Docker Compose (앱 · MySQL · Redis · Caddy) |
| 외부 노출 포트 | 22, 80, 443 |

## Lightsail을 고른 이유

EC2는 확장성과 설정 자유도가 높지만, 초기 서비스에서는 비용을 예측하기 어렵다.
Lightsail은 **월 정액**이라 비용이 예측 가능하고, 서버 · 네트워크 · 스토리지 구성이 단순하다. 트래픽이 적은 초기 단계에서 운영 비용과 관리 부담을 줄이려고 Lightsail을 선택했다.

---

## 2026-07-09 · 인스턴스 생성

Lightsail에서 Ubuntu 인스턴스를 만들었다.

![인스턴스 생성](https://github.com/user-attachments/assets/fecbee9b-1272-4bb2-981a-b8af659f60ad)

![인스턴스 정보](https://github.com/user-attachments/assets/a719f4e9-9125-4510-8901-8d85c01025c5)

### 고정 IP(Static IP) 연결

Lightsail 인스턴스의 기본 공인 IP는 고정이 아니다. 인스턴스를 중지했다가 다시 시작하면 IP가 바뀔 수 있다.
API 주소가 바뀌지 않도록 **고정 IP를 만들어 인스턴스에 연결**했다.

![고정 IP 설정](https://github.com/user-attachments/assets/6c9bbac8-c4e8-4f06-a6db-f35ecb4545e2)

![고정 IP 연결 완료](https://github.com/user-attachments/assets/7f638fbe-f8a0-4782-8bca-1493ecb9a9c3)

### 방화벽 설정

외부에서 접근해야 하는 포트만 열었다.

| 포트 | 용도 |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |

MySQL(3306)과 Redis(6379)는 외부에 열지 않았다. **Docker 내부 네트워크에서만** 접근하도록 했다.

![방화벽](https://github.com/user-attachments/assets/bba73124-298a-4069-8f5f-bf3d40c6005a)

---

## 2026-07-13 · 서버 기본 구성

서버에 접속해 운영에 필요한 기본 패키지를 설치했다.

- Git, Curl
- Docker, Docker Compose

![Docker 설치 확인](https://github.com/user-attachments/assets/41257f68-a32d-4131-ae15-ea79b4aefa71)

---

## 2026-07-16 · 트러블슈팅: SSH 키(PEM) 권한 오류

`scp`로 서버에 파일을 보내다가 키 권한 오류가 났다.

![PEM 권한 오류](https://github.com/user-attachments/assets/c0c63033-6e0f-4733-9dbb-f831a04ac226)

**원인**: PEM 키 파일 권한이 너무 넓게 열려 있었다. SSH는 다른 사용자가 읽을 수 있는 키를 보안상 위험하다고 보고 사용을 거부한다.

**해결**: Windows에서 키 파일 권한을 **현재 사용자만 읽을 수 있도록** 제한했다. 이후 접속에 성공했다.

> 실제 명령에는 로컬 사용자명, 경로, 키 파일명이 들어가서 기록에서 뺐다.

![접속 성공](https://github.com/user-attachments/assets/a9d074e0-324c-442f-9836-0b898200e5c9)

---

## 2026-07-28 · 배포 방식 변경과 트러블슈팅: 서버의 GitHub 인증

처음에는 로컬에서 ZIP 파일을 만들어 서버에 올리고, 서버에서 압축을 풀어 실행했다.
매번 파일을 만들어 올리는 과정이 번거로워서 **서버가 GitHub에서 직접 소스를 받아 배포**하는 방식으로 바꿨다.

서버에서 저장소를 clone하다가 인증 오류가 났다.

![GitHub 인증 오류](https://github.com/user-attachments/assets/26a8502b-d0f1-41e2-b6f9-062daf4801d4)

**원인**: GitHub는 Git 작업에서 계정 비밀번호 인증을 더 이상 지원하지 않는다. 비공개 저장소에 접근하려면 Personal Access Token(PAT)이 필요하다.

**해결**: 필요한 권한만 준 PAT를 만들어 비밀번호 대신 사용했다.

![PAT 생성](https://github.com/user-attachments/assets/ab673bbc-404c-40ff-8f66-c05645a3ac99)

![PAT 권한](https://github.com/user-attachments/assets/b5a3d992-fd76-4f82-9c20-0e5d2da71f49)

---

## 결과

Docker Compose로 앱 · MySQL · Redis · Caddy 컨테이너를 띄웠다. 클라이언트에서 Lightsail 서버 API 호출까지 성공했다.

현재 서버 배포 흐름은 다음과 같다.

```bash
cd ~/friend
git pull
docker compose up -d --build
docker compose logs -f app
```

## 정리

- **고정 IP**: Lightsail의 공인 IP는 재시작하면 바뀔 수 있다. 서비스 주소로 쓸 IP는 처음부터 고정 IP로 잡는다.
- **최소 포트만 개방**: DB와 캐시는 외부에 열지 않고 컨테이너 내부 네트워크로만 연결한다.
- **접속 키 관리**: SSH 키와 GitHub 토큰은 최소 권한으로 만들고, 기록에는 실제 값과 경로를 남기지 않는다.
