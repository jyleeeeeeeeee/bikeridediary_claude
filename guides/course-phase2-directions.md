# Naver Directions 5/15 통합 가이드

작성일: 2026-08-04 (최신 갱신: 실제 구현 반영)
스코프: 라이딩 코스 2차 — 경로 계산 (D5=B 사용자 "경로 미리보기" 버튼 클릭 시 1회 호출, 자동 호출 없음)

---

## 실제 구현 상태 (사용자 커스텀 반영)

**단일 클라이언트로 통합**: `infra/naver/maps/NaverMapsClient` — search(Geocoding) + directions(Directions 5/15) 한 클래스에 배치.
가이드 초안의 `NaverDirectionsClient` 분리안 대신 채택. 헤더/인증/헬퍼 재사용 이득.

**바이크 서비스 특성 반영**:
- `option=traavoidcaronly` — 자동차전용도로 회피 (오토바이 진입금지 도로 자동 회피)
- `cartype=1` — 승용차 (Naver Directions에 오토바이 옵션 없어 승용차로 요청 후 회피 옵션으로 보완)

**waypoints 개수 기반 자동 API 분기**:
- `waypoints.size() <= 5` → Directions 5 (`properties.directionUrl()`)
- `waypoints.size() > 5` → Directions 15 (`properties.direction15Url()`)
- 근거: 공식 문서상 "5"/"15"는 waypoints(VIA) 개수 상한. Directions 5가 저렴하니 우선 사용.

---

## 사전 확인

`NaverMapsProperties`는 `infra/naver/maps/NaverMapsProperties.java`에 이미 존재:

```java
@ConfigurationProperties(prefix = "naver.maps")
public record NaverMapsProperties(
    String clientId,
    String clientSecret,
    String directionUrl,      // Directions 5 endpoint
    String direction15Url,    // Directions 15 endpoint
    String geocodingUrl,
    String reverseGeocodingUrl
) {}
```

`application.yml`에 `direction-url`, `direction15-url` 세팅됨. 각 프로필 yml에서
`naver.maps.client-id` / `naver.maps.client-secret` 환경변수로 주입.

---

## 파일 목록

| 경로 | 상태 | 설명 |
|-----|------|-----|
| `infra/naver/maps/NaverMapsClient.java` | 구현 완료 | search + directions 통합 |
| `infra/naver/maps/dto/NaverDirectionsResponse.java` | 구현 완료 | `route.traavoidcaronly` 필드 |
| `infra/naver/maps/dto/NaverGeocodeResponse.java` | 기존 | search 응답 |
| `global/exception/ErrorCode.java` | 수정 | ErrorCode 2개 추가 (아래) |

---

## 1. NaverDirectionsResponse DTO (실제 구현)

파일: `src/main/java/com/bikeridediary/infra/naver/maps/dto/NaverDirectionsResponse.java`

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record NaverDirectionsResponse(
        int code,
        String message,
        Route route
) {
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Route(
            // option=traavoidcaronly 응답 (자동차 전용, 고속도로 제외)
            List<RoutePath> traavoidcaronly
    ) {}

    @JsonIgnoreProperties(ignoreUnknown = true)
    public record RoutePath(
            Summary summary,
            @JsonProperty("path") List<List<Double>> path
    ) {}

    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Summary(
            int distance,                // 미터
            int duration,                // 밀리초 (D2=A로 현재 미저장)
            List<List<Double>> bbox      // [[minLng,minLat],[maxLng,maxLat]] — 앱 fitBounds용
    ) {}
}
```

**bbox 확인**: Naver Directions summary는 `bbox` 필드를 제공한다. 응답 예:
```json
"summary": {
  "distance": 14300,
  "duration": 1800000,
  "bbox": [[127.0073, 37.6029], [127.0146, 37.6611]],
  ...
}
```
서버가 path를 순회해 min/max 계산할 필요 없음. Naver가 준 그대로 CoursePreviewResponse에 전달.

**중요**: Naver Directions 응답의 `route` 하위 필드명은 요청 `option` 값과 **동일**하게 옵니다.
- `option=trafast` → `route.trafast[0]`
- `option=traoptimal` → `route.traoptimal[0]`
- `option=traavoidcaronly` → `route.traavoidcaronly[0]`

옵션 변경 시 DTO 필드명도 함께 바꿔야 함.

---

## 2. NaverMapsClient (실제 구현 요약)

파일: `src/main/java/com/bikeridediary/infra/naver/maps/NaverMapsClient.java`

핵심 로직:

```java
public NaverDirectionsResponse directions(
        BigDecimal startLng, BigDecimal startLat,
        BigDecimal goalLng, BigDecimal goalLat,
        List<Waypoint> waypoints) {

    UriComponentsBuilder builder = UriComponentsBuilder
            .fromHttpUrl(waypoints.size() <= 5 ? properties.directionUrl() : properties.direction15Url())
            .queryParam("start", startLng.toPlainString() + "," + startLat.toPlainString())
            .queryParam("goal", goalLng.toPlainString() + "," + goalLat.toPlainString())
            .queryParam("cartype", 1)
            .queryParam("option", "traavoidcaronly");

    if (!waypoints.isEmpty()) {
        String waypointStr = waypoints.stream()
                .map(w -> w.longitude().toPlainString() + "," + w.latitude().toPlainString())
                .collect(Collectors.joining("|"));
        builder.queryParam("waypoints", waypointStr);
    }

    // ... executeGetRequest 호출 + 응답 검증 (traavoidcaronly 필드 확인)
}

public record Waypoint(BigDecimal latitude, BigDecimal longitude) {}
```

응답 검증 부분에서 **`response.route().traavoidcaronly()`** 참조해야 함 (아래 "⚠️ 컴파일 오류 조치" 참조).

---

## ⚠️ 컴파일 오류 조치 (2026-08-04 현재)

DTO는 `trafast` → `traavoidcaronly`로 바뀌었지만, `NaverMapsClient.directions()` 검증부가 여전히 `.trafast()` 참조 중. **컴파일 에러 상태**.

`NaverMapsClient.java` 91-95줄 수정:

```java
// AS-IS (컴파일 에러)
if (response.route() == null
        || response.route().trafast() == null
        || response.route().trafast().isEmpty()) {
    log.error("[NaverDirections] trafast 경로 없음");
    throw new BusinessException(ErrorCode.COURSE_DIRECTIONS_FAILED);
}

// TO-BE
if (response.route() == null
        || response.route().traavoidcaronly() == null
        || response.route().traavoidcaronly().isEmpty()) {
    log.error("[NaverDirections] traavoidcaronly 경로 없음");
    throw new BusinessException(ErrorCode.COURSE_DIRECTIONS_FAILED);
}
```

---

## 3. ErrorCode 추가

파일: `src/main/java/com/bikeridediary/global/exception/ErrorCode.java`

코스 블록에 추가:

```java
COURSE_DIRECTIONS_FAILED(HttpStatus.BAD_GATEWAY, "COURSE_DIRECTIONS_FAILED", "경로를 계산할 수 없습니다. 출발지·경유지·도착지를 확인해 주세요"),
COURSE_DIRECTIONS_WAYPOINTS_LIMIT(HttpStatus.BAD_REQUEST, "COURSE_DIRECTIONS_WAYPOINTS_LIMIT", "경유지는 최대 15개까지 지정할 수 있습니다"),
```

`HttpStatus.BAD_GATEWAY` 사용 이유: 외부 API(Naver) 오류는 우리 서버 문제가 아니므로 502 적합.

---

## 4. CourseService에서 응답 파싱

```java
// path JSON 생성 헬퍼
private String buildPathJson(NaverDirectionsResponse response) {
    List<List<Double>> path = response.route().traavoidcaronly().get(0).path();
    try {
        return objectMapper.writeValueAsString(path);
    } catch (JsonProcessingException e) {
        throw new BusinessException(ErrorCode.COURSE_DIRECTIONS_FAILED);
    }
}

private int buildDistanceMeters(NaverDirectionsResponse response) {
    return response.route().traavoidcaronly().get(0).summary().distance();
}
```

`ObjectMapper`는 Spring Boot가 자동 등록. 생성자 주입.

CourseService에서 클라이언트 호출:

```java
NaverDirectionsResponse response = naverMapsClient.directions(
        start.longitude(), start.latitude(),
        end.longitude(), end.latitude(),
        vias  // List<NaverMapsClient.Waypoint>
);
```

---

## 5. 좌표 형식 정리

| 항목 | 형식 | 예시 |
|-----|-----|-----|
| Directions API 파라미터 | `lng,lat` (경도 먼저) | `127.0276368,37.4979502` |
| Directions API 응답 path | `[[lng, lat], ...]` | `[[127.027, 37.497], ...]` |
| 앱/DB 저장 | `lat, lng` 각각 BigDecimal | `latitude=37.4979502, longitude=127.0276368` |
| 앱 NLatLng | `NLatLng(lat, lng)` | 위도 먼저 |

**주의**: Naver API는 경도(lng) 먼저, NLatLng은 위도(lat) 먼저. 파라미터 조립 시 혼동 금지.

---

## 6. Naver Directions 5 vs 15 분기

공식 문서 기준:
- **Directions 5**: `waypoints` 파라미터 최대 5개 (파이프 `|` 구분)
- **Directions 15**: `waypoints` 파라미터 최대 15개

여기서 waypoints는 role=VIA인 지점만 (START/GOAL은 별도 파라미터 `start`, `goal`). 즉:
- VIA 0~5개 → Directions 5 사용 가능 (저렴)
- VIA 6~15개 → Directions 15 필수

`waypoints.size() <= 5` 분기 로직이 공식 스펙과 일치. 저렴한 API 우선 사용.

---

## 7. D5=B 호출 정책 (2026-08-04 최종 확정, A에서 재변경)

- **버튼 클릭 호출**: 사용자가 "경로 미리보기" 버튼을 눌렀을 때만 `POST /api/v1/courses/preview` 호출
- **자동 호출 안 함**: waypoint 좌표 미세 조정으로 같은 경로인데 API 낭비되는 것 방지
- **stale 처리**: waypoint 변경 시 이전 preview 결과 무효화. 저장 버튼 비활성 + "경로를 다시 확인해 주세요" 안내
- **저장은 preview 이후에만 가능**: 옵션 B(path 앱 전송)와 결합. 사용자가 지도에서 확인한 경로 그대로 저장
- **Naver Directions 유료 한도**: 무료 60,000회/월. 버튼 방식이라 사용자당 편집 세션에서 1~3회 호출 수준 예상
- **Directions 5 우선 사용**: waypoints 5개 이하 대다수 시나리오에서 저렴한 5 사용

---

## 8. 예상 함정

1. **option 변경 시 DTO 필드명 동기화 필수**: `option=trafast`로 바꾸면 `Route.trafast` 필드, `traoptimal`이면 `Route.traoptimal` 필드 필요. 미스매치 시 항상 null → 즉시 예외.
2. **path 크기**: 100km 코스 기준 path 배열이 약 2,000개 이상 좌표. `List<List<Double>>` 역직렬화 시 Jackson 기본 설정으로 충분하지만, RestTemplate 기본 버퍼(256KB)가 타이트할 수 있음. 필요 시 `RestTemplateBuilder`에서 조정.
3. **경유지 없는 경우**: `waypoints` 파라미터를 아예 생략해야 함 (빈 문자열로 보내면 API 오류). 현재 `!waypoints.isEmpty()` 체크로 처리됨.
4. **role=VIA만 waypoints 파라미터에 포함**: role=START는 `start`, role=GOAL은 `goal` 파라미터로 별도 전달. Service 레이어에서 role 필터링 필요.
5. **traavoidcaronly가 유일한 옵션인 이유**: 오토바이는 자동차전용도로 진입 금지. `trafast`는 고속도로 포함해서 실제 라이더가 못 타는 경로 반환 위험. `traavoidcaronly`가 도메인에 딱 맞음.
6. **D5=B 채택 → 호출 빈도 낮음**: 편집 세션당 preview 1~3회 예상. 로그 레벨은 프로덕션에서 INFO 유지.
