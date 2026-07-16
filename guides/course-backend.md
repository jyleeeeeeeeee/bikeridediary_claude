# 라이딩 코스 도메인 — 백엔드 구현 가이드

> backend-dev 에이전트 산출. 사용자가 brd_be에 직접 구현. 파일 경로별 복붙 가능한 완성 코드.

---

## 목표

라이딩 코스(Course) 도메인 MVP 백엔드 구현 — 조회 + 즐겨찾기 + 소유 마커. 생성/편집은 제외.

---

## 파일 목록

| 순서 | 경로 | 신규/수정 | 설명 |
|-----|------|----------|------|
| 1 | `domain/course/entity/CourseEntity.java` | 신규 | 코스 기본 정보 |
| 2 | `domain/course/entity/CourseWaypointEntity.java` | 신규 | 경유지 |
| 3 | `domain/course/entity/CourseFavoriteId.java` | 신규 | 즐겨찾기 복합키 |
| 4 | `domain/course/entity/CourseFavoriteEntity.java` | 신규 | 즐겨찾기 매핑 |
| 5 | `domain/course/repository/CourseRepository.java` | 신규 | JPQL 포함 |
| 6 | `domain/course/repository/CourseWaypointRepository.java` | 신규 | 경유지 조회 |
| 7 | `domain/course/repository/CourseFavoriteRepository.java` | 신규 | 즐겨찾기 CRUD |
| 8 | `domain/course/dto/CourseWaypointResponse.java` | 신규 | |
| 9 | `domain/course/dto/CourseSummaryResponse.java` | 신규 | 리스트용 (path 제외) |
| 10 | `domain/course/dto/CourseDetailResponse.java` | 신규 | 상세용 |
| 11 | `domain/course/service/CourseService.java` | 신규 | 5개 메서드 |
| 12 | `domain/course/controller/CourseController.java` | 신규 | 5개 엔드포인트 |
| 13 | `global/exception/ErrorCode.java` | 수정 | FAVORITE 3개 추가 |
| 14 | `global/config/SecurityConfig.java` | 수정 | `/api/v1/courses` 인가 정책 |
| 15 | `docs/course-api.md` | 신규 | API 스펙 문서 |

---

## 1. CourseEntity

`src/main/java/com/bikeridediary/domain/course/entity/CourseEntity.java`

```java
package com.bikeridediary.domain.course.entity;

import com.bikeridediary.domain.common.entity.BaseEntity;
import com.bikeridediary.domain.user.entity.UserEntity;
import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.util.UUID;

// 라이딩 코스 엔티티 — 경로 데이터(path)와 경유지(waypoints)를 보유
@Entity
@Table(name = "courses")
@EntityListeners(AuditingEntityListener.class)
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CourseEntity extends BaseEntity {

    // 코스 ID (클라이언트 UUID)
    @Id
    @Column(name = "id")
    private UUID id;

    // 코스 작성자 (FK, nullable — 시드/큐레이션 코스는 작성자 없음)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = true)
    private UserEntity userEntity;

    // 코스 이름
    @Column(name = "name", nullable = false, length = 200)
    private String name;

    // 총 거리 (미터 단위)
    @Column(name = "distance_meters")
    private Integer distanceMeters;

    // 경로 좌표 배열 (JSON 문자열 — [[lng,lat],[lng,lat]...], TEXT 컬럼)
    @Column(name = "path", columnDefinition = "TEXT")
    private String path;

    // 공개 여부 (true=탐색탭 노출, false=작성자만 조회 가능)
    @Column(name = "is_public", nullable = false)
    private boolean isPublic = false;

    // 원본 코스 ID (남의 코스를 복제해서 저장할 때 참조, 자기참조 FK)
    @Column(name = "source_course_id")
    private UUID sourceCourseId;

    // 이 코스가 특정 사용자에게 속하는지 확인 (권한 검증용)
    // 시드 코스(userEntity=null)는 그 누구의 소유도 아님
    public boolean isOwner(UUID userId) {
        if (this.userEntity == null || userId == null) return false;
        return this.userEntity.getId().equals(userId);
    }

    // 코스 생성 팩토리
    public static CourseEntity createWithId(
            UUID id,
            UserEntity userEntity,
            String name,
            Integer distanceMeters,
            String path,
            boolean isPublic,
            UUID sourceCourseId
    ) {
        CourseEntity e = new CourseEntity();
        e.id = id;
        e.userEntity = userEntity;
        e.name = name;
        e.distanceMeters = distanceMeters;
        e.path = path;
        e.isPublic = isPublic;
        e.sourceCourseId = sourceCourseId;
        return e;
    }
}
```

## 2. CourseWaypointEntity

`src/main/java/com/bikeridediary/domain/course/entity/CourseWaypointEntity.java`

```java
package com.bikeridediary.domain.course.entity;

import com.bikeridediary.domain.place.entity.PlaceEntity;
import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.util.UUID;

// 코스 경유지 엔티티 — 코스의 출발지/경유지/목적지를 순서(seq)로 관리
@Entity
@Table(name = "course_waypoints")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CourseWaypointEntity {

    // 경유지 ID (UUID)
    @Id
    @Column(name = "id")
    private UUID id;

    // 소속 코스 (FK, 코스 삭제 시 CASCADE)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "course_id", nullable = false)
    private CourseEntity courseEntity;

    // 등록된 place 참조 (옵셔널) — 임의 지점은 null
    // ON DELETE SET NULL: place 삭제되어도 좌표 스냅샷으로 코스는 유효
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "place_id", nullable = true)
    private PlaceEntity placeEntity;

    // 순서 인덱스 (0-based, 정렬용)
    @Column(name = "seq", nullable = false)
    private short seq;

    // 역할 (START: 출발지, END: 목적지, VIA: 경유지)
    @Column(name = "role", nullable = false, length = 20)
    private String role;

    // 지점 이름 스냅샷 (place 사용 시 등록 시점 place.placeName 복사, 임의 지점은 사용자 입력)
    @Column(name = "name", length = 200)
    private String name;

    // 위도 (소수점 7자리, 약 1.1cm 정밀도)
    @Column(name = "latitude", nullable = false, precision = 9, scale = 7)
    private BigDecimal latitude;

    // 경도 (소수점 7자리, 약 1.1cm 정밀도)
    @Column(name = "longitude", nullable = false, precision = 10, scale = 7)
    private BigDecimal longitude;

    // 임의 지점 waypoint 생성 (지도 롱프레스, GPX 임포트 등 place 없는 지점)
    // 사용자가 좌표/이름 지정
    public static CourseWaypointEntity create(
            CourseEntity courseEntity,
            short seq,
            String role,
            String name,
            BigDecimal latitude,
            BigDecimal longitude
    ) {
        CourseWaypointEntity e = new CourseWaypointEntity();
        e.id = UUID.randomUUID();
        e.courseEntity = courseEntity;
        e.placeEntity = null;
        e.seq = seq;
        e.role = role;
        e.name = name;
        e.latitude = latitude;
        e.longitude = longitude;
        return e;
    }

    // 등록된 place를 waypoint로 사용 시 생성
    // 이름/좌표는 place에서 스냅샷으로 자동 복사 (place가 나중에 수정되어도 코스는 등록 시점 유지)
    public static CourseWaypointEntity createWithPlace(
            CourseEntity courseEntity,
            PlaceEntity placeEntity,
            short seq,
            String role
    ) {
        CourseWaypointEntity e = new CourseWaypointEntity();
        e.id = UUID.randomUUID();
        e.courseEntity = courseEntity;
        e.placeEntity = placeEntity;
        e.seq = seq;
        e.role = role;
        e.name = placeEntity.getPlaceName();
        e.latitude = placeEntity.getLatitude();
        e.longitude = placeEntity.getLongitude();
        return e;
    }
}
```

## 3. CourseFavoriteId

`src/main/java/com/bikeridediary/domain/course/entity/CourseFavoriteId.java`

```java
package com.bikeridediary.domain.course.entity;

import jakarta.persistence.Embeddable;
import lombok.*;

import java.io.Serializable;
import java.util.UUID;

// 즐겨찾기 복합 PK — (course_id, user_id)
@Embeddable
@Getter
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode
public class CourseFavoriteId implements Serializable {
    private UUID courseId;
    private UUID userId;
}
```

## 4. CourseFavoriteEntity

`src/main/java/com/bikeridediary/domain/course/entity/CourseFavoriteEntity.java`

```java
package com.bikeridediary.domain.course.entity;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;
import java.util.UUID;

// 즐겨찾기 엔티티 — 공개된 남의 코스만 즐겨찾기 가능
@Entity
@Table(name = "course_favorites")
@EntityListeners(AuditingEntityListener.class)
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CourseFavoriteEntity {

    @EmbeddedId
    private CourseFavoriteId id;

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    public static CourseFavoriteEntity create(UUID courseId, UUID userId) {
        CourseFavoriteEntity e = new CourseFavoriteEntity();
        e.id = new CourseFavoriteId(courseId, userId);
        return e;
    }
}
```

## 5. CourseRepository

`src/main/java/com/bikeridediary/domain/course/repository/CourseRepository.java`

```java
package com.bikeridediary.domain.course.repository;

import com.bikeridediary.domain.course.entity.CourseEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface CourseRepository extends JpaRepository<CourseEntity, UUID> {

    // MY탭 — 내가 만든 코스 (hard delete 정책이라 deleted_at 필터 없음)
    List<CourseEntity> findByUserEntityIdOrderByCreatedAtDesc(UUID userId);

    // MY탭 — 내가 즐겨찾기한 코스 (남의 코스 + 시드 코스 모두 포함, 내 것만 제외)
    @Query("""
            SELECT c FROM CourseEntity c
            JOIN CourseFavoriteEntity f ON f.id.courseId = c.id
            WHERE f.id.userId = :userId
              AND c.isPublic = TRUE
              AND (c.userEntity IS NULL OR c.userEntity.id <> :userId)
            ORDER BY f.createdAt DESC
            """)
    List<CourseEntity> findFavoritedByOthers(@Param("userId") UUID userId);

    // 탐색탭 — 공개 코스 전체
    List<CourseEntity> findByIsPublicTrueOrderByCreatedAtDesc();

    // 탐색탭 — 코스명 부분일치 검색
    @Query("""
            SELECT c FROM CourseEntity c
            WHERE c.isPublic = TRUE
              AND LOWER(c.name) LIKE LOWER(CONCAT('%', :keyword, '%'))
            ORDER BY c.createdAt DESC
            """)
    List<CourseEntity> searchPublicByName(@Param("keyword") String keyword);

    // 상세 조회 — User fetch join (waypoints는 별도 Repository로 분리 조회)
    @Query("""
            SELECT DISTINCT c FROM CourseEntity c
            LEFT JOIN FETCH c.userEntity
            WHERE c.id = :id
            """)
    Optional<CourseEntity> findByIdWithUser(@Param("id") UUID id);
}
```

> **waypoints fetch join 안 하는 이유**: 컬렉션 fetch join + distinct 조합은 페이징 불가 경고 발생하고 코드 복잡. MVP에서는 분리 조회가 낫다 (Service에서 처리).

## 6. CourseWaypointRepository

`src/main/java/com/bikeridediary/domain/course/repository/CourseWaypointRepository.java`

```java
package com.bikeridediary.domain.course.repository;

import com.bikeridediary.domain.course.entity.CourseWaypointEntity;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.UUID;

public interface CourseWaypointRepository extends JpaRepository<CourseWaypointEntity, UUID> {

    // 코스의 경유지 목록 (seq 오름차순)
    List<CourseWaypointEntity> findByCourseEntityIdOrderBySeqAsc(UUID courseId);
}
```

## 7. CourseFavoriteRepository

`src/main/java/com/bikeridediary/domain/course/repository/CourseFavoriteRepository.java`

```java
package com.bikeridediary.domain.course.repository;

import com.bikeridediary.domain.course.entity.CourseFavoriteEntity;
import com.bikeridediary.domain.course.entity.CourseFavoriteId;
import org.springframework.data.jpa.repository.JpaRepository;

public interface CourseFavoriteRepository extends JpaRepository<CourseFavoriteEntity, CourseFavoriteId> {
    boolean existsById(CourseFavoriteId id);
    void deleteById(CourseFavoriteId id);
}
```

## 8. CourseWaypointResponse

`src/main/java/com/bikeridediary/domain/course/dto/CourseWaypointResponse.java`

> `placeId`가 null이 아니면 앱에서 place 상세 화면으로 딥링크 가능.
> `placeCategoryCode`로 마커 아이콘/색상 결정 (CAFE/SERVICE_CENTER 등).

```java
package com.bikeridediary.domain.course.dto;

import com.bikeridediary.domain.course.entity.CourseWaypointEntity;
import com.bikeridediary.domain.place.entity.PlaceEntity;

import java.math.BigDecimal;
import java.util.UUID;

public record CourseWaypointResponse(
        UUID id,
        short seq,
        String role,
        String name,
        BigDecimal latitude,
        BigDecimal longitude,
        // 참조된 place ID (임의 지점은 null)
        UUID placeId,
        // 참조된 place의 카테고리 코드 (앱에서 마커 아이콘 결정용, null 가능)
        String placeCategoryCode
) {
    public static CourseWaypointResponse from(CourseWaypointEntity entity) {
        PlaceEntity place = entity.getPlaceEntity();
        return new CourseWaypointResponse(
                entity.getId(),
                entity.getSeq(),
                entity.getRole(),
                entity.getName(),
                entity.getLatitude(),
                entity.getLongitude(),
                place == null ? null : place.getId(),
                place == null ? null : place.getPlaceCategoryEntity().getCategoryCode()
        );
    }
}
```

> **N+1 주의**: `getPlaceEntity()`가 LAZY라 상세 조회 시 waypoint마다 place 쿼리가 나갈 수 있음. `CourseWaypointRepository`에 fetch join 메서드 추가 필요:
> ```java
> @Query("""
>         SELECT w FROM CourseWaypointEntity w
>         LEFT JOIN FETCH w.placeEntity p
>         LEFT JOIN FETCH p.placeCategoryEntity
>         WHERE w.courseEntity.id = :courseId
>         ORDER BY w.seq ASC
>         """)
> List<CourseWaypointEntity> findByCourseEntityIdWithPlaceOrderBySeqAsc(@Param("courseId") UUID courseId);
> ```
> Service의 `getDetail`에서 이 메서드 사용.

## 9. CourseSummaryResponse

`src/main/java/com/bikeridediary/domain/course/dto/CourseSummaryResponse.java`

> UserEntity의 닉네임 필드명이 다르면 `getNickname()` 부분을 실제 필드명으로 교체.

```java
package com.bikeridediary.domain.course.dto;

import com.bikeridediary.domain.course.entity.CourseEntity;
import com.bikeridediary.domain.user.entity.UserEntity;

import java.time.LocalDateTime;
import java.util.UUID;

// 리스트용 DTO — path 제외 (수십 KB 절약)
public record CourseSummaryResponse(
        UUID id,
        String name,
        Integer distanceMeters,
        // 작성자 닉네임 — 시드/큐레이션 코스는 빈 문자열("")
        String authorNickname,
        boolean isPublic,
        boolean ownedByMe,
        boolean isFavorited,
        LocalDateTime createdAt
) {
    public static CourseSummaryResponse from(CourseEntity entity, UUID requestUserId, boolean isFavorited) {
        UserEntity author = entity.getUserEntity();
        return new CourseSummaryResponse(
                entity.getId(),
                entity.getName(),
                entity.getDistanceMeters(),
                author == null ? "" : author.getNickname(),
                entity.isPublic(),
                requestUserId != null && author != null && author.getId().equals(requestUserId),
                isFavorited,
                entity.getCreatedAt()
        );
    }
}
```

## 10. CourseDetailResponse

`src/main/java/com/bikeridediary/domain/course/dto/CourseDetailResponse.java`

```java
package com.bikeridediary.domain.course.dto;

import com.bikeridediary.domain.course.entity.CourseEntity;
import com.bikeridediary.domain.user.entity.UserEntity;

import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

public record CourseDetailResponse(
        UUID id,
        String name,
        Integer distanceMeters,
        // 작성자 닉네임 — 시드/큐레이션 코스는 빈 문자열("")
        String authorNickname,
        String path,
        boolean isPublic,
        UUID sourceCourseId,
        List<CourseWaypointResponse> waypoints,
        boolean ownedByMe,
        boolean isFavorited,
        LocalDateTime createdAt,
        LocalDateTime updatedAt
) {
    public static CourseDetailResponse from(
            CourseEntity entity,
            List<CourseWaypointResponse> waypoints,
            UUID requestUserId,
            boolean isFavorited
    ) {
        UserEntity author = entity.getUserEntity();
        return new CourseDetailResponse(
                entity.getId(),
                entity.getName(),
                entity.getDistanceMeters(),
                author == null ? "" : author.getNickname(),
                entity.getPath(),
                entity.isPublic(),
                entity.getSourceCourseId(),
                waypoints,
                requestUserId != null && author != null && author.getId().equals(requestUserId),
                isFavorited,
                entity.getCreatedAt(),
                entity.getUpdatedAt()
        );
    }
}
```

## 11. ErrorCode 추가

`src/main/java/com/bikeridediary/global/exception/ErrorCode.java` (수정)

기존 `COURSE_*` 블록 아래에 추가. `COURSE_NOT_FOUND`, `COURSE_ACCESS_DENIED` 등은 이미 존재하므로 건너뜀.

```java
COURSE_FAVORITE_ALREADY_EXISTS(HttpStatus.CONFLICT, "COURSE_FAVORITE_ALREADY_EXISTS", "이미 즐겨찾기한 코스입니다"),
COURSE_FAVORITE_NOT_FOUND(HttpStatus.NOT_FOUND, "COURSE_FAVORITE_NOT_FOUND", "즐겨찾기 기록을 찾을 수 없습니다"),
COURSE_FAVORITE_OWN_COURSE(HttpStatus.BAD_REQUEST, "COURSE_FAVORITE_OWN_COURSE", "내 코스는 즐겨찾기할 수 없습니다"),
```

## 12. CourseService

`src/main/java/com/bikeridediary/domain/course/service/CourseService.java`

```java
package com.bikeridediary.domain.course.service;

import com.bikeridediary.domain.course.dto.*;
import com.bikeridediary.domain.course.entity.*;
import com.bikeridediary.domain.course.repository.*;
import com.bikeridediary.domain.user.repository.UserRepository;
import com.bikeridediary.global.exception.BusinessException;
import com.bikeridediary.global.exception.ErrorCode;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.*;
import java.util.stream.Collectors;

import static com.bikeridediary.global.exception.ErrorCode.*;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class CourseService {

    private final CourseRepository courseRepository;
    private final CourseWaypointRepository waypointRepository;
    private final CourseFavoriteRepository favoriteRepository;
    private final UserRepository userRepository;

    public List<CourseSummaryResponse> getMyList(UUID userId) {
        verifyUserExists(userId);

        List<CourseEntity> myCourses = courseRepository
                .findByUserEntityIdOrderByCreatedAtDesc(userId);
        List<CourseEntity> favorited = courseRepository.findFavoritedByOthers(userId);

        List<CourseSummaryResponse> result = new ArrayList<>();
        for (CourseEntity c : myCourses) {
            result.add(CourseSummaryResponse.from(c, userId, false));
        }
        for (CourseEntity c : favorited) {
            result.add(CourseSummaryResponse.from(c, userId, true));
        }
        return result;
    }

    public List<CourseSummaryResponse> getPublicList(UUID userId, String keyword) {
        List<CourseEntity> courses = (keyword == null || keyword.isBlank())
                ? courseRepository.findByIsPublicTrueOrderByCreatedAtDesc()
                : courseRepository.searchPublicByName(keyword.trim());

        Set<UUID> myFavoriteIds = (userId == null)
                ? Set.of()
                : courseRepository.findFavoritedByOthers(userId)
                        .stream().map(CourseEntity::getId).collect(Collectors.toSet());

        return courses.stream()
                .map(c -> CourseSummaryResponse.from(c, userId, myFavoriteIds.contains(c.getId())))
                .toList();
    }

    public CourseDetailResponse getDetail(UUID courseId, UUID userId) {
        CourseEntity course = courseRepository.findByIdWithUser(courseId)
                .orElseThrow(() -> new BusinessException(COURSE_NOT_FOUND));

        validateDetailAccess(course, userId);

        List<CourseWaypointResponse> waypoints = waypointRepository
                .findByCourseEntityIdOrderBySeqAsc(courseId)
                .stream()
                .map(CourseWaypointResponse::from)
                .toList();

        boolean isFavorited = !course.isOwner(userId)
                && favoriteRepository.existsById(new CourseFavoriteId(courseId, userId));

        return CourseDetailResponse.from(course, waypoints, userId, isFavorited);
    }

    @Transactional
    public boolean addFavorite(UUID courseId, UUID userId) {
        CourseEntity course = courseRepository.findByIdWithUser(courseId)
                .orElseThrow(() -> new BusinessException(COURSE_NOT_FOUND));

        if (course.isOwner(userId)) throw new BusinessException(COURSE_FAVORITE_OWN_COURSE);
        if (!course.isPublic())     throw new BusinessException(COURSE_ACCESS_DENIED);

        CourseFavoriteId favId = new CourseFavoriteId(courseId, userId);
        if (favoriteRepository.existsById(favId)) {
            throw new BusinessException(COURSE_FAVORITE_ALREADY_EXISTS);
        }

        favoriteRepository.save(CourseFavoriteEntity.create(courseId, userId));
        return true;
    }

    @Transactional
    public boolean removeFavorite(UUID courseId, UUID userId) {
        CourseFavoriteId favId = new CourseFavoriteId(courseId, userId);
        if (!favoriteRepository.existsById(favId)) {
            throw new BusinessException(COURSE_FAVORITE_NOT_FOUND);
        }
        favoriteRepository.deleteById(favId);
        return false;
    }

    // 코스 hard delete — 작성자 본인만 가능
    // CASCADE로 waypoints, favorites 자동 삭제
    // source_course_id ON DELETE SET NULL로 파생 코스는 유지 (source 참조만 NULL로)
    @Transactional
    public void deleteCourse(UUID courseId, UUID userId) {
        CourseEntity course = courseRepository.findByIdWithUser(courseId)
                .orElseThrow(() -> new BusinessException(COURSE_NOT_FOUND));
        if (!course.isOwner(userId)) {
            throw new BusinessException(COURSE_ACCESS_DENIED);
        }
        courseRepository.delete(course);
    }

    private void validateDetailAccess(CourseEntity course, UUID userId) {
        if (course.isOwner(userId)) return;
        if (course.isPublic()) return;
        boolean favoritedByMe = favoriteRepository
                .existsById(new CourseFavoriteId(course.getId(), userId));
        if (favoritedByMe) return;
        throw new BusinessException(COURSE_ACCESS_DENIED);
    }

    private void verifyUserExists(UUID userId) {
        userRepository.findByIdAndDeletedAtIsNull(userId)
                .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));
    }
}
```

## 13. CourseController

`src/main/java/com/bikeridediary/domain/course/controller/CourseController.java`

```java
package com.bikeridediary.domain.course.controller;

import com.bikeridediary.domain.course.dto.*;
import com.bikeridediary.domain.course.service.CourseService;
import com.bikeridediary.global.auth.CustomUserDetails;
import com.bikeridediary.global.response.ApiResponse;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.UUID;

@Tag(name = "코스", description = "라이딩 코스 조회 및 즐겨찾기 관리")
@RestController
@RequestMapping("/api/v1/courses")
@RequiredArgsConstructor
public class CourseController {

    private final CourseService courseService;

    @Operation(summary = "MY탭 — 내가 만든 코스 + 즐겨찾기한 남의 코스")
    @GetMapping("/my")
    public ResponseEntity<ApiResponse<List<CourseSummaryResponse>>> getMyList(
            @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        return ResponseEntity.ok(ApiResponse.ok(courseService.getMyList(userDetails.getUserId())));
    }

    @Operation(summary = "탐색탭 — 공개 코스 목록 (keyword로 코스명 검색 가능)")
    @GetMapping
    public ResponseEntity<ApiResponse<List<CourseSummaryResponse>>> getPublicList(
            @RequestParam(required = false) String keyword,
            @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        UUID userId = userDetails != null ? userDetails.getUserId() : null;
        return ResponseEntity.ok(ApiResponse.ok(courseService.getPublicList(userId, keyword)));
    }

    @Operation(summary = "코스 상세 조회 (waypoints + path 포함)")
    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<CourseDetailResponse>> getDetail(
            @PathVariable UUID id,
            @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        return ResponseEntity.ok(ApiResponse.ok(courseService.getDetail(id, userDetails.getUserId())));
    }

    @Operation(summary = "즐겨찾기 등록")
    @PostMapping("/{id}/favorite")
    public ResponseEntity<ApiResponse<Boolean>> addFavorite(
            @PathVariable UUID id,
            @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        return ResponseEntity.ok(ApiResponse.ok(courseService.addFavorite(id, userDetails.getUserId())));
    }

    @Operation(summary = "즐겨찾기 해제")
    @DeleteMapping("/{id}/favorite")
    public ResponseEntity<ApiResponse<Boolean>> removeFavorite(
            @PathVariable UUID id,
            @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        return ResponseEntity.ok(ApiResponse.ok(courseService.removeFavorite(id, userDetails.getUserId())));
    }

    @Operation(summary = "코스 삭제 (작성자만, hard delete)")
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteCourse(
            @PathVariable UUID id,
            @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        courseService.deleteCourse(id, userDetails.getUserId());
        return ResponseEntity.noContent().build();
    }
}
```

## 14. SecurityConfig 수정

`src/main/java/com/bikeridediary/global/config/SecurityConfig.java` (수정)

```java
private final String[] GET_PERMIT_ALL_ENDPOINTS = {
        "/api/v1/courses",           // 탐색탭 — 게스트도 열람 가능
        "/api/v1/bike-models/**",
};
```

기존의 `/api/v1/courses/public/**`은 제거.

> `GET /api/v1/courses`를 permitAll하면 `userDetails`가 null로 들어올 수 있으므로 Controller에서 이미 null 체크. Service의 `getPublicList`도 userId null 방어 완료.

## 15. docs/course-api.md

`src/main/resources/docs/course-api.md` 또는 `docs/course-api.md`

```markdown
# Course API Spec

Base path: `/api/v1/courses`

## Authentication
- `GET /courses` — 게스트 허용 (isFavorited는 항상 false)
- 나머지 전체 — `Authorization: Bearer <access_token>` 필수

## 1. MY탭
GET /api/v1/courses/my

Response 200:
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "name": "남해 남해대교 루프",
      "distanceMeters": 120000,
      "authorNickname": "홍길동",
      "isPublic": true,
      "ownedByMe": true,
      "isFavorited": false,
      "createdAt": "2026-07-16T10:00:00"
    },
    {
      "id": "uuid",
      "name": "설악산 한계령 루트",
      "distanceMeters": 85000,
      "authorNickname": "",
      "isPublic": true,
      "ownedByMe": false,
      "isFavorited": true,
      "createdAt": "2026-07-15T09:00:00"
    }
  ]
}

> `authorNickname`이 빈 문자열("")이면 시드/큐레이션 코스 (작성자 없음).

## 2. 탐색탭
GET /api/v1/courses?keyword=남해

Response: MY탭과 동일 구조

## 3. 상세
GET /api/v1/courses/{id}

Response:
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "...",
    "distanceMeters": 120000,
    "authorNickname": "홍길동",
    "path": "[[127.123,34.123],[127.234,34.234]]",
    "isPublic": true,
    "sourceCourseId": null,
    "waypoints": [
      { "id":"...", "seq":0, "role":"START", "name":"남해IC", "latitude":34.1234567, "longitude":127.1234567, "placeId":null, "placeCategoryCode":null },
      { "id":"...", "seq":1, "role":"VIA",   "name":"바다뷰카페", "latitude":34.2000000, "longitude":127.2000000, "placeId":"uuid-of-place", "placeCategoryCode":"CAFE" }
    ],
    "ownedByMe": true,
    "isFavorited": false,
    "createdAt": "...",
    "updatedAt": "..."
  }
}

Errors: 404 COURSE_NOT_FOUND, 403 COURSE_ACCESS_DENIED

## 4. 즐겨찾기 등록
POST /api/v1/courses/{id}/favorite → { "data": true }
Errors: 404 COURSE_NOT_FOUND, 400 COURSE_FAVORITE_OWN_COURSE, 403 COURSE_ACCESS_DENIED, 409 COURSE_FAVORITE_ALREADY_EXISTS

## 5. 즐겨찾기 해제
DELETE /api/v1/courses/{id}/favorite → { "data": false }
Errors: 404 COURSE_FAVORITE_NOT_FOUND

## 6. 코스 삭제 (hard delete)
DELETE /api/v1/courses/{id}
Response: 204 No Content
- 작성자 본인만 가능
- waypoints + favorites CASCADE 자동 삭제 (다른 사용자 즐겨찾기 목록에서도 사라짐)
- 파생 코스는 유지 (source_course_id → NULL)
Errors: 404 COURSE_NOT_FOUND, 403 COURSE_ACCESS_DENIED
```

---

## 구현 순서 권장

1. DDL 반영 (`docker compose down -v && docker compose up -d`)
2. Entity 3종 + CourseFavoriteId 작성 후 컴파일 (`./gradlew compileJava`)
3. Repository 3종
4. DTO 4종
5. ErrorCode FAVORITE 3개 추가
6. Service
7. Controller + SecurityConfig 수정
8. 단위 테스트
9. Swagger 수동 확인 (`http://localhost:8081/swagger-ui.html`)

---

## 체크리스트

- [ ] 모든 엔티티 필드 한글 주석
- [ ] CourseWaypointEntity.latitude/longitude `@Column(precision=9/10, scale=7)` 명시
- [ ] CourseEntity의 `@JoinColumn(user_id)` **nullable=true** (시드 코스 대비)
- [ ] `CourseEntity.isOwner()` null 안전 처리
- [ ] DTO에서 `entity.getUserEntity()` 접근 전 null 체크, null이면 authorNickname="", ownedByMe=false
- [ ] `findFavoritedByOthers` JPQL에 `(c.userEntity IS NULL OR c.userEntity.id <> :userId)` 조건 반영
- [ ] CourseWaypointEntity에 `@ManyToOne(nullable=true) PlaceEntity placeEntity` + `@JoinColumn(name="place_id")`
- [ ] CourseWaypointResponse에 placeId, placeCategoryCode 필드 추가
- [ ] 상세 조회 waypoint 쿼리에 place fetch join (N+1 방지)
- [ ] Hard delete: courses 테이블에 `deleted_at` 컬럼 없음, Entity에도 없음
- [ ] 모든 Repository 쿼리에서 `deleted_at IS NULL` 조건 제거됨
- [ ] `DELETE /api/v1/courses/{id}` 엔드포인트 + 소유권 검증
- [ ] CourseService 클래스 `@Transactional(readOnly = true)`, 쓰기 메서드에만 `@Transactional` 추가
- [ ] `getPublicList` userId null 분기 처리
- [ ] SecurityConfig `GET_PERMIT_ALL_ENDPOINTS` 수정 + 기존 `/api/v1/courses/public/**` 제거
- [ ] ErrorCode FAVORITE 3개 추가
- [ ] docs/course-api.md 작성

---

## 예상 함정

**1. source_course_id 자기참조 FK**
`@ManyToOne` 대신 단순 UUID 필드로 처리(위 코드 반영). LAZY 로딩 체인 방지.

**2. 복합키 @EmbeddedId**
`existsById(CourseFavoriteId)`, `deleteById(CourseFavoriteId)` 그대로 동작. `@IdClass` 대신 `@EmbeddedId` 채택 이유: Repository 시그니처 명확.

**3. path 필드 String vs @Convert**
MVP는 String 그대로. 코스 생성/편집 API 만들 때 `@Convert` 도입 여부 결정.

**4. getPublicList 쿼리 수**
목록 + 즐겨찾기 = 2회. MVP 규모 충분. 페이징 도입 시점에 JOIN 통합 검토.

**5. @Transactional(readOnly = true) 클래스 레벨 + 쓰기 메서드 오버라이드**
`addFavorite`/`removeFavorite`에 `@Transactional` 빠뜨리면 즐겨찾기 저장 안 됨.

---

## flutter-dev에게 넘긴 API 스펙 요약

```
[인증 필요]
GET    /api/v1/courses/my              → CourseSummaryResponse[]

[게스트 허용]
GET    /api/v1/courses?keyword=        → CourseSummaryResponse[]

[인증 필요]
GET    /api/v1/courses/{id}            → CourseDetailResponse
POST   /api/v1/courses/{id}/favorite   → { "data": true }
DELETE /api/v1/courses/{id}/favorite   → { "data": false }
DELETE /api/v1/courses/{id}            → 204 No Content (hard delete, 작성자만)
```
