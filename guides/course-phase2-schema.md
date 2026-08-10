# 라이딩 코스 2차 스코프 - 스키마 변경 결정

작성일: 2026-08-04  
담당: dba  
근거 계획: `plans/riding-course-phase2.md` (사용자 승인 D1~D7 전부 추천안 채택)

---

## 결정 요약

**schema.sql 변경 없음. 현행 스키마로 2차 스코프(코스 생성/편집) 전체를 지원한다.**

승인된 결정이 스키마에 미치는 영향:
- D1=A (path TEXT 현행 유지) -> 컬럼 변경 없음
- D2=A (distance_meters만 저장) -> 컬럼 추가 없음
- D3=A (place_id 옵셔널 FK, 현행 그대로) -> 스키마 이미 충족
- D4~D7 -> 앱 UI/API 흐름 결정이라 스키마 무관

---
## 검증 결과

### 1. courses 테이블 (schema.sql 라인 232-248)

| 컬럼 | 타입 | 2차 스코프 필요 | 상태 |
|------|------|----------------|------|
| `id` | UUID PK | 클라이언트 UUID | 존재 |
| `user_id` | UUID nullable FK -> users | 작성자 (시드 코스는 null 허용) | 존재, ON DELETE SET NULL |
| `name` | VARCHAR(100) NOT NULL | 코스 이름 | 존재 |
| `distance_meters` | INTEGER NOT NULL | Directions summary에서 파싱해 저장 | 존재 |
| `path` | TEXT NOT NULL | `[[lng,lat],...]` JSON 문자열 | 존재 |
| `is_public` | BOOLEAN NOT NULL DEFAULT TRUE | 공개/비공개 | 존재 |
| `source_course_id` | UUID FK -> courses(id) SET NULL | 복사편집 파생 코스 추적 | 존재 |
| `created_at`, `updated_at` | TIMESTAMP | 감사 필드 | 존재 |

엔티티(`CourseEntity.java`) 필드 대응 확인:
- `path`: `@Column(name="path", columnDefinition="TEXT")` - 스키마 TEXT와 일치
- `distanceMeters`: `@Column(name="distance_meters")` - Integer 타입, 컬럼 INTEGER와 일치
- `sourceCourseId`: `@Column(name="source_course_id")` - UUID, 자기참조 FK 정상
- `isPublic`: `@Column(name="is_public", nullable=false)` - 일치

D2=A 채택으로 `duration_seconds`, `toll_fare`, `summary_json` 컬럼 **추가 없음**.

### 2. course_waypoints 테이블 (schema.sql 라인 255-271)

| 컬럼 | 타입 | 확인 사항 | 상태 |
|------|------|----------|------|
| `id` | UUID PK | 서버 생성 UUID | 존재 |
| `course_id` | UUID NOT NULL FK -> courses(id) CASCADE | 코스 삭제 시 waypoint 함께 삭제 | 존재, CASCADE 확인 |
| `seq` | SMALLINT NOT NULL | 0-based 순서 인덱스, 재배열 시 서버가 재계산 | 존재 |
| `role` | VARCHAR(10) NOT NULL | CHECK IN ('START', 'VIA', 'GOAL') — Naver Directions `goal` 파라미터와 통일 | **END→GOAL 마이그레이션 필요** |
| `place_id` | UUID nullable FK -> places(id) SET NULL | D3=A: 임의 지점은 null | 존재, SET NULL 확인 |
| `name` | VARCHAR(100) | 스냅샷 - place 수정/삭제 후에도 코스 유지 | 존재 |
| `latitude` | NUMERIC(9,7) NOT NULL | 약 1.1cm 해상도 | 존재 |
| `longitude` | NUMERIC(10,7) NOT NULL | 약 1.1cm 해상도 | 존재 |
| UNIQUE(course_id, seq) | 제약 | 같은 코스 내 seq 중복 방지 | 존재 (uq_waypoint_course_seq) |

엔티티(`CourseWaypointEntity.java`) 필드 대응 확인:
- `latitude`: `@Column(precision=9, scale=7)` - 스키마 NUMERIC(9,7) 일치. **precision/scale 명시됨**
- `longitude`: `@Column(precision=10, scale=7)` - 스키마 NUMERIC(10,7) 일치. **precision/scale 명시됨**
- `placeEntity`: `@JoinColumn(name="place_id", nullable=true)` - 옵셔널 FK 정상
- `role`: `@Column(nullable=false, length=20)` - CHECK 제약은 DB 레벨에 있음
- 팩토리 2개 존재: `create()`(임의 지점), `createWithPlace()`(place 스냅샷) - 2차 두 경로 모두 지원

### 3. course_favorites 테이블 (schema.sql 라인 276-283)

2차 스코프에서 favorites 로직 변경 없음. 복합 PK `(course_id, user_id)`로 정의됨.
엔티티: `@EmbeddedId CourseFavoriteId` + `@JdbcTypeCode(SqlTypes.UUID)` - Hibernate 6.x UUID batch 이슈 대응 완료.

### 4. 인덱스 현황

현재 courses 관련 인덱스 (schema.sql 라인 301-322):

| 인덱스 | 대상 컬럼 | 커버 쿼리 |
|--------|----------|----------|
| `idx_courses_user_id` | `(user_id)` | 내 코스 목록 조회 |
| `idx_courses_public` | `(is_public) WHERE is_public=TRUE` | 탐색 탭 공개 코스 목록 |
| `idx_courses_updated_at` | `(updated_at DESC) WHERE is_public=TRUE` | 공개 코스 최신순 |
| `idx_course_favorites_user_id` | `(user_id)` | MY탭 즐겨찾기 목록 |
| `idx_course_waypoints_place_id` | `(place_id) WHERE place_id IS NOT NULL` | place 역방향 조회 |

**2차에서 추가할 인덱스 없음.** `POST /courses`, `PATCH /courses/{id}` 쓰기 경로는 인덱스 영향 없음.

---

## 확장 시 절차

### D2 확장 - 소요시간/유료요금 컬럼 추가 시

트래픽이 붙어 소요시간 UI를 추가하는 시점에 아래 DDL을 `schema.sql` courses 섹션 끝에 추가한다.

```sql
-- D2 확장: Naver Directions 응답 요약 필드 추가 (2차에서는 보류)
ALTER TABLE courses ADD COLUMN IF NOT EXISTS duration_seconds INTEGER;
ALTER TABLE courses ADD COLUMN IF NOT EXISTS toll_fare        INTEGER;
```

엔티티에 추가할 필드:

```java
// 예상 소요 시간 (초 단위, Naver Directions summary.duration)
@Column(name = "duration_seconds")
private Integer durationSeconds;

// 유료도로 요금 (원 단위, Naver Directions summary.tollFare)
@Column(name = "toll_fare")
private Integer tollFare;
```

summary 전체를 보존하려면(D2=C로 전환) `summary_json TEXT` 컬럼 추가. 컬럼에 인덱스를 태울 계획이 없으면 TEXT로 충분하고 JSONB로 만들 필요 없다.

### path 인덱싱 - 현재 없음, 향후 검토

현재 `courses.path` 에 인덱스가 없다. TEXT 컬럼이므로 일반 B-tree 인덱스는 도움이 안 되고, GIN 인덱스는 JSONB 타입에서만 유효하다.

- **현재 방침**: path는 SELECT 대상이지 검색 조건이 아니므로 인덱스 불필요.
- **PostGIS 도입 시**: path를 `GEOMETRY(LINESTRING, 4326)` 컬럼으로 마이그레이션하면 GIST 인덱스로 공간 쿼리(경로 근처 주유소, 반경 내 코스 검색) 지원 가능. 이 마이그레이션은 places 좌표 컬럼과 함께 단일 사이클로 처리할 것.
- **path 컬럼을 JSONB로 전환 시**: `ALTER TABLE courses ALTER COLUMN path TYPE JSONB USING path::jsonb` 후 GIN 인덱스 가능. 단 읽기 성능 이득은 제한적이고 저장 크기 증가, 마이그레이션 위험이 있어 현재는 유보.

### waypoint 15개 상한 - DB 강제 없음

Naver Directions 15의 하드 리밋은 경유지 15개(출발지/도착지 제외)다. DB에 CHECK 제약으로 강제하지 않는다. 이유:

1. `UNIQUE (course_id, seq)` 제약이 있어 seq 범위로 개수를 제한하려면 트리거가 필요하다. 트리거는 Spring Boot init 스크립트와 궁합이 나쁘다(PostgreSQL 함수 별도 생성 필요).
2. 강제는 앱 UI(15개 도달 시 추가 버튼 비활성화)와 백엔드 서비스 레이어(`if (waypoints.size() > 15) throw`)로 충분하다.

backend-dev 사항: `CourseService.createCourse()`에서 VIA 개수 상한 검증 추가 (VIA ≤ 15, `viaCount > 15` 체크). `ErrorCode.COURSE_DIRECTIONS_WAYPOINTS_LIMIT` 신규 추가 (backend 가이드 참조).

---
## 리스크

### 1. 장거리 코스 path 크기

- 100km 코스: Directions 응답 경로 좌표 약 1,000~3,000개 -> JSON 약 50~150KB
- 500km 코스: 5,000~15,000개 좌표 -> 250~750KB
- PostgreSQL TEXT 컬럼은 자동 TOAST(임계치 2KB 초과 시 외부 저장) 처리되므로 저장 자체에 크기 제한 없음.
- 문제는 **목록 API에서 path까지 SELECT될 경우**: 목록 쿼리가 전체 엔티티를 로드하면 path까지 불러온다.

대응 방향 (2차 구현 시 backend-dev 적용):

```java
// 목록 조회용 Projection - path 제외
public interface CourseListView {
    UUID getId();
    String getName();
    Integer getDistanceMeters();
    Boolean getIsPublic();
    LocalDateTime getUpdatedAt();
}
// 상세 조회에서만 CourseEntity 전체 로드
```

또는 `@Basic(fetch = FetchType.LAZY)` 조합. 단 Hibernate 필드 레벨 lazy는 바이트코드 향상이 필요하므로 Projection이 더 현실적이다.

### 2. place_id가 null인 waypoint 편집 시 처리

편집 화면 진입 시 `place_id` -> null 된 waypoint는 좌표/이름 스냅샷으로만 남아 있다. 앱은 임의 지점으로 취급해야 한다.

판단 기준:
- `placeId == null` -> 임의 지점 UI로 표시
- `placeId != null` -> place 선택 지점 UI로 표시

백엔드: 상세 조회 응답에 `waypoints[].placeId` null이면 앱이 임의 지점 취급. 별도 스키마 변경 없음.

### 3. PATCH 시 waypoints 교체 전략

2차 스코프에서는 **전체 교체** 방식을 권장한다. waypoints 수가 최대 17개라 성능 부담 없고, seq 재배열이 자주 일어나는 구조라 diff가 복잡도 대비 이득이 없다.

`UNIQUE (course_id, seq)` 제약이 있으므로 교체 순서 주의:
1. 기존 waypoints 전체 DELETE
2. 새 waypoints INSERT (seq 0부터 재할당)

JPA 활용 시: `courseWaypointRepository.deleteByCourseEntity(courseEntity)` 후 `saveAll(newWaypoints)`.

### 4. distance_meters 타입 주의

스키마: `INTEGER` / 엔티티: `Integer`
Naver Directions summary.distance 단위는 미터(m), 응답 타입은 int.
최대 가능 거리: 약 2,100km = 2,100,000m. `Integer.MAX_VALUE`(약 2.1억 m)로 충분. BIGINT로 변경 불필요.

---

## 체크리스트

- [x] `courses.path TEXT NOT NULL` 존재 확인
- [x] `courses.distance_meters INTEGER NOT NULL` 존재 확인
- [x] `courses.source_course_id FK ON DELETE SET NULL` 존재 확인
- [x] `course_waypoints.place_id FK ON DELETE SET NULL` 존재 확인
- [x] `course_waypoints.role CHECK IN ('START','VIA','END')` 존재 확인 — **END→GOAL 리네임 필요** (backend 가이드 참조)
- [x] `UNIQUE (course_id, seq)` 존재 확인
- [x] `CourseWaypointEntity.latitude/longitude` `@Column(precision, scale)` 명시 확인
- [x] D2=A 채택으로 추가 컬럼 없음
- [x] 15개 상한 DB 강제 없음 - 서비스 레이어 강제 권장
- [ ] 목록 API에서 path 제외 Projection 적용 (backend-dev 2차 구현 시)
- [ ] `ErrorCode.COURSE_DIRECTIONS_WAYPOINTS_LIMIT` 추가 (backend-dev 구현 시)

---

## backend-dev에게 넘길 사항

1. **목록 조회 Projection**: `GET /courses/my`, `GET /courses`(탐색) 응답에서 `path` 필드 제외. `CourseListView` 인터페이스 또는 DTO 생성자 쿼리 사용. path 크기가 수백KB일 수 있어 목록에서 전송하면 응답 크기/DB 부담 모두 있음.

2. **VIA 개수 상한 검증**: `CourseService.createCourse()` 및 `updateCourse()` 진입 시 VIA `viaCount > 15` 체크 (Naver Directions 15 스펙: waypoints 파라미터는 VIA만 세며 최대 15개). `ErrorCode.COURSE_DIRECTIONS_WAYPOINTS_LIMIT` 추가 (메시지: "경유지는 최대 15개까지 지정할 수 있습니다").

3. **PATCH waypoints 교체 순서**: `courseWaypointRepository.deleteByCourseEntity(courseEntity)` 후 새 waypoints `saveAll`. `UNIQUE (course_id, seq)` 제약으로 인해 DELETE 먼저 후 INSERT 순서 필수.

4. **regeneratePath 플래그 처리**: PATCH 요청 DTO에 `boolean regeneratePath` 포함. `false`이면 `name`, `isPublic`만 업데이트하고 Directions API 호출 및 waypoints/path/distanceMeters 갱신 생략. `true`이면 waypoints 교체 + Directions 호출 + path/distanceMeters 갱신.

5. **Naver Directions 응답 파싱**: `route.trafast[0].summary.distance` -> `distanceMeters`(Integer). `route.trafast[0].path` -> `List<List<Double>>` -> `objectMapper.writeValueAsString()` -> TEXT 저장.
