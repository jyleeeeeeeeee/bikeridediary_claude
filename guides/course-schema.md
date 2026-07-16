# 라이딩 코스 도메인 — DB 스키마 가이드

> dba 에이전트 산출. 사용자가 `brd_be/src/main/resources/schema.sql`, `data.sql`에 직접 반영.

---

## 목표

라이딩 코스 도메인 3개 테이블(courses, course_waypoints, course_favorites)을 schema.sql에 추가하고, place/place_categories DDL도 함께 보완하여 schema.sql을 완성된 상태로 만든다.

---

## 파일 수정 목록

| 파일 | 신규/수정 | 설명 |
|------|----------|------|
| `brd_be/src/main/resources/schema.sql` | 수정 | places, place_categories, place_wishes, courses, course_waypoints, course_favorites 테이블 + 인덱스 추가 |
| `brd_be/src/main/resources/data.sql` | 수정 검토 | place_categories 시드 추가 필수. 코스 초기 데이터는 사용자 판단 (MVP는 생략 권장) |

---

## 현황 파악

`schema.sql`을 보면 현재 places, place_categories, place_wishes 3개 테이블이 **누락**되어 있습니다. PlaceEntity와 PlaceCategoryEntity는 이미 백엔드에 구현되어 있고 DB에서 직접 seed된 상태로 운영 중인 것으로 추정됩니다. 코스 DDL 추가 전에 이 3개를 schema.sql에 먼저 채워 넣어야 `CREATE TABLE IF NOT EXISTS` 방식이라도 재실행 시 안전합니다.

---

## DDL 스니펫

### 1. schema.sql 헤더 주석 갱신

```sql
-- ============================================================
-- BikeRideDiary (바라다) PostgreSQL 스키마
-- 대상 Entity: UserEntity, BikeEntity, MaintenanceEntity,
--              MaintenanceScheduleEntity, FuelingEntity,
--              ManufacturerEntity, BikeModelEntity,
--              PlaceCategoryEntity, PlaceEntity,
--              CourseEntity, CourseWaypointEntity, CourseFavoriteEntity
-- RefreshToken은 Redis에 저장되므로 PostgreSQL 스키마에 포함하지 않음
-- ============================================================
```

### 2. schema.sql에 추가할 블록 (기존 8번 인덱스 섹션 앞에 삽입)

```sql
-- ============================================================
-- 8. place_categories (장소 카테고리 마스터)
-- ============================================================
CREATE TABLE IF NOT EXISTS place_categories (
    category_code VARCHAR(50) PRIMARY KEY,
    category_name VARCHAR(50)  NOT NULL,
    display_order INTEGER      NOT NULL DEFAULT 0,
    created_at    TIMESTAMP    NOT NULL DEFAULT now(),
    updated_at    TIMESTAMP
);

-- ============================================================
-- 9. places (라이더 큐레이션 POI)
-- ============================================================
CREATE TABLE IF NOT EXISTS places (
    id             UUID          DEFAULT gen_random_uuid() PRIMARY KEY,
    place_name     VARCHAR(100)  NOT NULL,
    user_id        UUID          REFERENCES users(id) ON DELETE SET NULL,
    star_point     REAL,
    wished_count   INTEGER       NOT NULL DEFAULT 0,
    category_code  VARCHAR(50)   NOT NULL REFERENCES place_categories(category_code) ON DELETE RESTRICT,
    latitude       NUMERIC(9,7)  NOT NULL,
    longitude      NUMERIC(10,7) NOT NULL,
    address        VARCHAR(200),
    road_address   VARCHAR(200),
    description    TEXT,
    photo_url      VARCHAR(500),
    phone          VARCHAR(30),
    kakao_place_id VARCHAR(50),
    naver_place_id VARCHAR(50),
    created_at     TIMESTAMP     NOT NULL DEFAULT now(),
    updated_at     TIMESTAMP,
    deleted_at     TIMESTAMP
);

-- ============================================================
-- 10. place_wishes (장소 찜)
-- ============================================================
CREATE TABLE IF NOT EXISTS place_wishes (
    place_id   UUID      NOT NULL REFERENCES places(id) ON DELETE CASCADE,
    user_id    UUID      NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMP NOT NULL DEFAULT now(),

    PRIMARY KEY (place_id, user_id)
);

-- ============================================================
-- 11. courses (라이딩 코스)
-- ============================================================
CREATE TABLE IF NOT EXISTS courses (
    id               UUID          DEFAULT gen_random_uuid() PRIMARY KEY,
    -- user_id nullable: 시드/큐레이션 코스는 작성자 없음 (관리자가 seed로 넣은 코스)
    -- ON DELETE SET NULL: 유저 탈퇴 시 코스 콘텐츠는 유지
    user_id          UUID          REFERENCES users(id) ON DELETE SET NULL,
    name             VARCHAR(100)  NOT NULL,
    distance_meters  INTEGER       NOT NULL,
    path             TEXT          NOT NULL,
    is_public        BOOLEAN       NOT NULL DEFAULT TRUE,
    -- 원본 코스 참조 — 원본 hard delete 시 SET NULL로 참조만 끊고 파생 코스는 유지
    source_course_id UUID          REFERENCES courses(id) ON DELETE SET NULL,
    created_at       TIMESTAMP     NOT NULL DEFAULT now(),
    updated_at       TIMESTAMP
    -- deleted_at 없음: courses는 hard delete 정책 (waypoints/favorites CASCADE 자동 삭제)
);

-- ============================================================
-- 12. course_waypoints (코스 경유지)
-- ============================================================
CREATE TABLE IF NOT EXISTS course_waypoints (
    id         UUID          DEFAULT gen_random_uuid() PRIMARY KEY,
    course_id  UUID          NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    seq        SMALLINT      NOT NULL,
    role       VARCHAR(10)   NOT NULL,
    -- 등록된 place 참조 (옵셔널) — 임의 지점(지도 롱프레스, GPX 임포트)은 NULL
    -- ON DELETE SET NULL: place 삭제되어도 좌표 스냅샷이 남아 코스는 유효
    place_id   UUID          REFERENCES places(id) ON DELETE SET NULL,
    -- 좌표/이름은 스냅샷으로 저장 (place 수정/삭제에도 코스는 그때 그대로 유지)
    name       VARCHAR(100),
    latitude   NUMERIC(9,7)  NOT NULL,
    longitude  NUMERIC(10,7) NOT NULL,

    CONSTRAINT chk_waypoint_role CHECK (role IN ('START', 'VIA', 'END')),
    CONSTRAINT uq_waypoint_course_seq UNIQUE (course_id, seq)
);

-- ============================================================
-- 13. course_favorites (코스 즐겨찾기)
-- ============================================================
CREATE TABLE IF NOT EXISTS course_favorites (
    course_id  UUID      NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    user_id    UUID      NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMP NOT NULL DEFAULT now(),

    PRIMARY KEY (course_id, user_id)
);
```

> **updated_at NOT NULL 여부**: 기존 테이블(maintenances 등)과 일관성을 위해 `TIMESTAMP`(nullable)로 두는 것을 권장 (위 DDL 반영됨).

### 3. schema.sql 인덱스 섹션에 추가

```sql
-- places: findByDeletedAtIsNullOrderByPlaceCategoryEntity_DisplayOrderAsc
CREATE INDEX IF NOT EXISTS idx_places_category_deleted_at
    ON places (category_code, deleted_at);

-- places: 좌표 반경 검색 (PostGIS 도입 전 NUMERIC 기반, 삭제된 레코드 제외)
CREATE INDEX IF NOT EXISTS idx_places_lat_lng
    ON places (latitude, longitude)
    WHERE deleted_at IS NULL;

-- place_wishes: MY탭에서 user 기준 조회 (PK가 (place_id, user_id)라 user 단독 인덱스 별도 필요)
CREATE INDEX IF NOT EXISTS idx_place_wishes_user_id
    ON place_wishes (user_id);

-- courses: 내 코스 조회 (user_id 기준)
CREATE INDEX IF NOT EXISTS idx_courses_user_id
    ON courses (user_id);

-- courses: 탐색 목록 - 공개 코스만 (partial index)
CREATE INDEX IF NOT EXISTS idx_courses_public
    ON courses (is_public)
    WHERE is_public = TRUE;

-- courses: 최신순 정렬 지원
CREATE INDEX IF NOT EXISTS idx_courses_updated_at
    ON courses (updated_at DESC)
    WHERE is_public = TRUE;

-- course_favorites: user 기준 MY탭 조회 (PK가 course_id 시작이라 user 단독 인덱스 필요)
CREATE INDEX IF NOT EXISTS idx_course_favorites_user_id
    ON course_favorites (user_id);

-- course_waypoints: place 역방향 조회 (특정 place를 지나는 코스 찾기 - 커뮤니티 확장 대비)
-- place_id IS NOT NULL인 row만 포함하는 partial index
CREATE INDEX IF NOT EXISTS idx_course_waypoints_place_id
    ON course_waypoints (place_id)
    WHERE place_id IS NOT NULL;
```

### 4. DROP 순서 (FK 역순)

```sql
DROP TABLE IF EXISTS course_favorites;
DROP TABLE IF EXISTS course_waypoints;
DROP TABLE IF EXISTS courses;
DROP TABLE IF EXISTS place_wishes;
DROP TABLE IF EXISTS places;
DROP TABLE IF EXISTS place_categories;
DROP TABLE IF EXISTS bike_models;
DROP TABLE IF EXISTS manufacturers;
DROP TABLE IF EXISTS fuelings;
DROP TABLE IF EXISTS maintenance_schedules;
DROP TABLE IF EXISTS maintenances;
DROP TABLE IF EXISTS bikes;
DROP TABLE IF EXISTS users;
```

### 5. data.sql에 place_categories 시드 추가 (필수)

FK 순서상 places INSERT보다 먼저:

```sql
-- ============================================================
-- 0. 장소 카테고리 초기 데이터
-- ============================================================
INSERT INTO place_categories (category_code, category_name, display_order) VALUES
('CAFE','카페',1,null,null,null),
('FAMOUS','명소',2,null,null,null),
('RESTAURANT','식당',3,null,null,null),
('SERVICE','센터',4,null,null,null),
('OTHER','기타',9999,null,null,'2026-07-15 07:18:09.831278')

ON CONFLICT (category_code) DO NOTHING;
```

---

## UNIQUE 제약 & CHECK 근거

- **courses.name — UNIQUE 미적용**: 다른 유저가 같은 이름 코스 만들 수 있음, 복사 편집 시 이름 유지 자연스러움
- **course_waypoints (course_id, seq) — UNIQUE 적용**: seq 중복 시 순서 깨짐. UNIQUE 제약이 B-tree 인덱스를 겸함
- **course_waypoints.role CHECK**: 로직 버그로 잘못된 role 저장 방지

## place 연결 설계 (스냅샷 패턴)

`course_waypoints.place_id`는 **옵셔널 참조**. 좌표(latitude/longitude)와 이름은 waypoint에도 저장 (place 데이터 스냅샷).

**왜 좌표 중복 저장하나**:
1. place 수정 대응: place 이름/좌표가 바뀌어도 코스는 등록 시점 그대로 유지 (불변성)
2. place 삭제 대응: FK만 끊고 waypoint 좌표는 살아있음 → 코스 여전히 유효
3. place 없는 지점 지원: 지도 롱프레스, 네이버 검색, GPX 임포트 등

**얻는 이점**:
- UI 딥링크: place_id 있으면 "이 카페 상세로 이동"
- place 추천: 코스 생성 시 근처 등록된 place 후보 노출
- 역방향 통계: "이 카페 지나는 코스 목록" (커뮤니티 확장 시)
- 아이콘 자동: place.category로 waypoint 마커 색상/아이콘 결정

---

## FK ON DELETE 정책

| FK | 정책 | 근거 |
|----|------|------|
| `courses.user_id → users(id)` | SET NULL | 유저 탈퇴 시 코스 콘텐츠는 유지 (커뮤니티 자산). 또한 **시드/큐레이션 코스는 최초부터 user_id = NULL** (관리자가 seed로 넣은 코스) |
| `courses.source_course_id → courses(id)` | SET NULL | 원본 hard delete 시 파생 코스는 독립 유지 (source 참조만 NULL로) |
| `course_waypoints.course_id → courses(id)` | CASCADE | 코스 hard delete 시 waypoints도 함께 삭제 |
| `course_waypoints.place_id → places(id)` | SET NULL | place 삭제되어도 waypoint 좌표 스냅샷은 살아있어 코스 유효. 딥링크만 끊김 |
| `course_favorites.*` | CASCADE | 코스/유저 hard delete 시 즐겨찾기도 함께 삭제 |

## 삭제 정책 (2026-07-16 확정)

**Hard delete**. 소프트 삭제(`deleted_at`) 사용 안 함.
- 작성자가 코스 삭제 → CASCADE로 waypoints + favorites 자동 삭제 (즐겨찾기한 다른 사용자에게서도 사라짐)
- 파생 코스(source_course_id 참조)는 유지되며 source_course_id만 NULL로 세팅
- 오직 작성자 본인만 삭제 가능 (Service 계층 소유권 검증)
- 시드/큐레이션 코스(user_id=NULL)는 일반 사용자가 삭제 불가

---

## 인덱스 설계 근거

| 인덱스 | 커버 쿼리 | 근거 |
|--------|----------|------|
| `idx_courses_user_id` | `findByUserEntityIdOrderByCreatedAtDesc` | 내 코스 목록 |
| `idx_courses_public` (partial) | 탐색 목록 전체 | is_public=true만 포함 → 인덱스 크기 대폭 축소 |
| `idx_courses_updated_at` (partial) | `ORDER BY updated_at DESC` | 최신순 정렬 |
| `uq_waypoint_course_seq` UNIQUE | 상세 조회 seq 정렬 | UNIQUE 제약이 인덱스 겸함 |
| `idx_course_waypoints_place_id` (partial) | place 역방향 조회 | place_id IS NOT NULL만 인덱스 → 대부분이 place 미참조여도 크기 최소 |
| `idx_course_favorites_user_id` | MY탭 즐겨찾기 조회 | PK가 (course_id, user_id) 시작이라 user 단독 조회 커버 불가 |

---

## backend-dev에게 전달할 사항

### CourseWaypointEntity 필드 필수

`latitude`, `longitude`를 반드시 `BigDecimal` + `@Column(precision, scale)` 명시. 안 하면 Hibernate 기본값 `NUMERIC(19,2)`가 적용되어 소수점 2자리로 잘림 (PlaceEntity에서 발생한 버그 재발 방지).

```java
@Column(name = "latitude",  nullable = false, precision = 9,  scale = 7)
private BigDecimal latitude;

@Column(name = "longitude", nullable = false, precision = 10, scale = 7)
private BigDecimal longitude;
```

### Repository 메서드 명명 힌트

```java
// idx_courses_user_id 활용 — hard delete라 deleted_at 조건 없음
List<CourseEntity> findByUserEntityIdOrderByCreatedAtDesc(UUID userId);

// idx_courses_public 활용
List<CourseEntity> findByIsPublicTrueOrderByCreatedAtDesc();

// uq_waypoint_course_seq 활용
List<CourseWaypointEntity> findByCourseEntityIdOrderBySeqAsc(UUID courseId);

// idx_course_favorites_user_id 활용
List<CourseFavoriteEntity> findByUserId(UUID userId);

// existsBy - count보다 빠름
boolean existsByIdCourseIdAndIdUserId(UUID courseId, UUID userId);
```

---

## PostGIS 도입 시 마이그레이션 방향

**courses.path 컬럼**: MVP `TEXT`(JSON) → 도입 시 아래 ALTER

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
ALTER TABLE courses ADD COLUMN path_geom GEOMETRY(LINESTRING, 4326);
-- 마이그레이션 후:
-- ALTER TABLE courses DROP COLUMN path;
-- ALTER TABLE courses RENAME COLUMN path_geom TO path;
CREATE INDEX IF NOT EXISTS idx_courses_path_geom ON courses USING GIST (path_geom);
```

**course_waypoints.latitude/longitude**: `NUMERIC(9,7)/(10,7)` 유지 무방 (표시/입력용).

PostGIS 도입 시점은 코스 도메인 본격 개발 시작 때 한 번에 처리 권장.

---

## seed 데이터 (코스)

**MVP 단계에서 코스 초기 시드 생략 권장.** 이유:
- path(좌표 배열) 필수라 수작업 INSERT 비용 높음
- GPX 업로드나 Directions API 완성 후 실제 라이딩으로 채우는 게 현실적
- 개발용 더미 코스는 `POST /api/v1/courses` API 완성 후 스크립트 insert가 유지보수 유리

---

## 체크리스트

- [ ] `place_categories` INSERT가 `places` INSERT보다 앞
- [ ] `courses` INSERT가 waypoints/favorites보다 앞
- [ ] `uq_waypoint_course_seq` UNIQUE 이후 `(course_id, seq)` 별도 인덱스 없음
- [ ] `idx_courses_public`, `idx_courses_updated_at` partial WHERE 절 동일
- [ ] CourseWaypointEntity `@Column(precision=9/10, scale=7)` 명시
- [ ] `place_categories` 시드에 `OTHER` 포함
- [ ] DROP 순서 FK 역순

---

## 확인 방법

```bash
docker compose down -v
docker compose up postgres
```

볼륨 제거 후 재기동 → init 스크립트 재실행 → 오류 없으면 OK. 또는:

```bash
psql -U postgres -d bikeridediary -f brd_be/src/main/resources/schema.sql
```
