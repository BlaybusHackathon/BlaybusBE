# 설스터디 (SeolStudy) - Backend

멘토-멘티 기반 학습 관리 플랫폼 백엔드입니다.  
멘티의 일일 학습 플래너 관리, 과제 제출, 멘토 피드백이 핵심 기능입니다.

---

## 기술 스택

| 분류 | 기술 |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 4.0.2 |
| Build | Gradle (Groovy DSL) |
| Security | Spring Security + JWT (Access Token, 1시간) |
| Database | H2 (개발, in-memory) / MySQL 8.x (운영, Amazon RDS) |
| ORM | Spring Data JPA |
| File Storage | 로컬 파일시스템 (MultipartFile) |
| Push Notification | Firebase Cloud Messaging (FCM) |
| API Docs | SpringDoc OpenAPI (Swagger) |
| Infra | Docker / Amazon ECR / Amazon EC2 |
| CI/CD | GitHub Actions |

---

## 패키지 구조

```
src/main/java/com/blaybus/blaybusbe/
├── global/
│   ├── config/          # SecurityConfig, SwaggerConfig
│   ├── security/        # CustomUserDetails, LoginFilter
│   │   └── jwt/         # JwtUtil, JwtFilter
│   ├── common/          # BaseTimeEntity
│   └── exception/       # GlobalExceptionHandler, CustomException, ErrorCode
└── domain/
    ├── user/            # 사용자 (로그인, 프로필 조회/수정)
    ├── mentoring/       # 멘토-멘티 매핑
    ├── plan/            # 일일 플래너 CRUD, 캘린더
    ├── task/            # 할 일 + 과제 (멘토 과제 출제, 타이머, 반복 과제)
    ├── submission/      # 과제 제출 (다중 이미지 + 메모)
    ├── feedback/        # 이미지 좌표 기반 피드백
    ├── comment/         # 피드백 댓글
    ├── zoom/            # 줌 미팅 피드백
    ├── weakness/        # 보완점/약점
    ├── material/        # 학습지 (PDF)
    ├── report/          # 주간 학습 리포트
    ├── notification/    # FCM 푸시 알림 + DB 이력
    ├── dashboard/       # 멘토/멘티 대시보드
    └── file/            # 범용 파일 업로드/다운로드
```

---

## 구현 현황

| 도메인 | 상태 | 설명 |
|---|---|---|
| Auth / User | ✅ | JWT 로그인, 회원 정보 조회/수정 |
| Plan | ✅ | 일일 플래너 CRUD, 캘린더 조회, 플래너 피드백 |
| Task | ✅ | 멘토 과제 출제 (주차+요일 기반), 멘티 할 일, 타이머 |
| Submission | ✅ | 과제 제출 (다중 이미지 + 메모), 제출 시 자동 DONE |
| Mentoring | ✅ | 멘토-멘티 매핑 조회 |
| StudyContent | ✅ | 학습지 등록/조회/삭제 |
| Weakness | ✅ | 보완점(약점) CRUD + 과목별 필터 |
| WeeklyReport | ✅ | 주간 학습 리포트 CRUD |
| Feedback | ✅ | 이미지 좌표 피드백 CRUD, 어제자/이전 피드백 모아보기 |
| Comment | ✅ | 피드백 댓글 CRUD |
| Notification | ✅ | FCM 푸시 알림 + DB 이력 (이벤트 연동) |
| Zoom | ✅ | 줌 미팅 피드백 CRUD + 목록 조회 |
| Dashboard | ✅ | 멘토/멘티 대시보드 (주간·월간 기간 필터, 과목별 진행률) |

---

## 빌드 및 실행

### 사전 요구사항

- Java 21
- Gradle

### 로컬 실행 (H2 인메모리 DB)

```bash
cd BlaybusBE

# 빌드
./gradlew clean build

# 실행
./gradlew bootRun
```

실행 후 Swagger UI: `http://localhost:8080/swagger-ui/index.html`

### Docker 실행

```bash
# 빌드 먼저 수행
./gradlew clean build -x test

# Docker 이미지 빌드
docker build -t seolstudy-be .

# 컨테이너 실행
docker run -p 8080:8080 seolstudy-be
```

---

## 환경 설정

`application-dev.yml`과 `application-prod.yml`은 보안상 GitHub Secrets에서 관리합니다.  
로컬 실행 시 `src/main/resources/` 아래에 직접 생성해야 합니다.

### 필요한 GitHub Secrets

| Secret 키 | 설명 |
|---|---|
| `APPLICATION_DEV_YML` | `application-dev.yml` (Base64 인코딩) |
| `APPLICATION_PROD_YML` | `application-prod.yml` (Base64 인코딩) |
| `FIREBASE_JSON` | Firebase 서비스 계정 JSON (Base64 인코딩) |
| `AWS_ACCESS_KEY_ID` | IAM 사용자 Access Key (ECR Push 권한) |
| `AWS_SECRET_ACCESS_KEY` | IAM 사용자 Secret Key |
| `ECR_REPOSITORY` | Amazon ECR 레포지토리 이름 |
| `EC2_HOST` | EC2 인스턴스 IP |
| `EC2_USERNAME` | EC2 SSH 사용자명 |
| `EC2_KEY` | EC2 SSH 개인 키 |

---

## CI/CD

`main` 브랜치에 Push되면 GitHub Actions가 자동으로 배포합니다.

```
Push to main
  → JDK 21 설치
  → application yml / Firebase JSON 생성
  → Gradle 빌드 (테스트 제외)
  → Docker 이미지 빌드
  → Amazon ECR Push
  → EC2 SSH 접속 후 컨테이너 재배포
```

---

## 핵심 비즈니스 규칙

- **회원가입 없음** — 계정은 DB 직접 생성
- **역할**: `MENTOR`, `MENTEE`, `WITH_DRAW`
- **멘토 1:N 멘티**, 멘티는 멘토 1명과만 매핑
- **필수 과제** (`is_mandatory=true`) — 멘토 출제 과제, 멘티 수정/삭제 불가
- **Task status**: `TODO` / `IN_PROGRESS` / `DONE`
- **과제 제출 시 자동 완료** — Submission 생성 시 Task status 자동으로 `DONE` 변경
- **타이머**: start/stop만 존재 (pause 없음), stop 시 세션 시간 누적 합산
- **DailyPlan 자동 생성** — 과제/할 일 생성 시 해당 날짜의 DailyPlan이 없으면 자동 생성
- **4가지 피드백**: 플래너 피드백 / 이미지 좌표 피드백 / 줌 미팅 피드백 / 주간 리포트 피드백

---

## 브랜치 전략

| 브랜치 | 용도 |
|---|---|
| `main` | 운영 배포 |
| `develop` | 개발 통합 |
| `feature/#숫자-도메인명` | 기능 개발 (예: `feature/#10-task`) |

---

## API 문서

로컬 실행 후 Swagger UI에서 전체 API 명세를 확인할 수 있습니다.

```
http://localhost:8080/swagger-ui/index.html
```

