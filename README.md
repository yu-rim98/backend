# 🎬 Joinflix — OTT 실시간 시청 파티 플랫폼

> 친구들과 함께 OTT 콘텐츠를 실시간으로 시청하며 음성과 채팅으로 소통할 수 있는 소셜 동시시청 서비스

📅 **진행 기간** : 2026.01 ~ 2026.02  
👥 **유형** : 팀 프로젝트

---

## 📌 프로젝트 개요

Joinflix는 사용자가 파티방을 생성하여 영상 재생을 실시간으로 동기화하고,  
WebRTC 기반 음성 통화와 WebSocket 채팅으로 함께하는 경험을 제공하는 서비스입니다.

---

## 🛠️ 기술 스택

| 분류 | 기술 |
|------|------|
| **Language** | Java 21 |
| **Framework** | Spring Boot 3.x |
| **ORM** | Spring Data JPA |
| **Security** | Spring Security, JWT |
| **실시간 통신** | WebSocket, STOMP, WebRTC Signaling |
| **Database** | MySQL |
| **Cache** | Redis |
| **Storage** | AWS S3 |
| **이메일** | Spring Mail (SMTP) |
| **결제** | PortOne (아임포트) |
| **빌드** | Gradle |

---

## 📁 패키지 아키텍처

```
src/main/java/com/sesac/joinflix
├── JoinflixApplication.java
│
├── domain/                         # 도메인 레이어
│   ├── auth/                       # 인증·인가 (로그인, 회원가입, 이메일 인증, 토큰 관리)
│   ├── chat/                       # 실시간 채팅, 영상 동기화, 음성 시그널링
│   ├── file/                       # 파일 업로드 (AWS S3)
│   ├── friend/                     # 친구 요청·수락·목록 관리
│   ├── membership/                 # 멤버십 플랜
│   ├── movie/                      # 영화·콘텐츠 정보
│   ├── notification/               # 인앱 알림 (파티 초대, 예약 알림 등)
│   ├── party/                      # 파티 생성·참여·초대·예약·퇴장
│   │   ├── controller/
│   │   ├── dto/
│   │   ├── entity/
│   │   ├── event/                  # 초대 이벤트 발행·리스너 (비동기 메일)
│   │   ├── repository/
│   │   ├── scheduler/              # 예약 파티 자동 활성화
│   │   └── service/
│   ├── payment/                    # PortOne 결제 검증·내역 관리
│   ├── refund/                     # 환불 처리
│   ├── review/                     # 리뷰
│   ├── user/                       # 사용자 정보·멤버십 상태
│   └── userhistory/                # 사용자 행동 로그
│
└── global/                         # 공통 레이어
    ├── config/                     # Security, Redis, WebSocket, S3, Async 설정
    ├── exception/                  # 공통 예외 처리 (CustomException, GlobalExceptionHandler)
    ├── handler/                    # STOMP 핸들러 (JWT 인증 인터셉터)
    ├── infra/mail/                 # 이메일 발송 인프라
    ├── security/                   # JWT Provider, Filter
    └── util/                       # Cookie, Network 유틸
```

---

## ✅ 담당 기능

### 🎉 파티 생성

- 공개방 / 비공개방(비밀번호) 구분 생성
- 즉시 생성 또는 30분 단위 예약 파티 생성
- 파티 상태: `ACTIVE` / `SCHEDULED`
- 비공개 파티 생성 시 친구 초대 동시 처리

### 🚪 파티 조회 / 입장

- **No-Offset(Cursor 기반) 페이징**으로 공개 파티 목록 조회
- JPA Fetch Join + 반정규화(`currentMemberCount`)로 N+1 문제 해소
- 입장 조건 검증: 유료 멤버십 확인, 예약 시간 검증, 최대 인원(4명) 비관적 락 기반 동시성 제어
- 공개방 직접 입장 / 비공개방 초대 + 비밀번호 검증 입장

### 👋 파티 퇴장 / 방장 위임

- 일반 참여자 퇴장 처리
- 방장 퇴장 시: 비공개방에서 남은 참여자에게 방장 위임 또는 방 폭파 처리
- **Grace Period 패턴**: 새로고침으로 인한 DISCONNECT 시 2초 유예 후 퇴장 처리 (`ScheduledExecutorService` + `ConcurrentHashMap<UserId, ScheduledFuture>`)

### 📅 예약 파티

- 예약 시간 30분 단위 유효성 검증
- `PartyScheduler`가 예약 시간에 파티를 `ACTIVE` 상태로 전환 및 초대 알림 발송
- 예약 시간 이전 입장 시도 차단

### 📨 파티 초대

- 친구 관계 기반 초대 대상 검증
- `ApplicationEventPublisher` 기반 이벤트 발행
- `@TransactionalEventListener(AFTER_COMMIT)` + `@Async` 조합으로 트랜잭션 커밋 후 비동기 이메일 발송
- 인앱 알림 동시 발송

### 💬 실시간 채팅

- WebSocket + STOMP 기반 파티 채팅
- 입장·퇴장·메시지 타입 구분 (`ENTER` / `TALK` / `LEAVE`)
- DB 저장 없는 In-memory 휘발성 구조 (실시간성 우선)
- STOMP CONNECT 시점 단 1회 JWT 인증 (매 메시지 인증 오버헤드 제거)

### 🎬 영상 재생 동기화

- 재생 / 일시정지 / 탐색 이벤트를 모든 파티원에게 실시간 전파
- Redis에 `currentTime` + `updatedAt` 저장, 신규 입장자 재생 위치 계산
- 방장 전용 영상 제어 옵션 (`hostControl` 설정)

### 🔔 알림

- 파티 초대, 예약 파티 활성화 등 인앱 알림 처리

---

## 🔥 기술적 문제 해결

### 1. 이벤트 기반 비동기 처리로 API 응답 속도 99% 개선
- 13명 초대 기준 응답 시간 **35.18초 → 0.31초**
- `@Async` + `@TransactionalEventListener(AFTER_COMMIT)` 도입

### 2. Cursor 페이징 + Fetch Join으로 쿼리 95% 감소
- 파티 목록 10건 조회 기준 **21회 → 1회**
- No-Offset 페이징, Fetch Join, `currentMemberCount` 반정규화 적용

### 3. 비관적 락으로 동시성 제어 및 데드락 해결
- JMeter 부하 테스트에서 발견한 데드락(Error 1213) 해결
- `@Lock(LockModeType.PESSIMISTIC_WRITE)` 적용으로 **데드락 0%, 정합성 100%** 달성

### 4. Grace Period 패턴으로 WebSocket 세션 정합성 확보
- 새로고침 시 발생하는 DISCONNECT → 즉시 퇴장 처리 오류 해결
- 2초 유예 후 SUBSCRIBE 재유입 시 퇴장 Task 취소

---

## ⚙️ 실행 방법

```bash
# 의존성 설치 및 빌드
./gradlew build

# 애플리케이션 실행
./gradlew bootRun
```

> `application.yml` 에 DB, Redis, S3, SMTP, PortOne 설정이 필요합니다.
