# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

# 프로젝트 컨텍스트 — 바라다 (BikeRideDiary)

> 작업 메모리(진행 상황·피드백·전략)는 git으로 동기화되는 별도 파일에 있습니다. 세션 시작 시 함께 읽으세요.
> @claude-memory.md

## 나에 대해

- 직업: 프리랜서 백엔드 개발자
- 주력 스택: Java, Spring Boot
- 프론트엔드 경험: HTML, CSS, JS, jQuery, Thymeleaf 수준
- 취미: 바이크 라이딩

---

## 프로젝트 개요

**서비스명 (한글)**: 바라다
**서비스명 (영문)**: BikeRideDiary
**약칭**: BRD
**도메인 후보**: barada.kr / barada.app
**패키지**: com.bikeridediary

바이크 라이더를 위한 풀스택 앱. 정비/소모품 교체 이력 관리와 라이딩 코스 기록·공유가 핵심 기능이다.
개인 도구로 시작해서 공개 커뮤니티 서비스로 성장시키는 것이 목표다.

슬로건 후보: 달리고, 기록하고, 바라다

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| 백엔드 | Spring Boot 3.x, Spring Security, Spring Data JPA, QueryDSL |
| 인증 | JWT + OAuth2 (카카오, 구글, Apple) |
| DB | PostgreSQL + PostGIS 확장 |
| 캐시 | Redis |
| 스토리지 | AWS S3 (GPX 파일, 이미지) |
| 웹 프론트 | 추후 검토 — 앱 출시 후 커뮤니티 성장 시 Vue 3 도입 예정 |
| 모바일 앱 | Flutter (Dart) — flutter_naver_map SDK, flutter_foreground_task ← 1순위 |
| 인프라 | Docker Compose, GitHub Actions, AWS |
| API 문서 | Swagger / OpenAPI 3.0 |

---

## 핵심 도메인

**바이크 (Bike)**
- 사용자가 여러 대의 바이크를 등록할 수 있음
- 제조사, 모델명, 연식, 카테고리, 총 주행거리 관리
- 제조사/모델 입력 방식은 단계별로 전환
  - MVP: 텍스트 직접 입력 (빠른 개발 + 사용 데이터 수집)
  - 2차: 사용 빈도 기반 DB 구축 (manufacturers / bike_models / bike_trims 테이블)
  - 3차: 자동완성 + 선택 UI, DB 없는 모델은 직접 입력 유지

**정비 이력 (Maintenance)**
- 소모품 교체 기록 (엔진오일, 타이어, 체인, 브레이크패드 등)
- 교체 당시 주행거리, 날짜, 메모, 비용
- 교체 주기 설정 (km 기준 또는 날짜 기준)
- 다음 교체 예정 알림

**라이딩 코스 (Riding Course)**
- 세 가지 방식으로 코스 생성 (아래 코스 생성 섹션 참고)
- 거리, 소요 시간, 평균 속도, 고도, 난이도
- 공개/비공개 설정
- 공개 코스는 다른 라이더가 탐색·저장 가능
- 좋아요, 댓글

**사용자 (User)**
- 소셜 로그인 전용 (이메일/비밀번호 로그인 없음)
- Apple 로그인 시 이메일 숨김 처리 필요 (가상 이메일 저장)

---

## 기능 목록

### 정비 관리
- 소모품 교체 기록 (엔진오일, 타이어, 체인, 브레이크패드 등) + 비용 기록
- 교체 주기 설정 (km 기준 / 날짜 기준) 및 다음 교체 예정 알림
- 연비 계산 (주유량 + 주행거리 입력 → 자동 계산 및 추이 기록)
- 주요 서류 및 정비 이력서 사진 기록 (S3 업로드)
- 보험 만료 알림
- 정기 검사 알림

### 코스 생성 (3가지 방식)
- GPX 파일 직접 업로드 (Strava, Komoot 등 외부 앱에서 가져오기)
- 라이딩 중 GPS 자동 기록 → 종료 후 코스로 저장
- 지도에서 직접 생성
  - flutter_naver_map SDK로 앱 내 지도 표시
  - 출발지 / 경유지(최대 15개) / 목적지 텍스트 입력
  - 네이버 Geocoding API로 주소 → 좌표 변환
  - Spring Boot 서버가 네이버 Directions 15 API 호출 (API Key 서버에서 관리)
  - 응답의 path 좌표 배열을 NPolylineOverlay로 지도에 표시
  - 확인 후 저장 시 path 좌표를 GPX로 변환하여 S3 저장

### 주유소 검색
- 현재 위치 기반 근처 주유소 검색
  - 필터: 현재 영업중 / 24시 영업, 셀프 주유 여부, 고급유/일반유 가능 여부
  - 카카오 로컬 API + 오피넷(한국석유공사) API 연동
- 라이딩 코스 경로 근처 주유소 검색
  - GPX 경로에서 일정 간격으로 포인트 샘플링 후 PostGIS 반경 검색

### 라이딩 기록 및 분석
- 라이딩 시 날씨/기온 자동 기록 (OpenWeather API 연동)
- 코스 분석: 고도, 총 거리, 평균 속도, 난이도 자동 태깅
- 통계: 월별 주행거리, 평균 속도, 누적 라이딩 횟수/거리
- 코스 저장 / 즐겨찾기
- 공개 코스 탐색, 좋아요, 댓글

### 라이딩 퍼포먼스 (Flutter 앱 전용, 센서 기반)
- 공통 사항
  - sensors_plus 패키지로 자이로스코프 + 가속도계 데이터 수신
  - 저역통과 필터(Low-pass filter)로 엔진 진동 노이즈 제거
  - 참고용 수치임을 UI에 명시 (BMW IMU 같은 전용 하드웨어 수준 아님)

- 뱅킹각 측정 (롤축)
  - 거치 상태에서 캘리브레이션 → 현재 롤값을 0으로 초기화
  - 오르막/내리막 경사는 피치축 영향이라 롤축 측정에 영향 미미 (오차 2~5도 수준)
  - 와인딩 코너에서도 측정 가능

- 윌리 각도 측정 (피치축)
  - 평지에서만 측정 가능 — 경사로에서는 도로 경사값이 피치값에 섞여 분리 불가
  - GPS로 실시간 경사도 계산, 일정 기준 이상이면 "측정 불가 구간" 표시 후 비활성화
  - 평지 직선 구간에서만 동작

- 라이딩 후 퍼포먼스 요약
  - 최대 뱅킹각, 최대 윌리 각도, 급가속/급제동 횟수 등 통계 표시

### 기능 우선순위
- MVP (1차): 소모품 교체 기록/비용/알림, 연비 계산, 보험/검사 알림, GPS 라이딩 기록, 날씨 기록, 월별 통계
- 2차: 코스 지도 생성, GPX 업로드, 주유소 검색, 코스 분석/난이도, 사진 기록, 코스 즐겨찾기
- 3차: 뱅킹각/윌리 측정, 코스 근처 주유소, 커뮤니티 기능 (팔로우, 같이 라이딩 모집 등)

---

## 외부 API 연동

| API | 용도 | 비고 |
|-----|------|------|
| 네이버 Directions 15 | 코스 생성 경로 계산 (경유지 최대 15개) | API Key 서버에서 관리 |
| 네이버 Geocoding | 주소 텍스트 → 좌표 변환 | Directions API 연동 전처리 |
| flutter_naver_map SDK | 앱 내 지도 표시, 경로 Polyline | 네이버 클라우드 Client ID 필요 |
| 카카오 로컬 API | 주유소 검색 | 국내 데이터 커버리지 우수 |
| 오피넷 API | 주유소 유가 / 상세 정보 | 한국석유공사 공공 API |
| OpenWeather API | 라이딩 시 날씨/기온 기록 | 무료 플랜으로 충분 |
| Firebase FCM | 푸시 알림 (보험/검사/소모품) | Flutter + Spring Boot 연동 |
| Apple Sign In | iOS 소셜 로그인 필수 | identity_token JWT 검증 방식 |

---

## 인증 구조

- OAuth2로 카카오/구글/Apple 로그인 처리
- 로그인 완료 후 자체 JWT 발급 (Access Token + Refresh Token)
- Apple은 identity_token(JWT)을 Apple 공개키로 직접 검증하는 방식 (액세스토큰 방식 아님)
- Apple은 최초 1회만 유저 정보 제공 → 첫 응답 때 반드시 저장

---

## 코드 컨벤션 및 선호사항

- 언어: 모든 주석, 변수명, 문서는 **영어**로 작성 (커밋 메시지 포함)
- 응답 포맷: 공통 ApiResponse 래퍼 사용
- 예외 처리: GlobalExceptionHandler로 중앙 처리
- API 버전: `/api/v1/` prefix 사용
- 테스트: JUnit5 + Mockito (단위 테스트 우선)
- 패키지 구조: 도메인 기반 레이어드 아키텍처 (com.bikeridediary)
- Entity 클래스명: 반드시 "Entity" suffix 붙임 (User → UserEntity, Bike → BikeEntity)
- Entity 필드 주석: 모든 필드에 한글 주석으로 설명 작성 (예: `// 제조사명 (MVP: 텍스트 직접 입력)`)
- JPA dirty checking 활용: `@Transactional` 내에서 update/delete 시 `repository.save()` 호출하지 않음
- `@Transactional` 컨벤션 (2026-08-05 확정): **클래스 레벨 어노테이션 금지, 메서드 레벨에만 명시**. 조회 메서드는 `@Transactional(readOnly = true)`, 쓰기 메서드는 `@Transactional`, DB 접근 없는 메서드(외부 API 호출뿐)는 어노테이션 생략. 이유: 각 메서드의 트랜잭션 성격이 어노테이션으로 즉시 보임 + `Propagation.NEVER` 같은 트릭 불필요 + 새 메서드 추가 시 의식적으로 결정하도록 강제

```
com.bikeridediary
├── domain
│   ├── userEntity
│   ├── bikeEntity
│   ├── maintenance
│   ├── course
│   ├── fueling        ← 연비/주유 기록
│   ├── station        ← 주유소 검색
│   ├── document       ← 서류/사진 기록
│   └── bikemodel      ← 제조사/모델 마스터 데이터
├── global
│   ├── auth
│   ├── config
│   ├── exception
│   └── response
└── infra
    ├── s3
    ├── fcm
    ├── weather        ← OpenWeather API
    ├── naver          ← 네이버 Directions / Geocoding API
    ├── kakao          ← 카카오 로컬 API
    └── opinet         ← 오피넷 API
```

---

## 현재 개발 단계

### 완료된 작업

1. 기능 명세서 작성
2. ERD 설계
3. Spring Boot 프로젝트 초기 세팅
4. 인증 모듈 구현 (JWT + OAuth2 카카오/네이버, 이메일 로그인)
5. 사용자(User) 도메인 구현
6. 바이크(Bike) 도메인 구현
7. 정비(Maintenance) 도메인 구현
   - Entity: MaintenanceEntity, MaintenanceScheduleEntity, MaintenanceType(15종)
   - BaseEntity 추상 클래스 (공통 audit 필드)
   - Repository, DTO, Service, Controller 전체 구현
   - 12개 파일 코드 검토 완료 (2026-06-02)
8. CustomUserDetails 도입 — Controller에서 `userDetails.getUserId()`로 UUID 직접 접근
9. MaintenanceScheduleEntity.update()에서 maintenanceType 파라미터 제거 — 스케줄 정비 종류는 고정
10. 정비 도메인 단위 테스트 작성 완료 (2026-06-04)
    - MaintenanceServiceTest: 12개 테스트 (CRUD + 권한 검증)
    - MaintenanceScheduleServiceTest: 16개 테스트 (CRUD + 권한 검증 + 중복 스케줄 방지)
11. SQL 스키마 작성 완료 (2026-06-05)
    - 4개 테이블: users, bikes, maintenances, maintenance_schedules
    - 복합 인덱스 6개 (Repository 쿼리 메서드 기반)
    - RefreshToken은 Redis — PostgreSQL 스키마 미포함
12. OpenAPI 3.0 어노테이션 추가 완료 (2026-06-05)
    - SwaggerConfig: JWT Bearer 인증 스키마 설정 (기존)
    - 4개 컨트롤러에 @Tag, @Operation 어노테이션 추가 (21개 엔드포인트)
    - 접근 경로: http://localhost:8080/swagger-ui.html
13. 보류 사항 해결 (2026-06-05)
    - @ValidScheduleInterval 커스텀 검증: intervalKm/intervalMonths 둘 다 null 방지
    - MaintenanceResponse, MaintenanceScheduleResponse에 updatedAt 필드 추가
14. Docker 설정 보완 (2026-06-05)
    - Dockerfile (JRE Alpine 기반) + .dockerignore 추가
    - docker-compose.yml에 app 서비스 추가 (postgres/redis 의존)
    - 02_schema.sql init 스크립트로 테이블 자동 생성
15. GitHub Actions CI 파이프라인 추가 (2026-06-05)
    - push/PR 시 빌드 + 테스트 자동 실행
    - Gradle 캐싱, 테스트 리포트 artifact 업로드
16. 버그 수정 및 안정화 (2026-06-08)
    - BikeRepository JPA 쿼리 파생 수정: `findByUserId` → `findByUserEntityId` (엔티티 필드명 불일치)
    - NaverProvider `@Value` 경로 변경: `spring.security.oauth2.client.registration.naver.*` → `naver.*` (OAuth2 자동 설정 충돌 방지)
    - UserEntity.createWithEmail() providerId를 이메일 기반 UUID v3로 생성 (NOT NULL 제약조건 위반 수정)
    - build.gradle `jar { enabled = false }` 추가 (plain JAR 생성 방지)
17. 파일 로깅 추가 (2026-06-08)
    - logback-spring.xml: app.log (전체) + error.log (ERROR만) + 일별 롤링
    - Docker 볼륨 마운트로 호스트에서 로그 파일 접근 가능
18. 멀티 프로필 구성 (2026-06-08)
    - application.yml을 공통 설정만 남기고 5개 프로필로 분리
    - local (IntelliJ 직접 실행), local-dev (로컬 Docker), dev/stg/prd (AWS)
    - docker-compose.yml을 SPRING_PROFILES_ACTIVE=local-dev 한 줄로 간소화
    - stg/prd에서 Swagger UI 비활성화

19. SecurityConfig에 CORS 설정 추가 (2026-06-11)
    - Flutter 웹 앱에서 백엔드 API 호출 시 CORS preflight 404 해결
    - `localhost:*` 패턴 허용, credentials 포함
20. Flutter 앱 전체 구현 (2026-06-11)
    - 프로젝트 위치: ../brd_app
    - 기술 스택: Riverpod(상태관리), Dio(HTTP), GoRouter(라우팅), flutter_secure_storage(JWT)
    - 아키텍처: Clean Architecture (data/domain/presentation 3계층)
    - Auth: 이메일 로그인/회원가입, JWT 토큰 저장/갱신, 인증 가드
    - Bike: 목록/상세/등록/수정/삭제, 대표 바이크 설정
    - Maintenance: 정비 기록 CRUD, 정비 스케줄 CRUD, overdue 알림
    - Home: 대시보드 (대표 바이크 요약, 정비 필요 알림, 빠른 메뉴)
    - 화면 12개, Dart 파일 약 40개
    - 웹(Chrome)에서 개발 중 (AVD 미해결)
    - api_config.dart: kIsWeb으로 웹/에뮬레이터 baseUrl 자동 분기

21. 연비/주유(Fueling) 도메인 구현 (2026-06-16)
    - FuelingEntity: BigDecimal(8,2) 정밀 주유량, FuelType enum (REGULAR/PREMIUM/DIESEL)
    - 연비 계산: 이전 주유 기록 주행거리와 현재 주행거리 차이 / 현재 주유량
    - CRUD + 통계 엔드포인트 (6개): GET/POST/PUT/DELETE /fuelings, GET /fuelings/stats
    - FuelingRepository: 이전 주유 기록 조회, 구간 주유량 합계 @Query
    - 단위 테스트 16개 (CRUD, 접근 권한, 연비 계산, 통계)
    - ErrorCode에 FUELING_ACCESS_DENIED 추가
    - schema.sql에 fuelings 테이블 + 인덱스 2개 추가
22. Flutter 앱 주유 기능 + 전체 UI 현대화 (2026-06-16)
    - 주유 데이터 레이어: model/repository/provider (FamilyAsyncNotifier)
    - 주유 목록: SliverAppBar 통계 헤더 + 바이크 선택 드롭다운 + 카드 리스트
    - 주유 폼: 주유량/단가 자동 계산, 연료 종류
    - 디자인 시스템: deep blue #1B2838 + orange accent #FF6B35
    - 전체 화면 그래디언트 헤더 + 카드 기반 레이아웃으로 통일
    - StatefulShellRoute 4탭 하단 네비게이션 (홈/바이크/주유/설정)
    - 로그인/회원가입: 그래디언트 + 흰색 카드 오버레이 디자인
    - 설정 화면: 프로필 카드 + 앱 정보 + 로그아웃

23. bikemodel 도메인 재구조화 + BikeCategory 마이그레이션 (2026-06-19)
    - manufacturers 테이블 PK: BIGSERIAL id → manufacturer_name VARCHAR(100)
    - bike_models FK: manufacturer_id → manufacturer_name
    - 시드 데이터: 47개 제조사, 120개 바이크 모델 (data.sql)
    - BikeCategory enum 삭제 → String 타입으로 전환 (bike_models.type 값 직접 사용)
    - BikeModelNameResponse DTO 추가 (name + type 함께 반환)
    - 백엔드 DTO/서비스/테스트 전체 수정, Flutter BikeTypeDisplay 유틸 클래스
    - p6spy 추가: 파라미터 바인딩된 SQL 로깅 (spy.properties)
    - CORS: 192.168.* 패턴 추가 (로컬 네트워크 접속용)
24. Flutter 앱 GitHub 업로드 + UI 개선 (2026-06-19)
    - brd_app 레포지토리 GitHub에 push 완료
    - 개발 환경: Android Studio → VS Code 전환 (경량화)
    - 실기기 연결: adb reverse tcp:8081 tcp:8081 (게스트 Wi-Fi AP 격리 우회)
    - 바이크 등록 화면: 진행 표시기에 번호 원(①②③) + 라벨(제조사/모델/상세) 추가
    - 바이크 등록 시 모델 선택 → 카테고리 자동 채움 (bike_models.type 연동)

25. 정비 기록 이미지 업로드 + 앱 디자인 개편 (2026-06-26)
    - 이미지 저장: MaintenanceEntity.image_urls 컬럼(TEXT, JSON 문자열로 URL 목록 저장)
    - ImageStorageService 인터페이스 + Local/S3 구현체(@Profile로 전환)
    - 멀티파트 업로드: Controller @RequestPart("data") + @RequestPart("images")
    - 이미지 서빙: FileController가 인증 + 소유권(userId 비교) 검증 후 서빙, /files/** 경로
    - 수정 시 삭제된 이미지 정리: existingImageUrls(유지 목록) 기준으로 빠진 파일 스토리지 삭제
    - 버그 수정: update 시 oldUrls 대신 keepUrls로 저장(3→2장 삭제 반영 안 되던 문제)
    - 멀티파트 전송 버그: dio BaseOptions에서 Content-Type 고정 제거(FormData 자동 boundary)
    - Flutter: image_picker로 갤러리/카메라 선택(최대 5장), AuthenticatedImage 위젯(JWT 헤더 포함 이미지 로드)
    - 디자인 개편: deep blue+orange → iOS 블루(#007AFF) 스타일 전체 적용
    - 새로고침 전 화면 캐시 invalidate + AlwaysScrollableScrollPhysics(짧은 화면도 당김 가능)
    - 보안 메모: application-local.yml의 opinet/api-ninjas 키가 이전 커밋부터 추적 중 → 키 회전 + 환경변수화 권장

26. 뱅킹각 측정 기능 추가 (2026-07-03, brd_app 커밋 97cf952 + a1825a4)
    - brd_app/lib/features/banking/ 신규 도메인 (10개 파일)
    - 센서 스트림/저역 필터/캘리브레이션, Cupertino 다이얼로그(확인=왼쪽, 취소=오른쪽, 다시 안 보기)
    - wakelock을 recording 상태 종속으로 관리 → 화면 이동해도 기록 유지 (앱 내에 있는 한)
    - shell 밖 전체 화면 라우트(/banking, /banking/sessions, /banking/sessions/:id)
    - 로컬 SQLite(brd_banking.db), 서버 백업 버튼 placeholder + synced_at 컬럼 준비
    - 크래시 fix 2건: 50Hz rebuild thrash(rebuild scope 분리 + Timer 기반 elapsed), OutlinedButton unbounded width(minimumSize override)

27. 오프라인 우선 아키텍처 Phase 1/2/4 (2026-07-03)
    - Phase 1 (brd_app 커밋 97cf952, 인프라):
      - core/local/app_database.dart 통합 SQLite(brd_local.db), 도메인별 migration slot
      - core/sync/sync_engine.dart Syncable 등록 기반, 오프라인→온라인 전이 시 자동 syncAll
      - core/sync/sync_types.dart SyncState enum + syncColumnsSql 공통 스니펫
      - connectivity_plus, uuid 패키지 추가, main.dart에서 startAutoSync 호출
    - Phase 2 (brd_app 커밋 dea1424, Auth 오프라인):
      - AuthState.isLocalGuest 추가, continueAsGuest DioException.connectionError/timeout 시 로컬 게스트 fallback
      - core/network/connectivity_provider.dart 앱 전역 온라인 감지
      - 로그인 화면 오프라인 배너, 홈 화면 _LocalGuestHome 분기(뱅킹만 사용 가능)
    - Phase 4 (brd_be 커밋 31457dd, 백엔드 스펙 문서):
      - docs/sync-api.md — 도메인별 upsert 엔드포인트 스펙, LWW, soft delete, idempotency
      - 클라이언트 UUID 정책, request/response 예시, 서버 로직, 클라이언트 sync 흐름
    - 결정: 한 기기 전제 / 클라이언트 UUID / LWW / soft delete / 이미지 로컬 우선 / 바이크·정비·주유 3개 도메인
    - Phase 3(도메인 이전) 인수인계 상태 — 백엔드 upsert 완성 후 진행 권장, 도메인당 7단계 작업 명세는 claude-memory.md 참조

28. Phase 3 바이크 도메인 로컬 우선 이전 완료 (2026-07-07, brd_app 커밋 10aa159/f0932ee, brd_be 커밋 fc30fd6)
    - 앱 클라이언트:
      - features/bike/data/local/bike_local_repository.dart (SQLite CRUD, softDelete/markSynced/markFailed)
      - features/bike/data/model/bike_response.dart에 optional syncState 필드 추가
      - features/bike/data/repository/bike_repository.dart의 sync() 메서드 추가 (POST /bikes/sync)
      - features/bike/domain/bike_sync_service.dart (Syncable 구현 + pullFromServerIfEmpty)
      - features/bike/domain/bike_provider.dart 로컬 우선 재작성 (create 시 client UUID 생성)
      - main.dart에서 BikeSyncService.register + 로그인 상태 전이 시 pull + syncAll
      - Auth logout 시 AppDatabase.clearAll() (한 기기 = 한 유저)
      - 바이크 목록 카드에 sync 상태 배지 (☁️ pending, ⚠️ failed)
      - fix: syncPending/pullFromServerIfEmpty 완료 후 provider invalidate (UI 자동 갱신)
    - 백엔드:
      - BikeEntity: @GeneratedValue 제거, createWithId 팩토리 추가, create()도 UUID 명시
      - BikeSyncRequest DTO (client 스키마 그대로 수용)
      - BikeService.sync(): 소유권 검증 → deletedAt early return → LWW updatedAt 비교 → isRepresentative 3분기
      - POST /api/v1/bikes/sync 매핑

29. 로그인 로딩 오버레이 + 지도/POI 인프라 (2026-07-07, brd_app 커밋 8b2eac4/3dd5610)
    - 로그인 화면 로딩 오버레이 (커밋 8b2eac4)
      - 5개 로그인 흐름 모두 authProvider.status == loading 시 반투명 배경 + 스피너
      - AbsorbPointer로 중복 요청 방지, select 문법으로 rebuild 최소화
    - 네이버 지도 + place 도메인 + 확장 메뉴 3버튼 (커밋 3dd5610)
      - flutter_naver_map 통합, main.dart에서 SDK 초기화 (.env의 NAVER_MAP_CLIENT_ID 필요)
      - CourseMapScreen (shell 밖 /courses 라우트)
      - place 도메인 (features/place): PlaceCategory enum(명소/카페/센터/주유), PlaceResponse, PlaceRepository
      - 지도 상단 카테고리 필터 배지 + 마커 색상 카테고리별 구분 + 하단 상세 시트
      - main_shell 확장 메뉴 3버튼 부채꼴 (코스탐색/내 바이크/뱅킹각)
      - 내 바이크 진입은 navigationShell.goBranch(2)로 브랜치 전환
    - brd_be 커밋 ac5888f: docs/place-api.md 백엔드 스펙
      - PlaceCategory enum, PlaceEntity 필드, 시드 데이터 예시
      - GET /api/v1/places?category=X 엔드포인트, permitAll 안내

30. 코스탐색 필터 재설계 + Naver 지역검색 통합 (2026-07-15)
    - 앱 (brd_app):
      - CourseMapScreen 필터 다중 선택: `Set<PlaceCategory> activeCategories` + `bool wishActive`. 전체 = 모두 활성일 때만 활성 표시, 전체 chip 클릭 = 토글(모두 on/off)
      - 필터 UI: 접기/펼치기 chip 오른쪽에 검색창 배치, 펼침 시 카테고리 세로 나열, 기본 펼침
      - 마커 관리 최적화: clear+rebuild → visibility toggle. `_syncMarkers`(초기 1회 + place CRUD 후 delta) + `_applyVisibility`(필터 변경 시 O(n) setIsVisible)
      - 검색 in-memory 필터로 전환: allPlacesProvider 캐시 부분 일치 검색. 서버 왕복/디바운스/로딩 스피너 전부 제거, 즉시 반응
      - 검색 선택 시 해당 카테고리 자동 활성화. 사용자가 카테고리 필터 조작하면 selectedSearchResult 자동 해제
      - PlaceCategory에 OTHER('OTHER', '기타', '📌') 추가
      - Naver 지역검색 흐름 구현: PlaceCandidate 모델, PlaceRepository.searchExternal, 신규 PlaceSearchAddScreen (검색→선택→크로스헤어 좌표 보정→카테고리 확정→저장)
      - Naver 카테고리 문자열 → 앱 카테고리 자동 추정 (카페/식당/센터/명소 키워드 매칭, 나머지 OTHER)
      - PlaceCreateScreen 삭제(교체), 미사용 _mapCameraCenterProvider 정리
    - 백엔드 (brd_be):
      - 엔드포인트: `GET /api/v1/places/search-external?query=`, `POST /api/v1/places`(addNewPlace)
      - infra/naver/search/: NaverSearchClient + NaverLocalSearchResponse/Item + NaverSearchProperties. `X-Naver-Client-Id/Secret` 헤더, UriComponentsBuilder로 UTF-8 인코딩
      - PlaceService.searchExternal: HTML 태그 제거(<b>...</b>) + WGS84×10⁷ 좌표 스케일링(BigDecimal /1e7, scale=7). CoordinateConverter 미사용
      - Naver 지역검색 응답 좌표 = WGS84 × 10⁷ 확인 (curl 검증). display 최대 5
      - RestTemplate: postForObject → exchange(..., HttpMethod.POST, ...) 통일 (KakaoProvider, NaverProvider)
      - **전면 @ConfigurationProperties 이관** (10개 record, 소비 클래스와 co-location):
        - global/config: FileStorageProperties, AwsProperties(S3 nested)
        - global/auth/jwt: JwtProperties
        - global/auth/oauth2: GoogleOAuth2Properties, NaverOAuth2Properties, AppleProperties
        - infra/naver/search: NaverSearchProperties
        - domain/weather/service: OpenWeatherProperties
        - domain/station/service: OpinetProperties
        - domain/bikemodel/service: ApiNinjasProperties
      - `@ConfigurationPropertiesScan` 활성화 (BikeRideDiaryApplication) → 새 record 자동 등록
      - `cloud.aws.*` 블록 제거 → `aws.*` 통합 (pre-existing 버그: Java가 cloud.aws.s3.bucket=your-bucket-name 기본값만 읽어 프로필별 aws.s3.bucket이 무시되던 문제 해결)
      - MaintenanceService의 참조 안 되던 file.upload-dir/base-url 죽은 필드 제거

31. 라이딩 코스(Course) 도메인 MVP + 서브에이전트 팀 도입 (2026-07-16)
    - **커스텀 에이전트 7명 등록** (`brd_claude/.claude/agents/`):
      - `pm`, `publisher`, `dba`, `backend-dev`, `flutter-dev`, `code-reviewer`, `qa`
      - dba/backend-dev는 조력자(가이드만, Read/Grep/Glob/Bash), 실제 수정은 사용자
      - flutter-dev/pm은 직접 구현 권한
    - **라이딩 코스 MVP 워크플로** (게이트 3단계):
      - ① pm 명세/태스크 분해 → 사용자 승인
      - ② publisher(목업) + dba(DDL) + backend-dev(백엔드 스펙) 병렬 → 사용자 승인
      - ③ flutter-dev(앱 구현) + code-reviewer + qa
    - **백엔드 (brd_be)** — 사용자 직접 구현 완료:
      - 3개 테이블: `courses`, `course_waypoints`, `course_favorites` (schema.sql)
      - courses: hard delete 정책(deleted_at 없음), user_id nullable(시드/큐레이션 대비), source_course_id 자기참조 SET NULL(파생 코스 유지)
      - waypoints: `place_id` 옵셔널 FK + 좌표/이름 스냅샷 저장(place 수정·삭제에도 코스 유효), `role CHECK IN ('START','VIA','END')`, UNIQUE (course_id, seq), lat/lng `NUMERIC(9,7)/(10,7)`
      - favorites: 복합 PK (course_id, user_id), `@EmbeddedId` 방식
      - 6개 엔드포인트: GET /courses/my, GET /courses(게스트 허용), GET /courses/{id}, POST/DELETE favorite, DELETE course
      - IDOR 방어: validateDetailAccess (내 것 or 공개 or 즐겨찾기)
      - 상세 조회 fetch join (waypoint N+1 방지)
      - ErrorCode 3개 추가: COURSE_FAVORITE_ALREADY_EXISTS/NOT_FOUND/OWN_COURSE
      - SecurityConfig: `/api/v1/courses` GET만 permitAll
      - Place 도메인: 신규 등록 시 placeName 중복 체크 (`existsBy`) 추가
    - **앱 (brd_app)**:
      - 확장 메뉴 3→4버튼, 기존 '코스 탐색'→'찾아보기' 라벨, 신규 '라이딩 코스'(🚴 Icons.route) 추가
      - 라우트 신규: `/riding-courses`, `/riding-courses/:id` (shell 밖)
      - `features/riding_course/` 도메인 신설: data/model 3종, repository, provider, presentation 5개 화면(홈/탭2/상세/위젯)
      - 홈: 커스텀 2탭바(내 코스/탐색), FAB(+ = "곧 지원 예정" 스낵바)
      - 탐색 탭: CupertinoSearchTextField in-memory 필터, 낙관적 즐겨찾기 토글 + 실패 시 롤백 + 스낵바
      - 상세 화면: 지도(폴리라인+마커 3종) + 정보 반반, 내 코스는 삭제 다이얼로그(Cupertino destructive), 남 코스는 즐겨찾기+복사편집(스낵바)
      - waypoint role은 백엔드 DB와 통일 (START/VIA/END)
      - `placeId`, `placeCategoryCode` 옵셔널 필드 파싱 (딥링크는 후속)
      - Repository fallback은 `kDebugMode` 한정 (프로덕션 빌드 하드코딩 배제, 실패 시 rethrow로 롤백 가능)
    - **부가 버그 수정**: place 이름/좌표 편집 시 course_map_screen의 마커 갱신 안 되던 문제 → `_syncMarkers`가 좌표/이름/카테고리 변경 감지 시 마커 삭제 후 재생성
    - **문서 (brd_claude)**:
      - `guides/course-schema.md`, `guides/course-backend.md` (사용자 직접 구현용 스니펫)
      - `mockups/riding-course/` HTML 목업 6개 + index (브라우저로 시각 확인)
    - **미결(후속 처리 예정)**:
      - `findFavoritedByOthers` JPQL의 `c.isPublic = TRUE` 조건 → 즐겨찾기 후 비공개 전환 시 MY탭 누락 문제 (QA 발견)
      - place 중복 체크 좌표 근접 조합(옵션 A 100m Haversine) 도입 여부
      - 코스 생성/편집(2차 스코프): Naver Directions 15 통합. `NaverMapsClient/Properties`는 준비 상태
      - GPX 업로드, GPS 자동 기록은 별도 사이클
      - 실기기 시나리오 검증 (Galaxy Z Flip3): 확장 메뉴 4버튼 겹침, 지도 렌더링 성능

32. 장소 승인 워크플로 + 페이징 + 부가 기능 대사이클 (2026-07-21 ~ 2026-07-24)
    - **장소 등록/수정 어드민 승인 워크플로**:
      - 백엔드: `place_change_requests` 테이블(단일, type/target/payload JSONB/status/reviewer), 유저 요청 + 어드민 승인/반려 API, `@EnableMethodSecurity` + `@PreAuthorize("hasRole('ADMIN')")`, `hypersistence-utils-hibernate-63` JSONB 매핑
      - **어드민 자동 승인 (D2=B, B안)**: 어드민이 요청 생성 시 같은 트랜잭션에서 auto-approve, 응답 status=APPROVED 즉시 반영
      - `users.role VARCHAR(20)` (USER/ADMIN), `UserRole` enum, `CustomUserDetails.getAuthorities()` role 반영
      - 앱: 로컬 저장(brd_local.db `local_places`) → "장소 제보하기" → PENDING/APPROVED → 지도 진입 시 자동 sync + APPROVED 로컬 hardDelete
      - `LocalPlace` `deletedAt` 컬럼, `LocalPlaceRepository.findDuplicate`(이름+50m Haversine), 서버 place와 통합 중복 체크(`findDuplicatePlace`)
      - 어드민 화면 (요청 목록·상세, `@PreAuthorize`), 내 요청 목록(설정 진입), 사용자 편집 화면들의 "요청 생성" 흐름
      - 어드민 장소 삭제 (soft/hard 선택 다이얼로그, hard는 CASCADE로 place_wishes/change_requests 함께 삭제)
      - 설명(description) 필드: `_ConfirmStep`, `NewPlaceFormScreen`, `PlaceInfoEditScreen` 3화면 + `UpdateInfoPayload.description` + `PlaceEntity.updateInfo(name, cat, desc)`
    - **주소 검색(NCP Geocoding)** — 세그먼트 "장소 검색" vs "주소로 등록", `NewPlaceFormScreen`(이름/카테고리 필수 입력), 백엔드 `NaverGeocodingClient` + `GET /api/v1/places/geocode`
    - **좌표 보정 위치 검색** — `LocationSearchSheet`(POI/주소 세그먼트), `place_coordinate_edit_screen` AppBar 🔍 → 카메라 이동
    - **페이징 (A+C 하이브리드)**:
      - `PageResponse<T>` + `PageResponse.ofSlice()` 공용 래퍼 (Slice의 `totalElements`/`totalPages` nullable)
      - Page 유지: 어드민 요청 큐, 내 요청, 주유/정비 이력 (총 개수 UI 활용)
      - Slice 전환: 공개 코스(`findByIsPublicTrue`), `searchPublicByName`, `findFavoritedByOthers` (count 오버헤드 회피)
      - `/courses/my` → `/my/owned` + `/my/favorites` 분리, N+1 회피 위해 favorite 여부 `findFavoritedCourseIdsIn` batch 조회
      - 앱 `Paged<T>` 모델 + `InfiniteScrollController` 공용 유틸, 어드민/내 요청 화면 무한 스크롤 완료
      - 나머지 화면(fueling/maintenance/course home/POI 검색)은 첫 페이지 20건만, 무한 스크롤 UI 미구현 (task #13 후속)
    - **인증/세션 개선**:
      - `GET /api/v1/users/me` + `AuthNotifier.checkAuth()`가 토큰 있으면 fetchMe로 유저 정보 채움 (앱 재시작 시 세션 복원 + role 반영)
      - `GlobalExceptionHandler`에 `AuthorizationDeniedException`/`AccessDeniedException` 핸들러 (403 반환. 이전 500 대신)
      - `UserResponse.role` 필드 (role=USER 기본)
      - 서버 오류 메시지 앱 정확 표시 (`{"error": {"message": ...}}` 경로 파싱, 진짜 네트워크 오류만 "네트워크 오류" 표시)
    - **DB — seq → no 리네임 + 전 테이블 적용 (14 테이블)**:
      - 각 테이블 `no BIGINT UNIQUE DEFAULT nextval('<t>_no_seq')` 컬럼 + 시퀀스 + UNIQUE 제약
      - Entity `@Column(name="no", insertable=false, updatable=false) @Generated(event=EventType.INSERT) private Long no;`
      - Repository `findByNo(Long no)`
      - `course_waypoints`는 기존 `seq SMALLINT`(순서) 유지 + `no BIGINT` 신규 추가
      - schema.sql에 `DO $$ ... $$` PL/pgSQL 블록 사용 시 Spring `ScriptUtils`가 `$$` 미지원 → 세미콜론에서 자름 → 각 테이블별 `ADD COLUMN IF NOT EXISTS` + `ALTER COLUMN ... SET DEFAULT` 개별 문장으로 재작성
      - **컬럼 순서**: 신규 CREATE TABLE엔 `no`가 첫 컬럼. 기존 로컬 DB는 ALTER ADD라 뒤에 붙음 (재배치 안 함, 실기능 무관)
    - **Hibernate 6.x UUID batch 이슈**:
      - `EntityBatchLoaderArrayParam`이 UUID[] → byte[] 캐스팅 실패 (`preferred_uuid_jdbc_type` 설정만으론 안 해결)
      - 각 엔티티 `@Id`에 `@JdbcTypeCode(SqlTypes.UUID)` 명시 (10개 엔티티 + `CourseFavoriteId` embeddable)
      - 어드민 요청 목록 `PlaceChangeRequestRepository.findByStatus`에 `@EntityGraph({"requester","targetPlace"})` → lazy 프록시 초기화 회피
    - **UI/UX 개선 다수**:
      - 좌표 값 노출 제거 (사용자 화면 전체), 어드민 좌표 요청 상세 지도 확대·드래그 허용
      - 로컬 pin 마커: `Icons.location_on` (iOS 블루), 서버 pin은 📍 이모지 유지
      - 크로스헤어 조정 렉 해결 (`_initial` 상수 + `onCameraChange` 유지 또는 `onCameraIdle`)
      - 회원가입/이메일 로그인 시트 하단 nav bar 여백, 뒤로가기 shell 최상위 double-tap-to-exit
      - `_ConfirmStep` 상단 좌표 카드 → 주소 표시, "🏍️" 이모지 오버플로우 FittedBox, 로그인 하단 버튼 SingleChildScrollView
      - "공개 등록 요청" 문구 → **"장소 제보"** 전체 통일 (버튼/스낵바/리스트 라벨)
      - `place_info_edit_screen` 키보드 overflow → `SingleChildScrollView` + `bottomNavigationBar`
    - **장소 제보 랭킹 화면 신설** (2026-07-24 세션 후반):
      - 백엔드 `GET /api/v1/places/rankings` — `countRegistrationsByUser` 활용 + rank 번호
      - `PlaceResponse.userId` 필드 추가 (배지·필터용)
      - 앱 `PlaceRankingScreen` (`/place-rankings`): 상단 그래디언트 헤더 "나의 점수 N점", 1/2/3위 🥇🥈🥉 이모지, 나머지 숫자, 내 항목 강조, count는 "N 점"으로 표시
      - 진입점: 지도(찾아보기) AppBar 새로고침 왼쪽 🏆 아이콘 (설정에서는 제거)
      - 지도 필터에 "🙋 내 장소" chip 추가: mineActive alone → 내 등록 전부, +카테고리 → AND 결합
    - **미결 / 후속**:
      - 앱 UI 무한 스크롤 확장 (task #13): fueling/maintenance/course/POI 화면들
      - JWT stateless 리팩터 (task #7): 매 요청 DB조회 → JWT claim에 role 포함
      - place_wishes JPA 엔티티 없음 (스키마만 seq/no 반영, 엔티티 생기면 동일 패턴 적용)
      - course_waypoints 조회용 no는 신규 추가, 기존 seq(순서)와 공존

33. JWT stateless 리팩터 + Phase 3 로컬 우선 완성 (2026-07-27 ~ 2026-07-28)
    - **JWT stateless** (brd_be 커밋 `04ccd2d`):
      - `JwtTokenProvider`: `generateAccessToken(userId, role)` + `extractUserRole` — role claim 추가
      - `JwtAuthenticationFilter`: `userDetailsService.loadUserByUsername` 제거 → claim만으로 `CustomUserDetails` 생성 (매 요청 DB 조회 0회)
      - `AuthService`: 5개 발급 지점(guest/signup/social/email/refresh) role 전달
      - refresh 흐름: `userRepository.findByIdAndDeletedAtIsNull` DB 조회 1회 추가 → 최신 role 반영 + soft-deleted 유저 자동 차단
      - 파급: role 변경/유저 삭제 반영 지연 = access 만료 주기(1시간). 즉시 강제 로그아웃 필요 시 Redis 블랙리스트 별도 스코프
    - **Phase 3 로컬 우선 완성** — 주유(Fueling) + 정비(Maintenance) 이전 + 이미지 처리
      - **백엔드 (brd_be 커밋 `04ccd2d`)**:
        - 신규 sync 엔드포인트: `POST /fuelings/sync`, `POST /maintenances/sync`(멀티파트), `POST /maintenance-schedules/sync` + 각 `GET /my` 초기 pull
        - `MaintenanceService.sync`: `existingImageUrls`에서 빠진 URL 파일 삭제 + 새 이미지 업로드 + `setImageUrls`로 최종 확정. `parseStringToList`/`toJson`/`responseMaintenance` 기존 헬퍼 재사용
        - `FuelingService.sync`: 이전 주유 기록 기반 `fuel_efficiency` 자동 재계산
        - `MaintenanceScheduleService.sync`: 기존 `buildResponse(schedule, bikeId, currentMileage)` 헬퍼로 overdue 자동 판정
        - 3개 sync 서비스 모두 `updateBikeMileage(bike)` 호출 (기존 create/update와 동일)
        - `MaintenanceEntity.setImageUrls` public setter 추가 (sync에서 이미지 확정용)
        - `MaintenanceScheduleEntity.@GeneratedValue` 제거 + `createWithId` 팩토리 추가
        - **`save()` 반환값(managed 엔티티) 사용 필수**: ID 수동 세팅 시 Spring Data JPA가 `merge()` 경로로 감 → `@PrePersist`는 managed 복사본에만 발생 → `target` (detached)의 `createdAt`은 null → 응답 파싱 실패. bike/fueling/maintenance sync 3곳 모두 `target = repo.save(toSave)` 패턴으로 fix
        - **`MaintenanceRepository` 파생 메서드 필드명 정정**: `findByBikeEntityId...` → `findByBikeId...` (`MaintenanceEntity.bike` vs `MaintenanceScheduleEntity.bikeEntity` 명명 불일치. 이전엔 lazy validation이라 startup 통과했다가 `@Query` 신규 추가로 startup validation 발동 → 4곳 rename + 호출부 3곳)
        - **SecurityConfig 정리**: `/api/v1/fuelings/**`, `/api/v1/maintenances/**`, `/api/v1/maintenance-schedules/**` **permitAll 제거** → 인증 필수. (permitAll이면 `@AuthenticationPrincipal`이 null → controller에서 NPE)
        - `BaseEntity`의 미사용 protected setter (setCreatedAt/setUpdatedAt) 정리
      - **앱 (brd_app 커밋 `4b2941f`)**:
        - SQLite v3 → v5: `fuelings`, `maintenances`, `maintenance_schedules` 테이블 신규
        - 3개 신규 `SyncService` (Syncable): `FuelingSyncService`, `MaintenanceSyncService` (이미지 diff + 멀티파트 업로드), `MaintenanceScheduleSyncService`
        - `pullFromServerIfEmpty`: `hasAnyRecords()` 판정 (기존 `listPendingRaw` 오판 fix)
        - **이미지 로컬 우선**: `MaintenanceLocalRepository.persistImage` — 앱 문서 폴더로 복사(cache 청소 방어). `AuthenticatedImage` 위젯이 로컬 파일 경로/서버 URL 자동 분기 (Image.file vs HTTP)
        - Provider 로컬 우선 재작성: `build()`가 로컬 SQLite만 조회 → **UI 절대 에러 안 남**. 안전망(try/catch) 유지
        - **바이크 mileage 로컬 갱신**: 정비/주유 create/update 시 `_bumpBikeMileageIfHigher(bikeId, mileage)` — 서버 sync 완료 전에도 UI 즉시 반영
        - `_triggerSync`에서 bike + 해당 도메인 sync 동시 트리거
        - **바이크 create sync await**: 다음 화면(정비/주유)이 즉시 바이크 참조해도 서버에 존재 보장. 오프라인이면 스킵
        - **오프라인 시도 스킵**: `_triggerSync`가 `connectivityProvider` 체크 → 오프라인이면 sync 시도 자체 안 함 (실패 로그 낭비 방지). 온라인 복구 시 SyncEngine 자동 재시도
        - **로딩 오버레이에서 sync/pull 제외**: dio 인터셉터가 `Options.extra['background']=true` 확인 → 로딩 카운터 스킵. 8개 요청(bike/fueling/maintenance × sync/my) 모두 이 플래그 세팅
        - Repository 축소: fueling/maintenance는 `sync()` + `getMy()`만 남기고 기존 CRUD 삭제 (SyncService만 서버 통신)
        - main.dart에서 4개 SyncService 등록 + 로그인 시 pull
        - UI 배지: 정비 기록/스케줄/주유 카드에 ☁️(pending)/⚠️(failed) sync 상태 표시
    - **부수 UI/UX 버그 fix (brd_app 커밋 `4b2941f`)**:
      - 확장 메뉴 서브 FAB 히트박스: SizedBox 높이 150→200. 안쪽 버튼(±20°)이 `dy≈113px` 이동 + Column 78px 필요 → 기존 150이라 아이콘이 Stack 밖으로 튀어나가 tap 무시됨. `Clip.none`은 시각만 클리핑 끄고 히트테스트는 여전히 Stack 크기 안에서만 동작
      - `_onBikes` `goBranch(2, initialLocation: true)` → 조건부 `(2 == currentIndex)` (다른 브랜치에서 오면 마지막 위치 유지)
      - 4개 FAB에 unique `heroTag` 부여 (`fab-bikes/fab-fueling/fab-maintenance/fab-riding-course`). `StatefulShellRoute.indexedStack`이 브랜치 동시 유지 → 기본 heroTag 충돌
    - **가이드 (brd_claude 커밋 `68cbfbd`)**:
      - `guides/jwt-stateless.md`, `guides/fueling-sync-backend.md`, `guides/maintenance-sync-backend.md`
      - 실제 코드베이스 반영 (기존 `responseMaintenance`/`buildResponse` 헬퍼 활용, JPA auditing으로 `syncTimestamps` 불필요, `MaintenanceResponse.from(entity, List<String>)` 실제 시그니처, 필드명 관례 불일치 주의)
    - **미결 (후속)**:
      - `FuelingServiceTest`가 이전 fueling 마이그레이션 과정에서 이미 깨짐 (삭제된 repository 메서드 참조). 이번 세션 스코프 아님 → 별도 정리
      - 뱅킹 세션 서버 백업은 별도 스코프 (Phase 3 목록에 있으나 이번 사이클 제외)
      - 실기기 오프라인 시나리오 (비행기 모드 → 로컬 저장 → 온라인 복구 자동 sync) 검증 남음 — 로컬 개발 환경에선 정상 동작 확인됨

34. 게스트 로그인 유도 + 로딩 오버레이 리팩터 + FuelingServiceTest 가이드 + 라이딩 코스 2차 pm 게이트 (2026-07-31)
    - **1. `/api/v1/places/**` 인증 확인**: 이미 처리 완료 상태 확인. `SecurityConfig`가 places를 `GET_PERMIT_ALL_ENDPOINTS`에만 두고 POST/PATCH/DELETE는 change-request 워크플로 + `@PreAuthorize`로 걸어둠. 별도 조치 불필요.
    - **2. 앱 게스트 로그인 유도 다이얼로그** (brd_app):
      - 신규 `lib/features/auth/presentation/require_auth.dart` — `Future<bool> requireAuth(BuildContext, WidgetRef)`. `isLocalGuest`면 Cupertino 다이얼로그(취소/로그인) 띄우고 확인 시 `logout()` → 라우터가 `/login`으로 redirect. 서버 인증이면 true 즉시 반환
      - `course_map_screen.dart` 3개 진입점에 가드: `_openAddPlace()`(장소 제보), 정보 수정 버튼, 좌표 보정 버튼. 다이얼로그 후 `rootCtx.mounted` 체크로 async gap 안전 처리
      - `local_place_detail_screen.dart` `_submitRequest()`에도 가드 추가
      - flutter analyze 클린 (course_map_screen에 pre-existing warning 2건은 이번 변경 아님)
    - **3. FuelingServiceTest 복구 가이드**: `brd_claude/guides/fueling-service-test.md` — 5개 변경점(페이징 메서드 교체, `getFuelings` 시그니처 Pageable+PageResponse, `updateBikeFuelEfficiency` 신규 스텁, `UserRepository` mock, `FuelingStatsResponse` 6필드) 체크리스트. 사용자 직접 수정 예정
    - **4. 로딩 오버레이 stuck 버그 fix** (brd_app):
      - 증상: `로딩 중...` 오버레이가 걸려 안 사라지는데 FAB/하단탭 터치는 됨. body만 덮고 outer Stack의 FAB이 위에 얹혀서 정상적으로 히트 통과(오버레이 자체는 body 내부 히트를 정상 차단). 근본 원인은 int 카운터의 불균형 leak
      - 조치: `loadingCountProvider` (int 카운터) → `pendingRequestsProvider` (`Set<int>` of 요청 ID)로 교체. 각 요청은 `options.extra['_loadingId']`에 고유 ID 저장. Set은 double-add/remove 자동 idempotent라 leak 원천 차단
      - `beginLoading(ref, id, url)` / `endLoading(ref, id, url, {ok})` 헬퍼. `kDebugMode`에서 `[Loading] +N /path (pending=M)` 로그 → stuck 재현 시 어떤 요청이 걸렸는지 즉시 추적 가능
      - 변경: `lib/core/network/loading_state.dart`, `lib/core/network/dio_client.dart`
    - **5. 라이딩 코스 2차 스코프 pm 게이트 1** (brd_claude 계획 문서):
      - `plans/riding-course-phase2.md` 생성 (pm 서브에이전트가 담당)
      - 스코프 In: Naver Directions 15 통합, waypoint 지도 편집(최대 15), 코스 생성/편집/미리보기, 복사편집 실제 구현, 거리 자동 계산
      - 스코프 Out: GPX, GPS 자동기록, 난이도/고도 분석, 소요시간 UI, 오프라인 편집
      - 사용자 결정 대기 중(D1~D7): path 저장 형식(A JSON), 요약 필드(A distance만), waypoint place 연동(A 둘 다), 순서 재배열(A 드래그), Directions 호출 시점(B 미리보기 버튼), 복사편집(A 이번 포함), Directions 실패(A 저장 차단). 사용자가 "다 추천대로" 하면 게이트 2(publisher+dba+backend-dev 병렬)로 진행
      - 관련: `courses.path TEXT` 컬럼 이미 있음(JSON 저장 재사용 가능), `NaverMapsProperties.direction15Url` 이미 세팅됨

35. 라이딩 코스 2차 스코프 완결 + 서브에이전트 개선 사이클 (2026-08-12)
    - **네이버 지도 딥링크 안내 다이얼로그 2단계** (brd_app `course_detail_screen.dart`):
      - 1단계: "출발지를 내 위치로 변경하시겠습니까?" (예/아니오). 예 선택 시 geolocator로 현재 위치 획득, 권한/서비스/타임아웃 실패 시 스낵바로 사유 안내하고 원래 START 좌표로 fallback
      - 2단계: "네이버 지도에서 이륜차 설정 필요 + 실제 경로 다를 수 있음" 안내 (확인/취소)
      - 확인 시에만 `nmap://route/car?...` 실행. 예 선택했으면 `slat/slng/sname='내 위치'`로 override, 아니오면 코스 저장 START 사용
      - `_buildQueryParams({Position? currentPosition})` 옵셔널 override 파라미터 추가
    - **라이딩 코스 2차 스코프 게이트 2/3 완결** (2026-07-29 pm 게이트 1 → 2026-08-12 최종):
      - D1~D7 전부 pm 추천안 채택 (D5는 사전 확정된 B 유지, D6 복사편집 이번 스코프 포함, D8 `regeneratePath` 플래그 채택)
      - **핵심 발견**: 백엔드/앱 모두 대부분 이미 구현된 상태였음. 이번 사이클은 계획 문서화 + 리뷰 + 소수 fix로 축소
      - 산출물 (`brd_claude/`):
        - `plans/riding-course-phase2.md` (pm 서브에이전트)
        - `guides/course-phase2-schema.md` (dba, 결론: DDL 변경 불필요. courses에 이미 bbox/description/count 컬럼 4종 ALTER 반영, END→GOAL 마이그레이션 schema.sql 라인 392~396에 포함)
        - `guides/course-phase2-directions.md`, `guides/course-phase2-backend.md` (backend-dev, 기존 구현 검증 + 잠재 이슈)
        - `mockups/riding-course-phase2/` 6개 HTML + index + style.css (publisher, 파일명 하이픈 스타일로 리네임됨)
    - **백엔드 fix — sourceCourseId IDOR 취약점** (brd_be `CourseService.createCourse` 라인 195~200):
      - 기존: `courseRepository.findById(sourceCourseId).ifPresent(src -> incrementCopyCount(src.getId()))` — 비공개 타인 코스여도 조용히 copy_count +1
      - fix: `findByIdWithUser(sourceCourseId).orElseThrow(COURSE_NOT_FOUND)` → `validateDetailAccess(source, userId)` → `incrementCopyCount`. 원본 없으면 404, 비공개 타인 코스면 403
      - `SecurityConfig.java` 죽은 주석/backtick 오타 정리 (사용자 직접)
    - **앱 fix**:
      - `course_repository.dart` 무의미한 try/catch(rethrow만) 3건 통째로 제거 (fetchMyCourses, fetchAllCourses, fetchCourse)
      - flutter analyze 클린 확인 (경고 3건 → 0)
    - **서브에이전트(QA/code-reviewer) 근본 개선** (`.claude/agents/qa.md`, `code-reviewer.md`):
      - 문제 발견: QA가 "START/GOAL 드래그 시 role이 VIA로 재계산되는 버그" 리포트 → 실제로는 사용자가 코스 방향 뒤집기 위해 S/G도 드래그하는 게 의도된 설계. code-reviewer가 "삭제 액션에 Icons.thumb_up" 지적 → **실제 git baseline엔 정말 thumb_up이 있었음(진짜 버그). Claude가 "리뷰어 오지적"이라 잘못 반박했다가 git diff로 확인하고 정정.** 두 케이스 모두 팩트 체크 부족
      - 조치: 두 agent 정의에 "리포트 전 필수 사전 조사 5단계"(plans/CLAUDE.md/memory/mockups/실제 코드 팩트 체크) + "가정 명시 필수"(단정 금지, 확인 못 한 사항은 "설계 의도 확인 필요" 섹션으로) + code-reviewer에 "과잉 방어 지적 금지"(CLAUDE.md 원칙 참조) 추가
      - 실제 사례 명시 (2026-08-12 라이딩 코스 QA)로 반복 방지
    - **메모리 신설** (`~/.claude/projects/.../memory/`):
      - `MEMORY.md` (index)
      - `feedback_subagent_review.md` — 서브에이전트 리포트를 팩트 체크 없이 사용자에게 전달 금지. 실제 코드/spec 확인 후 필터링해서 전달

36. 이월 미결 대청소 사이클 (2026-08-12)
    - **탐색 탭 무한 스크롤** (brd_app `c8b9540`):
      - `CourseRepository.fetchAllCoursesPaged({page, size, keyword})` 신규 (`Paged<T>` 반환)
      - `ExploreCoursesTab` StatefulWidget + `InfiniteScrollController` 리팩터 (my_place_requests 패턴)
      - 검색은 서버 `keyword` param submit 시 page 0부터 재조회, 낙관적 즐겨찾기 토글 유지
      - **orphan cleanup**: 무참조 provider 3개 삭제 (`allCoursesProvider`, `exploreSearchProvider`, `courseSearchProvider`), 관련 `ref.invalidate` 5곳 정리, `fetchAllCourses(non-paged)` 삭제
    - **stale 이월 목록 대청소 (팩트 체크)** — 32번 사이클부터 이월된 8개 항목을 실제 코드로 검증:
      - ✅ **`findFavoritedByOthers` isPublic 조건**: 이미 완료. 실제 JPQL에 조건 없음 (사이클 중간에 조용히 fix됨)
      - ✅ **place 중복 100m Haversine**: 이미 완료. `PlaceService.checkDuplicate()`가 bounding box 1차 + Haversine 2차 정밀화로 완성
      - ✅ **fueling/maintenance 무한 스크롤**: 대상 아님 확정. Phase 3 로컬 우선(SQLite 전체 로드)이라 서버 페이징 개념 자체 없음
      - ✅ **place 검색 무한 스크롤**: 대상 아님 확정. 지도 마커용 전체 로드 + in-memory 검색이 UX 요구사항
      - ✅ **뱅킹 세션 서버 백업**: 스코프 아웃 확정 (2026-08-12). 사용자 결정으로 로컬 전용
      - ✅ **카카오맵 딥링크**: 사용자 결정으로 pass (2026-08-12)
      - ✅ **FuelingServiceTest 복구**: 사용자 결정으로 pass (2026-08-12)
      - ✅ **주유소 지도 통합**: 사용자 결정으로 pass (2026-08-12). 필요 시 별도 사이클에서 pm 게이트로 스코핑
    - **서브에이전트 리뷰 규칙 정정** (`code-reviewer.md`, `claude-memory.md`):
      - 이전 사이클(35번)에서 code-reviewer의 "Icons.thumb_up" 지적을 Claude가 "오지적"으로 반박 → 실제 git baseline에 정말 thumb_up 있었음. 반박도 팩트 체크 필수라는 규칙 추가
      - `.claude/agents/code-reviewer.md`에서 잘못된 "오지적 예시" 제거하고 "반박 시에도 팩트 체크" 문구로 교체

### 다음 단계

- **라이딩 코스 2차 스코프 실기기 검증**: Galaxy Z Flip3에서 신규/편집/복사편집/미리보기/네이버 지도 딥링크 2단 다이얼로그 흐름 확인 (사용자 검증 완료 시 삭제)
- **로딩 오버레이 leak stuck 재발 시 로그 확인**: `[Loading] +N <path>`만 있고 매칭 `-N` 없는 요청 URL이 진짜 leak 소스
- **주요 미결/후속** (36번 사이클 대청소 후):
  - 실기기 오프라인 게스트 시나리오 검증 (비행기 모드 → 가입없이 시작하기 → 뱅킹)
  - **외부 API 호출 로깅 (모든 기능 완성 후 착수 예정, 2026-08-04 결정)**: Naver Directions/Geocoding/Search, Kakao Local, Opinet, OpenWeather 등 외부 API 호출 시 어떤 사용자가·언제·어떤 파라미터로 호출했는지 DB 기록. 목적: 사용량 모니터링(Naver 무료 60,000/월 등 한도 추적), 이상 사용/오남용 탐지, 유저별 차단 근거. `api_call_logs` 테이블 신규 or AOP 기반 인터셉터. **지금 착수 금지 — 후속 사이클에서**
  - 주유소 지도 통합 (2026-08-12 사용자 결정으로 이번 세션 pass. 재개 시 pm 게이트로 정식 스코핑 필요)
  - `FuelingServiceTest` 복구 (2026-08-12 pass. `guides/fueling-service-test.md`에 가이드 있음)
  - 카카오맵 딥링크 (2026-08-12 pass. 필요 시 재개)

---

## Claude에게 요청하는 작업 방식

- 코드 생성 시 전체 파일 단위로 작성해줘 (일부만 발췌 말고)
- 새 기능 추가 전에 기존 코드 구조 먼저 파악하고 일관성 유지해줘
- Spring Boot 관련 베스트 프랙티스 적극 반영해줘
- Vue / Flutter는 내가 배우면서 하는 거라 코드에 설명 주석 달아줘
- 보안 이슈 발견하면 먼저 알려줘
