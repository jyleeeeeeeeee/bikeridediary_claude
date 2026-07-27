# 장소 승인 워크플로 — DB 스키마 (dba)

> 대상 파일: `brd_be/src/main/resources/schema.sql`
> 관련: `place-approval-workflow.md`, `place-approval-backend.md`
> Postgres 15+ (JSONB 필요)

---

## 1. users.role 컬럼 추가

`users` 테이블에 role 컬럼 추가. USER / ADMIN 두 값.

### schema.sql 반영

`users` CREATE TABLE 정의 안에 컬럼 추가:

```sql
CREATE TABLE IF NOT EXISTS users (
    id                UUID         DEFAULT gen_random_uuid() PRIMARY KEY,
    provider          VARCHAR(20)  NOT NULL,
    provider_id       VARCHAR(255) NOT NULL,
    email             VARCHAR(255),
    password          VARCHAR(255),
    nickname          VARCHAR(50)  NOT NULL,
    profile_image_url VARCHAR(255),
    fcm_token         VARCHAR(255),
    role              VARCHAR(20)  NOT NULL DEFAULT 'USER',  -- 신규
    created_at        TIMESTAMP    NOT NULL DEFAULT now(),
    updated_at        TIMESTAMP,
    deleted_at        TIMESTAMP,

    CONSTRAINT uq_users_provider UNIQUE (provider, provider_id),
    CONSTRAINT chk_users_role CHECK (role IN ('USER', 'ADMIN'))  -- 신규
);
```

### 기존 배포 환경 마이그레이션 (schema.sql 하단 "레거시 컬럼 정리" 섹션에 추가)

```sql
-- 2026-07-21: users.role 컬럼 추가 (승인 워크플로 도입)
ALTER TABLE users ADD COLUMN IF NOT EXISTS role VARCHAR(20) NOT NULL DEFAULT 'USER';
ALTER TABLE users DROP CONSTRAINT IF EXISTS chk_users_role;
ALTER TABLE users ADD CONSTRAINT chk_users_role CHECK (role IN ('USER', 'ADMIN'));
```

---

## 2. place_change_requests 테이블 신설

단일 테이블 + type 컬럼 + JSONB payload (D2=B 확정).

```sql
-- ============================================================
-- 15. place_change_requests (장소 변경 요청 큐)
-- ============================================================
-- 유저의 신규 장소 등록 요청 / 좌표 수정 / 정보 수정을 어드민 승인 큐로 관리.
-- 승인되면 places 테이블 반영 후 status=APPROVED, 거절되면 status=REJECTED.
-- (D6=A: 승인 클릭 트랜잭션 내에서 즉시 places 반영)
CREATE TABLE IF NOT EXISTS place_change_requests (
    id               UUID         DEFAULT gen_random_uuid() PRIMARY KEY,

    -- 요청 종류: CREATE / UPDATE_COORDINATES / UPDATE_INFO
    type             VARCHAR(30)  NOT NULL,

    -- 수정 대상 place (UPDATE_* 계열만 값 있음, CREATE는 NULL)
    -- ON DELETE CASCADE: place 삭제되면 관련 요청도 정리 (히스토리 유지 필요 시 SET NULL로 변경)
    target_place_id  UUID         REFERENCES places(id) ON DELETE CASCADE,

    -- 요청자 (NOT NULL). 유저 탈퇴 시 요청 히스토리도 사라져도 무방하므로 CASCADE.
    requester_id     UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- type별 payload (JSONB)
    --   CREATE:              { clientUuid, placeName, category, latitude, longitude,
    --                          address, roadAddress, description, phone, photoUrl }
    --   UPDATE_COORDINATES:  { latitude, longitude }
    --   UPDATE_INFO:         { placeName, category }
    payload          JSONB        NOT NULL,

    -- PENDING / APPROVED / REJECTED
    status           VARCHAR(20)  NOT NULL DEFAULT 'PENDING',

    -- 어드민이 남기는 승인/거절 사유
    review_note      TEXT,

    -- 검토한 어드민 (nullable, PENDING 상태에서는 NULL)
    reviewed_by      UUID         REFERENCES users(id) ON DELETE SET NULL,
    reviewed_at      TIMESTAMP,

    created_at       TIMESTAMP    NOT NULL DEFAULT now(),

    CONSTRAINT chk_pcr_type   CHECK (type IN ('CREATE', 'UPDATE_COORDINATES', 'UPDATE_INFO')),
    CONSTRAINT chk_pcr_status CHECK (status IN ('PENDING', 'APPROVED', 'REJECTED')),
    -- CREATE 요청은 target 없어야 하고, UPDATE_* 는 target 필수
    CONSTRAINT chk_pcr_target CHECK (
        (type = 'CREATE' AND target_place_id IS NULL) OR
        (type IN ('UPDATE_COORDINATES', 'UPDATE_INFO') AND target_place_id IS NOT NULL)
    )
);
```

## 3. 인덱스

```sql
-- 어드민 요청 목록: status=PENDING 기준 최신순 (부분 인덱스)
CREATE INDEX IF NOT EXISTS idx_pcr_status_created
    ON place_change_requests (status, created_at DESC)
    WHERE status = 'PENDING';

-- 요청자 본인 목록 조회
CREATE INDEX IF NOT EXISTS idx_pcr_requester
    ON place_change_requests (requester_id, created_at DESC);

-- 특정 place에 대한 PENDING 존재 여부 조회 (D8: 중복 방지)
CREATE INDEX IF NOT EXISTS idx_pcr_target_pending
    ON place_change_requests (target_place_id)
    WHERE status = 'PENDING' AND target_place_id IS NOT NULL;

-- 중복 방지 UNIQUE (부분): 같은 target_place_id + PENDING 조합 유일
-- UPDATE_COORDINATES와 UPDATE_INFO를 통틀어 target 하나에 대해 PENDING 1개만 허용
-- (뷰가 두 개 요청을 동시에 승인하면 순서 문제 있어 아예 UPDATE 계열 통틀어 1건 강제)
CREATE UNIQUE INDEX IF NOT EXISTS uq_pcr_target_pending
    ON place_change_requests (target_place_id)
    WHERE status = 'PENDING' AND target_place_id IS NOT NULL;
```

## 4. FK 정책 정리 (근거)

| FK | ON DELETE | 근거 |
|----|-----------|------|
| `target_place_id → places(id)` | CASCADE | place가 삭제되면 관련 요청도 무의미. 감사 로그 필요하면 SET NULL로 바꾸고 audit 테이블 별도 |
| `requester_id → users(id)` | CASCADE | 유저 탈퇴 시 요청도 삭제 (개인정보 정리 관점) |
| `reviewed_by → users(id)` | SET NULL | 어드민 탈퇴해도 요청 이력은 남기고 검토자만 null 처리 |

## 5. 기존 places 테이블 마이그레이션 (no-op)

- `places` 스키마 변경 없음. 컬럼/인덱스 그대로.
- 승인 완료된 CREATE 요청은 `places` INSERT 시 `id`를 payload의 `clientUuid`로 지정 (백엔드 로직).
- 승인 완료된 UPDATE_* 는 기존 `places` row UPDATE.

## 6. 앱 SQLite 참고 스키마 (flutter-dev용, 여기는 참고만)

```sql
-- brd_local.db, AppDatabase migration v3
CREATE TABLE local_places (
    id TEXT PRIMARY KEY,                    -- 클라이언트 UUID v4 (승인 시 서버 places.id로 그대로 사용)
    place_name TEXT NOT NULL,
    category_code TEXT NOT NULL,            -- FAMOUS/CAFE/RESTAURANT/SERVICE/OTHER
    latitude REAL NOT NULL,
    longitude REAL NOT NULL,
    address TEXT,
    road_address TEXT,
    description TEXT,
    phone TEXT,
    photo_url TEXT,
    created_at INTEGER NOT NULL,            -- millis
    updated_at INTEGER NOT NULL,

    -- 서버 요청 상태 트래킹 (요청 안 한 순수 로컬 pin은 null)
    request_id TEXT,                        -- 서버 place_change_requests.id
    request_status TEXT,                    -- null | PENDING | APPROVED | REJECTED
    request_synced_at INTEGER,              -- 마지막으로 서버 상태 pull한 시각

    deleted_at INTEGER                      -- 로컬 삭제 시각 (승인되어 서버로 이관된 경우 여기서 정리)
);
CREATE INDEX idx_local_places_request_status ON local_places(request_status);
CREATE INDEX idx_local_places_deleted ON local_places(deleted_at);
```

## 주의사항

1. **UNIQUE 인덱스와 CHECK 조합**: PENDING 유니크는 UPDATE 계열에만 적용된다 (target_place_id IS NOT NULL 조건). CREATE는 target=null이라 이 유니크에 안 걸림. CREATE 개수 제한은 앱/서버 애플리케이션 레이어에서.
2. **JSONB 검증**: DB 레벨에서는 payload 구조를 강제하지 않는다 (JSON Schema 확장 미도입). Service 레이어에서 type별 payload 검증 필수.
3. **PostgreSQL 버전**: JSONB는 9.4+, gen_random_uuid는 13+ 기본 내장. 현재 스키마와 동일 전제.
4. **트랜잭션**: 승인 시 places INSERT/UPDATE + 요청 status 변경은 단일 트랜잭션. Service `@Transactional`로 커버 (backend 문서 참조).
5. **role 컬럼 default**: 기존 유저 전원 USER로 마이그레이션. 어드민 지정은 별도 SQL:
   ```sql
   UPDATE users SET role = 'ADMIN' WHERE email = 'jyl9311@gmail.com';
   ```
   → 해당 유저는 앱에서 로그아웃 후 재로그인해야 JWT에 role claim 실림.
