# 주소 검색 (Geocoding) 통합 가이드 — 백엔드

> 사용자가 미등록 장소(집, 신규 스팟 등)를 등록하기 위해 주소로 좌표 조회하는 흐름.
> 흐름: 앱 → `GET /api/v1/places/geocode?query=서울시 강남구...` → 백엔드 → NCP Geocoding API → 좌표+주소 응답 → 앱은 폼으로 이동해 이름/카테고리 입력 → 로컬 저장

## 사전 조건 (이미 완료)

- `application.yml`에 `naver.maps.geocoding-url: https://maps.apigw.ntruss.com/map-geocode/v2` 세팅됨
- `naver.maps.client-id/client-secret` = NCP Maps 인증키 (env var: `NAVER_MAPS_CLIENT_ID/SECRET`)
- `NaverMapsProperties` record에 `geocodingUrl`, `clientId`, `clientSecret` 필드 이미 있음

## 구현 순서

### 1. NCP Geocoding API 응답 DTO

`brd_be/src/main/java/com/bikeridediary/infra/naver/maps/dto/NaverGeocodeResponse.java`

```java
package com.bikeridediary.infra.naver.maps.dto;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

import java.util.List;

// NCP Geocoding v2 응답. 필요한 필드만 명시.
@JsonIgnoreProperties(ignoreUnknown = true)
public record NaverGeocodeResponse(
        String status,
        List<Address> addresses,
        int meta_totalCount,
        String errorMessage
) {
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Address(
            String roadAddress,     // 도로명주소 (예: "서울특별시 강남구 테헤란로 152")
            String jibunAddress,    // 지번주소 (예: "서울특별시 강남구 역삼동 737")
            String englishAddress,
            String x,               // 경도 (문자열, WGS84)
            String y                // 위도 (문자열, WGS84)
    ) {}
}
```

**중요**: NCP Geocoding은 좌표를 x=경도/y=위도 **문자열**로 반환. `Double.parseDouble` 필요.

### 2. NaverGeocodingClient

`brd_be/src/main/java/com/bikeridediary/infra/naver/maps/NaverGeocodingClient.java`

```java
package com.bikeridediary.infra.naver.maps;

import com.bikeridediary.infra.naver.maps.dto.NaverGeocodeResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.util.UriComponentsBuilder;

import java.nio.charset.StandardCharsets;

@Slf4j
@RequiredArgsConstructor
@Component
public class NaverGeocodingClient {

    private final NaverMapsProperties properties;
    private final RestTemplate restTemplate;

    public NaverGeocodeResponse geocode(String query) {
        HttpHeaders headers = new HttpHeaders();
        headers.set("X-NCP-APIGW-API-KEY-ID", properties.clientId());
        headers.set("X-NCP-APIGW-API-KEY", properties.clientSecret());

        String url = UriComponentsBuilder.fromUriString(properties.geocodingUrl() + "/geocode")
                .queryParam("query", query)
                .queryParam("count", 5)
                .encode(StandardCharsets.UTF_8)
                .build()
                .toString();

        return restTemplate
                .exchange(url, HttpMethod.GET, new HttpEntity<>(headers), NaverGeocodeResponse.class)
                .getBody();
    }
}
```

### 3. 앱 응답 DTO

`brd_be/src/main/java/com/bikeridediary/domain/place/dto/GeocodeResultResponse.java`

```java
package com.bikeridediary.domain.place.dto;

import java.math.BigDecimal;

// 주소 검색 결과 (앱에 반환할 형태)
public record GeocodeResultResponse(
        String roadAddress,   // 도로명주소 (nullable — 신규 도로명 없는 지역)
        String jibunAddress,  // 지번주소
        BigDecimal latitude,
        BigDecimal longitude
) {
    public static GeocodeResultResponse from(
            com.bikeridediary.infra.naver.maps.dto.NaverGeocodeResponse.Address addr
    ) {
        return new GeocodeResultResponse(
                addr.roadAddress() == null || addr.roadAddress().isBlank() ? null : addr.roadAddress(),
                addr.jibunAddress(),
                new BigDecimal(addr.y()).setScale(7, java.math.RoundingMode.HALF_UP),
                new BigDecimal(addr.x()).setScale(7, java.math.RoundingMode.HALF_UP)
        );
    }
}
```

**주의**: places 테이블 좌표 scale이 7이므로 GeocodeResultResponse도 scale 7로 통일 (`claude-memory.md`의 BigDecimal precision/scale 규칙).

### 4. PlaceService에 geocode 메서드

`PlaceService.java`에 추가:

```java
private final NaverGeocodingClient naverGeocodingClient;  // 필드 추가

public List<GeocodeResultResponse> geocode(String query) {
    NaverGeocodeResponse response = naverGeocodingClient.geocode(query);
    if (response == null || !"OK".equals(response.status()) || response.addresses() == null) {
        return List.of();
    }
    return response.addresses().stream()
            .map(GeocodeResultResponse::from)
            .toList();
}
```

### 5. PlaceController 엔드포인트

`PlaceController.java`에 추가:

```java
@Operation(summary = "주소 검색 (NCP Geocoding)")
@GetMapping("/geocode")
public ResponseEntity<ApiResponse<List<GeocodeResultResponse>>> geocode(
        @RequestParam("query") String query
) {
    return ResponseEntity.ok(ApiResponse.ok(placeService.geocode(query)));
}
```

### 6. SecurityConfig

`GET /api/v1/places/geocode`는 로그인 유저만 사용하도록 유지 (현재 SecurityConfig 그대로 두면 `authenticated()` 걸림). 게스트도 허용하고 싶으면 `GET_PERMIT_ALL_ENDPOINTS`에 추가.

## 테스트 예시

```bash
curl -H "Authorization: Bearer <JWT>" \
  "http://localhost:8081/api/v1/places/geocode?query=서울특별시 강남구 테헤란로 152"
```

기대 응답:
```json
{
  "data": [
    {
      "roadAddress": "서울특별시 강남구 테헤란로 152",
      "jibunAddress": "서울특별시 강남구 역삼동 737",
      "latitude": 37.5006540,
      "longitude": 127.0364710
    }
  ]
}
```

## 주의사항

1. **NCP Geocoding 무료 한도**: 5,000회/일. 초과 시 유료 (건당 원). 앱에서 검색어 debounce 필수(400ms 이상).
2. **좌표 문자열 파싱**: NCP는 x/y를 String으로 반환. NPE 방지 위해 y값 null 체크 후 `new BigDecimal(str)`. 파싱 실패 시 예외 처리(BusinessException).
3. **주소 결과 없음**: `status=OK`이면서 `addresses=[]`인 경우 있음. 앱은 "검색 결과 없음" 안내.
4. **status 값**:
   - `OK`: 성공
   - `INVALID_REQUEST`: 잘못된 파라미터
   - `SYSTEM_ERROR`: NCP 서버 오류
   - errorMessage 필드에 상세 있음
5. **좌표계**: NCP Geocoding은 이미 **WGS84** 반환 (변환 불필요). 지역검색과 달리 x10^7 스케일 아님.
