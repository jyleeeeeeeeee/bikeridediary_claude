# Place 찜(Wish) API 백엔드 가이드

작성일: 2026-08-12
담당: backend-dev (사용자 직접 구현)
근거: 지도(찾아보기) 화면의 place 상세 시트에 하트 토글 UI가 추가됨. 백엔드 endpoint 필요.

## 현황

- **스키마**: `place_wishes` 테이블 완비 (schema.sql 222~229번 줄, PK `(place_id, user_id)`, CASCADE, `no` 컬럼, `idx_place_wishes_user_id` 인덱스)
- **엔티티/서비스/컨트롤러**: 없음
- **PlaceEntity에 `wishedCount` 필드 + `increaseWish()/decreaseWish()` 메서드는 이미 있음**

## 앱 계약 (이미 구현됨)

```
POST   /api/v1/places/{placeId}/wish   → 찜 추가. 이미 있으면 idempotent OK (or 409)
DELETE /api/v1/places/{placeId}/wish   → 찜 해제. 없어도 idempotent OK (or 404)
```

인증 필수. `@AuthenticationPrincipal CustomUserDetails.getUserId()` 사용.

**추가 요구**: 기존 place 조회 응답(`PlaceResponse`)에 `isWished: boolean` 필드 추가 필요. 현재 로그인 유저의 찜 여부.
- 미인증 요청이면 `isWished = false` 항상.
- `GET /api/v1/places` 목록 조회 시 유저의 wish set을 batch 조회 (N+1 방지).

## 파일 생성/수정 목록

| # | 경로 | 신규/수정 |
|---|------|----------|
| 1 | `domain/place/entity/PlaceWishEntity.java` | 신규 |
| 2 | `domain/place/entity/PlaceWishId.java` | 신규 (복합 PK) |
| 3 | `domain/place/repository/PlaceWishRepository.java` | 신규 |
| 4 | `domain/place/service/PlaceWishService.java` | 신규 |
| 5 | `domain/place/controller/PlaceWishController.java` | 신규 (or PlaceController에 통합) |
| 6 | `domain/place/dto/PlaceResponse.java` | 수정 (`isWished` 필드 추가) |
| 7 | `domain/place/service/PlaceService.java` | 수정 (`list()`에서 wish batch 조회) |
| 8 | `global/exception/ErrorCode.java` | 수정 (PLACE_WISH_ALREADY_EXISTS 등 필요 시) |

## 스니펫

### 1. PlaceWishId.java (복합 PK)

`src/main/java/com/bikeridediary/domain/place/entity/PlaceWishId.java`

```java
package com.bikeridediary.domain.place.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Embeddable;
import lombok.AccessLevel;
import lombok.EqualsAndHashCode;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

import java.io.Serializable;
import java.util.UUID;

@Embeddable
@Getter
@EqualsAndHashCode
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class PlaceWishId implements Serializable {

    @Column(name = "place_id")
    @JdbcTypeCode(SqlTypes.UUID)
    private UUID placeId;

    @Column(name = "user_id")
    @JdbcTypeCode(SqlTypes.UUID)
    private UUID userId;

    public PlaceWishId(UUID placeId, UUID userId) {
        this.placeId = placeId;
        this.userId = userId;
    }
}
```

### 2. PlaceWishEntity.java

`src/main/java/com/bikeridediary/domain/place/entity/PlaceWishEntity.java`

```java
package com.bikeridediary.domain.place.entity;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.Generated;
import org.hibernate.generator.EventType;

import java.time.LocalDateTime;
import java.util.UUID;

// 장소 찜 매핑 (place_wishes 테이블)
// PK: (place_id, user_id) 복합 — @EmbeddedId 방식 (CourseFavoriteEntity와 동일 패턴)
@Entity
@Table(name = "place_wishes")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class PlaceWishEntity {

    @EmbeddedId
    private PlaceWishId id;

    // 조회용 친숙 번호
    @Column(name = "no", insertable = false, updatable = false)
    @Generated(event = EventType.INSERT)
    private Long no;

    // 찜 생성 시각
    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;

    public static PlaceWishEntity create(UUID placeId, UUID userId) {
        PlaceWishEntity e = new PlaceWishEntity();
        e.id = new PlaceWishId(placeId, userId);
        e.createdAt = LocalDateTime.now();
        return e;
    }
}
```

### 3. PlaceWishRepository.java

`src/main/java/com/bikeridediary/domain/place/repository/PlaceWishRepository.java`

```java
package com.bikeridediary.domain.place.repository;

import com.bikeridediary.domain.place.entity.PlaceWishEntity;
import com.bikeridediary.domain.place.entity.PlaceWishId;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;
import java.util.UUID;

public interface PlaceWishRepository extends JpaRepository<PlaceWishEntity, PlaceWishId> {

    boolean existsById(PlaceWishId id);

    /**
     * 페이지 단위 place 목록에 대한 찜 여부 batch 조회 (N+1 방지).
     * ExploreCoursesTab의 findFavoritedCourseIdsIn과 동일 패턴.
     */
    @Query("""
            SELECT w.id.placeId FROM PlaceWishEntity w
            WHERE w.id.userId = :userId
              AND w.id.placeId IN :placeIds
            """)
    List<UUID> findWishedPlaceIdsIn(
            @Param("userId") UUID userId,
            @Param("placeIds") List<UUID> placeIds);
}
```

### 4. PlaceWishService.java

`src/main/java/com/bikeridediary/domain/place/service/PlaceWishService.java`

```java
package com.bikeridediary.domain.place.service;

import com.bikeridediary.domain.place.entity.PlaceEntity;
import com.bikeridediary.domain.place.entity.PlaceWishEntity;
import com.bikeridediary.domain.place.entity.PlaceWishId;
import com.bikeridediary.domain.place.repository.PlaceRepository;
import com.bikeridediary.domain.place.repository.PlaceWishRepository;
import com.bikeridediary.global.exception.BusinessException;
import com.bikeridediary.global.exception.ErrorCode;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

@Service
@RequiredArgsConstructor
public class PlaceWishService {

    private final PlaceRepository placeRepository;
    private final PlaceWishRepository placeWishRepository;

    /**
     * 찜 추가. 이미 있으면 idempotent (아무 것도 안 함).
     * PlaceEntity.wishedCount 원자적 증가.
     */
    @Transactional
    public void addWish(UUID placeId, UUID userId) {
        PlaceEntity place = placeRepository.findById(placeId)
                .orElseThrow(() -> new BusinessException(ErrorCode.PLACE_NOT_FOUND));

        PlaceWishId id = new PlaceWishId(placeId, userId);
        if (placeWishRepository.existsById(id)) return;  // idempotent

        placeWishRepository.save(PlaceWishEntity.create(placeId, userId));
        place.increaseWish();  // PlaceEntity dirty checking 활용
    }

    /**
     * 찜 해제. 없어도 idempotent.
     */
    @Transactional
    public void removeWish(UUID placeId, UUID userId) {
        PlaceEntity place = placeRepository.findById(placeId)
                .orElseThrow(() -> new BusinessException(ErrorCode.PLACE_NOT_FOUND));

        PlaceWishId id = new PlaceWishId(placeId, userId);
        if (!placeWishRepository.existsById(id)) return;  // idempotent

        placeWishRepository.deleteById(id);
        place.decreaseWish();
    }
}
```

### 5. PlaceWishController.java

`src/main/java/com/bikeridediary/domain/place/controller/PlaceWishController.java`

```java
package com.bikeridediary.domain.place.controller;

import com.bikeridediary.domain.place.service.PlaceWishService;
import com.bikeridediary.global.auth.CustomUserDetails;
import com.bikeridediary.global.response.ApiResponse;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@Tag(name = "장소 찜", description = "장소 찜 추가/해제")
@RestController
@RequestMapping("/api/v1/places/{placeId}/wish")
@RequiredArgsConstructor
public class PlaceWishController {

    private final PlaceWishService placeWishService;

    @Operation(summary = "장소 찜 추가 (idempotent)")
    @PostMapping
    public ResponseEntity<ApiResponse<Void>> addWish(
            @PathVariable UUID placeId,
            @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        placeWishService.addWish(placeId, userDetails.getUserId());
        return ResponseEntity.ok(ApiResponse.ok(null));
    }

    @Operation(summary = "장소 찜 해제 (idempotent)")
    @DeleteMapping
    public ResponseEntity<ApiResponse<Void>> removeWish(
            @PathVariable UUID placeId,
            @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        placeWishService.removeWish(placeId, userDetails.getUserId());
        return ResponseEntity.ok(ApiResponse.ok(null));
    }
}
```

### 6. PlaceResponse에 `isWished` 필드 추가

`src/main/java/com/bikeridediary/domain/place/dto/PlaceResponse.java`

```java
public record PlaceResponse(
        // ... 기존 필드들 ...
        boolean isWished    // 현재 로그인 유저의 찜 여부. 미인증 요청이면 false.
) {
    public static PlaceResponse from(PlaceEntity entity, boolean isWished) {
        return new PlaceResponse(
                // ... 기존 필드 매핑 ...
                isWished
        );
    }
}
```

### 7. PlaceService.list() 리팩터 (wish batch 조회)

`PlaceService.list()`를 다음처럼 변경:

```java
@Transactional(readOnly = true)
public List<PlaceResponse> list(String categoryCode, UUID currentUserId) {
    List<PlaceEntity> places = categoryCode == null
            ? placeRepository.findByDeletedAtIsNullOrderByPlaceCategoryEntity_DisplayOrderAsc()
            : placeRepository.findByPlaceCategoryEntity_CategoryCodeAndDeletedAtIsNull(categoryCode);

    // 로그인 유저의 wish set batch 조회 (N+1 방지). 미인증이면 빈 Set.
    Set<UUID> wishedIds = (currentUserId == null || places.isEmpty())
            ? Set.of()
            : Set.copyOf(placeWishRepository.findWishedPlaceIdsIn(
                    currentUserId,
                    places.stream().map(PlaceEntity::getId).toList()));

    return places.stream()
            .map(p -> PlaceResponse.from(p, wishedIds.contains(p.getId())))
            .toList();
}
```

컨트롤러에서 `@AuthenticationPrincipal(errorOnInvalidType = false) CustomUserDetails userDetails`로 받아 `userDetails != null ? userDetails.getUserId() : null` 전달.

**주의**: `/api/v1/places` GET은 현재 `permitAll`(게스트 접근 허용) 상태. `SecurityContextHolder`에서 `principal`이 익명(`String "anonymousUser"`)일 수 있음. `instanceof CustomUserDetails` 체크로 안전 추출.

### 8. ErrorCode (필요 시)

이미 `PLACE_NOT_FOUND`가 있으면 재사용. 없다면 추가.

## SecurityConfig

`/api/v1/places/{placeId}/wish`는 인증 필수. `@AuthenticationPrincipal`이 null이면 NPE 나므로 아래 확인:
- `SecurityConfig`의 `GET_PERMIT_ALL_ENDPOINTS`에는 `/api/v1/places/**`가 이미 있으나, POST/DELETE는 `anyRequest().authenticated()`로 걸림 → 자동으로 인증 필수
- 별도 SecurityConfig 수정 불필요

## 확인 방법

```bash
# 1. 유저 로그인 후 찜 추가
TOKEN=$(curl -s -X POST http://localhost:8081/api/v1/auth/login/email \
  -H "Content-Type: application/json" \
  -d '{"email":"user@test.com","password":"..."}' | jq -r '.data.accessToken')

curl -X POST -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8081/api/v1/places/<PLACE_UUID>/wish"

# 2. 찜 상태 확인 (list 응답에 isWished:true 나와야 함)
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8081/api/v1/places?category=CAFE" | jq '.data[] | {id, isWished}'

# 3. 찜 해제
curl -X DELETE -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8081/api/v1/places/<PLACE_UUID>/wish"

# 4. 인증 없이 → 401
curl -X POST "http://localhost:8081/api/v1/places/<PLACE_UUID>/wish"

# 5. DB 확인
psql -c "SELECT * FROM place_wishes ORDER BY created_at DESC LIMIT 10;"
psql -c "SELECT id, place_name, wished_count FROM places WHERE wished_count > 0;"
```

## 체크리스트

- [ ] `PlaceWishEntity` `@EmbeddedId PlaceWishId` (CourseFavoriteEntity 패턴)
- [ ] `PlaceWishId`에 `@JdbcTypeCode(SqlTypes.UUID)` 명시 (Hibernate 6.x UUID 이슈)
- [ ] `PlaceWishRepository.findWishedPlaceIdsIn` batch 조회 (N+1 방지)
- [ ] `PlaceService.list()`에 `currentUserId` 파라미터 추가 + wish batch 조회
- [ ] `PlaceResponse`에 `isWished` 필드 추가
- [ ] `PlaceWishService.addWish/removeWish`는 idempotent (이미 있음/없음 조용히 통과)
- [ ] `PlaceEntity.increaseWish/decreaseWish` 활용 (dirty checking, save 호출 없음)
- [ ] Controller `@AuthenticationPrincipal CustomUserDetails userDetails`
- [ ] `/api/v1/places/{id}/wish` POST/DELETE는 SecurityConfig 별도 수정 불필요 (자동 인증 필수)
- [ ] Swagger에서 두 엔드포인트 확인

## 확장 아이디어 (별도 스코프)

- `GET /api/v1/places/my/wishes` — 내 찜 목록 (MY 탭 확장 시)
- `wished_count` 정합성 배치 재계산 (트래픽 붙은 뒤 대응)
