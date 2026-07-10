# Claude 작업 메모리 (git 동기화)

> 이 파일은 `CLAUDE.md`에서 `@claude-memory.md`로 import되어 세션 시작 시 자동 로드됩니다.
> Claude 홈(`~/.claude`)의 메모리는 git 공유가 안 되므로, 정본은 이 파일에 둡니다.
> 다른 환경에서도 git pull 하면 그대로 읽힙니다. 최종 갱신: 2026-06-30.

---

## 🚨 프로젝트 일시 중단 (2026-06-30)

- 바라다(brd_be/brd_app) 작업 **일시 보류**.
- 분기: 뱅킹각 측정 기능만 따로 분리해 단독 앱 시작 → **CheckBanking** (`C:\Users\jyl93\check_banking`, 원래 D:였으나 Kotlin cross-drive 이슈로 이동).
- 분리 이유: 센서 기반 정확도 검증이 MVP의 핵심이므로 다른 도메인과 섞지 않고 집중 개발.
- 복귀 시점 미정. 뱅킹각 앱 검증 후 본 바라다에 라이딩 퍼포먼스 기능으로 다시 합칠 가능성 있음.
- 작업할 때 디렉터리를 `C:\Users\jyl93\check_banking`으로 옮겨서 진행 (해당 프로젝트 자체 CLAUDE.md/claude-memory.md 존재).

---

## 작업 방식 / 피드백 (반드시 준수)

- **메모리/기록 저장 위치 = 이 파일**: 사용자가 "진행 상황 기록해 / 기억해 / 메모리에 저장해" 등을 요청하면, 홈(`~/.claude`)의 메모리가 아니라 **이 파일(`brd_be/claude-memory.md`)에 기록하고 git 커밋**할 것 (다른 환경 공유를 위해). 완료 작업 로그성 내용은 `CLAUDE.md`의 "현재 개발 단계"에 추가.
- **퍼미션 확인 금지**: 파일 수정·Bash·프로세스 종료 등 모든 작업을 확인 없이 바로 실행. 위험 작업(force push 등)도 사용자가 요청했으면 바로. 사용자가 매번 확인받는 걸 매우 싫어함.
- **앱은 가이드만 받지 않고 자율 완성형 구현**: Flutter/앱 코드는 강의식(Learn by Doing, TODO(human), 교육적 중단) 금지. 설명 요청이 아닌 한 바로 완성형으로. Insight는 사용자가 관심 가질 것만 간결히.
- **역할 분담**: 앱(brd_app)은 Claude가 구현, 백엔드(brd_be)는 Claude가 **가이드만** 주고 사용자가 직접 구현. (단 사용자가 명시적으로 "반영/수정해"라고 하면 백엔드도 직접 수정)
- **"커밋해" = 두 프로젝트 모두**: brd_be + brd_app 양쪽 git status 확인 후 변경된 모든 repo 커밋.
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
- 사용자 초안 스키마: place(place_id, place_name, place_author, star_point, wished_count) + place_categories(category_code, category_name)
- 미완: 좌표(lat/lng), category_code FK, address, description, photo_url, deleted_at 등 필수 컬럼 보완 필요
- 리뷰/후기: 카카오/네이버 리뷰 API 없음. 크롤링 법적 리스크. **자체 리뷰 시스템 + AI 요약** 방향 검토, 결정 유보
- 딥링크 방향: 상세 시트에 카카오맵/네이버 지도 딥링크 버튼 (url_launcher), 후속 작업
- 주유소 카테고리: 지도 UI 토글엔 있으나 데이터 소스 미연결. 기존 station API(오피넷)와 통합 필요
- 확장 메뉴 3버튼 (main_shell): 좌 코스탐색(/courses) / 중 내 바이크(navigationShell.goBranch(2)) / 우 뱅킹각(/banking)

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
- **백엔드 시크릿 주의**: `brd_be/src/main/resources/application-local.yml`이 git 추적 중이고 `opinet.api-key`, `api-ninjas.api-key`가 실제 키. 이전 커밋부터 원격에 올라가 있음. public이면 키 회전 + 환경변수 분리 권장. (원칙: API 키는 git에 올리지 않음)
- 백엔드 원격 이전: `bikeridediary` → `bikeridediary_be.git`. `git remote set-url` 갱신 권장.

---
---

# 프로젝트: CheckBanking (뱅킹각 측정 단독 앱)

> 원본: `C:\Users\jyl93\check_banking\claude-memory.md`. 최종 갱신: 2026-06-30 (UI 다듬기 끝, 실기기 빌드 직전).

## 작업 방식 / 피드백 (반드시 준수)

- **메모리 저장 위치 = 이 파일** (`C:\Users\jyl93\check_banking\claude-memory.md`). 홈(~/.claude) 아님.
- **퍼미션 확인 금지** — 파일 수정/Bash/프로세스 종료 등 바로 실행.
- **앱 코드는 자율 완성형** — Learn by Doing/TODO(human) 강의식 금지.
- **센서 알고리즘 변경 시 한 줄 설명** — 왜 그 선택인지 (학습 중인 영역이라).

---

## 프로젝트 진행 상황

### 시작 (2026-06-30)
- 모(母) 프로젝트 바라다(`C:\Users\jyl93\bikeridediary`)에서 라이딩 퍼포먼스(뱅킹각) 기능만 분리.
- 분리 사유: 센서 정확도 검증이 별도 스프린트 가치가 있음. 다른 도메인과 섞으면 디버깅 어려움.
- 첫 세션에 MVP 전체 구현 완료 (flutter analyze 0 issues).

### 프로젝트 위치 변경 (2026-06-30)
- 원래 `D:\check_banking` → **`C:\Users\jyl93\check_banking`**로 이동.
- 이유: Kotlin 증분 컴파일러의 `RelocatableFileToPathConverter`가 프로젝트(D:)와 pub 캐시(C:) 간 cross-drive 상대경로 계산 실패하는 알려진 이슈. 같은 드라이브 위치로 이동해 해결.
- 부수 효과: `kotlin.incremental=false` 임시 플래그 원복 가능 (현재 원복 완료).
- **D:\check_banking 원본은 사용자가 수동 삭제** 필요 (Claude Code 하네스가 D:\ 루트 직속 폴더 삭제를 안전장치로 거부).

### MVP 구현 결정 사항
- **가속도계 only** (자이로 미사용): 자이로 통합은 드리프트 누적되어 라이딩 1시간 이상 시 부정확. 가속도계 + 저역통과 필터로 시작해보고 부족하면 complementary filter(가속도+자이로) 도입.
- **필터 alpha = 0.08**: 50Hz 가속도계에서 cutoff 약 5Hz. 엔진 진동(보통 20~40Hz) 거르고 라이딩 동작(0.5~3Hz)은 통과. 실기기 검증 후 0.05~0.15 사이에서 튜닝 필요.
- **메모리 버퍼 → 종료 시 일괄 저장**: 50Hz × 1시간 ≈ 2.9MB(double+int=16B). 디스크 IO 매 샘플 부담보다 메모리 안전. transaction + batch insert.
- **포트레이트 거치 가정**: 핸들바/탱크 마운트 표준. atan2(gx, √(gy²+gz²))로 피치 영향 최소화. 다른 거치 방향은 향후 옵션화.

### UI 진화 기록 (2026-06-30 세션)
- **초기 게이지**: 큰 숫자 한가운데 + 좌/우 화살표 → 너무 평면적이라 폐기
- **타코미터 게이지로 전환**: CustomPaint + SweepGradient + 눈금 + 빨간 바늘. 자동차 RPM 스타일.
- **부호 컨벤션 수정**: 폰 우측 엣지를 들면 좌측 뱅킹(음수)이 나오도록 `atan2` 결과 negate. (처음 구현 땐 부호가 반대로 나옴)
- **색 zone 임계값 정착**: 15° 초록 / 30° 노랑 / 45° 주황 / 그 이상 빨강. (실험: MotoGP 기준 40/50/55 시도했지만 일반 사용자 감각엔 너무 후함 — 15/30/45가 더 보수적이라 채택)
- **공통 함수 `bankingZoneColor(absAngle)`**: 게이지 호 그라데이션, 중앙 숫자, 좌/우 최대각 텍스트, 게이지 위 max marker가 모두 이 한 함수에서 색 결정 → 임계값 조정 1곳만 만지면 전체 일관
- **SweepGradient로 자연스러운 색 전환**: 4개 zone을 별도 호로 그리는 대신 9-stop SweepGradient 하나로 → 색 사이 부드럽게 블렌딩 + drawCall 1번으로 성능 ↑
- **눈금 = 바늘까지 연장된 방사형 스포크**: 외곽 짧은 눈금 → arc에서 needle tip까지 가는 긴 선. minor는 alpha 24%로 노이즈 줄임. major는 흰색 100% + 굵음
- **바늘 길이 1/3 축소**: `(radius - 28) * 2/3`. 짧은 바늘이 타코미터스러운 클래식한 비율
- **소수점 제거**: 모든 각도 표시 `toStringAsFixed(0)`. 센서 정밀도 ±2~5° 고려하면 소수점은 false precision

### 세션 녹화 UX 결정
- **샘플 수 표시 → 경과 시간(mm:ss)**: 사용자 입장에서 "지금 몇 분 녹화 중"이 더 유의미. sampleCount는 DB엔 남기되 UI에선 안 보임.
- **버튼 레이아웃**: 캘리브 배너 카드 → 폐기. `[기록 시작 | 각도 리셋]` 동일 너비 2버튼이 최대각 표시 카드 바로 아래에 위치. 녹화 중엔 `[중지(mm:ss) | 취소]`
- **각도 리셋 = 캘리브레이션**: 별도 큰 배너 없이 평상시 버튼 하나로 통합. SnackBar로 피드백.

### 미해결 / 검증 필요
- **실기기 검증 안 됨**: emulator는 센서 시뮬레이션이 제한적이라 의미 없음. 갤럭시 Z Flip3(공기계)에 빌드해서 거치 → 캘리브레이션 → 흔들기 → 실제 라이딩 순서로 검증 필요.
- **iOS 빌드 미검증**: Mac 없어서 Android만 빌드/테스트 가능. iOS는 자동화 가능 시점에 별도 처리.
- **wakelock_plus 작동 여부**: 라이딩 1시간 동안 화면 안 꺼지는지 실기기 확인 필요.
- **D:\check_banking 잔존**: 사용자가 수동 삭제 필요 (Claude Code 하네스가 D:\ 루트 폴더 삭제 차단).
- **GitHub 원격 미연결**: 현재 로컬 git만 있음(첫 커밋 `4b9cd40`). 다른 컴퓨터에서 작업하려면 GitHub repo 만들고 push 필요.

### Plugin KGP 경고
- `sensors_plus`, `wakelock_plus`가 옛 Kotlin Gradle Plugin 적용 방식. 빌드 시 경고만 뜨고 정상 동작. 1년 내 stable Flutter가 차단할 가능성 → 플러그인 업데이트 확인 필요(현재는 무시 OK).

### 패키지 ID 불일치 메모
- Android: `com.check_banking.app` (사용자 지정 그대로)
- iOS: `com.checkbanking.app` (Apple이 underscore 금지 → 자동 변환)
- 통일하려면 Android 쪽도 underscore 제거 권장. 현재는 분리 유지.

---

## 작업 환경 (CheckBanking)

- IDE: VS Code (Android Studio는 메모리 부담 커서 사용 안 함)
- 실기기: 삼성 갤럭시 Z Flip3 (공기계, SIM 없음)
- Flutter 3.44.1 / Dart 3.12.1
- Windows 11, PowerShell

### 실기기 연결
- USB 케이블 연결 → `adb devices`로 확인
- 게스트 Wi-Fi가 AP 격리라 ADB over WiFi 안 됨 → USB만 사용

---

## 모(母) 프로젝트 참고 (CheckBanking → 바라다)

- 바라다(brd_be + brd_app): `C:\Users\jyl93\bikeridediary`
- 라이딩 퍼포먼스 기능을 본 앱에 다시 합칠지는 CheckBanking 검증 후 결정
- 합치는 경우: 이 프로젝트의 `banking_provider.dart` + 알고리즘 통째로 이식 가능하도록 의존성 최소화 유지 (현재는 sensors_plus + 표준 라이브러리만)
