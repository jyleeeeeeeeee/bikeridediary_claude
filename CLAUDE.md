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

### 다음 단계

- **Phase 3 클라이언트 도메인 이전 진행**: 주유(Fueling) → 정비(Maintenance) 순, 백엔드 upsert 필요
- **주유소 지도 통합**: place UI 토글의 주유 카테고리를 기존 station API에 연결 (또는 place로 흡수)
- **카카오맵/네이버 지도 딥링크 버튼**: 하단 시트에 url_launcher로 추가 (TODO 표시됨)
- **place_categories 시드 확인**: schema.sql/data.sql에 없음 — DB에 직접 seed된 상태로 추정. OTHER row가 실제 존재하는지 확인 필요 (없으면 앱 '기타' 저장 시 FK 위반)
- **SecurityConfig의 /api/v1/places/** 정리**: 지금 permitAll이라 POST/PATCH도 무인증. 인증 정리를 나중에 다른 엔드포인트들과 일괄 처리 예정
- Flutter 앱 실기기 오프라인 게스트 시나리오 검증 (비행기 모드 → 가입없이 시작하기 → 뱅킹)
- 라이딩 코스(Course) 도메인 (GPX 기록/업로드, Naver Directions 15 활용)

---

## Claude에게 요청하는 작업 방식

- 코드 생성 시 전체 파일 단위로 작성해줘 (일부만 발췌 말고)
- 새 기능 추가 전에 기존 코드 구조 먼저 파악하고 일관성 유지해줘
- Spring Boot 관련 베스트 프랙티스 적극 반영해줘
- Vue / Flutter는 내가 배우면서 하는 거라 코드에 설명 주석 달아줘
- 보안 이슈 발견하면 먼저 알려줘
