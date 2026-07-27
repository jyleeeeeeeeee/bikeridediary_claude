# 장소 승인 워크플로 --- QA 시나리오 완성판

> 작성: 2026-07-21 (qa 에이전트 확장)
> 기반: place-approval-workflow.md / place-approval-backend.md / place-approval-app.md
> 우선순위: P0=출시 블로커, P1=중요, P2=nice-to-have

---

## 1. Happy Path (골든패스)

### T1. W1 일반 유저 --- 로컬 저장 > 공개 요청 > 어드민 승인 > 지도 반영

사전 조건: USER 계정 A(로그인), ADMIN 계정 B(다른 기기 로그인). local_places 비어있음. 온라인.
우선순위: P0

| 단계 | 행동 | 기대 결과 |
|------|------|----------|
| 1 | [A] CourseMapScreen > 검색 아이콘 > Naver 지역검색 입력 | 후보 목록 5건 이하 표시 |
| 2 | [A] 후보 선택 > 좌표 보정 > 카테고리 CAFE > 내 지도에 저장 | 스낵바: 내 지도에 저장했어요. 공개 등록은 상세에서. |
| 3 | [A] 지도 화면 복귀 | 반투명 마커 + 내 뱃지 노출. 서버 마커와 시각 구분 |
| 4 | [A] 반투명 마커 탭 > LocalPlaceDetailScreen | 장소명, 카테고리, 좌표, request_status=null |
| 5 | [A] 공개 등록 요청 버튼 탭 | POST /place-change-requests 200. 스낵바: 검토 대기 중. local_places.request_status=PENDING |
| 6 | [A] 설정 > 내 장소 요청 | 방금 요청 PENDING 상태로 목록 표시 |
| 7 | [B] 설정 > 요청 관리 > AdminPlaceRequestsScreen | A의 CREATE 요청 PENDING 목록 표시 |
| 8 | [B] 요청 탭 > 상세 화면 | payload 지도 마커, 장소명/카테고리 정보 정상 표시 |
| 9 | [B] 승인 버튼 탭 | POST /admin/place-change-requests/{id}/approve 200. places INSERT. 요청 APPROVED |
| 10 | [A] 지도 새로고침 (pull-to-refresh) | 서버 마커(불투명)로 교체됨. 반투명 로컬 마커 겹침 여부 확인 |
| 11 | [A] 내 장소 요청 | 해당 요청 APPROVED 상태 확인 |

DB 검증: SELECT * FROM places WHERE id=clientUuid > 1건. SELECT status FROM place_change_requests > APPROVED.

---
n사전 조건: 서버 places에 place P 존재. USER A, ADMIN B 로그인.
우선순위: P0

| 단계 | 행동 | 기대 결과 |
|------|------|----------|
| 1 | [A] 서버 place P 탭 > 하단 시트 > 좌표 수정 요청 | PlaceCoordinateEditScreen. 버튼 라벨: 좌표 수정 요청 |
| 2 | [A] 위치 보정 > 좌표 수정 요청 | POST 200. 스낵바: 좌표 수정 요청을 보냈어요. 어드민 승인 후 반영됩니다. |
| 3 | [A] place P 상세 재진입 | 수정 요청 대기 중 뱃지. 편집 버튼 비활성화 |
| 4 | [B] AdminPlaceRequestsScreen | UPDATE_COORDINATES 요청 PENDING 표시 |
| 5 | [B] 상세 > 지도 before(빨강)/after(파랑) 마커 + 이동 거리 | 두 마커 정상 표시 |
| 6 | [B] 승인 | places.latitude/longitude 갱신. 요청 APPROVED |
| 7 | [A] 지도 새로고침 | place P 마커가 새 좌표로 이동 |

---

### T3. W2 일반 유저 --- 정보 수정 요청 > 어드민 거절

사전 조건: 서버 place P 존재. USER A, ADMIN B 로그인.
우선순위: P1

| 단계 | 행동 | 기대 결과 |
|------|------|----------|
| 1 | [A] place P 상세 > 정보 수정 요청 > 이름 변경 | PlaceInfoEditScreen |
| 2 | [A] 정보 수정 요청 | POST 200. 스낵바: 정보 수정 요청을 보냈어요. |
| 3 | [B] 상세 > before/after 대비표 > 거절 + 사유 이름 부정확 | reject 200. 요청 REJECTED |
| 4 | [A] 내 장소 요청 | REJECTED + 사유 이름 부정확 표시 |
| 5 | [A] place P 상세 | 원래 이름 유지. 편집 버튼 재활성화 |

---

### T4. 어드민 자동 승인 --- 즉시 반영

사전 조건: ADMIN 계정 B 로그인. 서버 place P 존재.
우선순위: P0

| 단계 | 행동 | 기대 결과 |
|------|------|----------|
| 1 | [B] place P > 좌표 수정 요청 > 위치 수정 | POST 응답 status=APPROVED. 스낵바: 좌표가 즉시 반영되었어요. |
| 2 | [B] 지도 | place P 마커가 이미 새 좌표로 이동 (allPlacesProvider invalidate) |
| 3 | [B] AdminPlaceRequestsScreen APPROVED 탭 | 자동 승인 기록 (reviewer=B, review_note=null) 확인 |

DB 검증: SELECT status, reviewed_by FROM place_change_requests ORDER BY created_at DESC LIMIT 1 > APPROVED.
ADMIN CREATE 흐름도 동일: 로컬 저장 > 공개 등록 요청 > status=APPROVED > 즉시 반영 스낵바.

---

## 2. 엣지 케이스

### E1. 중복 PENDING 요청 (같은 place UPDATE 두 번)

사전 조건: 서버 place P에 USER A의 UPDATE_COORDINATES PENDING 존재.
우선순위: P0

| 시나리오 | 기대 동작 | 확인 방법 |
|---------|----------|----------|
| [A] place P 상세 편집 버튼 탭 | 버튼 비활성화 또는 진입 차단 | 앱 UI 확인 |
| 앱 우회 POST /place-change-requests 직접 호출 | HTTP 409. ErrorCode=PLACE_REQUEST_ALREADY_PENDING | Postman |

DB 검증: SELECT count(*) FROM place_change_requests WHERE target_place_id=P.id AND status=PENDING > 항상 1 이하.

---

### E2. CREATE 요청 상한 초과 (20건 초과)

사전 조건: USER A PENDING CREATE 요청 정확히 20건 보유.
우선순위: P1

| 시나리오 | 기대 동작 | 확인 방법 |
|---------|----------|----------|
| 21번째 로컬 pin 공개 등록 요청 | HTTP 429. ErrorCode=PLACE_REQUEST_LIMIT_EXCEEDED. 스낵바: 대기 중인 요청이 많아요. | Postman + 앱 UI |
| 기존 PENDING 취소 후 재시도 | 정상 요청 생성 | |

---

### E3. 요청 취소 흐름 (PENDING만 가능)

우선순위: P2
스펙 확인 필요: 취소 API가 현재 스코프에 포함되었는지 flutter-dev에 확인.

| 시나리오 | 기대 동작 |
|---------|----------|
| PENDING 요청 취소 | DELETE /place-change-requests/{id} 200. local_places.request_status null 초기화 |
| APPROVED 요청 취소 시도 | 409 또는 앱에서 버튼 미노출 (이미 places 반영됨) |
| REJECTED 요청 취소 시도 | 취소 불가. 다시 요청 흐름으로 유도 |

---

### E4. REJECTED 후 재요청

사전 조건: USER A CREATE 요청 REJECTED 상태. 로컬 pin 존재.
우선순위: P1

| 단계 | 행동 | 기대 결과 |
|------|------|----------|
| 1 | [A] LocalPlaceDetailScreen | REJECTED 뱃지 + reviewNote 표시 |
| 2 | [A] 다시 요청 탭 | 새 PlaceChangeRequest row 생성. 이전 REJECTED row 유지. local_places.request_status=PENDING |
| 3 | [B] AdminPlaceRequestsScreen | 새 PENDING 요청 표시 (이전 REJECTED와 별개 row) |

---

### E5. 어드민이 유저 PENDING 있는 place 직접 수정

사전 조건: USER A가 place P에 UPDATE_COORDINATES PENDING. ADMIN B가 같은 place P 직접 수정.
우선순위: P1 (backend.md 주의사항 8 참조)

| 단계 | 행동 | 기대 결과 |
|------|------|----------|
| 1 | [B] place P 좌표 수정 요청 | 자동 승인 > B 좌표로 places 갱신 |
| 2 | A PENDING 요청 여전히 PENDING | 어드민이 A 요청 승인 시 A payload가 B 변경분을 덮어씀 |

DB 검증: B 승인 직후 > B 좌표. A PENDING 승인 후 재조회 > A 좌표로 덮어써짐.
운영 주의: 어드민 자동 승인 시 같은 target의 유저 PENDING auto-REJECT 로직 없음 (G4 참조).

---

### E6. 존재하지 않는 place UUID로 UPDATE 요청

우선순위: P0

| 행동 | 기대 결과 |
|------|----------|
| POST /place-change-requests { type: UPDATE_COORDINATES, targetPlaceId: fake-uuid } | HTTP 404. ErrorCode=PLACE_NOT_FOUND |

---

### E7. 로컬 저장 후 앱 재설치 > 로컬 pin 소실

사전 조건: USER A 로컬 pin 3개 저장 (request_status 혼재). 앱 재설치.
우선순위: P1

| 시나리오 | 기대 결과 |
|---------|----------|
| 재설치 후 로그인 | brd_local.db 초기화 > local_places 전부 소실 |
| PENDING 요청 있었던 pin | 서버 요청은 유지됨. 어드민 승인 시 places에 등록됨 |
| 순수 로컬 pin (요청 없음) | 영구 소실. 복구 불가 |

앱 UX 확인: 로컬 저장 시 공개 등록 요청 유도 안내 존재 여부.

---

### E8. 로그아웃 > 로컬 pin 정리

사전 조건: USER A 로그인. 로컬 pin 2개.
우선순위: P0

| 단계 | 행동 | 기대 결과 |
|------|------|----------|
| 1 | [A] 설정 > 로그아웃 | AppDatabase.clearAll() > local_places 전체 삭제 |
| 2 | 다른 계정 B 로그인 | A의 로컬 pin 지도 미표시 |

DB 검증: 로그아웃 직후 SELECT count(*) FROM local_places > 0.

---

### E9. 오프라인 시나리오

사전 조건: 비행기 모드. USER A 로그인 상태.
우선순위: P0

| 시나리오 | 기대 동작 | 우선순위 |
|---------|----------|----------|
| 오프라인 로컬 저장 | SQLite INSERT 성공. 반투명 마커 표시. 서버 통신 없음 | P1 |
| 오프라인 공개 등록 요청 | 네트워크 오류 스낵바. local_places 상태 변경 없음 | P0 |
| 오프라인 어드민 목록 | 로딩 실패 + 네트워크 없음 안내 | P1 |
| 온라인 복귀 후 재시도 | 정상 요청 생성 | P0 |

실기기: Galaxy Z Flip3에서 비행기 모드 전환 후 앱 재진입 없이 연속 테스트.

---

### E10. 요청 중 대상 place가 삭제됨

사전 조건: USER A의 UPDATE PENDING. 어드민이 place P 삭제.
우선순위: P2

| 시나리오 | 기대 동작 | 확인 방법 |
|---------|----------|----------|
| place P 삭제 | CASCADE로 해당 요청도 삭제 | DB 직접 확인 |
| [A] 내 장소 요청 pull-to-refresh | 해당 요청 미표시 | 앱 UI |
| [B] AdminPlaceRequestsScreen 해당 요청 직접 접근 | 404 또는 미표시 | |

---

### E11. 로컬 pin 승인 후 자동 정리 미구현 (수동 삭제 필요)

사전 조건: USER A CREATE 요청 APPROVED. 지도에 서버/로컬 마커 공존.
우선순위: P1

| 시나리오 | 기대 동작 |
|---------|----------|
| 지도 확인 | 같은 좌표에 서버 마커(불투명)와 로컬 마커(반투명) 겹침 |
| [A] LocalPlaceDetailScreen > 로컬에서 제거 | local_places hard delete. 반투명 마커 사라짐. 서버 마커만 잔존 |

현재 스펙 갭: 승인 후 자동 정리 없음. 사용자 수동 삭제 필요. (G3 참조)

---

## 3. 보안

### S1. 일반 USER가 어드민 API 호출

사전 조건: USER 계정 JWT.
우선순위: P0

| 요청 | 기대 결과 | 확인 방법 |
|------|----------|----------|
| GET /api/v1/admin/place-change-requests (USER JWT) | HTTP 403 | Postman |
| POST /api/v1/admin/place-change-requests/{id}/approve (USER JWT) | HTTP 403 | Postman |
| POST /api/v1/admin/place-change-requests/{id}/reject (USER JWT) | HTTP 403 | Postman |

핵심 확인: SecurityConfig에 @EnableMethodSecurity 추가 여부. 없으면 @PreAuthorize 무시 > USER도 200 반환 > 출시 블로커.

---

### S2. 게스트/무인증 호출

우선순위: P0

| 요청 | 기대 결과 |
|------|----------|
| GET /api/v1/admin/place-change-requests (인증 없음) | HTTP 401 |
| POST /api/v1/place-change-requests (인증 없음) | HTTP 401 |
| GET /api/v1/place-change-requests/mine (인증 없음) | HTTP 401 |

---

### S3. 다른 유저의 요청 취소 시도 (IDOR)

사전 조건: USER A 요청 ID를 USER B가 취소 시도. 취소 API 구현 시 적용.
우선순위: P0

| 행동 | 기대 결과 |
|------|----------|
| DELETE /place-change-requests/{A요청ID} + B JWT | HTTP 403 또는 404 |

확인 방법: Service에서 requester_id == 로그인 userId 소유권 검증 코드 확인.

---

### S4. payload 조작 --- clientUuid가 기존 place UUID

사전 조건: USER A가 이미 존재하는 place X UUID를 clientUuid로 하여 CREATE 요청.
우선순위: P0

| 단계 | 행동 | 기대 결과 |
|------|------|----------|
| 1 | POST /place-change-requests { type: CREATE, payload: { clientUuid: 기존 place X UUID } } | 요청 PENDING 생성 허용 |
| 2 | [B] 어드민 승인 시도 | applyToPlaces에서 existsById 체크 > PLACE_ALREADY_EXIST로 승인 거부 |

스펙 갭 G1: existsById 체크가 권장으로만 명시. 미구현 시 기존 place 덮어쓰기 위험. 배포 전 필수 확인.

---

### S5. 어드민 승격 후 재로그인 없이 앱 UI 반영 여부

사전 조건: USER A 로그인 상태. DB에서 role ADMIN 변경.
우선순위: P1

| 단계 | 행동 | 기대 결과 |
|------|------|----------|
| 1 | DB UPDATE users SET role=ADMIN | 즉시 DB 반영 |
| 2 | [A] 설정 화면 확인 | 요청 관리 카드 미표시 (앱 캐시 role=USER) |
| 3 | [A] 어드민 API 직접 호출 | HTTP 200 (서버는 매 요청 DB lookup으로 즉시 반영) |
| 4 | [A] 로그아웃 > 재로그인 | 설정 화면에 요청 관리 카드 표시 |

---
## 4. Concurrency and Transactions

### C1. Two admins approve same request simultaneously

Precondition: ADMIN B1, B2 on same PENDING request detail screen. Priority: P0
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | [B1] tap approve | 200. places updated. status=APPROVED |
| 2 | [B2] tap approve at same time | HTTP 409. ErrorCode=PLACE_REQUEST_ALREADY_REVIEWED. Snackbar: already processed |

DB: SELECT status FROM place_change_requests WHERE id=req_id > APPROVED only 1 row. places INSERT also 1 row.
Note: Under extreme timing both transactions may read PENDING simultaneously. Add optimistic locking if needed.

---

### C2. Transaction rollback when places INSERT fails

Priority: P0

| Scenario | Expected Result |
|---------|-----------------|
| places INSERT fails (FK violation etc.) | @Transactional rollback > request.status stays PENDING |
| Retry approve after rollback | Processed normally |

Verification: inject DB error intentionally > after rollback SELECT status > PENDING

---
## 5. UI/UX (App)

### U1. Snackbar: admin auto-approve vs user pending

Priority: P1

| Scenario | Expected Snackbar |
|---------|-----------------|
| ADMIN submits coordinate update | Coordinate applied immediately. |
| USER submits coordinate update | Coordinate update request sent. Pending admin approval. |
| ADMIN submits local pin public request | Applied immediately. |
| USER submits local pin public request | Pending review badge |

Check: PlaceChangeRequest.status == approved branch exists in code.

---

### U2. Local pin marker visual distinction

Priority: P1

| State | Style |
|------|-------|
| Local pin (no request) | Semi-transparent (alpha 0.6) + small white badge [Me] top-right |
| Local pin PENDING | Semi-transparent + badge gray |
| Local pin REJECTED | Semi-transparent + badge red |
| Server place | Opaque. Marker ID local-uuid vs place-uuid distinction |

Device: Galaxy Z Flip3 folded (small screen) - [Me] badge still visible.

---

### U3. Pending request - edit button disabled

Priority: P1

| Scenario | Expected |
|---------|----------|
| place P has PENDING UPDATE | Edit buttons disabled + pending notice |
| After PENDING rejected | Buttons re-enabled |

---

### U4. REJECTED - reviewNote display

Priority: P1

| Location | Expected Display |
|------|-----------------|
| LocalPlaceDetailScreen | Rejected: [reviewNote] (null > no reason provided) |
| MyPlaceRequestsScreen item dialog | Full reviewNote shown |

---

### U5. MyPlaceRequestsScreen pull-to-refresh

Priority: P1

| Step | Action | Expected |
|------|--------|----------|
| 1 | Enter screen | listMine() shows latest state |
| 2 | Pull-to-refresh | Server re-query. APPROVED/REJECTED states reflected immediately |
| 3 | Local pin request_status | syncRequestStatus() updates local DB |

---

### U6. New screens x4 bottom SafeArea (Galaxy Z Flip3)

Priority: P0

| Screen | Check |
|------|-------|
| AdminPlaceRequestDetailScreen approve/reject buttons | SafeArea(top: false) or viewPadding.bottom padding - no gesture nav bar overlap |
| MyPlaceRequestsScreen scroll list bottom | EdgeInsets bottom includes viewPadding.bottom |
| LocalPlaceDetailScreen action buttons | Same |
| AdminPlaceRequestsScreen list bottom | Same |

Verify: On device, buttons not hidden behind gesture nav bar black area.

---

## 6. Regression (Existing Features)

### R1. Existing place GET remains permitAll

Priority: P0

| Request | Expected |
|---------|----------|
| Unauthenticated GET /api/v1/places | 200. Existing seed data returned |
| Unauthenticated GET /api/v1/places/search-external | 200 |

Change: /api/v1/places/** removed from PERMIT_ALL. GET only added to GET_PERMIT_ALL. POST/PATCH endpoints removed.

---

### R2. Existing seed data migration intact

Priority: P0

| Query | Expected |
|---------|----------|
| SELECT count(*) FROM places | Same count as before deployment |
| SELECT count(*) FROM place_categories | Same count |

---

### R3. Social login 5 types - role=USER by default

Priority: P0

| Login Type | Expected |
|-----------|----------|
| Email | UserResponse.role=USER |
| Guest | Offline local guest or server role=USER |
| Kakao | UserResponse.role=USER |
| Google | UserResponse.role=USER |
| Apple | UserResponse.role=USER |

Check: UserEntity factory applies DB default USER when role not explicitly set.

---

### R4. Existing place_wishes functionality works

Priority: P1

| Feature | Expected |
|------|----------|
| Add/remove from place_wishes | Works normally. No FK impact |
| Wish list query | Normal response |
| findFavoritedByOthers isPublic=TRUE bug | Previous cycle unresolved. Re-verify in R4 (G5) |

---

### R5. Bike/Course/Fueling/Maintenance unchanged

Priority: P0

| Feature | Check |
|------|-------|
| Bike CRUD | Create/update/delete normal. Local sync normal |
| Course view/favorite | GET list, favorite POST/DELETE normal |
| Fueling stats | GET /fuelings/stats normal response |
| Maintenance images | Upload/update/delete normal |

---

## 7. Device Testing Required (Galaxy Z Flip3)

| # | Scenario | Verification Method | Priority |
|---|---------|---------------------|----------|
| M1 | T1 Golden Path W1 full flow (local save>request>approve>map update) | 2 devices or admin account in parallel | P0 |
| M2 | T2 Golden Path W2 coordinate update before/after map markers | Verify marker coordinate change visually | P0 |
| M3 | T4 Admin auto-approve immediate update | status=APPROVED + map instantly refreshed | P0 |
| M4 | E9 Offline scenario (airplane mode) | Local save OK / request failure notice | P0 |
| M5 | U6 New 4 screens bottom SafeArea | gesture nav bar overlap visual check | P0 |
| M6 | E8 Logout clears local pins | Other account login - previous pins absent | P0 |
| M7 | U2 Local marker visual distinction (folded state) | Flip3 folded small screen - Me badge visible | P1 |
| M8 | S1 USER JWT > admin API 403 | Postman (adb reverse tcp:8081 tcp:8081) | P0 |
| M9 | R3 Social login 5 types UserResponse.role=USER | Network logs or Swagger | P0 |
| M10 | C1 Simultaneous approval simulation | 2 devices tapping simultaneously | P1 |

---

## 8. Performance / Load Concerns

| # | Scenario | Expected Benchmark | Verification |
|---|---------|-------------------|--------------|
| P1 | GET /admin/place-change-requests with 10000 PENDING rows | Response under 2 seconds | EXPLAIN ANALYZE + idx_pcr_status_created partial index hit |
| P2 | 500 server places + 50 local pins map render | 60fps maintained | Galaxy Z Flip3 scroll dropped frame visual check |
| P3 | allPlacesProvider invalidate frequency | Only 1 invalidate per admin auto-approve | Riverpod DevTools or logs |

---

## Spec Gaps / Implementation Gaps Found

| # | Item | Content | Recommended Action | Priority |
|---|------|---------|-------------------|----------|
| G1 | clientUuid existsById check | applyToPlaces CREATE branch existsById check only recommended, not confirmed implemented. Risk: overwrites existing place | Confirm implementation before deployment | P0 |
| G2 | Cancel request API undefined | Not mentioned in workflow.md. E3 cancel flow may be out of scope | Ask flutter-dev whether cancel button should be shown | P2 |
| G3 | No auto-cleanup after local pin approved | app.md note 2 says manual delete. Poor UX | Add auto-cleanup logic in next cycle | P2 |
| G4 | No auto-REJECT user PENDING when admin auto-approves | backend.md note 8 says out of scope | Document as admin operational note. Implement in next cycle | P1 |
| G5 | findFavoritedByOthers isPublic=TRUE bug | After favorite then set private, MY tab missing. Previous cycle unresolved | Re-verify in R4 | P1 |
| G6 | OTHER category seed data existence | place_categories OTHER row not confirmed. Missing row causes FK violation on save | SELECT * FROM place_categories WHERE category_code=OTHER - must confirm | P0 |
| G7 | Cancel API IDOR check | S3 applies when cancel API implemented. Skip if not yet | When cancel API added: verify requester_id ownership check | P0 (when implemented) |

---

## Automation Test Candidates

### For backend-dev (JUnit5 + Mockito)

PlaceChangeRequestServiceTest:
- USER CREATE request creation success

- CREATE request exceeds 20 limit > PLACE_REQUEST_LIMIT_EXCEEDED

- UPDATE type duplicate PENDING > PLACE_REQUEST_ALREADY_PENDING

- ADMIN request auto-approves + applyToPlaces called (status=APPROVED returned)

- approve() success: places updated + status=APPROVED

- approve() on already APPROVED > PLACE_REQUEST_ALREADY_REVIEWED

- reject() success: places unchanged + status=REJECTED

- CREATE payload clientUuid already exists > approval rejected (after G1 implemented)

- applyToPlaces failure > @Transactional rollback (status=PENDING preserved)

- IDOR: cancel another user request > 403


AdminPlaceChangeRequestControllerTest (MockMvc):
- USER JWT on GET /admin/place-change-requests > 403

- No auth on GET /admin/* > 401

- ADMIN JWT on GET /admin/* > 200


### For flutter-dev (flutter_test)

- local_place_repository_test.dart: insert / listActive / softDelete / attachRequest / syncRequestStatus

- place_coordinate_edit_screen_test.dart: response status=approved shows immediately applied snackbar

- local_place_detail_screen_test.dart: button state per requestStatus (public request / pending / retry / remove local) rendering
