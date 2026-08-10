# 라이딩 코스 2차 스코프 — 코스 생성/편집

작성일: 2026-07-29
상태: pm 명세 (사용자 승인 대기)

## 배경

1차 MVP(2026-07-16)로 코스 조회/즐겨찾기/삭제는 완성. 코스를 "만드는" 흐름이 없음(FAB → "곧 지원 예정" 스낵바).

## 1. 명세 (What)

### In-Scope
1. **네이버 Directions 15 통합** — 서버가 API 호출, 응답 path를 코스 데이터로 저장
2. **waypoint 지도 편집 UI (앱)** — 출발지/경유지(최대 15)/도착지 지정
   - 지도에서 롱프레스로 임의 지점 찍기
   - 등록된 place에서 선택 (place_id 스냅샷)
   - 순서 재배열, 삭제
3. **코스 저장/편집 백엔드 엔드포인트**
   - `POST /api/v1/courses` (생성)
   - `PATCH /api/v1/courses/{id}` (편집, 작성자 본인만)
   - path 미리보기용 `POST /api/v1/courses/preview` (저장 없이 Directions 호출)
4. **"복사 편집" 흐름 완성** — 남의 코스 상세에서 복사 → 편집 화면 진입 → 자기 코스로 저장 (source_course_id 세팅)
5. **거리(distance_meters) 자동 계산** — Directions 응답 summary에서 파싱
6. **미리보기 → 확정 저장 UX** — 편집 중 지도에 실시간 폴리라인 표시

### Out-of-Scope (별도 사이클)
- GPX 업로드/다운로드
- GPS 자동 기록으로 코스 저장
- 코스 난이도/고도 분석
- 소요시간, 예상 도착시간, 유료도로 옵션 UI
- 오프라인 편집 (Naver Directions는 온라인 필수)
- 코스 공개→비공개 전환 시 즐겨찾기 정합성 (findFavoritedByOthers 이슈, 별도 태스크로 남김)

## 2. 핵심 결정 필요 사항

### D1. path 좌표 저장 방식
현재 `courses.path TEXT`에 JSON 문자열로 `[[lng,lat],...]` 저장 중. Directions 응답은 수천 개 좌표 배열이라 크기가 큼.

| 옵션 | 설명 | 크기(예상) | tradeoff |
|---|---|---|---|
| A | 현행 유지 (JSON `[[lng,lat],...]`) | 100km 코스 ≈ 200KB | 파싱 단순, 크기 큼 |
| B | Naver의 인코딩된 polyline 문자열 (Google Encoded Polyline Algorithm 유사) 있으면 그대로 저장 | ≈ 20~30KB | Naver가 제공하면 이득, 없으면 A |
| C | 좌표 압축 저장 (delta encoding) — 서버에서 직접 압축 | ≈ 40~50KB | 코드 복잡, PostgreSQL BYTEA 검토 |

**추천: A (현행 유지)**. 2차 스코프에선 데이터 크기가 병목은 아니고, 1차와 스키마 호환. B/C는 사용자 수 증가 시 재검토.

### D2. Naver Directions 응답 요약 저장
Directions 응답의 `route.trafast[0].summary`에 총 거리/시간/유료도로 등 포함.

| 옵션 | 저장 필드 |
|---|---|
| A | `distance_meters`만 저장 (현행 스키마 그대로) |
| B | + `duration_seconds`, `toll_fare` 컬럼 추가 |
| C | + summary 전체를 `summary_json TEXT`로 저장 |

**추천: A**. 2차 UI에 소요시간/요금 표시가 없어 지금 저장 안 해도 됨. 필요해지면 컬럼 추가는 저비용.

### D3. waypoint의 place_id 연동
지도에서 선택할 때 등록된 place / 임의 지점 두 경로.

| 옵션 | UX |
|---|---|
| A | 지도 롱프레스 = 임의 지점. 하단시트에서 place 선택 = place_id 세팅. 두 흐름 분리 |
| B | 지도 롱프레스만 지원. 근처 place가 있으면 자동 매칭 (반경 50m) |
| C | place 선택 흐름만 지원 (임의 지점 불가) — 자유도 낮음 |

**추천: A**. 1차 MVP place 인프라 재사용, 자유도 최대. B의 자동 매칭은 오탐 위험.

### D4. 편집 UI에서 waypoint 순서 재배열
| 옵션 | UX |
|---|---|
| A | 리스트에서 드래그(ReorderableListView) |
| B | 각 항목에 위/아래 화살표 버튼 |
| C | 지도의 마커 드래그 (순서는 자동 계산: 시작 최근접순) |

**추천: A**. Flutter 표준 위젯, 학습곡선 낮음. C는 UX 애매(자동 순서가 사용자 의도와 다를 수 있음).

### D5. Naver Directions 호출 시점
| 옵션 | 시점 |
|---|---|
| A | waypoint 변경마다 자동 호출 (실시간 폴리라인 반영) |
| B | "경로 미리보기" 버튼 누를 때만 호출 |
| C | 저장 버튼 눌렀을 때만 호출 (미리보기 없음) |

**추천: B**. Naver Directions는 유료(무료 60,000/월). 매 변경마다 호출은 낭비. 편집 완료 후 명시적으로 호출. 단, waypoint 변경 시 이전 미리보기는 stale로 표시.

**최종 결정 (2026-08-04, 재변경): B** — 좌표 미세 변경 시 같은 경로인데 API가 쓸데없이 호출되는 낭비 방지. 사용자가 명시적으로 "경로 미리보기" 버튼 클릭 시에만 호출. waypoint 변경 시 이전 preview 결과는 stale 상태로 표시(저장 버튼 비활성 + "경로를 다시 확인해 주세요" 안내).

### D6. "복사 편집" (남의 코스에서 파생)
1차 MVP에서 상세 화면에 버튼은 있으나 스낵바만.

| 옵션 | 구현 |
|---|---|
| A | 이번 스코프 포함 — 복사 시 waypoints + name 프리필, source_course_id 세팅 |
| B | 별도 사이클 (이번은 신규 생성만) |

**추천: A**. 편집 UI가 완성되면 프리필만 추가하면 됨. source_course_id 컬럼도 이미 있음.

### D7. Directions API 실패 대응
| 옵션 | 처리 |
|---|---|
| A | 실패 시 저장 자체를 막음 (사용자에게 재시도 요구) |
| B | 실패 시 waypoints만으로 저장(path는 waypoints 직선 연결) — 나중에 재계산 |
| C | 로컬 우선(Phase 3처럼): waypoints만 로컬 저장, 온라인 복구 시 서버가 Directions 호출해 path 채움 |

**추천: A**. 코스의 핵심 가치가 path인데 없이 저장하면 반쪽. 2차는 온라인 전용 명시. C는 Phase 3 확장이라 별도 스코프.

### D8. 편집 vs 신규 저장 (기존 코스 편집 시 path)
자기 코스 편집 시 waypoints를 그대로 두고 이름/공개여부만 바꾸는 경우 vs waypoints를 바꿔 path도 재계산해야 하는 경우.

**결정 필요 없음(제안)**: PATCH 요청 payload에 `regeneratePath: boolean` 플래그. waypoints 변경 시 앱이 true로 보냄. false면 서버는 name/isPublic만 업데이트, Directions 호출 안 함.

## 3. 태스크 분해

### 게이트 ① pm 명세 (현재)
| # | 담당 | 작업 | 산출물 |
|---|-----|-----|--------|
| 0 | pm | 이 문서 | plans/riding-course-phase2.md |

**사용자 승인 필요**: D1~D7 옵션 결정.

### 게이트 ② 병렬 (승인 후)

| # | 담당 | 작업 | 선행 |
|---|-----|-----|------|
| 1 | dba | 스키마 변경 필요 여부 확정 (D2 결정에 따라). 추천안(A) 채택 시 변경 없음. `guides/course-phase2-schema.md`에 결정과 근거 기록 | D2 |
| 2 | backend-dev | Naver Directions 15 통합 가이드 — `NaverDirectionsClient` 신설 (기존 `NaverGeocodingClient` 패턴 재사용), request/response DTO, 에러 매핑 | D1, D7 |
| 3 | backend-dev | Course 생성/편집/미리보기 서비스 로직 가이드 — waypoints diff, LWW 검증, path 생성, source_course_id 세팅 | 2 |
| 4 | backend-dev | Controller 3개 엔드포인트 가이드 (POST/PATCH/POST preview), DTO, ErrorCode, SecurityConfig 확인 | 3 |
| 5 | publisher | 코스 생성/편집 화면 HTML 목업 6개 — 신규(빈상태), waypoint 리스트, place 선택 시트, 지도 편집, 미리보기, 저장 확정, 복사편집 프리필 | D3, D4, D5 |

산출물:
- `guides/course-phase2-directions.md` (Naver Directions 호출 스니펫)
- `guides/course-phase2-backend.md` (Service/Controller 스니펫)
- `mockups/riding-course-phase2/` (HTML 6개 + index)

**사용자 승인 필요**: 백엔드 가이드로 사용자 직접 구현 → 실제 배포 → 앱 착수.

### 게이트 ③ 앱 구현 + 검토

| # | 담당 | 작업 | 선행 |
|---|-----|-----|------|
| 6 | flutter-dev | `features/riding_course/presentation/course_edit_screen.dart` — 지도 + waypoint 리스트 + 재배열 + 미리보기 | 4, 5 |
| 7 | flutter-dev | `PlaceSelectionSheet` — 지도에서 등록 place 검색/선택 (기존 place repository 재사용) | 6 |
| 8 | flutter-dev | Repository/Provider 확장 — `createCourse`, `updateCourse`, `previewPath` | 6 |
| 9 | flutter-dev | 라우팅 — `/riding-courses/new`, `/riding-courses/:id/edit`, `/riding-courses/:id/copy` (신규 프리필) | 6 |
| 10 | flutter-dev | 홈 FAB 실제 연결 (스낵바 제거), 상세 화면 편집 버튼(내 코스만), 복사편집 버튼 실제 연결 | 9 |
| 11 | code-reviewer | 백엔드 + 앱 코드 리뷰 (BigDecimal precision/scale, path 크기, IDOR, Naver 시크릿 노출) | 10 |
| 12 | qa | 시나리오 검증 — 정상 생성/편집/복사, waypoint 15개 상한, Directions 실패, 오프라인 진입, 남의 코스 편집 차단, 삭제 후 파생 코스 유지 | 10 |

## 4. 3단계 게이트

```
[게이트 1] pm 명세 (이 문서)
  ↓ 사용자 승인 (D1~D7 확정)
[게이트 2] publisher(목업) + dba(스키마) + backend-dev(백엔드 스펙) 병렬
  ↓ 사용자 승인 (백엔드 직접 구현/배포 완료)
[게이트 3] flutter-dev(앱 구현) → code-reviewer + qa
```

## 5. 위험 요소

- **Naver Directions 유료**: 무료 60,000/월. 실시간 호출(D5=A) 채택 시 사용자당 소비량 폭증 위험 → B 추천
- **path 크기**: 장거리 코스(500km+)에서 응답 JSON 500KB+ 가능. Dio 기본 타임아웃/크기 제한 확인 필요
- **waypoint 15개 상한**: Naver Directions 15의 하드 리밋. 앱 UI에서도 15개 도달 시 추가 차단 필요
- **네이버 지도 SDK와 좌표 정밀도**: `BigDecimal(scale=7)` ↔ `NLatLng(double)` 변환 시 손실 최소 (7자리 유지)
- **편집 중 오프라인 진입**: waypoints 로컬 편집은 되나 미리보기/저장 불가. UX 명확한 안내 필요
- **place 삭제된 waypoint의 편집**: place_id → null 되었지만 좌표/이름 스냅샷은 유지. 편집 시 해당 waypoint는 임의 지점 취급

## 6. claude-memory.md / CLAUDE.md 갱신 안내 (완료 시)

CLAUDE.md "완료된 작업"에 항목 34로 추가:
- 34. 라이딩 코스 2차 스코프 (코스 생성/편집)
  - Naver Directions 15 통합, 미리보기, path 저장 방식(선택 옵션)
  - 앱 편집 UI, waypoint 재배열, place 선택 시트
  - 복사 편집 흐름 완성
  - 결정 요약(D1~D7), 미결(GPX/GPS, 소요시간 저장 시점)

CLAUDE.md "다음 단계" 갱신:
- 삭제: "라이딩 코스 2차 스코프: 코스 생성/편집 UI ..." (완료)
- 추가: "코스 소요시간/유료요금 표시 (D2 확장)", "장거리 코스 path 크기 최적화 검토"
- 유지: `findFavoritedByOthers` isPublic 조건, place 중복 좌표 근접 조합, 주유소 지도 통합, 카카오/네이버 딥링크, place_categories 시드, SecurityConfig /places/**, 뱅킹 서버 백업, FuelingServiceTest, 무한 스크롤 확장, 실기기 오프라인 시나리오

claude-memory.md "지도/POI 인프라" 섹션 근처에 "라이딩 코스 편집" 서브섹션 추가:
- Naver Directions 호출 시점, path 저장 방식, 편집 시 regeneratePath 플래그 정책

## 7. 결정 필요 사항 요약 (사용자 응답 대기)

- [ ] **D1** path 저장: A(현행 JSON) 추천
- [ ] **D2** 요약 저장: A(distance만) 추천
- [ ] **D3** waypoint place 연동: A(둘 다 지원) 추천
- [ ] **D4** 순서 재배열: A(드래그) 추천
- [x] **D5** Directions 호출 시점: **B(경로 미리보기 버튼 클릭 시) 최종 확정** (2026-08-04, A에서 재변경)
- [ ] **D6** 복사편집: A(이번 스코프 포함) 추천
- [ ] **D7** Directions 실패: A(저장 차단) 추천

전부 추천안 채택하면 "다 A/B 추천대로"라고만 답해도 됨.
