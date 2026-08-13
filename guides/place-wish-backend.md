# Place 찜(Wish) API 백엔드 가이드

작성일: 2026-08-12 (2026-08-13 팩트 체크 재작성)
담당: backend-dev (사용자 직접 구현)
근거: 앱(brd_app)에는 이미 `PlaceRepository.addWish/removeWish` + `PlaceResponse.isWished` + 지도 필터 `wishActive`가 구현되어 있으나, 백엔드는 스켈레톤만 있고 실제 로직이 없어 동작하지 않음.

---

## 1. 현재 상태 팩트 체크 (2026-08-13)

### DB / 엔티티 / 관례
- **schema.sql**: `place_wishes` 완비 (222~229번 줄, PK `(place_id, user_id)`, CASCADE, `no BIGINT DEFAULT nextval`, `idx_place_wishes_user_id` 인덱스).
- **`PlaceEntity`**: `wishedCount int` 필드 + **`incrementWishedCount()` / `decrementWishedCount()`** 인스턴스 메서드 존재. (이름 주의 — increaseWish 아님)
- **동일 패턴 레퍼런스**: `CourseFavoriteEntity` + `CourseFavoriteId` (`@EmbeddedId`, `@JdbcTypeCode(SqlTypes.UUID)`). PlaceWish도 같은 방식으로 통일.

### 백엔드 파일별 상태

| 파일 | 상태 | 조치 |
|------|------|------|
| `PlaceWishEntity.java` | **컴파일 안 됨** — `@IdClass`/`@EmbeddedId` 없이 `@Id` 두 개 + `e.id = new PlaceWishId(...)`는 존재하지 않는 필드에 대입 | 재작성 (스니펫 §3.2) |
| `PlaceWishId.java` | @Column 매핑까지 OK | 그대로 사용 가능 (부분 정리는 §3.1) |
| `PlaceWishRepository.java` | 껍데기 — `findByUserId` 쿼리에 `WHERE` 절 없음, 필요한 exists/delete 메서드 없음 | 재작성 (스니펫 §3.3) |
| `PlaceWishService.java` | **완전 미구현 + 자기 자신 주입 오류** (`private final PlaceWishService placeWishService;`) → 스프링 순환참조로 앱 부팅 실패 | 재작성 (스니펫 §3.4) |
| `PlaceController.java` | wish 3개 엔드포인트 이미 있음. **다만 `@AuthenticationPrincipal UUID userId`** 사용 — 프로젝트 관례는 `CustomUserDetails userDetails` + `userDetails.getUserId()` | 시그니처 수정 (스니펫 §3.5) |
| `PlaceResponse.java` | `isWished` 필드 **없음** — 앱은 이미 `isWished`를 파싱하지만 항상 `false`로 뜸 | 필드 + 팩토리 오버로드 추가 (스니펫 §3.6) |
| `PlaceService.list()` | `wish` 정보 채우지 않음. 단, **컨트롤러에 `GET /places` 매핑 자체가 없음** (앱은 호출하지만 404 상태로 보임) — wish 붙이는 김에 컨트롤러 매핑도 같이 살릴지 검토 | §3.7 |
| `SecurityConfig` | `/api/v1/places/**` GET이 `GET_PERMIT_ALL_ENDPOINTS`에 있어 **`GET /api/v1/places/wishes/me`가 인증 없이 통과됨** → `@AuthenticationPrincipal`이 null → NPE | 매칭 조정 (§4) |
| `ErrorCode` | `PLACE_NOT_FOUND` 이미 존재. 추가 필요 없음 (add/remove는 idempotent라 wish 자체 에러 코드 불필요) | 그대로 |

### 앱 (brd_app) 관련 상태
- `PlaceRepository.addWish(id) / removeWish(id)`: 구현 완료 (`POST/DELETE /places/{id}/wish`).
- `PlaceResponse.isWished`: 필드 + fromJson 파싱 완료. 백엔드가 안 내려주면 default `false`.
- `PlaceFilterState.wishActive`: 필터 옵션 존재.
- **미구현**: 지도 상세 시트의 하트 토글 버튼, "내 찜 목록" 화면. 이번 가이드 스코프 아님 (백엔드 완성 후 앱 별도 사이클).

---

## 2. API 계약

| 메서드 | 경로 | 인증 | 응답 | 비고 |
|-------|------|------|------|------|
| POST | `/api/v1/places/{placeId}/wish` | 필수 | `ApiResponse<Void>` | idempotent (이미 있어도 200 OK, wishedCount 재증가 안 함) |
| DELETE | `/api/v1/places/{placeId}/wish` | 필수 | `ApiResponse<Void>` | idempotent (없어도 200 OK, wishedCount 재감소 안 함) |
| GET | `/api/v1/places/wishes/me` | 필수 | `ApiResponse<List<PlaceResponse>>` | 내 찜 목록. 카테고리 정렬(displayOrder) → 최근순 |

**목록/상세 조회의 `isWished`**: 로그인 유저면 batch로 채워서 응답. 미인증(익명)이면 항상 `false`.

---

## 3. 파일별 스니펫

### 3.1 PlaceWishId (기존 파일 정리)

`src/main/java/com/bikeridediary/domain/place/entity/PlaceWishId.java`

기존 파일 사용 가능. `@Embeddable` 어노테이션만 추가 (누락되어 있음). `AllArgsConstructor`는 명시적 생성자로 대체해도 무방.

```java
package com.bikeridediary.domain.place.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Embeddable;
import lombok.AllArgsConstructor;
import lombok.EqualsAndHashCode;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

import java.io.Serializable;
import java.util.UUID;

// 장소 찜 복합 PK — (place_id, user_id). CourseFavoriteId 동일 패턴.
@Embeddable
@Getter
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode
public class PlaceWishId implements Serializable {

    @Column(name = "place_id")
    @JdbcTypeCode(SqlTypes.UUID)
    private UUID placeId;

    @Column(name = "user_id")
    @JdbcTypeCode(SqlTypes.UUID)
    private UUID userId;
}
```

### 3.2 PlaceWishEntity (재작성)

`src/main/java/com/bikeridediary/domain/place/entity/PlaceWishEntity.java`

**전체 교체**. 기존 파일은 `@IdClass`/`@EmbeddedId` 없이 `@Id` 두 개 + 존재하지 않는 `id` 필드 대입으로 컴파일 오류.

```java
package com.bikeridediary.domain.place.entity;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.Generated;
import org.hibernate.generator.EventType;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;
import java.util.UUID;

// 장소 찜 매핑 (place_wishes 테이블). CourseFavoriteEntity 동일 패턴.
@Entity
@Table(name = "place_wishes")
@EntityListeners(AuditingEntityListener.class)
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class PlaceWishEntity {

    // 조회용 친숙 번호 (자동 증가, DB DEFAULT nextval)
    @Column(name = "no", insertable = false, updatable = false)
    @Generated(event = EventType.INSERT)
    private Long no;

    @EmbeddedId
    private PlaceWishId id;

    // 찜한 장소 참조 (조회 전용).
    // INSERT/UPDATE는 @EmbeddedId의 placeId 컬럼이 담당하고,
    // 이 필드는 같은 place_id 컬럼을 read-only로 매핑해 JOIN FETCH 대상으로만 사용.
    // ⚠️ @MapsId 사용 금지 — placeEntity 없이 저장 시 "assign id from null" 오류 발생.
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "place_id", insertable = false, updatable = false)
    private PlaceEntity placeEntity;

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    public static PlaceWishEntity create(UUID placeId, UUID userId) {
        PlaceWishEntity e = new PlaceWishEntity();
        e.id = new PlaceWishId(placeId, userId);
        return e;
    }
}
```

**주의**:
- **`@MapsId` 붙이면 안 됨**. @MapsId는 "placeId를 placeEntity에서 파생하라"고 지시하기 때문에 팩토리에서 `placeEntity` 없이 `id`만 세팅해 저장하면 `attempted to assign id from null one-to-one property` 오류.
- `insertable=false, updatable=false`가 핵심. place_id 컬럼은 `@EmbeddedId.placeId`가 관리하고, `placeEntity`는 조회 전용. 같은 컬럼 두 번 매핑처럼 보이지만 Hibernate가 read-only 힌트로 인식해 안전.
- `userEntity` ManyToOne은 필요 없음 (내 찜 목록에서 place 정보만 필요).

### 3.3 PlaceWishRepository (재작성)

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

    // 내 찜 목록 — PlaceWishEntity를 반환. placeEntity(+ category)는 JOIN FETCH로
    // 미리 로드해 서비스에서 w.getPlaceEntity() 호출 시 N+1 없이 접근 가능.
    // 정렬: 찜 시각 DESC.
    @Query("""
            SELECT w FROM PlaceWishEntity w
            JOIN FETCH w.placeEntity p
            JOIN FETCH p.placeCategoryEntity c
            WHERE w.id.userId = :userId
              AND p.deletedAt IS NULL
            ORDER BY w.createdAt DESC
            """)
    List<PlaceWishEntity> findByIdUserIdWithPlace(@Param("userId") UUID userId);

    // 유저의 찜한 place_id 전량 조회 (Set으로 바로 감싸 isWished 매핑에 사용).
    // 지도(찾아보기)는 페이징 없이 places 전체 로드 + in-memory 필터 UX라
    // IN절로 좁힐 실익 없음 (유저별 wish 수는 많아야 수십~수백).
    // SELECT w.id.placeId 프로젝션으로 wish 엔티티 전체를 로드하지 않음.
    @Query("SELECT w.id.placeId FROM PlaceWishEntity w WHERE w.id.userId = :userId")
    List<UUID> findPlaceIdsByUserId(@Param("userId") UUID userId);
}
```

**참고**: `existsById(PlaceWishId)`, `deleteById(PlaceWishId)`, `save(PlaceWishEntity)`는 `JpaRepository` 기본 제공 — 별도 선언 불필요.

### 3.4 PlaceWishService (재작성)

`src/main/java/com/bikeridediary/domain/place/service/PlaceWishService.java`

**전체 교체**. 기존 파일은 자기 자신을 주입해서 스프링 순환참조 오류.

```java
package com.bikeridediary.domain.place.service;

import com.bikeridediary.domain.place.dto.PlaceResponse;
import com.bikeridediary.domain.place.entity.PlaceEntity;
import com.bikeridediary.domain.place.entity.PlaceWishEntity;
import com.bikeridediary.domain.place.entity.PlaceWishId;
import com.bikeridediary.domain.place.repository.PlaceRepository;
import com.bikeridediary.domain.place.repository.PlaceWishRepository;
import com.bikeridediary.global.exception.BusinessException;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.UUID;

import static com.bikeridediary.global.exception.ErrorCode.PLACE_NOT_FOUND;

@Service
@RequiredArgsConstructor
public class PlaceWishService {

    private final PlaceRepository placeRepository;
    private final PlaceWishRepository placeWishRepository;

    // 찜 추가 — 이미 있으면 idempotent (증가 없음). 갱신된 PlaceResponse 반환.
    // 앱은 이 응답으로 로컬 places 캐시의 해당 place를 교체 (isWished, wishedCount 반영).
    @Transactional
    public PlaceResponse addWish(UUID placeId, UUID userId) {
        PlaceEntity place = placeRepository.findByIdAndDeletedAtIsNull(placeId)
                .orElseThrow(() -> new BusinessException(PLACE_NOT_FOUND));

        PlaceWishId id = new PlaceWishId(placeId, userId);
        if (!placeWishRepository.existsById(id)) {
            placeWishRepository.save(PlaceWishEntity.create(placeId, userId));
            place.incrementWishedCount(); // dirty checking으로 UPDATE
        }
        return PlaceResponse.from(place, true);
    }

    // 찜 해제 — 없으면 idempotent (감소 없음). 갱신된 PlaceResponse 반환.
    @Transactional
    public PlaceResponse removeWish(UUID placeId, UUID userId) {
        PlaceEntity place = placeRepository.findByIdAndDeletedAtIsNull(placeId)
                .orElseThrow(() -> new BusinessException(PLACE_NOT_FOUND));

        PlaceWishId id = new PlaceWishId(placeId, userId);
        if (placeWishRepository.existsById(id)) {
            placeWishRepository.deleteById(id);
            place.decrementWishedCount();
        }
        return PlaceResponse.from(place, false);
    }

    // 내 찜 목록 — PlaceWishEntity에서 fetch join된 placeEntity 꺼내 매핑
    @Transactional(readOnly = true)
    public List<PlaceResponse> listMyWishes(UUID userId) {
        return placeWishRepository.findByIdUserIdWithPlace(userId).stream()
                .map(w -> PlaceResponse.from(w.getPlaceEntity(), true)) // 내 찜이니 isWished=true
                .toList();
    }
}
```

**의도적 선택**:
- `PlaceEntity`의 기존 `incrementWishedCount() / decrementWishedCount()` 인스턴스 메서드(dirty checking)를 그대로 활용. 원자적 `@Modifying UPDATE` 방식은 트래픽 붙은 뒤 전환 검토.
- soft-deleted place에 대한 wish 시도는 `PLACE_NOT_FOUND`로 통일 (isDeleted 별도 코드 만들지 않음).

### 3.5 PlaceController 시그니처 수정

`src/main/java/com/bikeridediary/domain/place/controller/PlaceController.java`

**기존 파일의 3개 엔드포인트 유지**, `@AuthenticationPrincipal` 타입만 `CustomUserDetails`로 변경. 프로젝트 표준 패턴(`AdminPlaceChangeRequestController`, `PlaceChangeRequestController` 동일).

```java
@Operation(summary = "장소 찜 추가 — 갱신된 PlaceResponse 반환 (idempotent)")
@PostMapping("/{placeId}/wish")
public ResponseEntity<ApiResponse<PlaceResponse>> addWish(
        @PathVariable UUID placeId,
        @AuthenticationPrincipal CustomUserDetails userDetails
) {
    return ResponseEntity.ok(ApiResponse.ok(
            placeWishService.addWish(placeId, userDetails.getUserId())));
}

@Operation(summary = "장소 찜 해제 — 갱신된 PlaceResponse 반환 (idempotent)")
@DeleteMapping("/{placeId}/wish")
public ResponseEntity<ApiResponse<PlaceResponse>> removeWish(
        @PathVariable UUID placeId,
        @AuthenticationPrincipal CustomUserDetails userDetails
) {
    return ResponseEntity.ok(ApiResponse.ok(
            placeWishService.removeWish(placeId, userDetails.getUserId())));
}

@Operation(summary = "내 찜 목록 조회")
@GetMapping("/wishes/me")
public ResponseEntity<ApiResponse<List<PlaceResponse>>> myWishes(
        @AuthenticationPrincipal CustomUserDetails userDetails
) {
    return ResponseEntity.ok(ApiResponse.ok(placeWishService.listMyWishes(userDetails.getUserId())));
}
```

### 3.6 PlaceResponse에 `isWished` 필드 추가

`src/main/java/com/bikeridediary/domain/place/dto/PlaceResponse.java`

기존 필드 유지 + 마지막에 `boolean isWished` 추가. 팩토리 오버로드로 기존 호출부(무맥락 조회) 호환.

```java
public record PlaceResponse(
        UUID id,
        UUID userId,
        String placeName,
        String category,
        BigDecimal latitude,
        BigDecimal longitude,
        String address,
        String roadAddress,
        String description,
        String photoUrl,
        String phone,
        String kakaoPlaceId,
        String naverPlaceId,
        boolean isWished    // 현재 로그인 유저의 찜 여부. 미인증/미조회 시 false.
) {
    // 로그인 컨텍스트 없는 조회용 (기본 false)
    public static PlaceResponse from(PlaceEntity entity) {
        return from(entity, false);
    }

    // 유저 컨텍스트 있는 조회용 (list batch + myWishes 등)
    public static PlaceResponse from(PlaceEntity entity, boolean isWished) {
        return new PlaceResponse(
                entity.getId(),
                entity.getUserEntity() == null ? null : entity.getUserEntity().getId(),
                entity.getPlaceName(),
                entity.getPlaceCategoryEntity().getCategoryCode(),
                entity.getLatitude(),
                entity.getLongitude(),
                entity.getAddress(),
                entity.getRoadAddress(),
                entity.getDescription(),
                entity.getPhotoUrl(),
                entity.getPhone(),
                entity.getKakaoPlaceId(),
                entity.getNaverPlaceId(),
                isWished
        );
    }
}
```

**주의**: 프로젝트에 `PlaceResponse.from(entity)` 호출부가 여럿 있음 (PlaceService.list, PlaceRankingResponse, admin 로직 등). 오버로드 방식이라 기존 호출부는 그대로 컴파일됨.

### 3.7 PlaceService.list()에 유저 컨텍스트 주입 (batch)

`PlaceService.list()`를 다음처럼 변경. 목록 조회 시 유저의 wish set을 한 번에 가져와 매핑 (N+1 방지).

```java
@Transactional(readOnly = true)
public List<PlaceResponse> list(String categoryCode, UUID currentUserId) {
    List<PlaceEntity> places = (categoryCode == null || categoryCode.isBlank())
            ? placeRepository.findByDeletedAtIsNullOrderByPlaceCategoryEntity_DisplayOrderAsc()
            : placeRepository.findByPlaceCategoryEntity_CategoryCodeAndDeletedAtIsNull(categoryCode);

    if (places.isEmpty()) return List.of();

    // 미인증이면 빈 Set, 아니면 유저의 찜한 place_id 전체를 Set으로
    java.util.Set<UUID> wishedIds = (currentUserId == null)
            ? java.util.Set.of()
            : java.util.Set.copyOf(placeWishRepository.findPlaceIdsByUserId(currentUserId));

    return places.stream()
            .map(p -> PlaceResponse.from(p, wishedIds.contains(p.getId())))
            .toList();
}
```

`PlaceService`에 `private final PlaceWishRepository placeWishRepository;` 필드 추가.

**컨트롤러에서 유저 전달** — `list` 매핑 자체가 현재 컨트롤러에 없다면 이번 참에 추가:

```java
@Operation(summary = "장소 목록 조회 (전체 또는 카테고리 필터)")
@GetMapping
public ResponseEntity<ApiResponse<List<PlaceResponse>>> list(
        @RequestParam(required = false) String category,
        @AuthenticationPrincipal CustomUserDetails userDetails  // 익명 시 null
) {
    UUID userId = userDetails != null ? userDetails.getUserId() : null;
    return ResponseEntity.ok(ApiResponse.ok(placeService.list(category, userId)));
}
```

**주의**:
- `GET /api/v1/places/**`는 SecurityConfig에서 `permitAll`이므로 익명 요청도 통과 → `@AuthenticationPrincipal`이 null일 수 있음. null 체크 필수.
- 매개변수 이름이 `category`(앱 계약, `PlaceRepository.list`에서 `category` param 사용)와 서비스의 `categoryCode`(내부 명)로 다름 — 컨트롤러에서 매핑만 잘하면 됨.

---

## 4. SecurityConfig 매칭 조정 (중요)

현재 `SecurityConfig`:

```java
private final String[] GET_PERMIT_ALL_ENDPOINTS = {
        "/api/v1/courses",
        "/api/v1/bike-models/**",
        "/api/v1/places/**",   // ← 이것 때문에 GET /places/wishes/me도 permitAll로 통과
};
```

`GET /api/v1/places/wishes/me`가 GET permitAll 매칭에 걸려 익명 요청도 통과 → `@AuthenticationPrincipal`이 null → NPE.

### 조치

`authorizeHttpRequests` 블록에서 **`GET_PERMIT_ALL_ENDPOINTS`보다 먼저** 명시적 매칭 추가:

```java
.authorizeHttpRequests(auth -> auth
        .requestMatchers(PERMIT_ALL_ENDPOINTS).permitAll()
        // 인증 필수 매칭을 GET permitAll보다 앞에 배치 (더 구체적인 매칭이 우선)
        .requestMatchers(HttpMethod.GET, "/api/v1/places/wishes/**").authenticated()
        .requestMatchers(HttpMethod.GET, GET_PERMIT_ALL_ENDPOINTS).permitAll()
        .requestMatchers(AUTHENTICATED_ENDPOINTS).authenticated()
        .anyRequest().authenticated()
)
```

**POST/DELETE `/places/{id}/wish`는 별도 매칭 불필요** — `anyRequest().authenticated()`로 자동 커버됨.

---

## 5. 구현 순서 권장

1. `PlaceWishId`에 `@Embeddable` 추가 (누락되어 있음)
2. `PlaceWishEntity` 전체 교체 (@EmbeddedId 방식)
3. `PlaceWishRepository` 전체 교체
4. `PlaceWishService` 전체 교체 (자기 주입 제거)
5. `PlaceResponse`에 `isWished` 필드 + 팩토리 오버로드
6. `PlaceService.list(String, UUID)` 시그니처 변경 + `placeWishRepository` 주입 + batch 매핑
7. `PlaceController` 3개 wish 엔드포인트 `@AuthenticationPrincipal CustomUserDetails userDetails`로 통일 + (필요 시) `GET /places` 매핑 추가
8. `SecurityConfig`에 `GET /api/v1/places/wishes/**` authenticated 매칭 (GET permitAll보다 앞)
9. 컴파일 → 부팅 확인 (기존 스켈레톤 오류로 안 올라가던 상태에서 정상 부팅)
10. Swagger에서 3개 wish 엔드포인트 확인

---

## 6. 확인 방법 (curl)

```bash
BASE=http://localhost:8081

# 이메일 로그인 → 토큰
TOKEN=$(curl -s -X POST $BASE/api/v1/auth/login/email \
  -H "Content-Type: application/json" \
  -d '{"email":"user@test.com","password":"..."}' | jq -r '.data.accessToken')

# 유효한 place ID 하나 확보 (DB에서)
PID=$(psql -tA -c "SELECT id FROM places WHERE deleted_at IS NULL LIMIT 1;")

# 1) 찜 추가 → 200 OK
curl -X POST -H "Authorization: Bearer $TOKEN" "$BASE/api/v1/places/$PID/wish"

# 2) 다시 추가 (idempotent) → 200 OK, wished_count 재증가 없음
curl -X POST -H "Authorization: Bearer $TOKEN" "$BASE/api/v1/places/$PID/wish"

# 3) list 응답에 isWished:true 나오는지
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/v1/places" \
  | jq '.data[] | select(.id=="'$PID'") | {id, placeName, isWished}'

# 4) 익명 요청 → isWished:false (인증 없어도 통과)
curl -s "$BASE/api/v1/places" | jq '.data[0] | {id, isWished}'

# 5) 내 찜 목록
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/v1/places/wishes/me" \
  | jq '.data | map({id, placeName, isWished})'

# 6) 익명이 내 찜 목록 접근 → 401 (SecurityConfig 매칭 확인)
curl -s -o /dev/null -w "%{http_code}\n" "$BASE/api/v1/places/wishes/me"
# 401 이어야 정상. 200 나오면 4번 조치 누락

# 7) 찜 해제 → 200 OK
curl -X DELETE -H "Authorization: Bearer $TOKEN" "$BASE/api/v1/places/$PID/wish"

# 8) 다시 해제 (idempotent) → 200 OK
curl -X DELETE -H "Authorization: Bearer $TOKEN" "$BASE/api/v1/places/$PID/wish"

# 9) DB 확인
psql -c "SELECT place_id, user_id, created_at FROM place_wishes ORDER BY created_at DESC LIMIT 5;"
psql -c "SELECT id, place_name, wished_count FROM places WHERE wished_count > 0 ORDER BY wished_count DESC;"
```

---

## 7. 체크리스트

- [ ] `PlaceWishId`에 `@Embeddable` 어노테이션 (누락 상태)
- [ ] `PlaceWishEntity` 전체 재작성 — `@EmbeddedId` + read-only `@ManyToOne PlaceEntity`(`insertable=false, updatable=false`) + `AuditingEntityListener` (⚠️ `@MapsId` 금지)
- [ ] `PlaceWishRepository.findByIdUserIdWithPlace` — `List<PlaceWishEntity>` 반환, placeEntity + category JOIN FETCH, deleted_at 필터, 찜 시각 DESC
- [ ] `PlaceWishRepository.findPlaceIdsByUserId` — `@Query` + `SELECT w.id.placeId` 프로젝션, `Set.copyOf`로 감싸 isWished 매핑
- [ ] `PlaceWishService` 자기 주입 제거, `addWish/removeWish` idempotent, `listMyWishes` 추가
- [ ] `PlaceEntity.incrementWishedCount/decrementWishedCount` 활용 (dirty checking, save 호출 X)
- [ ] `PlaceController` 3개 엔드포인트 `@AuthenticationPrincipal CustomUserDetails userDetails`로 통일
- [ ] `PlaceResponse`에 `isWished` 필드 + `from(entity)` / `from(entity, isWished)` 두 팩토리
- [ ] `PlaceService.list(String, UUID)` batch wish 매핑
- [ ] `SecurityConfig`에 `GET /api/v1/places/wishes/**` authenticated 매칭 (GET permitAll보다 먼저)
- [ ] 컴파일/부팅 → Swagger 확인
- [ ] curl §6 시나리오 9개 통과

---

## 8. 별도 스코프 (이번 사이클 아님)

- **앱 UI 후속**: 지도 상세 시트에 하트 토글 버튼 + 낙관적 업데이트/롤백, "내 찜 목록" 탭 추가 (라이딩 코스 즐겨찾기 패턴 재사용). 백엔드 완성 후 앱 사이클에서.
- **`wished_count` 정합성 배치 재계산**: 트래픽 커지면 (1) `UPDATE places SET wished_count = (SELECT COUNT(*) FROM place_wishes WHERE place_id = places.id)` 주기 배치, 또는 (2) DB 트리거 전환.
- **원자적 카운터**: 현재 dirty checking(`instance.incrementWishedCount()`)이라 이론상 race 가능. 필요 시 `@Modifying @Query("UPDATE PlaceEntity p SET p.wishedCount = p.wishedCount + 1 WHERE p.id = :id")` 방식으로 이관.
- **자체 리뷰 시스템**: `place_reviews` 테이블 신설 + `starPoint` 평균 캐시 전환. 결정 유보 상태.
