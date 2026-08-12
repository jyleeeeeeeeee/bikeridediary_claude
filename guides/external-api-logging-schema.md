# 외부 API 호출 로깅 — DB 스키마 가이드

작성일: 2026-08-12
담당: dba
근거 계획: `plans/external-api-logging.md` (D1~D7 확정)
참조: `brd_be/src/main/resources/schema.sql` (기존 컨벤션)

## 결론

`api_call_logs` 테이블 + 시퀀스 + 인덱스 3개 + 기존 DB용 ALTER 2줄만 schema.sql에 추가.
엔티티 매핑 힌트만 이 문서에 있고, 실제 엔티티/서비스/컨트롤러 코드는 `external-api-logging-backend.md` 참조.

## 파일 수정 목록

| 파일 | 신규/수정 | 설명 |
|------|---------|------|
| `brd_be/src/main/resources/schema.sql` | 수정 | 시퀀스·테이블·인덱스 + 기존 DB용 ALTER 블록 |

## DDL 스니펫

### 1. schema.sql — 섹션 0 (시퀀스 선언부)에 추가

파일 20~33번 줄의 `CREATE SEQUENCE IF NOT EXISTS` 블록 끝에 한 줄 추가.

```sql
CREATE SEQUENCE IF NOT EXISTS api_call_logs_no_seq;
```

### 2. schema.sql — 테이블 정의부에 추가 (place_change_requests 뒤, 새 섹션으로)

`place_change_requests` 테이블 정의(472번 줄) 끝 `);` 이후에 붙임.

```sql
-- ============================================================
-- 16. api_call_logs (외부 API 호출 로그)
-- 목적: 사용량 모니터링 / 이상 탐지 / 유저별 차단 근거
-- 보관 정책: 90일 후 스케줄러가 DELETE (D6=B 결정)
-- ============================================================
CREATE TABLE IF NOT EXISTS api_call_logs (
    no               BIGINT        UNIQUE DEFAULT nextval('api_call_logs_no_seq'),
    id               UUID          DEFAULT gen_random_uuid() PRIMARY KEY,
    -- 호출 유저 (인증 없이 호출된 경우 null — 게스트 요청, 시스템 배치 등)
    -- ON DELETE SET NULL: 유저 탈퇴 시 로그 자체는 유지 (사용량 집계 무결성)
    user_id          UUID          REFERENCES users(id) ON DELETE SET NULL,
    -- API 식별자 (NAVER_DIRECTIONS, NAVER_GEOCODING, NAVER_REVERSE_GEOCODING,
    --                NAVER_SEARCH, KAKAO_LOCAL, OPINET, OPENWEATHER)
    api_name         VARCHAR(50)   NOT NULL,
    -- 실제 호출 URL path (쿼리스트링 제외)
    endpoint         VARCHAR(200)  NOT NULL,
    -- HTTP 메서드
    http_method      VARCHAR(10)   NOT NULL,
    -- 응답 HTTP 상태 코드 (네트워크 예외 시 null)
    status_code      INTEGER,
    -- 호출~응답 소요 시간 (밀리초)
    response_time_ms INTEGER       NOT NULL,
    -- 마스킹된 요청 파라미터 (apiKey/clientSecret/clientId 자동 제거 후 저장)
    -- GIN 인덱스 없음 — JSON 내부 필드 검색 현재 불필요
    request_params   JSONB,
    -- 실패 시 예외 메시지 (성공이면 null)
    error_message    TEXT,
    -- 호출 시각
    called_at        TIMESTAMP     NOT NULL DEFAULT now()
);
```

### 3. schema.sql — 인덱스 섹션(14번)에 추가

```sql
-- api_call_logs: API별 최신 호출 조회 (관리자 API ?apiName=X + 최신순 정렬)
CREATE INDEX IF NOT EXISTS idx_api_logs_api_name_called_at
    ON api_call_logs (api_name, called_at DESC);

-- api_call_logs: 유저별 사용량 조회 (?userId=X 필터). partial — 익명(null) 제외해 인덱스 크기 절약
CREATE INDEX IF NOT EXISTS idx_api_logs_user_id_called_at
    ON api_call_logs (user_id, called_at DESC)
    WHERE user_id IS NOT NULL;

-- api_call_logs: 기간 조회 + 스케줄러 DELETE WHERE called_at < ?
-- 참고: 90일 미만 행이 전체의 100%인 시점(초기)엔 seq scan이 더 빠를 수 있음.
-- 데이터 누적 후(수십만 건) DELETE 배치가 느려지면 이 인덱스가 효과를 냄.
CREATE INDEX IF NOT EXISTS idx_api_logs_called_at
    ON api_call_logs (called_at);
```

### 4. schema.sql — 기존 DB용 ALTER 블록(섹션 11)에 추가

```sql
ALTER TABLE api_call_logs ADD COLUMN IF NOT EXISTS no BIGINT;
ALTER TABLE api_call_logs ALTER COLUMN no SET DEFAULT nextval('api_call_logs_no_seq');
```

## 엔티티 매핑 힌트

`ApiCallLogEntity` 위치: `com.bikeridediary.domain.apicalllog.entity.ApiCallLogEntity`

주요 어노테이션 (실제 전체 코드는 backend 가이드 참조):

```java
@Entity
@Table(name = "api_call_logs")
public class ApiCallLogEntity {

    @Column(name = "no", insertable = false, updatable = false)
    @Generated(event = EventType.INSERT)
    private Long no;

    @Id
    @Column(name = "id")
    @JdbcTypeCode(SqlTypes.UUID)   // Hibernate 6.x batch 이슈 대응 필수
    private UUID id;

    // ManyToOne UserEntity 대신 UUID 직접 저장 권장 (AOP가 영속성 컨텍스트 밖에서 세팅)
    @Column(name = "user_id")
    @JdbcTypeCode(SqlTypes.UUID)
    private UUID userId;

    @Enumerated(EnumType.STRING)
    @Column(name = "api_name", nullable = false, length = 50)
    private ApiName apiName;

    // JSONB — hypersistence-utils (이미 build.gradle 등록됨)
    @Type(JsonType.class)
    @Column(name = "request_params", columnDefinition = "jsonb")
    private Map<String, Object> requestParams;

    // TEXT — @Lob 금지 (PostgreSQL에서 OID 타입으로 매핑되어 pg_largeobject에 저장됨)
    @Column(name = "error_message", columnDefinition = "TEXT")
    private String errorMessage;

    // BaseEntity 상속 안 함 — 불변 로그 레코드. updated_at/deleted_at 불필요
}
```

**중요 포인트**:
1. `user_id`는 `@ManyToOne UserEntity` 대신 `@Column private UUID userId` 사용. AOP가 UserEntity 프록시 없이 UUID만 받아 세팅해야 함. FK 무결성은 스키마의 `REFERENCES users(id) ON DELETE SET NULL`로 이미 확보.
2. `@Type(JsonType.class)` + `columnDefinition = "jsonb"` 같이 선언 — 둘 중 하나만 쓰면 Hibernate가 JSONB 힌트 놓칠 수 있음. `PlaceChangeRequestEntity.payload` 패턴 그대로.
3. `@Lob` 금지. `@Column(columnDefinition="TEXT")` 사용.
4. `BaseEntity` 상속 안 함. `called_at`이 `created_at` 역할.

## 인덱스 근거

| 인덱스 | 커버 쿼리 | 근거 |
|--------|---------|------|
| `(api_name, called_at DESC)` | `GET /admin/api-logs?apiName=X&from=&to=` | api_name equality + called_at 범위/정렬 복합 커버. Cardinality: 7개 정도로 낮지만 복합 인덱스 선두로 유효 |
| `(user_id, called_at DESC) WHERE user_id IS NOT NULL` | `GET /admin/api-logs?userId=<UUID>` | partial 인덱스로 null 행 제외 → 인덱스 크기 절약. 게스트/시스템 호출로 null 비율이 높을 것 |
| `(called_at)` | 스케줄러 `DELETE WHERE called_at < :threshold` + 기간 범위 조회 | 아래 90일 보관 검토 참조 |

## 90일 보관 스케줄러 — DB 부하 검토

**스케줄러 실행 SQL**:
```sql
DELETE FROM api_call_logs WHERE called_at < now() - interval '90 days';
```

**`(called_at)` 단일 인덱스의 실효성**:

- 초기(수십만 건 미만, 90일 미만 데이터가 99%): 옵티마이저가 seq scan 선택할 가능성 큼. DELETE 대상이 전체 1% 미만일 때 인덱스가 유리하지만 초기엔 삭제 대상이 거의 없어 인덱스 존재만으로도 비용 큰 차이 없음
- **인덱스가 실효성 갖는 시점**: 90일분 데이터 누적 후. 매일 하루치씩 DELETE 대상 생기고 인덱스로 해당 범위만 빠르게 locate
- **현재 판단**: 인덱스 유지. 유지 비용(INSERT 시 인덱스 갱신) 작고, 누적 후 확실히 효과

**대량 DELETE 부담 완화 (누적 후 필요 시)**:

초기엔 불필요. 데이터가 수백만 건 이상 쌓이면 아래 방식으로 교체.

```sql
-- LIMIT 기반 분할 DELETE (스케줄러에서 반복 호출)
DELETE FROM api_call_logs
WHERE id IN (
    SELECT id FROM api_call_logs
    WHERE called_at < now() - interval '90 days'
    LIMIT 1000
);
```

분할 DELETE는 잠금 범위를 줄여 실시간 INSERT와 경합 최소화. PostgreSQL MVCC가 동시성은 처리하지만 단건 DELETE 트랜잭션이 오래 걸리면 autovacuum 부담 가능.

## PostgreSQL 성능/부하 우려

**JSONB `request_params` 크기**:
- Naver Directions 요청: start/goal + waypoints 최대 15개 좌표. JSON 약 500~1000 bytes. 무해
- Naver Search: query 문자열. 수십 bytes
- OpenWeather: lat/lon. 수십 bytes
- 응답 body는 저장 안 함 (스코프 아웃). 크기 방어 확보

**GIN 인덱스 필요 여부**:
- 현재 조회는 `api_name`, `user_id`, `called_at` 필터만. `request_params` 내부 필드 검색 없음
- 필요 시 나중에 도입

**INSERT 빈도**:
- 사용자 액션 기반, 초당 수십 건 이하 예상. 인덱스 3개 유지 비용 무시 가능

## FK 정책 요약

- `user_id → users(id) ON DELETE SET NULL`: 유저 탈퇴 시 로그는 보존, user_id만 null 처리 (사용량 집계 무결성)
- `updated_at`, `deleted_at` 없음: 불변 이력 로그. soft delete 불필요

## schema.sql 삽입 위치 요약

| 위치 | 내용 |
|------|------|
| 섹션 0 (시퀀스 블록, 20~33번 줄 끝) | `CREATE SEQUENCE IF NOT EXISTS api_call_logs_no_seq;` 1줄 |
| 섹션 15 신규 (place_change_requests `);` 직후) | `api_call_logs` CREATE TABLE |
| 섹션 14 인덱스 블록 끝 | 인덱스 3개 |
| 섹션 11 ALTER 블록 끝 (457번 줄 이후) | ADD COLUMN + SET DEFAULT 2줄 |

## 체크리스트

- [ ] 시퀀스 선언: `api_call_logs_no_seq` 1줄
- [ ] CREATE TABLE: `no` 컬럼이 첫 번째 컬럼 위치 (컨벤션)
- [ ] FK `ON DELETE SET NULL` 확인 (`user_id → users(id)`)
- [ ] `updated_at`, `deleted_at` 없음 확인 (로그 특성)
- [ ] 인덱스 3개 각각 `CREATE INDEX IF NOT EXISTS` 개별 문장 (DO $$ 블록 아님)
- [ ] partial 인덱스 `WHERE user_id IS NOT NULL` 정확
- [ ] ALTER 섹션에 `ADD COLUMN IF NOT EXISTS no BIGINT` + `SET DEFAULT` 2줄

## 확인 방법

```bash
docker compose down -v && docker compose up postgres
# 또는
psql -U <user> -d <db> -f brd_be/src/main/resources/schema.sql

psql -c "\d api_call_logs"
psql -c "SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'api_call_logs';"
```
