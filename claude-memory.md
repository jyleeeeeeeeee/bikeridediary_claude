# Claude 작업 메모리 (git 동기화)

> 이 파일은 `CLAUDE.md`에서 `@claude-memory.md`로 import되어 세션 시작 시 자동 로드됩니다.
> Claude 홈(`~/.claude`)의 메모리는 git 공유가 안 되므로, 정본은 이 파일에 둡니다.
> 다른 환경에서도 git pull 하면 그대로 읽힙니다. 최종 갱신: 2026-07-13.

---

## 작업 방식 / 피드백 (반드시 준수)

- **메모리/기록 저장 위치 = 이 파일**: 사용자가 "진행 상황 기록해 / 기억해 / 메모리에 저장해" 등을 요청하면, 홈(`~/.claude`)의 메모리가 아니라 **이 파일(`brd_claude/claude-memory.md`)에 기록하고 git 커밋**할 것 (다른 환경 공유를 위해). 완료 작업 로그성 내용은 `CLAUDE.md`의 "현재 개발 단계"에 추가.
- **퍼미션 확인 금지**: 파일 수정·Bash·프로세스 종료 등 모든 작업을 확인 없이 바로 실행. 위험 작업(force push 등)도 사용자가 요청했으면 바로. 사용자가 매번 확인받는 걸 매우 싫어함.
- **앱은 가이드만 받지 않고 자율 완성형 구현**: Flutter/앱 코드는 강의식(Learn by Doing, TODO(human), 교육적 중단) 금지. 설명 요청이 아닌 한 바로 완성형으로. Insight는 사용자가 관심 가질 것만 간결히.
- **역할 분담**: 앱(brd_app)은 Claude가 구현, 백엔드(brd_be)는 Claude가 **가이드만** 주고 사용자가 직접 구현. (단 사용자가 명시적으로 "반영/수정해"라고 하면 백엔드도 직접 수정)
- **"커밋해/푸시해" = 세 프로젝트 모두**: brd_claude + brd_be + brd_app 세 repo 모두 git status 확인 후 변경된 곳 커밋+푸시. 특정 프로젝트만 지목하면 그 프로젝트만.
- **엔티티 클래스명**: 모든 `@Entity`는 `Entity` suffix (UserEntity, BikeEntity…). 참조 파일(Repository/Service/Controller/DTO)도 함께 갱신.
- **엔티티 필드 주석**: 모든 필드에 한글 주석으로 설명. 기술 상세는 괄호로.
- **캐시 무효화**: CRUD 구현 시 관련 Riverpod provider 캐시 반드시 invalidate (목록/상세/연관 통계 모두). 새로고침 화면은 invalidate 후 재요청 + `AlwaysScrollableScrollPhysics`(짧은 화면도 당김 가능).

---

## 프로젝트 진행 상황

### 앱(brd_app) 개발 환경
- IDE: VS Code. 실기기: Galaxy Z Flip3(공기계, SIM 없음).
- 실기기 연결: `adb reverse tcp:8081 tcp:8081` → localhost:8081 (게스트 Wi-Fi는 AP 격리라 이 방법이 유일).
- 백엔드 포트 8081. api_config.dart가 플랫폼별 URL 분기.
- 디자인: **iOS 블루(#007AFF) 스타일** (구 deep blue #1B2838 + orange는 2026-06 폐기).
- 아키텍처: Riverpod + Dio + GoRouter + flutter_secure_storage, Clean Architecture 3계층.

### 구현 완료
- Auth: 이메일/게스트 로그인, JWT 저장/갱신, 인증 가드.
- Bike: CRUD, 대표 바이크, 모델 선택 시 카테고리 자동(bike_models.type 연동).
- Maintenance: 정비 기록/스케줄 CRUD, overdue 알림, **이미지 업로드/조회**(아래 참조).
- Fueling: 주유 CRUD, 만탱크법 연비 계산, 통계.
- Home 대시보드, Settings.

### 정비 이미지 업로드/조회 (완료)
- 저장: `MaintenanceEntity.image_urls`(TEXT)에 URL 목록을 JSON 문자열로. ObjectMapper(빈 주입) 직렬화/역직렬화. `List.toString()`은 JSON 아니므로 금지.
- 업로드: Controller `@RequestPart("data")` + `@RequestPart(value="images", required=false)`. 앱은 항상 멀티파트(dio FormData). dio BaseOptions에서 Content-Type 고정 제거(자동 boundary).
- 서빙: `FileController`가 `/files/{userId}/{filename}`에서 인증+소유권(URL userId == 로그인 userId) 검증 후 서빙. 앱은 `AuthenticatedImage` 위젯이 JWT 헤더 실어 로드.
- 수정: 앱이 `existingImageUrls`(유지 URL)+새 `images` 전송. 백엔드는 keepUrls에 없는 기존 파일 삭제 후 keepUrls+새 업로드 저장. (저장은 oldUrls 아닌 keepUrls 기준 — 버그 주의)
- 미해결: S3ImageStorageService.download()가 로컬 파일 로직 잔존 → S3 전환 시 스트림 기반 교체 필요.

### bikemodel DB 구조 (완료, 2026-06-19)
- manufacturers PK: BIGSERIAL → manufacturer_name VARCHAR(100). image_file 컬럼(static/logos/ 참조).
- bike_models FK: manufacturer_id → manufacturer_name. BikeModelNameResponse(name+type) DTO.
- BikeCategory enum 삭제 → String. 앱은 BikeTypeDisplay 유틸로 한글 매핑.
- data.sql: 제조사 47 + 모델 120 시드. p6spy로 바인딩 쿼리 로깅.

### 라이딩 GPS — 포그라운드 전용 (설정만 복구됨)
- 백그라운드 위치는 플레이스토어 심사 이슈로 **의도적 배제**. riding_provider.dart는 원래부터 포그라운드 전용(flutter_foreground_task 미사용).
- 현재 상태: geolocator 패키지 + Android ACCESS_FINE/COARSE(BACKGROUND 제외) + iOS WhenInUse(Always 제외)만 복구. **riding 코드 자체는 `brd_app/_disabled_features/riding`에 있고 lib/features로 복구 안 됨, 라우터/탭 연결도 안 됨.** 복구 절차: `brd_app/_disabled_features/RESTORE_GUIDE.md`.
- 라이딩 본격 복구 시 백그라운드 위치 권한 추가하지 말 것.

### 주유소 검색 (진행 중)
- 앱은 완료, 백엔드 오피넷(한국석유공사) API 사용자 구현 중. station은 일부 비활성(`_disabled_features/station`).

### 소셜 로그인 (완료, 2026-07-01 확인)
- 앱: 이메일/게스트/카카오/구글/애플 5종 다 구현. auth_provider.dart의 loginWithKakao/Google/Apple + auth_repository.socialLogin.
- 백엔드: `/auth/login/{provider}` 통합 엔드포인트 + KakaoProvider/GoogleProvider/AppleProvider/NaverProvider(백엔드만) 완성. AuthLoginRequest credential 단일 필드로 통합됨.
- 네이버는 백엔드만 있고 앱 미연동.

### 오프라인/로컬 우선 아키텍처 (2026-07-01 Phase 1/2/4 완료, Phase 3 인수인계)
- 배경: 5종 로그인이 다 있어도 전부 온라인 필수 → 네트워크 없으면 앱 진입도 불가. 게스트조차 서버 게스트라 오프라인에서 무용.
- 결정 (사용자 확정): 한 기기 전제 / 클라이언트 UUID / last-write-wins / soft delete(deleted_at) / 이미지 로컬 우선 / 바이크·정비·주유 3개 도메인 리팩터링 (뱅킹은 이미 로컬 우선).

**Phase 1 완료** (커밋 `97cf952`, 인프라):
- `connectivity_plus`, `uuid` 패키지
- `core/local/app_database.dart` — 통합 brd_local.db, 도메인별 migration slot
- `core/sync/sync_engine.dart` — Syncable 등록 기반, 오프라인→온라인 전이 시 자동 syncAll
- `core/sync/sync_types.dart` — SyncState enum, syncColumnsSql 공통 스니펫
- main.dart에서 startAutoSync 호출 (등록 도메인 없어 no-op)

**Phase 2 완료** (커밋 `dea1424`, Auth 오프라인):
- AuthState에 `isLocalGuest` 추가
- continueAsGuest() 오프라인 fallback (DioException.connectionError/timeout → 로컬 게스트 flag만 세팅)
- `core/network/connectivity_provider.dart` — 앱 전역 온라인 감지
- 로그인 화면 오프라인 배너
- 홈 화면 로컬 게스트 분기 → `_LocalGuestHome`(뱅킹만 안내)

**Phase 4 완료** (백엔드 스펙 문서): `brd_be/docs/sync-api.md`
- 도메인별 upsert 엔드포인트 스펙, LWW, soft delete, idempotency 명시
- 각 도메인 request/response, 서버 로직, 에러 처리, 클라이언트 sync 흐름
- 백엔드 사용자 구현 순서 권장: 바이크 → 주유 → 정비 → 뱅킹

**Phase 3 진행 상황 (2026-07-07)**:
- **바이크 도메인 완료** (brd_app 10aa159/f0932ee, brd_be fc30fd6)
  - 앱 & 백엔드 sync 엔드포인트 양쪽 완성, 통합 검증 A/B/C 시나리오 통과
  - 참조 구현 완성 — 주유/정비도 같은 패턴 복제 가능
- **남은 도메인**: 주유(Fueling) → 정비(Maintenance, 이미지 포함) → 뱅킹 서버 백업
- 도메인당 7단계 작업 (바이크 참고하면 예측 가능):
  1. 로컬 스키마 (app_database.dart의 _migrations에 v3/v4/v5 추가)
  2. LocalRepository (SQLite CRUD, softDelete/markSynced/markFailed)
  3. RemoteRepository에 `sync()` 메서드 추가 (POST /{domain}/sync)
  4. SyncService (Syncable 구현 + pullFromServerIfEmpty)
  5. Provider 로컬 우선 재작성 (client UUID 생성, invalidate 순서)
  6. UI sync 상태 배지 (☁️ pending, ⚠️ failed)
  7. main.dart에서 SyncEngine.register + 로그인 pull 훅 추가
- **필수 fix 패턴**: syncPending과 pullFromServerIfEmpty 둘 다에서 로컬 갱신 후 provider invalidate 호출 필수 (안 하면 UI 재시작 전엔 반영 안 됨)
- 정비 도메인 이미지: 로컬 임시 파일 경로 저장 → sync 시 기존 멀티파트 업로드 재사용 → 서버 응답 URL로 로컬 값 교체
- 참조 무결성: 정비/주유의 bikeId는 바이크 sync 완료 후 시도. SyncEngine 순회 순서(바이크 먼저 등록)로 자연스럽게 해결
- 백엔드 sync 엔드포인트 (docs/sync-api.md 참고, 사용자 직접 구현): 주유 → 정비 → 뱅킹 순
- 백엔드 BikeEntity 리팩터 노트: @GeneratedValue 제거 → createWithId 팩토리 추가 + 기존 create()도 UUID.randomUUID() 명시. 다른 도메인도 동일 패턴 적용

### 지도/POI 인프라 (2026-07-07 착수)
- 네이버 지도 통합 (flutter_naver_map, .env의 NAVER_MAP_CLIENT_ID)
- CourseMapScreen shell 밖 /courses 라우트
- place 도메인 (features/place): 큐레이션 POI (명소/카페/센터). 유저별 데이터 아니라 로컬 우선 아키텍처 대상 아님 — 순수 서버 조회.
- UI: 카테고리 필터 배지 + 마커 + 하단 상세 시트
- 백엔드 스펙: docs/place-api.md
- 리뷰/후기: 카카오/네이버 리뷰 API 없음. 크롤링 법적 리스크. **자체 리뷰 시스템 + AI 요약** 방향 검토, 결정 유보
- 딥링크 방향: 상세 시트에 카카오맵/네이버 지도 딥링크 버튼 (url_launcher), 후속 작업
- 주유소 카테고리: 지도 UI 토글엔 있으나 데이터 소스 미연결. 기존 station API(오피넷)와 통합 필요
- 확장 메뉴 3버튼 (main_shell): 좌 코스탐색(/courses) / 중 내 바이크(navigationShell.goBranch(2)) / 우 뱅킹각(/banking)

### Place DDL 최종 결정 (2026-07-13, 아직 schema.sql 반영 전)
- 인덱스 3개 확정:
  - `idx_places_category_deleted_at (category_code, deleted_at)`
  - `idx_places_lat_lng (latitude, longitude) WHERE deleted_at IS NULL` (partial)
  - `idx_place_wishes_user (user_id)` (복합 PK가 (place_id, user_id)라 user 단독 조회용 별도 필요)
- FK ON DELETE 정책:
  - `places.user_id → users(id) ON DELETE SET NULL` (유저 탈퇴 시 큐레이션 장소는 author만 null 처리)
  - `places.category_code → place_categories(category_code) ON DELETE RESTRICT` (실수로 카테고리 지우면 places 전체 참조 깨짐 방지)
  - `place_wishes.place_id/user_id ON DELETE CASCADE`
- 좌표 타입: `latitude NUMERIC(9,7)`, `longitude NUMERIC(10,7)` (카카오맵 API가 소수 7자리 반환, 약 1.1cm 해상도).
  - **엔티티 반영 필요**: PlaceEntity의 `double latitude/longitude` → `BigDecimal` + `@Column(precision=..., scale=7)`, create/update 팩토리 파라미터도 BigDecimal로.
- PostGIS 도입 여부는 **나중에 결정** (코스 도메인 시작 시점과 맞추면 마이그레이션 1회로 끝날 것)
- wishedCount 정합성 (배치 recount vs 트리거)는 트래픽 붙은 뒤 대응.

### Place 시드 데이터 수집 (2026-07-13, DB 반영 대기)
- 방법: **카카오맵 내부 JSONP 엔드포인트** `https://search.map.kakao.com/mapsearch/map.daum?callback=cb&q=<keyword>&msFlag=A&sort=0&page=<n>` 사용
  - **비공식 API — 카카오 약관 위반 소지, 스펙 변경 리스크**. Referer=map.kakao.com, User-Agent 필수. JSONP wrapper `/**/cb(...)` 제거 후 파싱.
  - 페이지당 15건, `page_count` 필드 = 전체 건수. `place[].lat`/`place[].lon` = WGS84 (x/y는 TM 좌표계라 사용 금지). `confirmid`로 dedupe.
  - 공식 대안: `dapi.kakao.com/v2/local/search/keyword` (최대 45건 상한이라 커버리지 좁음)
- 결과 파일 (git 미추적, `brd_claude/_kakao_*.sql`):
  - `_kakao_bike_cafes.sql` — **통합 159건** INSERT (place_name, category_code='CAFE', latitude, longitude만)
  - `_kakao_rider_delta.sql` — 라이더 카페 신규 17건만
- 키워드별 수집량:
  - "바이크 카페": 142건 (10페이지)
  - "라이더 카페": 32건 raw → 신규 17건 (15건은 바이크 카페와 중복)
- **DB 반영 전 필수 작업**:
  1. Place DDL 반영 (스키마 + PlaceEntity 좌표 타입 BigDecimal 전환)
  2. `place_categories`에 'CAFE' 먼저 seed (FK 위반 방지)
  3. **수동 큐레이션** — 카카오 리뷰 태그 기반 매칭이라 오탐 섞임 (예: 빽다방, 코코부코 등 라이더 무관 카페 몇 개). 특히 라이더 delta 17건 중 애매한 5개 검토.
- PowerShell 재현 스크립트 요지: `Invoke-WebRequest` + `[System.Text.Encoding]::UTF8.GetString($r.RawContentStream.ToArray())` + `text.Substring(7, len-9)` + `System.Web.Script.Serialization.JavaScriptSerializer(MaxJsonLength=104857600)`. PS 5.1 기본 `ConvertFrom-Json`은 큰 페이로드에서 "Invalid JSON primitive" 오류 자주 남 — JavaScriptSerializer 사용 권장.

**커밋 원칙**: 도메인별 세부 커밋.

---

## 경쟁앱 / 전략

### 바이킹스 벤치마크 (2026-06-29)
- 바이킹스(앱스토어 id6766225776) = 라이딩 "전(前)" 앱(라이더 날씨지수, 코스탐색/로드뷰, 같이타기/채팅, 주유소 유가). 바라다 = 라이딩 "후(後)" 앱(정비/주유/연비 기록·관리). 정반대 포지션, 상호 보완.
- 전략: 정면 경쟁(특히 코스 큐레이션) 말 것. 코어(기록/관리) 굳히고 API로 쉬운 것만 흡수(주유소·날씨지수) 후 내 기록 강점과 연결.
- 인사이트: 바이킹스 코스도 콘텐츠 직접 제작 아님 — 360영상=유튜브 임베드, 노면=유저 크라우드소싱, 난이도=GPX 곡률 자동분석, 주유소=오피넷. 따라하기 어렵지 않음. 초기 코스 시드만 직접 채우면 됨.
- 권장 순서: 소셜로그인 → 주유소검색 마무리 → 라이딩GPS복구 → 날씨/라이딩지수 → 코스 → 커뮤니티. (바이킹스 기능 ≈ 내 2~3차 계획과 일치)

---

## 레퍼런스

- **API-Ninjas 모터사이클 API**: endpoint `https://api.api-ninjas.com/v1/motorcycles`, 키는 application-local.yml `api-ninjas.api-key`. offset 페이징(30개씩), free 1req/sec. Royal Enfield 데이터 없음, CFMoto는 "CF Moto"로 요청. (현재는 DB 기반 조회로 전환됨)
- **백엔드 시크릿 (2026-07-14 env var 이관 완료)**: `application-local.yml`의 openweather/opinet/api-ninjas 키를 `${VAR:dummy}` 형태로 이관. 로컬 실행 시 IntelliJ Run Config 또는 shell env에 `OPENWEATHER_API_KEY`, `OPINET_API_KEY`, `API_NINJAS_API_KEY` 설정 필요. 이전 커밋 히스토리엔 실제 키 남아있으나 사용자 판단으로 회전 없이 진행. dev/stg/prd yml은 이전부터 이미 env var 사용 중. (원칙: 새 API 키는 git 커밋 금지)
- 백엔드 원격: `https://github.com/jyleeeeeeeeee/bikeridediary_be.git` (2026-07-14 `bikeridediary` → `bikeridediary_be` 갱신 완료)
