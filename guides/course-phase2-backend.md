# 라이딩 코스 생성/편집 백엔드 구현 가이드

작성일: 2026-08-04 (최신 갱신: NaverMapsClient 통합 반영, VIA 상한 정정, END→GOAL 리네임)
스코프: 라이딩 코스 2차 — `POST /courses`, `PATCH /courses/{id}`, `POST /courses/preview`

정책 요약:
- **D5=B**: preview는 사용자 "경로 미리보기" 버튼 클릭 시에만 호출
- **옵션 B**: create/update는 앱이 preview로 받은 path/distance를 그대로 전송, 서버는 Naver 재호출 없이 저장

---

## 실제 구현 상태 (사용자 커스텀 반영)

- `infra/naver/maps/NaverMapsClient` — search + directions 통합 클라이언트
- `infra/naver/maps/dto/NaverDirectionsResponse` — `route.traavoidcaronly` 필드 (option 값과 동일한 이름)
- `NaverMapsClient.Waypoint` record — VIA 좌표 wrapper

---

## ⚠️ END → GOAL 리네임 (스키마/엔티티/앱 전체 영향)

Naver Directions API가 `goal` 파라미터명을 쓰므로 waypoint role도 **GOAL**로 통일. 서비스 도메인 용어를 API와 맞춤.

### 마이그레이션 절차

**1. `schema.sql` 변경**

기존 CHECK 제약 재정의 + 데이터 마이그레이션:

```sql
-- 기존 role 값 마이그레이션
UPDATE course_waypoints SET role = 'GOAL' WHERE role = 'END';

-- 기존 CHECK 제약 삭제 후 재생성
ALTER TABLE course_waypoints DROP CONSTRAINT IF EXISTS course_waypoints_role_check;
ALTER TABLE course_waypoints ADD CONSTRAINT course_waypoints_role_check
    CHECK (role IN ('START','VIA','GOAL'));
```

`schema.sql`의 신규 `CREATE TABLE` 정의에서도 `CHECK IN ('START','VIA','END')` → `('START','VIA','GOAL')` 로 변경.

**2. `CourseWaypointEntity` 변경**

role 상수/enum이 있으면 END → GOAL. 팩토리 파라미터/필드/JavaDoc 모두 반영.

**3. 앱 코드 영향 (별도 사이클)**

- `features/riding_course/data/model/course_waypoint.dart` role 상수
- `presentation/riding_course_detail_screen.dart` 마커 아이콘/색상 분기
- 서버 응답 파싱 부분

이 가이드에서는 백엔드만 기준. 앱은 게이트 3(flutter-dev)에서 반영.

---

## 파일 생성/수정 목록

| 순서 | 경로 | 신규/수정 | 설명 |
|-----|-----|----------|-----|
| 1 | `schema.sql` | 수정 | `description TEXT` 추가 + role CHECK 재정의 (END→GOAL) |
| 2 | `domain/course/entity/CourseEntity.java` | 수정 | `description` 필드 + `update()` / `updatePath()` |
| 3 | `domain/course/entity/CourseWaypointEntity.java` | 수정 | role 상수 END→GOAL |
| 4 | `domain/course/dto/WaypointRequest.java` | 신규 | waypoint 입력 DTO |
| 5 | `domain/course/dto/CourseCreateRequest.java` | 신규 | 코스 생성 요청 DTO |
| 6 | `domain/course/dto/CourseUpdateRequest.java` | 신규 | 코스 편집 요청 DTO |
| 7 | `domain/course/dto/CoursePreviewRequest.java` | 신규 | 경로 미리보기 요청 DTO |
| 8 | `domain/course/dto/CoursePreviewResponse.java` | 신규 | 경로 미리보기 응답 DTO |
| 9 | `domain/course/repository/CourseWaypointRepository.java` | 수정 | `deleteByCourseEntityId` 추가 |
| 10 | `domain/course/service/CourseService.java` | 수정 | create/update/preview + 헬퍼 |
| 11 | `domain/course/controller/CourseController.java` | 수정 | 3개 엔드포인트 추가 |
| 12 | `global/exception/ErrorCode.java` | 수정 | 3개 추가 (Directions 관련) |

---

## 1. WaypointRequest DTO

파일: `src/main/java/com/bikeridediary/domain/course/dto/WaypointRequest.java`

```java
package com.bikeridediary.domain.course.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

import java.math.BigDecimal;
import java.util.UUID;

/**
 * waypoint 하나의 입력 DTO.
 * role: "START" | "VIA" | "GOAL" — DB CHECK 제약, Naver Directions API의 goal 파라미터와 통일.
 * placeId: 등록된 place에서 선택한 경우 UUID, 임의 지점은 null.
 */
public record WaypointRequest(
        // 역할 (START/VIA/GOAL)
        @NotBlank String role,
        // 순서 인덱스 (0-based, 앱이 재부여한 연속값)
        short seq,
        // 위도 (소수점 7자리 이내)
        @NotNull BigDecimal latitude,
        // 경도 (소수점 7자리 이내)
        @NotNull BigDecimal longitude,
        // 지점 이름 (place 선택 시 place.placeName, 임의 지점은 사용자 입력 또는 주소)
        String placeName,
        // 등록된 place ID (임의 지점은 null)
        UUID placeId,
        // place 카테고리 코드 (앱 마커 아이콘용, null 가능)
        String placeCategoryCode
) {}
```

---

## 2. CourseCreateRequest DTO

파일: `src/main/java/com/bikeridediary/domain/course/dto/CourseCreateRequest.java`

```java
package com.bikeridediary.domain.course.dto;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

import java.util.List;
import java.util.UUID;

public record CourseCreateRequest(
        // 코스 이름
        @NotBlank @Size(max = 200) String name,
        // 설명 (선택)
        String description,
        // 공개 여부
        boolean isPublic,
        // waypoints (START 1개 + GOAL 1개 + VIA 0~15개, seq는 앱이 재부여한 0-based 연속값)
        @NotEmpty @Valid List<WaypointRequest> waypoints,
        // 복사 편집 시 원본 코스 ID (신규 생성은 null)
        UUID sourceCourseId,
        // 옵션 B: preview에서 받은 경로 JSON 문자열 [[lng,lat],...] — 앱이 로컬 보관 후 저장 시 재전송
        @NotBlank String path,
        // 옵션 B: preview에서 받은 총 거리 (미터)
        @NotNull Integer distanceMeters
) {}
```

---

## 3. CourseUpdateRequest DTO

파일: `src/main/java/com/bikeridediary/domain/course/dto/CourseUpdateRequest.java`

```java
package com.bikeridediary.domain.course.dto;

import jakarta.validation.Valid;
import jakarta.validation.constraints.Size;

import java.util.List;

public record CourseUpdateRequest(
        // 코스 이름 (null이면 변경 없음)
        @Size(max = 200) String name,
        // 설명 (null이면 변경 없음)
        String description,
        // 공개 여부 (null이면 변경 없음)
        Boolean isPublic,
        // 변경된 waypoints (null이면 변경 없음, regeneratePath=true와 함께 와야 유효)
        @Valid List<WaypointRequest> waypoints,
        // waypoints 변경 시 path/distance도 함께 갱신할지 여부 (앱이 preview 재실행한 경우 true)
        boolean regeneratePath,
        // 옵션 B: regeneratePath=true 시 앱이 preview로 받은 신규 path (false면 무시)
        String path,
        // 옵션 B: regeneratePath=true 시 앱이 preview로 받은 신규 총 거리 (false면 무시)
        Integer distanceMeters
) {}
```

---

## 4. CoursePreviewRequest / Response DTO

파일: `src/main/java/com/bikeridediary/domain/course/dto/CoursePreviewRequest.java`

```java
package com.bikeridediary.domain.course.dto;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotEmpty;

import java.util.List;

public record CoursePreviewRequest(
        // waypoints (START + GOAL 최소 2개, seq는 앱이 재부여한 0-based 연속값)
        @NotEmpty @Valid List<WaypointRequest> waypoints
) {}
```

파일: `src/main/java/com/bikeridediary/domain/course/dto/CoursePreviewResponse.java`

```java
package com.bikeridediary.domain.course.dto;

import java.util.List;

public record CoursePreviewResponse(
        // Naver Directions 응답의 path 좌표 JSON 문자열 [[lng,lat],...]
        String path,
        // 총 거리 (미터)
        int distanceMeters,
        // 전체 경로 경계 영역 [[minLng,minLat],[maxLng,maxLat]] — 앱 지도 fitBounds용
        List<List<Double>> bbox
) {}
```

---

## 5. CourseEntity 변경

파일: `src/main/java/com/bikeridediary/domain/course/entity/CourseEntity.java`

**`description` 필드 신규**:

```java
// 코스 설명 (선택, TEXT 컬럼)
@Column(name = "description", columnDefinition = "TEXT")
private String description;
```

**`update()` / `updatePath()` 메서드 추가** (기존 `isOwner()` 아래):

```java
// 코스 기본 정보 업데이트 (JPA dirty checking — save() 호출 불필요)
public void update(String name, String description, boolean isPublic) {
    if (name != null) this.name = name;
    // description: null이면 변경 없음, 빈 문자열이면 null로 저장(제거)
    if (description != null) {
        this.description = description.isBlank() ? null : description;
    }
    this.isPublic = isPublic;
}

// 옵션 B: 앱이 preview 재실행한 결과로 path/distance 업데이트 (dirty checking)
public void updatePath(String path, Integer distanceMeters) {
    this.path = path;
    this.distanceMeters = distanceMeters;
}
```

**`createWithId()` 팩토리에 `description` 파라미터 추가**:

```java
public static CourseEntity createWithId(
    UUID id,
    UserEntity userEntity,
    String name,
    String description,        // 신규 파라미터
    Integer distanceMeters,
    String path,
    boolean isPublic,
    UUID sourceCourseId
) {
    CourseEntity e = new CourseEntity();
    e.id = id;
    e.userEntity = userEntity;
    e.name = name;
    e.description = description;
    e.distanceMeters = distanceMeters;
    e.path = path;
    e.isPublic = isPublic;
    e.sourceCourseId = sourceCourseId;
    return e;
}
```

**스키마 반영** (`schema.sql`):

```sql
ALTER TABLE courses ADD COLUMN IF NOT EXISTS description TEXT;
```

---

## 6. CourseService — 신규 메서드 3개

파일: `src/main/java/com/bikeridediary/domain/course/service/CourseService.java`

### 트랜잭션 컨벤션 (프로젝트 전체 공통, 2026-08-05 확정)

**클래스 레벨 `@Transactional` 사용 금지 — 메서드마다 명시**.

```java
@Service
@RequiredArgsConstructor
public class CourseService {  // 클래스 레벨 어노테이션 없음
    
    // 조회 메서드: @Transactional(readOnly = true) 명시
    @Transactional(readOnly = true)
    public CourseDetailResponse getDetail(...) { }
    
    // 쓰기 메서드: @Transactional 명시
    @Transactional
    public CourseDetailResponse createCourse(...) { }
    
    @Transactional
    public void deleteCourse(...) { }
    
    // DB 접근 없는 메서드(외부 API 호출뿐): 어노테이션 생략
    public CoursePreviewResponse previewCourse(...) { }
}
```

이유:
- 각 메서드의 트랜잭션 성격(조회/쓰기/없음)이 어노테이션으로 즉시 보임
- 클래스 레벨 상속이 없으니 `Propagation.NEVER` 같은 트릭 불필요
- 새 메서드 추가 시 트랜잭션 타입을 의식적으로 결정해야 함 (누락 시 트랜잭션 없이 실행 → 즉시 발견)

### 6-0. imports / 의존성

```java
import com.bikeridediary.domain.course.dto.*;
import com.bikeridediary.domain.place.entity.PlaceEntity;
import com.bikeridediary.domain.place.repository.PlaceRepository;
import com.bikeridediary.infra.naver.maps.NaverMapsClient;
import com.bikeridediary.infra.naver.maps.NaverMapsClient.Waypoint;
import com.bikeridediary.infra.naver.maps.dto.NaverDirectionsResponse;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Comparator;
```

필드 추가 (**모두 `private final`** — `@RequiredArgsConstructor`가 생성자 자동 생성):

```java
private final PlaceRepository placeRepository;
private final NaverMapsClient naverMapsClient;      // preview 전용 (create/update에서 미사용)
private final UserRepository userRepository;
private final ObjectMapper objectMapper;
```

⚠️ **`private final` 빠뜨리면 `@RequiredArgsConstructor`가 안 잡아 주입 안 됨 → 런타임 NPE**. 필드 선언 시 반드시 `final` 포함.

### 6-1. createCourse() — 옵션 B, Naver 호출 없음

```java
@Transactional
public CourseDetailResponse createCourse(UUID userId, CourseCreateRequest request) {
    validateWaypoints(request.waypoints());
    sanityCheckPath(request.path(), request.distanceMeters(), request.waypoints());

    UserEntity user = userRepository.findByIdAndDeletedAtIsNull(userId)
            .orElseThrow(() -> new BusinessException(USER_NOT_FOUND));

    // ID 수동 세팅 → save() 반환값 사용 필수 (merge 경로에서 @PrePersist가 원본이 아닌 반환 엔티티에만 적용됨)
    CourseEntity toSave = CourseEntity.createWithId(
            UUID.randomUUID(),
            user,
            request.name(),
            request.description(),
            request.distanceMeters(),   // 앱 preview 결과 그대로
            request.path(),             // 앱 preview 결과 그대로
            request.isPublic(),
            request.sourceCourseId()
    );
    CourseEntity saved = courseRepository.save(toSave);

    saveWaypoints(saved, request.waypoints());

    return buildDetailResponse(saved.getId(), userId);
}
```

### 6-2. updateCourse() — 옵션 B, regeneratePath=true 시 앱이 재전송한 path 사용

```java
@Transactional
public CourseDetailResponse updateCourse(UUID userId, UUID courseId, CourseUpdateRequest request) {
    CourseEntity course = courseRepository.findByIdWithUser(courseId)
            .orElseThrow(() -> new BusinessException(COURSE_NOT_FOUND));

    if (!course.isOwner(userId)) {
        throw new BusinessException(COURSE_ACCESS_DENIED);
    }

    // 기본 정보 업데이트 (dirty checking)
    boolean isPublic = request.isPublic() != null ? request.isPublic() : course.isPublic();
    course.update(request.name(), request.description(), isPublic);

    if (request.regeneratePath() && request.waypoints() != null && !request.waypoints().isEmpty()) {
        validateWaypoints(request.waypoints());
        if (request.path() == null || request.path().isBlank() || request.distanceMeters() == null) {
            // regeneratePath=true인데 path/distance 누락 = 앱 버그
            throw new BusinessException(COURSE_INVALID_WAYPOINTS);
        }
        sanityCheckPath(request.path(), request.distanceMeters(), request.waypoints());

        course.updatePath(request.path(), request.distanceMeters());

        // UNIQUE (course_id, seq) 충돌 방지 → DELETE 먼저, INSERT 나중
        courseWaypointRepository.deleteByCourseEntityId(courseId);
        saveWaypoints(course, request.waypoints());
    }

    return buildDetailResponse(courseId, userId);
}
```

### 6-3. previewCourse() — 유일한 Naver 호출 지점

미리보기는 사용자별 처리 로직 없음(rate limit도 클라이언트 debounce로 대체). `userId` 파라미터 불필요.
DB 접근 없음 → 트랜잭션 어노테이션 생략.

```java
public CoursePreviewResponse previewCourse(CoursePreviewRequest request) {
    validateWaypoints(request.waypoints());
    DirectionsResult dirs = callDirections(request.waypoints());
    return new CoursePreviewResponse(dirs.pathJson(), dirs.distanceMeters(), dirs.bbox());
}
```

### 6-4. 공통 헬퍼

```java
// waypoints 유효성 검증 (START 1, GOAL 1, VIA ≤15, seq unique)
private void validateWaypoints(List<WaypointRequest> waypoints) {
    long startCount = waypoints.stream().filter(w -> "START".equals(w.role())).count();
    long goalCount  = waypoints.stream().filter(w -> "GOAL".equals(w.role())).count();
    long viaCount   = waypoints.stream().filter(w -> "VIA".equals(w.role())).count();

    if (startCount != 1 || goalCount != 1) {
        throw new BusinessException(COURSE_INVALID_WAYPOINTS);
    }
    // Naver Directions 15 스펙: waypoints 파라미터(VIA만) 최대 15개
    if (viaCount > 15) {
        throw new BusinessException(COURSE_DIRECTIONS_WAYPOINTS_LIMIT);
    }

    long uniqueSeqs = waypoints.stream().mapToInt(WaypointRequest::seq).distinct().count();
    if (uniqueSeqs != waypoints.size()) {
        throw new BusinessException(COURSE_INVALID_WAYPOINTS);
    }
}

// Naver Directions 호출 (preview 전용)
// NaverMapsClient가 VIA 개수에 따라 Directions 5/15 자동 분기 (size<=5 → 5, else 15)
private DirectionsResult callDirections(List<WaypointRequest> waypoints) {
    WaypointRequest start = waypoints.stream()
            .filter(w -> "START".equals(w.role())).findFirst().orElseThrow();
    WaypointRequest goal = waypoints.stream()
            .filter(w -> "GOAL".equals(w.role())).findFirst().orElseThrow();
    List<Waypoint> vias = waypoints.stream()
            .filter(w -> "VIA".equals(w.role()))
            .sorted(Comparator.comparingInt(WaypointRequest::seq))
            .map(w -> new Waypoint(w.latitude(), w.longitude()))
            .toList();

    NaverDirectionsResponse response = naverMapsClient.directions(
            start.longitude(), start.latitude(),
            goal.longitude(), goal.latitude(),
            vias
    );

    // option=traavoidcaronly로 요청 → 응답 필드도 traavoidcaronly (option 값과 이름 동일)
    var route = response.route().traavoidcaronly().get(0);
    String pathJson;
    try {
        pathJson = objectMapper.writeValueAsString(route.path());
    } catch (JsonProcessingException e) {
        throw new BusinessException(COURSE_DIRECTIONS_FAILED);
    }

    int distance = route.summary().distance();
    List<List<Double>> bbox = route.summary().bbox();
    return new DirectionsResult(pathJson, distance, bbox);
}

// waypoints 엔티티 저장 (place_id 있으면 place 조회 후 스냅샷, 없으면 좌표만 저장)
private void saveWaypoints(CourseEntity course, List<WaypointRequest> waypoints) {
    for (WaypointRequest w : waypoints) {
        CourseWaypointEntity waypointEntity;
        if (w.placeId() != null) {
            PlaceEntity place = placeRepository.findById(w.placeId())
                    .orElseThrow(() -> new BusinessException(PLACE_NOT_FOUND));
            waypointEntity = CourseWaypointEntity.createWithPlace(
                    course, place, w.seq(), w.role()
            );
        } else {
            waypointEntity = CourseWaypointEntity.create(
                    course, w.seq(), w.role(), w.placeName(), w.latitude(), w.longitude()
            );
        }
        courseWaypointRepository.save(waypointEntity);
    }
}

// 상세 응답 조회 헬퍼 (fetch join + 즐겨찾기 여부)
private CourseDetailResponse buildDetailResponse(UUID courseId, UUID userId) {
    CourseEntity course = courseRepository.findByIdWithUser(courseId)
            .orElseThrow(() -> new BusinessException(COURSE_NOT_FOUND));
    List<CourseWaypointResponse> waypointResponses = courseWaypointRepository
            .findByCourseEntityIdWithPlaceOrderBySeqAsc(courseId)
            .stream().map(CourseWaypointResponse::from).toList();
    boolean isFavorited = !course.isOwner(userId)
            && courseFavoriteRepository.existsById(new CourseFavoriteId(courseId, userId));
    return CourseDetailResponse.from(course, waypointResponses, userId, isFavorited);
}

// 옵션 B sanity check: 앱이 전송한 path/distance가 waypoints와 대략 일치하는지 최소 검증
// 사용자 본인 코스라 조작 인센티브 낮음 → 파싱 가능 + start/goal 좌표 근사 일치만 확인
private void sanityCheckPath(String pathJson, int distanceMeters, List<WaypointRequest> waypoints) {
    if (distanceMeters <= 0) {
        throw new BusinessException(COURSE_INVALID_WAYPOINTS);
    }
    List<List<Double>> path;
    try {
        path = objectMapper.readValue(pathJson, new TypeReference<>() {});
    } catch (JsonProcessingException e) {
        throw new BusinessException(COURSE_INVALID_WAYPOINTS);
    }
    if (path.isEmpty() || path.get(0).size() < 2 || path.get(path.size() - 1).size() < 2) {
        throw new BusinessException(COURSE_INVALID_WAYPOINTS);
    }

    WaypointRequest start = waypoints.stream()
            .filter(w -> "START".equals(w.role())).findFirst().orElseThrow();
    WaypointRequest goal = waypoints.stream()
            .filter(w -> "GOAL".equals(w.role())).findFirst().orElseThrow();

    // path[0] = [lng, lat], path[last] = [lng, lat]
    double pathStartLng = path.get(0).get(0);
    double pathStartLat = path.get(0).get(1);
    double pathGoalLng  = path.get(path.size() - 1).get(0);
    double pathGoalLat  = path.get(path.size() - 1).get(1);

    // 0.001도 ≈ 100m 이내 오차 허용 (Directions는 인접 도로로 스냅되므로 정확 일치 안 함)
    if (Math.abs(pathStartLng - start.longitude().doubleValue()) > 0.001
            || Math.abs(pathStartLat - start.latitude().doubleValue()) > 0.001
            || Math.abs(pathGoalLng - goal.longitude().doubleValue()) > 0.001
            || Math.abs(pathGoalLat - goal.latitude().doubleValue()) > 0.001) {
        throw new BusinessException(COURSE_INVALID_WAYPOINTS);
    }
}

// Directions 결과 내부 record (bbox는 앱 지도 fitBounds용, DB 저장 안 함)
private record DirectionsResult(String pathJson, int distanceMeters, List<List<Double>> bbox) {}
```

---

## 7. CourseWaypointRepository — 삭제 메서드 추가

파일: `src/main/java/com/bikeridediary/domain/course/repository/CourseWaypointRepository.java`

```java
// update 시 기존 waypoints 전량 삭제 (재저장 패턴). @Modifying 불필요 (파생 메서드가 delete 지원)
void deleteByCourseEntityId(UUID courseId);
```

---

## 8. ErrorCode 추가

파일: `src/main/java/com/bikeridediary/global/exception/ErrorCode.java`

기존 코스 블록에 추가:

```java
COURSE_DIRECTIONS_FAILED(HttpStatus.BAD_GATEWAY, "COURSE_DIRECTIONS_FAILED",
        "경로를 계산할 수 없습니다. 출발지·경유지·도착지를 확인해 주세요"),
COURSE_DIRECTIONS_WAYPOINTS_LIMIT(HttpStatus.BAD_REQUEST, "COURSE_DIRECTIONS_WAYPOINTS_LIMIT",
        "경유지는 최대 15개까지 지정할 수 있습니다"),
COURSE_INVALID_WAYPOINTS(HttpStatus.BAD_REQUEST, "COURSE_INVALID_WAYPOINTS",
        "waypoint 구성이 올바르지 않습니다 (출발지 1개, 도착지 1개 필수)"),
```

---

## 9. CourseController — 3개 엔드포인트 추가

파일: `src/main/java/com/bikeridediary/domain/course/controller/CourseController.java`

기존 `@Tag` 설명 수정: `"라이딩 코스 조회/즐겨찾기/생성/편집 관리"`

```java
@Operation(summary = "코스 생성 (옵션 B: 앱이 preview 결과 함께 전송)")
@PostMapping
public ResponseEntity<ApiResponse<CourseDetailResponse>> createCourse(
        @AuthenticationPrincipal CustomUserDetails userDetails,
        @RequestBody @Valid CourseCreateRequest request
) {
    return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.ok(courseService.createCourse(userDetails.getUserId(), request)));
}

@Operation(summary = "코스 수정 (작성자만, regeneratePath=true 시 앱이 preview 재실행 결과 전송)")
@PatchMapping("/{id}")
public ResponseEntity<ApiResponse<CourseDetailResponse>> updateCourse(
        @PathVariable UUID id,
        @AuthenticationPrincipal CustomUserDetails userDetails,
        @RequestBody @Valid CourseUpdateRequest request
) {
    return ResponseEntity.ok(
            ApiResponse.ok(courseService.updateCourse(userDetails.getUserId(), id, request)));
}

@Operation(summary = "경로 미리보기 (저장 없이 Directions 호출, path + distance + bbox 반환)")
@PostMapping("/preview")
public ResponseEntity<ApiResponse<CoursePreviewResponse>> previewCourse(
        @AuthenticationPrincipal CustomUserDetails userDetails,
        @RequestBody @Valid CoursePreviewRequest request
) {
    // userDetails는 인증 강제용 (SecurityConfig가 permitAll 아님을 명시)
    return ResponseEntity.ok(ApiResponse.ok(courseService.previewCourse(request)));
}
```

---

## 10. SecurityConfig 확인

현재 `GET /api/v1/courses`만 permitAll. POST/PATCH/DELETE는 기본 인증 필요.
`/api/v1/courses/preview`는 POST라 인증 필요 — **추가 설정 없음**.

SecurityConfig의 `GET_PERMIT_ALL_ENDPOINTS` 배열에 `/api/v1/courses`(GET 전용)만 있는지 재확인 후 진행.

---

## 백엔드 정책 영향 (D5=B + 옵션 B, 2026-08-04)

- **Naver Directions 호출 지점 단 하나**: `POST /courses/preview`. create/update는 Naver 재호출 없이 앱이 전송한 path/distance 그대로 저장
- **저장 실패 위험 최소화**: 사용자가 미리보기 지도에서 본 경로 = DB 저장 경로 100% 보장
- **호출 빈도 낮음**: 편집 세션당 preview 1~3회. Naver 무료 60,000회/월 여유 충분
- **stale 관리는 클라이언트 책임**: waypoint 변경 후 preview 재호출 안 하면 저장 차단 (앱이 stale 감지)
- **path 저장 신뢰**: 앱이 조작한 path 저장 가능성 있으나 사용자 본인 코스라 인센티브 낮음. `sanityCheckPath()`로 파싱 가능 + start/goal 좌표 ±100m 최소 검증
- **로그 레벨**: `NaverMapsClient` DEBUG 로그가 preview마다 찍힘. 프로덕션은 INFO로 (실패만 로깅)

---

## 구현 순서 권장

1. `schema.sql`: `ALTER TABLE courses ADD COLUMN IF NOT EXISTS description TEXT` + role CHECK 재정의 (END→GOAL)
2. 기존 데이터 마이그레이션: `UPDATE course_waypoints SET role = 'GOAL' WHERE role = 'END'`
3. `CourseWaypointEntity` role 상수 END → GOAL
4. `CourseEntity`에 `description` 필드 + `update()` / `updatePath()` + `createWithId()` 파라미터 추가
5. DTO 5개 신규 (WaypointRequest, CourseCreateRequest, CourseUpdateRequest, CoursePreviewRequest/Response)
6. `ErrorCode` 3개 추가
7. `CourseWaypointRepository.deleteByCourseEntityId` 추가
8. `CourseService`에 헬퍼 포함 신규 메서드 추가 (컴파일 오류 확인)
9. `CourseController`에 3개 엔드포인트 추가
10. Swagger에서 3개 엔드포인트 확인
11. `POST /api/v1/courses/preview` 먼저 수동 테스트 (Directions 응답 확인, path JSON 크기 실측)

---

## 체크리스트

- [ ] `schema.sql`: description 컬럼 + role CHECK 제약 END→GOAL 재정의
- [ ] `UPDATE course_waypoints SET role = 'GOAL' WHERE role = 'END'` 실행 (기존 데이터 없으면 skip)
- [ ] `CourseWaypointEntity` role 상수/enum END → GOAL
- [ ] `CourseEntity` description 필드 + `createWithId()` 파라미터 추가
- [ ] `save()` 반환값 사용: `CourseEntity saved = courseRepository.save(toSave)` (ID 수동 세팅 → merge 경로 방어)
- [ ] IDOR 방어: `course.isOwner(userId)` 검증
- [ ] `NaverMapsClient.directions()` 호출 시 lng, lat 순서 (경도 먼저) 혼동 없는지 확인
- [ ] VIA만 `waypoints` 파라미터에 포함 (START/GOAL은 `start`/`goal` 파라미터로 분리)
- [ ] `PlaceRepository`, `NaverMapsClient`, `ObjectMapper` 의존성 주입 추가
- [ ] `deleteByCourseEntityId` Repository 메서드 추가
- [ ] `CourseCreateRequest`/`CourseUpdateRequest`에 `path`, `distanceMeters` 필드 존재
- [ ] `sanityCheckPath()` 헬퍼 구현 (파싱 + start/goal 좌표 ±0.001도)
- [ ] `create()`/`update(regeneratePath=true)`에서 Naver 호출 안 하고 request의 path/distance 그대로 저장

---

## flutter-dev에게 넘길 API 스펙

### POST /api/v1/courses

인증: Bearer 필수

Request (옵션 B — preview에서 받은 path/distanceMeters 함께 전송):
```json
{
  "name": "북한산 일주",
  "description": "정릉~우이~도선사 코스",
  "isPublic": false,
  "sourceCourseId": null,
  "waypoints": [
    { "role": "START", "seq": 0, "latitude": 37.6029, "longitude": 127.0105, "placeName": "정릉 입구", "placeId": null },
    { "role": "VIA",   "seq": 1, "latitude": 37.6611, "longitude": 127.0146, "placeName": "우이동", "placeId": "uuid-here" },
    { "role": "GOAL",  "seq": 2, "latitude": 37.6521, "longitude": 127.0073, "placeName": "도선사", "placeId": null }
  ],
  "path": "[[127.0105,37.6029],[127.0112,37.6035],...]",
  "distanceMeters": 14300
}
```

Response: `CourseDetailResponse` (기존 상세 조회 응답과 동일)

### PATCH /api/v1/courses/{id}

인증: Bearer 필수 (작성자 본인만)

Request:
```json
{
  "name": "북한산 일주 (수정)",
  "description": null,
  "isPublic": true,
  "regeneratePath": true,
  "waypoints": [ ... ],
  "path": "[[127.0105,37.6029],...]",
  "distanceMeters": 14300
}
```

- `regeneratePath: false` → `waypoints`, `path`, `distanceMeters` 무시. name/description/isPublic만 업데이트
- `regeneratePath: true` → `waypoints`, `path`, `distanceMeters` 모두 필수

Response: `CourseDetailResponse`

### POST /api/v1/courses/preview

인증: Bearer 필수

Request:
```json
{
  "waypoints": [
    { "role": "START", "seq": 0, "latitude": 37.6029, "longitude": 127.0105, "placeName": "정릉 입구" },
    { "role": "GOAL",  "seq": 1, "latitude": 37.6521, "longitude": 127.0073, "placeName": "도선사" }
  ]
}
```

Response:
```json
{
  "success": true,
  "data": {
    "path": "[[127.0105,37.6029],[127.0112,37.6035],...]",
    "distanceMeters": 14300,
    "bbox": [[127.0366708, 37.4813912], [128.5771841, 38.2276884]]
  }
}
```

- `path`: JSON 문자열 (파싱 후 `List<List<double>>`). Naver 형식은 `[lng, lat]` 순서. NLatLng 생성 시 `NLatLng(lat, lng)` 역전 필요
- `bbox`: 경로 전체를 감싸는 최소 사각형 `[[minLng,minLat],[maxLng,maxLat]]`. 앱에서 지도 초기 카메라를 경로 전체에 맞춰 잡을 때 사용 (`LatLngBounds` + `CameraUpdate.fitBounds`)

---

## D5=B + 옵션 B 클라이언트 정책 (앱 게이트 3 참조)

1. 사용자가 "경로 미리보기" 버튼 클릭 → 빈 waypoint 필터링 + seq 재부여 → `POST /preview`
2. preview 성공 시 앱 로컬에 `lastValidPayload`, `lastPath`, `lastDistance` 보관 + 지도에 폴리라인 렌더링
3. 저장 버튼은 `lastValidPayload != null && waypoints == lastEditedSnapshot` 조건 활성 (preview 이후 편집 없어야)
4. 저장 시 `lastValidPayload` + `lastPath` + `lastDistance`를 `POST /courses`에 함께 전송 (Naver 재호출 X)
5. waypoint 변경(추가/삭제/좌표변경/순서변경) 감지 시 stale 처리 → 저장 버튼 비활성 + "경로를 다시 확인해 주세요" 안내
6. 빈 칸이 섞인 상태는 UI에만 존재. 필터링 + seq 재부여 후 payload = DB 저장값 (사용자가 본 것 = 저장된 것)
