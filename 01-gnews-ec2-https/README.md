# gnews — EC2 서비스에 도메인 · HTTPS 적용 (Route 53 + ACM + ALB)

[← 전체 기록](../README.md) · 기간: 2026-01 ~ 2026-02

## 환경

| 항목 | 구성 |
|---|---|
| 서버 | EC2 (Ubuntu) |
| 웹 서버 | Nginx (:80) → 애플리케이션 (:8080) |
| DB · 캐시 | MySQL, Redis (EC2 안의 Docker 컨테이너) |

## 목표

이미 EC2에 올라가 있던 서비스에 **도메인과 HTTPS를 적용**하고, AWS 관리형 서비스로 **실서비스 구조**를 만든다.

```
Internet
  ↓
Route 53 (도메인)
  ↓
Application Load Balancer (HTTPS 종료)
  ↓
EC2 (Nginx :80)
  ↓
애플리케이션 (:8080)
```

- **인증서와 HTTPS 처리는 ALB가 맡는다.** EC2에는 인증서를 두지 않는다.
- **EC2는 ALB를 통해서만 HTTP를 받는다.** 보안 그룹으로 직접 접근을 막는다.

---

## 구축 과정

### 1. Route 53 — 호스팅 영역 생성

도메인 관리를 Route 53에서 하기 위해 호스팅 영역(Hosted Zone)을 만들었다.

<img width="1289" alt="Route 53 호스팅 영역" src="https://github.com/user-attachments/assets/2fa7c2a2-8bf1-469b-b733-c8fba0e89385" />

### 2. 도메인 구매처에 네임서버(NS) 등록

Route 53이 만들어 준 **NS 레코드 4개**를 도메인 구매처에 그대로 등록했다.

<img width="1875" alt="네임서버 등록" src="https://github.com/user-attachments/assets/e4cc4519-6f27-4a11-9a2a-80f2bd36ca81" />

> 이 단계를 빠뜨리면 이후 ACM 인증과 ALB 연결이 동작하지 않는다.

### 3. ACM 인증서 발급

HTTPS용 SSL 인증서를 발급했다. **인증서는 ALB와 같은 리전(us-east-1)에서 만들어야 ALB에 연결할 수 있다.**

<img width="1902" alt="ACM 인증서" src="https://github.com/user-attachments/assets/989a20ec-1084-4492-96bb-e28bdfbd3063" />

### 4. Route 53에 DNS 검증 레코드 생성

ACM 화면의 **Create records in Route 53**으로 검증용 CNAME을 만들었다. DNS 검증이 끝나면 인증서가 `Issued` 상태가 된다.

<img width="1883" alt="DNS 검증" src="https://github.com/user-attachments/assets/2f0d84a9-13e2-44e2-bb88-af72d2f99db1" />

### 5. 대상 그룹(Target Group) 생성

ALB가 트래픽을 보낼 대상이다. EC2 인스턴스를 `HTTP:80`으로 등록했다.

<img width="1885" alt="대상 그룹" src="https://github.com/user-attachments/assets/34aa3818-9368-481d-8079-0f85e799ea6e" />

### 6. EC2 보안 그룹 생성

| 포트 | 허용 대상 |
|---|---|
| SSH (22) | 내 IP만 |
| HTTP (80) | **ALB 보안 그룹만** |

<img width="1323" alt="EC2 보안 그룹" src="https://github.com/user-attachments/assets/841252a5-1a5b-4edd-9448-5b255d9ad753" />

### 7. EC2에 보안 그룹 연결

<img width="1499" alt="보안 그룹 연결" src="https://github.com/user-attachments/assets/021754b1-8340-4266-b487-12a367bccb7e" />

### 8. ALB 보안 그룹 생성

외부 트래픽을 받는 용도로 `HTTP(80)`, `HTTPS(443)`를 `0.0.0.0/0`에 열었다.

<img width="1416" alt="ALB 보안 그룹" src="https://github.com/user-attachments/assets/ce746711-fdc0-4377-bc78-e6f3370909ea" />

### 9. Application Load Balancer 생성

- 인터넷 연결(Internet-facing) ALB
- 리스너 구성
  - `HTTP 80` → `HTTPS 443`로 **리다이렉트**
  - `HTTPS 443` → 대상 그룹으로 **전달**
- 3단계에서 발급한 ACM 인증서 연결

<img width="1501" alt="ALB 1" src="https://github.com/user-attachments/assets/4171781a-a3c7-4033-880b-b400f2adab96" />
<img width="1534" alt="ALB 2" src="https://github.com/user-attachments/assets/ce33e16d-f83f-4f1b-97fe-ef15d37b916a" />
<img width="1515" alt="ALB 3" src="https://github.com/user-attachments/assets/8f09b6fb-71fc-451b-b5d4-0ec085ebe9a1" />
<img width="1509" alt="ALB 4" src="https://github.com/user-attachments/assets/f41052b3-186d-4723-8f37-21622d371172" />

### 대상 그룹 상태 확인

처음에는 대상이 **Unhealthy**였다. EC2 보안 그룹을 인스턴스에 연결해 ALB에서 오는 80 포트를 허용한 뒤 **Healthy**로 바뀌었다.

### 10. Route 53 → ALB 연결

루트 도메인과 `www` 모두 **A / ALIAS 레코드**로 ALB에 연결했다.

<img width="1231" alt="ALIAS 레코드" src="https://github.com/user-attachments/assets/1a24cca6-3c0a-404d-aede-8afff8204ef9" />

### 11. HTTPS 적용 확인

- 도메인으로 접속하면 HTTPS가 적용된다.
- HTTP로 접속하면 HTTPS로 리다이렉트된다.

<img width="1782" alt="HTTPS 성공" src="https://github.com/user-attachments/assets/a39aad9e-63d3-4474-8c38-5231ec2f0785" />

---

## 트러블슈팅

| 문제 | 원인 | 해결 |
|---|---|---|
| 대상 그룹이 Unhealthy | EC2가 ALB의 헬스체크(80)를 받지 못함 | EC2 보안 그룹에서 80 포트를 ALB 보안 그룹에만 허용하고 인스턴스에 연결 |

## 정리

- Route 53 · ACM · ALB · EC2를 연결한 **표준 HTTPS 구조**를 직접 구성했다.
- **보안 그룹을 체인으로 연결**했다(ALB SG → EC2 SG). 인스턴스는 외부에 직접 노출되지 않는다.
- HTTPS 종료를 ALB에 맡기면 인증서 갱신을 ACM이 자동으로 처리한다.
