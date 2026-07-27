# 장소 등록/수정 어드민 승인 워크플로 — 아키텍처 개요

> 작성: 2026-07-21 (pm 세션)
> 관련 문서: `place-approval-schema.md` (dba), `place-approval-backend.md` (backend-dev), `place-approval-app.md` (flutter-dev), `place-approval-qa.md` (qa)

---

## 배경 / 문제 정의

현재 place 도메인은 로그인한 아무 사용자나 다음 두 가지를 서버에 즉시 반영할 수 있다:

- `POST /api/v1/places` — 신규 장소 등록 (Naver 지역검색 결과 → 서버 저장)
- `PATCH /api/v1/places/{id}/coordinates` — 임의 장소의 좌표 수정
- `PATCH /api/v1/places/{id}/info` — 임의 장소의 이름/카테고리 수정

문제:

1. **보안**: `SecurityConfig.PERMIT_ALL_ENDPOINTS`에 `/api/v1/places/**`가 통째로 들어가 있어 좌표 수정도 무인증. 누구든 임의 place의 좌표를 태평양으로 보낼 수 있음. (claude-memory.md의 "미해결 사항" 참조)
2. **데이터 품질**: 큐레이션 POI인데 유저가 무통제 수정. 오탐/스팸/장난 방어 수단 없음.
3. **오프라인 우선 아키텍처 미스매치**: 유저가 자기 지역의 사적인 장소(단골 카페 등)를 등록하고 싶을 때, 지금은 서버에 바로 올리는 흐름뿐이라 "내 지도에만 보이는 개인 pin"이 불가능.

## 목표

- 신규 장소 등록: 유저 로컬에는 즉시 저장, 서버 공개는 어드민 승인 후.
- 수정: 좌표/정보 모두 즉시 반영 폐기, 어드민 승인 후 반영.
- 어드민: 별도 웹 없이 앱 안에서 role=ADMIN인 유저만 볼 수 있는 관리 화면.
- 오프라인/로컬 우선 원칙과 정합.

## 확정 사항 (D1~D9)

| # | 질문 | 확정 답 | 요지 |
|---|-----|---------|------|
| D1 | 신규 등록 UX | **A** | 로컬 SQLite에 먼저 저장 → 상세에서 "공개 등록 요청" 버튼으로 서버 요청 생성 |
| D2 | 요청 저장 스키마 | **B** | 단일 `place_change_requests` 테이블 + type 컬럼 + `payload JSONB` |
| D3 | 어드민 지정 | **A + B** | `users.role` 컬럼 (USER/ADMIN) + JWT claim `role` |
| D4 | 어드민 UI 위치 | **A** | 앱 내 (설정 화면에서 role=ADMIN이면 "요청 관리" 진입 노출). 별도 웹 어드민 나중 |
| D5 | 좌표/정보 수정 흐름 | **A** | 좌표·정보 모두 즉시 반영 폐기, 요청 큐로 통합 |
| D6 | 승인 후 데이터 반영 시점 | **A** | 승인 클릭 시 트랜잭션 내 즉시 places 반영 |
| D7 | 요청 소유자 UX | **A** | 요청자가 자기 요청 목록/상태 확인 가능 (신규 화면 `/my-place-requests`) |
| D8 | 중복 요청 방지 | **A** | 같은 `target_place_id` + status=PENDING 요청 1개만 허용 (UPDATE 계열). CREATE는 target=null이라 요청자별 상한(예: PENDING 20건)만 적용 |
| D9 | 로컬 place UUID 정책 | **A** | 앱이 UUID v4 생성 → 요청 payload에 담아 서버 전송 → 승인 시 그 UUID를 그대로 `places.id`에 INSERT |

## 워크플로 (텍스트 다이어그램)

### W1. 신규 장소 등록 (CREATE)

```
[유저] Naver 검색 → 후보 선택 → 좌표 보정 → "내 지도에 저장"
   ↓
[앱] local_places (SQLite)에 INSERT (client UUID v4)
   ↓
[앱] CourseMapScreen이 서버 places + local_places 합쳐 렌더링 (로컬은 반투명 마커)
   ↓
[유저] 로컬 place 상세에서 "공개 등록 요청" 탭
   ↓
[앱] POST /api/v1/place-change-requests { type: CREATE, payload: {...} }
   ↓
[서버] place_change_requests INSERT (status=PENDING)
   ↓
[앱] local_places 해당 row의 request_status = PENDING 업데이트, UI 뱃지 표시
   ↓
[어드민] /admin/place-requests에서 PENDING 목록 확인 → 승인 or 거절
   ↓ (승인)
[서버] 트랜잭션: places INSERT (id = payload.clientUuid) + 요청 status=APPROVED
   ↓
[앱] 다음 pull(수동/앱 재시작)에 서버 places에 나타남 → 로컬 pin은 삭제 or 서버 것과 동기화
```

### W2. 좌표/정보 수정 (UPDATE_COORDINATES / UPDATE_INFO)

```
[유저] 서버 place 상세 → "좌표 수정 요청" or "정보 수정 요청"
   ↓
[앱] POST /api/v1/place-change-requests { type: UPDATE_COORDINATES, targetPlaceId, payload: {lat, lng} }
   ↓
[서버] 같은 target_place_id + PENDING 요청 있으면 409 CONFLICT
       없으면 INSERT (status=PENDING)
   ↓
[앱] 상세 화면에 "수정 요청 대기 중" 뱃지 표시 (내 요청 목록에서도 확인)
   ↓
[어드민] 요청 확인 → 승인
   ↓
[서버] 트랜잭션: places UPDATE + 요청 status=APPROVED
```

### 상태 전이

```
PENDING → APPROVED (어드민 승인, places 반영 후)
PENDING → REJECTED (어드민 거절, review_note에 사유)
[terminal] APPROVED, REJECTED — 재활성화 없음 (재요청 시 새 request row 생성)
```

## 페르소나별 여정

### 일반 유저 (role=USER)

1. **로컬 pin 등록**: Naver 검색 → 좌표 보정 → 로컬 저장. 즉시 내 지도에 뜸.
2. **공개 요청 (옵션)**: 로컬 place 상세에서 "공개 등록 요청". 요청 상태 뱃지(대기 중/승인됨/거절됨).
3. **서버 place 수정 요청**: 서버 place 상세에서 "좌표 수정 요청" 또는 "정보 수정 요청". 이미 대기 중인 요청이 있으면 429/409 안내.
4. **내 요청 목록**: 설정 → "내 장소 요청" 진입. 상태별로 필터.

### 어드민 (role=ADMIN)

1. 설정 화면에 "요청 관리 (N건 대기)" 카드 노출 (role=ADMIN 조건).
2. `/admin/place-requests` — PENDING 목록. type별 아이콘, requester 닉네임, target place 이름(UPDATE면), 요청 시각.
3. 상세 화면 — payload 미리보기 (지도로 좌표 시각화, before/after diff), 승인/거절 버튼, 거절 시 review_note 입력.
4. 승인 → 즉시 places 반영. 거절 → status만 변경 (요청자가 자기 목록에서 사유 확인).

## 시스템 구성 요약

### DB (dba)

- `users.role VARCHAR(20) NOT NULL DEFAULT 'USER'` (USER, ADMIN)
- `place_change_requests` 신규 테이블 (id UUID, type, target_place_id, requester_id, payload JSONB, status, review_note, reviewed_by, reviewed_at, created_at)
- 인덱스 3개, FK 정책, PENDING 유니크 인덱스 (부분)

### 백엔드 (backend-dev)

- `UserRole` enum 추가, `UserEntity.role` 필드
- `JwtTokenProvider.generateAccessToken(userId, role)` — claim `role` 추가
- `CustomUserDetails`가 `ROLE_ADMIN` authority 부여
- `PlaceChangeRequestEntity/Repository/Service/Controller`
- 기존 `POST /places`, `PATCH /places/{id}/coordinates`, `PATCH /places/{id}/info` **제거** (또는 어드민 전용 백도어로 남길지 결정 필요 — 초안은 제거)
- `SecurityConfig` 정리: `/api/v1/places/**` PERMIT_ALL에서 제거, GET만 열기, `/api/v1/admin/**`은 `hasRole('ADMIN')`

### 앱 (flutter-dev)

- `AppDatabase` v3: `local_places` 테이블
- `local_places`에 request 상태 컬럼 (request_id, request_status)
- `LocalPlaceRepository`
- `CourseMapScreen` 렌더링: 서버 + 로컬 마커 합침 (로컬은 스타일 구분)
- 신규 화면: 로컬 place 상세, 서버 place 수정 요청, 내 요청 목록, 어드민 요청 목록/상세
- 기존 `PlaceCoordinateEditScreen`, `PlaceInfoEditScreen` UI는 재활용하되 저장 로직을 요청 생성으로 교체
- `AuthResponse`/`User` 모델에 role 필드, `authProvider`에서 노출

### QA (qa)

- 시나리오 뼈대 문서만 `place-approval-qa.md`에 두고, 구현 완료 후 qa 에이전트가 세부 시나리오 확장

## 위험 요소 / 주의사항

- **기존 데이터 마이그레이션**: 기존 places는 그대로 두면 됨 (schema 변경 없음). 다만 이미 유저가 즉시 저장한 이상 좌표들이 있을 수 있으니, 배포 후 어드민 검수 필요.
- **PENDING 유니크 제약**: UPDATE 계열은 `(target_place_id, status='PENDING')` 부분 유니크 인덱스로 강제. CREATE는 target=null이라 이 제약 미적용 → 요청자별 CREATE PENDING 개수 상한(예: 20건)을 앱/서버 양쪽에서 확인.
- **role 부여 방법**: 초기 어드민은 DB 직접 UPDATE (SQL 스니펫 backend 문서에 포함). 관리자 지정 UI는 스코프 외.
- **payload JSONB 검증**: type별 필수 필드 다름. 서버 Service 레이어에서 type별 분기 검증. Jackson으로 sealed hierarchy 대신 map으로 받고 dispatch 방식이 단순.
- **로컬 place → 승인 후 서버 동기화**: 승인되면 서버 places에 같은 UUID로 존재하게 됨. 앱은 pull 시 서버 places 우선하고 local_places는 request_status=APPROVED가 된 것들을 삭제(또는 flag 유지). Phase 3 bike sync와 다른 점: bike는 서버와 로컬이 1:1이지만 place는 로컬이 서버와 별도 테이블이므로 관리가 더 단순.
- **어드민 앱 접근성**: role=ADMIN인 유저가 앱을 로그아웃/재로그인해야 새 JWT에 role claim이 실림. 최초 어드민 지정 시 강제 재로그인 안내.
- **오프라인 요청**: 요청 자체는 서버 통신 필요. 오프라인 시 로컬 저장까지만 되고 요청 버튼은 "네트워크 없음" 안내.

## 스코프 컷 (이번 사이클 밖)

- 웹 어드민 콘솔 (별도 프로젝트)
- 요청 알림 (FCM으로 요청자에게 승인/거절 push) — 후속
- 좌표 변경량 자동 판정 (100m 이상이면 자동 rejected 등)
- 어드민 감사 로그 (누가 언제 무엇을 승인했는지 별도 audit 테이블)
- 어드민의 place 직접 수정 백도어 엔드포인트 (지금은 요청→자동 승인 우회 가능)
- 신고/스팸 처리

## 다음 단계

1. 이 문서 검토 → 사용자 승인
2. 3개 담당 문서로 담당자별 병렬 진행:
   - `place-approval-schema.md` → dba (사용자 직접 구현)
   - `place-approval-backend.md` → backend-dev (사용자 직접 구현)
   - `place-approval-app.md` → flutter-dev (Claude 구현)
3. 통합 후 qa 시나리오 확장 → code-reviewer 최종
