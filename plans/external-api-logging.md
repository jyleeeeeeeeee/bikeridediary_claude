# 외부 API 호출 로깅

작성일: 2026-08-12
상태: 결정 확정 (D1~D7), 게이트 2 진행 중

## 배경

Naver Directions/Geocoding/Search, Kakao Local, Opinet, OpenWeather 등 외부 API 호출을 DB 기록.

**목적**:
- 사용량 모니터링 (Naver 무료 60,000/월 등 한도 추적)
- 이상 사용/오남용 탐지
- 유저별 차단 근거 확보
- API별 응답시간·에러율 파악

## 확정 결정

| # | 결정 | 근거 |
|---|------|------|
| D1 | **A**: 모든 외부 API 대상 | 인프라 한 번 만들면 확장 비용 적음 |
| D2 | **A**: PostgreSQL `api_call_logs` 테이블 | 관리자 조회 UI 확장 가능, 트래픽 폭증 전엔 부담 미미 |
| D3 | **A**: Spring AOP + `@LogExternalApi` 어노테이션 | 각 client 수정 없이 어노테이션만 |
| D4 | 기본 필드 + `requestParams JSONB` + `errorMessage` | 좌표·유저 활동 분석용 파라미터 저장, API 키/시크릿 절대 저장 안 함 |
| D5 | **A**: 관리자 API (`GET /api/v1/admin/api-logs?...`) | UI는 필요해지면 어드민 앱에 추가 |
| D6 | **B**: 90일 후 자동 삭제 (스케줄러) | 파티션 오버킬, 스케줄러 delete 충분 |
| D7 | 인덱스 3개: `(api_name, called_at DESC)`, `(user_id, called_at DESC) WHERE user_id NOT NULL` partial, `(called_at)` | API별/유저별/기간 조회 커버 |

## 스코프

### In-scope
- `api_call_logs` 테이블 + 3개 인덱스
- Spring AOP + `@LogExternalApi(apiName = "...")` 어노테이션 (apiName은 String — enum 아님. `ApiNames` 상수 클래스 참조 권장. 2026-08-12 결정)
- 기존 외부 API 클라이언트 5종에 어노테이션 부착:
  - `NaverMapsClient.directions()`
  - `NaverGeocodingClient` (있으면 관련 메서드)
  - `NaverSearchClient.searchLocal()`
  - `KakaoLocalClient` (station 도메인 관련. 없으면 pass)
  - `OpinetClient` (station 도메인 관련. 없으면 pass)
  - `OpenWeatherClient` (있으면)
- 관리자 조회 API: `GET /api/v1/admin/api-logs?apiName=&userId=&from=&to=&page=&size=`
- 90일 보관 스케줄러 (Spring `@Scheduled`)
- 민감정보 마스킹 (요청 파라미터에서 `apiKey`/`clientSecret`/`clientId` 필드 자동 제거)

### Out-of-scope
- 어드민 앱 UI (별도 사이클, 필요 시)
- 실시간 알림 (한도 근접 시 슬랙 등)
- 대시보드 (Grafana)
- 응답 body 저장 (크기 방어)

## 저장 필드

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID PK | 로그 ID |
| no | BIGINT UNIQUE DEFAULT nextval | 조회용 친숙 번호 (프로젝트 컨벤션) |
| user_id | UUID nullable FK → users(id) ON DELETE SET NULL | 호출 유저 (인증 없이 호출된 경우 null) |
| api_name | VARCHAR(50) NOT NULL | 예: `NAVER_DIRECTIONS`, `KAKAO_LOCAL` |
| endpoint | VARCHAR(200) NOT NULL | 실제 URL path (쿼리 제외) |
| http_method | VARCHAR(10) NOT NULL | GET/POST 등 |
| status_code | INTEGER | 응답 HTTP status, 예외 시 null |
| response_time_ms | INTEGER NOT NULL | 소요 시간 |
| request_params | JSONB | 마스킹된 파라미터 (api-key/secret 제거) |
| error_message | TEXT | 실패 시 예외 메시지 |
| called_at | TIMESTAMP NOT NULL DEFAULT now() | 호출 시각 |

## 담당 분배 (게이트 2 병렬)

| # | 담당 | 산출물 |
|---|-----|-------|
| 1 | dba | `guides/external-api-logging-schema.md` — DDL + 인덱스 + 보관 정책 스니펫 |
| 2 | backend-dev | `guides/external-api-logging-backend.md` — AOP 애스펙트 + `@LogExternalApi` 어노테이션 + `ApiCallLogEntity`/Repository/Service + 관리자 조회 API + 스케줄러 + 각 client 어노테이션 부착 위치 |

## 게이트 3

사용자가 게이트 2 가이드로 백엔드 직접 구현 → 실기기 테스트 대신 curl로 로깅 확인 → 완료

## 위험 요소

- **`requestParams` JSONB 크기**: Naver Directions는 path 좌표를 요청 아닌 응답으로 받으니 requestParams는 작음. 문제 없음
- **AOP 성능 오버헤드**: 외부 API 호출 자체가 200~1000ms 규모라 AOP 오버헤드(<1ms) 무의미
- **로그 삽입 실패 시 원 요청 영향**: AOP는 `@AfterReturning`/`@AfterThrowing`에서 로그 삽입. 로그 삽입 실패해도 원 API 호출 결과는 정상 반환하도록 예외 catch → warn log
- **마스킹 누락**: `apiKey`, `client-id`, `client-secret`, `X-NCP-APIGW-API-KEY*` 헤더는 애초에 client 코드가 헤더로 넣지 파라미터로 안 넣음. 파라미터 마스킹은 방어적 성격
- **동시성**: 스케줄러 delete와 실시간 INSERT 충돌 → PostgreSQL MVCC로 자연스레 처리
