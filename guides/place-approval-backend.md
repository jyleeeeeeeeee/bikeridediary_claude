# 장소 승인 워크플로 — 백엔드 구현 가이드 (backend-dev)

> 사용자가 직접 구현. 아래 스니펫은 그대로 파일에 복사 붙여넣기 가능한 완성형.
> 대상 프로젝트: `brd_be` (Spring Boot 3.x)
> 관련: `place-approval-workflow.md`, `place-approval-schema.md`, `place-approval-app.md`

---

## 작업 순서

1. `UserRole` enum
2. `UserEntity`에 role 필드 + 팩토리 파라미터
3. `CustomUserDetails` — role 필드 보관, authorities에 ROLE_ADMIN/ROLE_USER 반영
4. `CustomUserDetailsService` — UserEntity.getRole() 읽어 CustomUserDetails 생성 (매 요청 시 DB role lookup)
5. `UserResponse`에 `role` 필드 추가 (앱이 role을 알기 위함)
6. `PlaceChangeRequestEntity` + 관련 enum (RequestType, RequestStatus)
7. `PlaceChangeRequestRepository`
8. DTO들 (Create/UpdateCoordinates/UpdateInfo request, Response, AdminResponse, ReviewRequest)
9. `PlaceChangeRequestService`
10. `PlaceChangeRequestController` (일반 유저용)
11. `AdminPlaceChangeRequestController` (어드민용)
12. `PlaceEntity`에 삭제 헬퍼 유지 (신규 UPDATE 메서드 없이 기존 updateCoordinates/updateInfo 재활용)
13. `PlaceController` 정리 (POST /places, PATCH /places/{id}/coordinates, PATCH /places/{id}/info 제거)
14. `PlaceService` 정리 (`addNewPlace`, `updateCoordinates`, `updateInfo` 제거 또는 package-private으로 좁혀 Service 내부에서만 호출)
15. `SecurityConfig` 수정 — **`@EnableMethodSecurity` 추가 필수**
16. `ErrorCode` 추가
17. 어드민 지정 SQL

> **JWT role claim 방식 폐기**: 초안엔 JwtTokenProvider에 role claim을 심는 방식이 있었으나, DB role lookup 방식으로 변경. `JwtTokenProvider`는 원본 그대로 유지 (수정 불필요). 이유: JWT 시그니처 변경 파급 최소화 + role 승격/강등 즉시 반영 (재로그인 불필요) + 매 요청 DB 조회 비용은 무시 가능(이미 CustomUserDetailsService에서 유저 조회 중).

---

## 1. UserRole enum

`brd_be/src/main/java/com/bikeridediary/domain/user/entity/UserRole.java`

```java
package com.bikeridediary.domain.user.entity;

// 사용자 권한 (USER: 일반, ADMIN: 어드민)
public enum UserRole {
    USER,
    ADMIN
}
```

## 2. UserEntity 변경

`brd_be/src/main/java/com/bikeridediary/domain/user/entity/UserEntity.java`

기존 필드 목록 뒤(`fcmToken` 아래)에 추가:

```java
    // 사용자 권한 (USER: 일반, ADMIN: 어드민 - 장소 요청 승인 권한)
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private UserRole role = UserRole.USER;
```

기존 팩토리는 role을 세팅하지 않는다 (default USER). 어드민 지정은 DB UPDATE로만.

## 3. JwtTokenProvider — 수정 불필요

**DB role lookup 방식 채택으로 JwtTokenProvider는 원본 그대로 유지.**

role은 JWT claim이 아니라 매 요청 시 `CustomUserDetailsService.loadUserByUsername`이 UserEntity에서 조회한다.
`generateAccessToken(UUID)` 시그니처 변경 없음 → 호출부(각 OAuth Provider, refresh 흐름) 수정 불필요.

장점:
- JWT 발급 로직 변경 없음 → 파급 최소
- 어드민 승격/강등 즉시 반영 (재로그인 불필요)
- 매 요청 DB 조회 비용은 이미 loadUserByUsername에서 발생 중이라 추가 부담 0

## 4. CustomUserDetails

`brd_be/src/main/java/com/bikeridediary/global/auth/CustomUserDetails.java` 전체 교체:

```java
package com.bikeridediary.global.auth;

import com.bikeridediary.domain.user.entity.UserRole;
import lombok.Getter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.List;
import java.util.UUID;

// 커스텀 UserDetails - UUID userId + role 보관
@Getter
public class CustomUserDetails implements UserDetails {

    private final UUID userId;
    private final UserRole role;
    private final Collection<? extends GrantedAuthority> authorities;

    public CustomUserDetails(UUID userId, UserRole role) {
        this.userId = userId;
        this.role = role;
        this.authorities = List.of(new SimpleGrantedAuthority("ROLE_" + role.name()));
    }

    @Override public String getPassword() { return ""; }
    @Override public String getUsername() { return userId.toString(); }
    @Override public boolean isAccountNonExpired() { return true; }
    @Override public boolean isAccountNonLocked() { return true; }
    @Override public boolean isCredentialsNonExpired() { return true; }
    @Override public boolean isEnabled() { return true; }
}
```

## 5. CustomUserDetailsService

`brd_be/src/main/java/com/bikeridediary/domain/user/service/CustomUserDetailsService.java` 전체 교체:

```java
package com.bikeridediary.domain.user.service;

import com.bikeridediary.domain.user.entity.UserEntity;
import com.bikeridediary.domain.user.repository.UserRepository;
import com.bikeridediary.global.auth.CustomUserDetails;
import com.bikeridediary.global.exception.BusinessException;
import com.bikeridediary.global.exception.ErrorCode;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

// JWT 인증용 UserDetailsService 구현
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Transactional(readOnly = true)
    public UserDetails loadUserById(String userId) throws UsernameNotFoundException {
        return this.loadUserByUsername(userId);
    }

    @Override
    public UserDetails loadUserByUsername(String userId) throws UsernameNotFoundException {
        UserEntity userEntity = userRepository.findByIdAndDeletedAtIsNull(UUID.fromString(userId))
                .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));

        return new CustomUserDetails(userEntity.getId(), userEntity.getRole());
    }
}
```

**role 반영 지점**: 이 클래스가 role의 유일한 진실 소스. 매 요청마다 JwtAuthenticationFilter가 loadUserByUsername을 호출하므로, DB에서 role을 UPDATE 하면 다음 요청부터 즉시 반영됨 (JWT 재발급 불필요).

`JwtAuthenticationFilter`는 원본 그대로 유지 (수정 없음).

## 5. UserResponse에 role 필드 추가

`brd_be/src/main/java/com/bikeridediary/domain/user/dto/UserResponse.java`

앱이 어드민 여부를 알아야 어드민 화면 노출/라우팅이 가능. 로그인 응답 / GET /users/me 등 UserResponse를 반환하는 모든 엔드포인트에서 자동 반영됨.

```java
package com.bikeridediary.domain.user.dto;

import com.bikeridediary.domain.user.entity.UserEntity;
import com.bikeridediary.domain.user.entity.UserRole;

import java.time.LocalDateTime;
import java.util.UUID;

// 사용자 정보 응답 DTO
public record UserResponse(
        // 사용자 ID
        UUID id,

        // OAuth2 제공자 (kakao, google, apple)
        String provider,

        // 닉네임
        String nickname,

        // 이메일
        String email,

        // 프로필 이미지 URL
        String profileImageUrl,

        // 가입 일시
        LocalDateTime createdAt,

        // 사용자 권한 (USER / ADMIN) — 앱의 어드민 화면 노출 제어용
        UserRole role
) {

    // UserEntity로부터 응답 DTO 생성
    public static UserResponse from(UserEntity userEntity) {
        return new UserResponse(
                userEntity.getId(),
                userEntity.getProvider(),
                userEntity.getNickname(),
                userEntity.getEmail(),
                userEntity.getProfileImageUrl(),
                userEntity.getCreatedAt(),
                userEntity.getRole()
        );
    }
}
```

**Jackson 직렬화**: `UserRole` enum은 기본 문자열 직렬화("USER" / "ADMIN")로 나감. 앱은 이 문자열을 파싱.

## 6. PlaceChangeRequestEntity

`brd_be/src/main/java/com/bikeridediary/domain/place/entity/PlaceChangeRequestEntity.java`

```java
package com.bikeridediary.domain.place.entity;

import com.bikeridediary.domain.common.entity.BaseEntity;
import com.bikeridediary.domain.user.entity.UserEntity;
import io.hypersistence.utils.hibernate.type.json.JsonType;
import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.Type;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;
import java.util.Map;
import java.util.UUID;

// 장소 변경 요청 큐 (CREATE / UPDATE_COORDINATES / UPDATE_INFO)
// 어드민이 승인/거절 처리하며, 승인 시 places 테이블에 반영된다.
@Entity
@Table(name = "place_change_requests")
@EntityListeners(AuditingEntityListener.class)
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class PlaceChangeRequestEntity {

    // 요청 ID
    @Id
    @Column(name = "id")
    private UUID id;

    // 요청 종류
    @Enumerated(EnumType.STRING)
    @Column(name = "type", nullable = false, length = 30)
    private PlaceChangeRequestType type;

    // 수정 대상 place (CREATE는 null)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "target_place_id")
    private PlaceEntity targetPlace;

    // 요청자
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "requester_id", nullable = false)
    private UserEntity requester;

    // type별 payload (JSONB)
    @Type(JsonType.class)
    @Column(name = "payload", nullable = false, columnDefinition = "jsonb")
    private Map<String, Object> payload;

    // 상태 (PENDING / APPROVED / REJECTED)
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private PlaceChangeRequestStatus status = PlaceChangeRequestStatus.PENDING;

    // 어드민 검토 노트
    @Column(name = "review_note", columnDefinition = "TEXT")
    private String reviewNote;

    // 검토한 어드민
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reviewed_by")
    private UserEntity reviewer;

    // 검토 시각
    @Column(name = "reviewed_at")
    private LocalDateTime reviewedAt;

    // 생성 시각 (수동 세팅. BaseEntity 안 씀 - updated_at/deleted_at 개념 없어서)
    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;

    // 신규 요청 생성 팩토리
    public static PlaceChangeRequestEntity create(
            PlaceChangeRequestType type,
            PlaceEntity targetPlace,
            UserEntity requester,
            Map<String, Object> payload
    ) {
        PlaceChangeRequestEntity e = new PlaceChangeRequestEntity();
        e.id = UUID.randomUUID();
        e.type = type;
        e.targetPlace = targetPlace;
        e.requester = requester;
        e.payload = payload;
        e.status = PlaceChangeRequestStatus.PENDING;
        e.createdAt = LocalDateTime.now();
        return e;
    }

    // 어드민 승인 처리
    public void approve(UserEntity reviewer, String note) {
        this.status = PlaceChangeRequestStatus.APPROVED;
        this.reviewer = reviewer;
        this.reviewNote = note;
        this.reviewedAt = LocalDateTime.now();
    }

    // 어드민 거절 처리
    public void reject(UserEntity reviewer, String note) {
        this.status = PlaceChangeRequestStatus.REJECTED;
        this.reviewer = reviewer;
        this.reviewNote = note;
        this.reviewedAt = LocalDateTime.now();
    }
}
```

`brd_be/src/main/java/com/bikeridediary/domain/place/entity/PlaceChangeRequestType.java`

```java
package com.bikeridediary.domain.place.entity;

// 장소 변경 요청 종류
public enum PlaceChangeRequestType {
    CREATE,              // 신규 장소 등록
    UPDATE_COORDINATES,  // 좌표 수정
    UPDATE_INFO          // 이름/카테고리 수정
}
```

`brd_be/src/main/java/com/bikeridediary/domain/place/entity/PlaceChangeRequestStatus.java`

```java
package com.bikeridediary.domain.place.entity;

// 장소 변경 요청 상태
public enum PlaceChangeRequestStatus {
    PENDING,   // 승인 대기
    APPROVED,  // 승인 완료 (places 반영됨)
    REJECTED   // 거절
}
```

**의존성 필요**: `build.gradle`에 hibernate-types 추가 (JSONB 매핑용).

```gradle
implementation 'io.hypersistence:hypersistence-utils-hibernate-63:3.7.0'
```

(스프링부트 3.x + Hibernate 6.3+ 기준. 버전은 최신 확인)

## 7. PlaceChangeRequestRepository

`brd_be/src/main/java/com/bikeridediary/domain/place/repository/PlaceChangeRequestRepository.java`

```java
package com.bikeridediary.domain.place.repository;

import com.bikeridediary.domain.place.entity.PlaceChangeRequestEntity;
import com.bikeridediary.domain.place.entity.PlaceChangeRequestStatus;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.UUID;

public interface PlaceChangeRequestRepository extends JpaRepository<PlaceChangeRequestEntity, UUID> {

    // 어드민 목록: 상태별 최신순
    List<PlaceChangeRequestEntity> findByStatusOrderByCreatedAtDesc(PlaceChangeRequestStatus status);

    // 내 요청 목록
    List<PlaceChangeRequestEntity> findByRequester_IdOrderByCreatedAtDesc(UUID requesterId);

    // 특정 place에 대한 PENDING UPDATE 요청 존재 여부 (D8 중복 방지)
    boolean existsByTargetPlace_IdAndStatus(UUID targetPlaceId, PlaceChangeRequestStatus status);

    // 요청자 PENDING 개수 (CREATE 요청 상한 검증용)
    long countByRequester_IdAndStatus(UUID requesterId, PlaceChangeRequestStatus status);
}
```

## 8. DTO들

### 8-1. Request DTO — type별로 분리

`brd_be/src/main/java/com/bikeridediary/domain/place/dto/CreatePlaceRequestPayload.java`

```java
package com.bikeridediary.domain.place.dto;

import java.math.BigDecimal;
import java.util.UUID;

// CREATE 요청 payload (앱이 생성한 clientUuid를 승인 시 places.id로 그대로 사용)
public record CreatePlaceRequestPayload(
        UUID clientUuid,
        String placeName,
        String category,        // FAMOUS/CAFE/RESTAURANT/SERVICE/OTHER
        BigDecimal latitude,
        BigDecimal longitude,
        String address,
        String roadAddress,
        String description,
        String phone,
        String photoUrl
) {}
```

`brd_be/src/main/java/com/bikeridediary/domain/place/dto/UpdateCoordinatesPayload.java`

```java
package com.bikeridediary.domain.place.dto;

import java.math.BigDecimal;

public record UpdateCoordinatesPayload(
        BigDecimal latitude,
        BigDecimal longitude
) {}
```

`brd_be/src/main/java/com/bikeridediary/domain/place/dto/UpdateInfoPayload.java`

```java
package com.bikeridediary.domain.place.dto;

public record UpdateInfoPayload(
        String placeName,
        String category
) {}
```

### 8-2. Controller 입력 DTO (통합)

`brd_be/src/main/java/com/bikeridediary/domain/place/dto/PlaceChangeRequestCreateRequest.java`

```java
package com.bikeridediary.domain.place.dto;

import com.bikeridediary.domain.place.entity.PlaceChangeRequestType;

import java.util.Map;
import java.util.UUID;

// 클라이언트가 요청 생성 시 보내는 DTO.
// payload는 type별로 다른 필드셋을 담는 map 형태로 받고, Service에서 type별 검증.
public record PlaceChangeRequestCreateRequest(
        PlaceChangeRequestType type,
        UUID targetPlaceId,          // CREATE는 null, UPDATE_*는 필수
        Map<String, Object> payload
) {}
```

### 8-3. Response DTO

`brd_be/src/main/java/com/bikeridediary/domain/place/dto/PlaceChangeRequestResponse.java`

```java
package com.bikeridediary.domain.place.dto;

import com.bikeridediary.domain.place.entity.PlaceChangeRequestEntity;
import com.bikeridediary.domain.place.entity.PlaceChangeRequestStatus;
import com.bikeridediary.domain.place.entity.PlaceChangeRequestType;

import java.time.LocalDateTime;
import java.util.Map;
import java.util.UUID;

// 요청자 본인용 응답 (내 요청 목록 / 상세)
public record PlaceChangeRequestResponse(
        UUID id,
        PlaceChangeRequestType type,
        UUID targetPlaceId,
        String targetPlaceName,     // UPDATE_*일 때만 값 (null 가능)
        Map<String, Object> payload,
        PlaceChangeRequestStatus status,
        String reviewNote,
        LocalDateTime reviewedAt,
        LocalDateTime createdAt
) {
    public static PlaceChangeRequestResponse from(PlaceChangeRequestEntity e) {
        return new PlaceChangeRequestResponse(
                e.getId(),
                e.getType(),
                e.getTargetPlace() != null ? e.getTargetPlace().getId() : null,
                e.getTargetPlace() != null ? e.getTargetPlace().getPlaceName() : null,
                e.getPayload(),
                e.getStatus(),
                e.getReviewNote(),
                e.getReviewedAt(),
                e.getCreatedAt()
        );
    }
}
```

`brd_be/src/main/java/com/bikeridediary/domain/place/dto/AdminPlaceChangeRequestResponse.java`

```java
package com.bikeridediary.domain.place.dto;

import com.bikeridediary.domain.place.entity.PlaceChangeRequestEntity;
import com.bikeridediary.domain.place.entity.PlaceChangeRequestStatus;
import com.bikeridediary.domain.place.entity.PlaceChangeRequestType;

import java.time.LocalDateTime;
import java.util.Map;
import java.util.UUID;

// 어드민 목록/상세용 응답 (요청자 정보 포함)
public record AdminPlaceChangeRequestResponse(
        UUID id,
        PlaceChangeRequestType type,
        UUID targetPlaceId,
        String targetPlaceName,
        UUID requesterId,
        String requesterNickname,
        Map<String, Object> payload,
        PlaceChangeRequestStatus status,
        String reviewNote,
        LocalDateTime reviewedAt,
        LocalDateTime createdAt
) {
    public static AdminPlaceChangeRequestResponse from(PlaceChangeRequestEntity e) {
        return new AdminPlaceChangeRequestResponse(
                e.getId(),
                e.getType(),
                e.getTargetPlace() != null ? e.getTargetPlace().getId() : null,
                e.getTargetPlace() != null ? e.getTargetPlace().getPlaceName() : null,
                e.getRequester().getId(),
                e.getRequester().getNickname(),
                e.getPayload(),
                e.getStatus(),
                e.getReviewNote(),
                e.getReviewedAt(),
                e.getCreatedAt()
        );
    }
}
```

### 8-4. 어드민 리뷰 DTO

`brd_be/src/main/java/com/bikeridediary/domain/place/dto/AdminReviewRequest.java`

```java
package com.bikeridediary.domain.place.dto;

// 어드민 승인/거절 입력 (note는 승인엔 선택, 거절엔 사실상 필수 - 앱에서 강제)
public record AdminReviewRequest(
        String note
) {}
```

## 9. PlaceChangeRequestService

`brd_be/src/main/java/com/bikeridediary/domain/place/service/PlaceChangeRequestService.java`

```java
package com.bikeridediary.domain.place.service;

import com.bikeridediary.domain.place.dto.*;
import com.bikeridediary.domain.place.entity.*;
import com.bikeridediary.domain.place.repository.PlaceChangeRequestRepository;
import com.bikeridediary.domain.place.repository.PlaceRepository;
import com.bikeridediary.domain.place_category.entity.PlaceCategoryEntity;
import com.bikeridediary.domain.place_category.repository.PlaceCategoryRepository;
import com.bikeridediary.domain.user.entity.UserEntity;
import com.bikeridediary.domain.user.repository.UserRepository;
import com.bikeridediary.global.exception.BusinessException;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import static com.bikeridediary.global.exception.ErrorCode.*;

@Service
@RequiredArgsConstructor
@Transactional
public class PlaceChangeRequestService {

    // 요청자별 PENDING CREATE 요청 상한 (스팸 방어)
    private static final long MAX_PENDING_PER_USER = 20;

    private final PlaceChangeRequestRepository requestRepository;
    private final PlaceRepository placeRepository;
    private final PlaceCategoryRepository placeCategoryRepository;
    private final UserRepository userRepository;
    private final ObjectMapper objectMapper;

    // ============================================================
    // 일반 유저 API
    // ============================================================

    // 요청 생성. ADMIN이면 같은 트랜잭션 안에서 즉시 자동 승인 → places 반영까지 완료
    public PlaceChangeRequestResponse create(UUID requesterId, PlaceChangeRequestCreateRequest req) {
        UserEntity requester = userRepository.findByIdAndDeletedAtIsNull(requesterId)
                .orElseThrow(() -> new BusinessException(USER_NOT_FOUND));

        boolean isAdmin = requester.getRole() == UserRole.ADMIN;

        // type별 payload 검증
        validatePayload(req.type(), req.payload(), req.targetPlaceId());

        PlaceEntity targetPlace = null;
        if (req.type() != PlaceChangeRequestType.CREATE) {
            // UPDATE_* 는 targetPlaceId 필수 + 실존 확인
            targetPlace = placeRepository.findById(req.targetPlaceId())
                    .orElseThrow(() -> new BusinessException(PLACE_NOT_FOUND));
            // 중복 PENDING 방지 (D8). 어드민은 즉시 승인되어 PENDING을 남기지 않으므로 스킵
            if (!isAdmin && requestRepository.existsByTargetPlace_IdAndStatus(
                    targetPlace.getId(), PlaceChangeRequestStatus.PENDING)) {
                throw new BusinessException(PLACE_REQUEST_ALREADY_PENDING);
            }
        } else if (!isAdmin) {
            // CREATE 요청 상한 (요청자별 PENDING 20건). 어드민은 상한 무관.
            long pending = requestRepository.countByRequester_IdAndStatus(
                    requesterId, PlaceChangeRequestStatus.PENDING);
            if (pending >= MAX_PENDING_PER_USER) {
                throw new BusinessException(PLACE_REQUEST_LIMIT_EXCEEDED);
            }
        }

        PlaceChangeRequestEntity saved = requestRepository.save(
                PlaceChangeRequestEntity.create(req.type(), targetPlace, requester, req.payload())
        );

        // ADMIN이면 즉시 자동 승인 처리. 감사 로그(change_requests)에는 status=APPROVED,
        // reviewer=self로 남음. 앱은 응답 status로 UI 스낵바 분기.
        if (isAdmin) {
            applyToPlaces(saved);
            saved.approve(requester, null);
        }

        return PlaceChangeRequestResponse.from(saved);
    }

    // 내 요청 목록
    @Transactional(readOnly = true)
    public List<PlaceChangeRequestResponse> listMine(UUID requesterId) {
        return requestRepository.findByRequester_IdOrderByCreatedAtDesc(requesterId)
                .stream()
                .map(PlaceChangeRequestResponse::from)
                .toList();
    }

    // ============================================================
    // 어드민 API
    // ============================================================

    // 어드민 목록 (상태 필터, 기본 PENDING)
    @Transactional(readOnly = true)
    public List<AdminPlaceChangeRequestResponse> listForAdmin(PlaceChangeRequestStatus status) {
        PlaceChangeRequestStatus effective = status == null ? PlaceChangeRequestStatus.PENDING : status;
        return requestRepository.findByStatusOrderByCreatedAtDesc(effective)
                .stream()
                .map(AdminPlaceChangeRequestResponse::from)
                .toList();
    }

    // 어드민 승인 - 트랜잭션 내에서 places 반영 후 status 변경 (D6=A)
    public AdminPlaceChangeRequestResponse approve(UUID requestId, UUID reviewerId, AdminReviewRequest review) {
        PlaceChangeRequestEntity request = requestRepository.findById(requestId)
                .orElseThrow(() -> new BusinessException(PLACE_REQUEST_NOT_FOUND));
        if (request.getStatus() != PlaceChangeRequestStatus.PENDING) {
            throw new BusinessException(PLACE_REQUEST_ALREADY_REVIEWED);
        }
        UserEntity reviewer = userRepository.findByIdAndDeletedAtIsNull(reviewerId)
                .orElseThrow(() -> new BusinessException(USER_NOT_FOUND));

        applyToPlaces(request);
        request.approve(reviewer, review == null ? null : review.note());
        return AdminPlaceChangeRequestResponse.from(request);
    }

    // 어드민 거절
    public AdminPlaceChangeRequestResponse reject(UUID requestId, UUID reviewerId, AdminReviewRequest review) {
        PlaceChangeRequestEntity request = requestRepository.findById(requestId)
                .orElseThrow(() -> new BusinessException(PLACE_REQUEST_NOT_FOUND));
        if (request.getStatus() != PlaceChangeRequestStatus.PENDING) {
            throw new BusinessException(PLACE_REQUEST_ALREADY_REVIEWED);
        }
        UserEntity reviewer = userRepository.findByIdAndDeletedAtIsNull(reviewerId)
                .orElseThrow(() -> new BusinessException(USER_NOT_FOUND));
        request.reject(reviewer, review == null ? null : review.note());
        return AdminPlaceChangeRequestResponse.from(request);
    }

    // ============================================================
    // Private
    // ============================================================

    // type별 payload → 실제 places 반영
    private void applyToPlaces(PlaceChangeRequestEntity request) {
        switch (request.getType()) {
            case CREATE -> {
                CreatePlaceRequestPayload p = objectMapper.convertValue(
                        request.getPayload(), CreatePlaceRequestPayload.class);
                PlaceCategoryEntity category = placeCategoryRepository.findById(p.category())
                        .orElseThrow(() -> new BusinessException(PLACE_CATEGORY_NOT_FOUND));
                // 앱이 생성한 clientUuid를 그대로 places.id로 (D9=A)
                PlaceEntity created = PlaceEntity.createWithId(
                        p.clientUuid(),
                        p.placeName(),
                        request.getRequester(),
                        category,
                        p.latitude(),
                        p.longitude(),
                        p.address(),
                        p.roadAddress(),
                        p.description(),
                        p.photoUrl(),
                        p.phone(),
                        null,
                        null
                );
                placeRepository.save(created);
            }
            case UPDATE_COORDINATES -> {
                UpdateCoordinatesPayload p = objectMapper.convertValue(
                        request.getPayload(), UpdateCoordinatesPayload.class);
                request.getTargetPlace().updateCoordinates(
                        new CoordinateUpdateRequest(p.latitude(), p.longitude()));
            }
            case UPDATE_INFO -> {
                UpdateInfoPayload p = objectMapper.convertValue(
                        request.getPayload(), UpdateInfoPayload.class);
                PlaceCategoryEntity category = placeCategoryRepository.findById(p.category())
                        .orElseThrow(() -> new BusinessException(PLACE_CATEGORY_NOT_FOUND));
                request.getTargetPlace().updateInfo(p.placeName(), category);
            }
        }
    }

    // type별 payload 형태 최소 검증
    private void validatePayload(PlaceChangeRequestType type, Map<String, Object> payload, UUID targetPlaceId) {
        if (payload == null) {
            throw new BusinessException(INVALID_INPUT);
        }
        switch (type) {
            case CREATE -> {
                if (targetPlaceId != null) throw new BusinessException(INVALID_INPUT);
                objectMapper.convertValue(payload, CreatePlaceRequestPayload.class); // 필드 매핑 실패 시 예외
            }
            case UPDATE_COORDINATES -> {
                if (targetPlaceId == null) throw new BusinessException(INVALID_INPUT);
                UpdateCoordinatesPayload p = objectMapper.convertValue(payload, UpdateCoordinatesPayload.class);
                if (p.latitude() == null || p.longitude() == null) throw new BusinessException(INVALID_INPUT);
            }
            case UPDATE_INFO -> {
                if (targetPlaceId == null) throw new BusinessException(INVALID_INPUT);
                UpdateInfoPayload p = objectMapper.convertValue(payload, UpdateInfoPayload.class);
                if (p.placeName() == null || p.placeName().isBlank() || p.category() == null) {
                    throw new BusinessException(INVALID_INPUT);
                }
            }
        }
    }
}
```

## 10. PlaceChangeRequestController (일반 유저용)

`brd_be/src/main/java/com/bikeridediary/domain/place/controller/PlaceChangeRequestController.java`

```java
package com.bikeridediary.domain.place.controller;

import com.bikeridediary.domain.place.dto.*;
import com.bikeridediary.domain.place.service.PlaceChangeRequestService;
import com.bikeridediary.global.auth.CustomUserDetails;
import com.bikeridediary.global.response.ApiResponse;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@Tag(name = "장소 변경 요청", description = "유저가 신규 등록/수정을 요청하고 어드민이 승인/거절한다")
@RestController
@RequestMapping("/api/v1/place-change-requests")
@RequiredArgsConstructor
public class PlaceChangeRequestController {

    private final PlaceChangeRequestService service;

    @Operation(summary = "요청 생성")
    @PostMapping
    public ResponseEntity<ApiResponse<PlaceChangeRequestResponse>> create(
            @RequestBody PlaceChangeRequestCreateRequest req,
            @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        return ResponseEntity.ok(ApiResponse.ok(
                service.create(userDetails.getUserId(), req)));
    }

    @Operation(summary = "내 요청 목록")
    @GetMapping("/mine")
    public ResponseEntity<ApiResponse<List<PlaceChangeRequestResponse>>> listMine(
            @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        return ResponseEntity.ok(ApiResponse.ok(
                service.listMine(userDetails.getUserId())));
    }
}
```

## 11. AdminPlaceChangeRequestController

`brd_be/src/main/java/com/bikeridediary/domain/place/controller/AdminPlaceChangeRequestController.java`

```java
package com.bikeridediary.domain.place.controller;

import com.bikeridediary.domain.place.dto.AdminPlaceChangeRequestResponse;
import com.bikeridediary.domain.place.dto.AdminReviewRequest;
import com.bikeridediary.domain.place.entity.PlaceChangeRequestStatus;
import com.bikeridediary.domain.place.service.PlaceChangeRequestService;
import com.bikeridediary.global.auth.CustomUserDetails;
import com.bikeridediary.global.response.ApiResponse;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.lang.Nullable;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.UUID;

@Tag(name = "어드민 - 장소 변경 요청", description = "어드민만 접근 가능")
@RestController
@RequestMapping("/api/v1/admin/place-change-requests")
@RequiredArgsConstructor
@PreAuthorize("hasRole('ADMIN')")
public class AdminPlaceChangeRequestController {

    private final PlaceChangeRequestService service;

    @Operation(summary = "요청 목록 (상태 필터, 기본 PENDING)")
    @GetMapping
    public ResponseEntity<ApiResponse<List<AdminPlaceChangeRequestResponse>>> list(
            @Nullable @RequestParam("status") PlaceChangeRequestStatus status
    ) {
        return ResponseEntity.ok(ApiResponse.ok(service.listForAdmin(status)));
    }

    @Operation(summary = "요청 승인")
    @PostMapping("/{id}/approve")
    public ResponseEntity<ApiResponse<AdminPlaceChangeRequestResponse>> approve(
            @PathVariable("id") UUID id,
            @RequestBody(required = false) AdminReviewRequest review,
            @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        return ResponseEntity.ok(ApiResponse.ok(
                service.approve(id, userDetails.getUserId(), review)));
    }

    @Operation(summary = "요청 거절")
    @PostMapping("/{id}/reject")
    public ResponseEntity<ApiResponse<AdminPlaceChangeRequestResponse>> reject(
            @PathVariable("id") UUID id,
            @RequestBody(required = false) AdminReviewRequest review,
            @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        return ResponseEntity.ok(ApiResponse.ok(
                service.reject(id, userDetails.getUserId(), review)));
    }
}
```

**주의**: `@PreAuthorize`가 동작하려면 `SecurityConfig`에 `@EnableMethodSecurity`가 반드시 필요. 섹션 14 SecurityConfig 참조.

## 12. PlaceController — 삭제할 엔드포인트

`brd_be/src/main/java/com/bikeridediary/domain/place/controller/PlaceController.java`

**제거할 메서드 3개**:

- `updateCoordinates` (라인 34-40)
- `updateInfo` (라인 43-49)
- `addNewPlace` (라인 61-68)

유지: `list`, `searchExternal`

## 13. PlaceService — 삭제/좁힐 메서드

`PlaceService.java`에서 다음 3개 메서드는 삭제 (Controller에서 안 부르니 컴파일 에러도 안 남):

- `updateCoordinates(UUID, CoordinateUpdateRequest)` (라인 52-60)
- `updateInfo(UUID, PlaceInfoUpdateRequest)` (라인 62-74)
- `addNewPlace(PlaceInsertRequest, UUID)` (라인 87-137)

기존 중복 검증 로직(`findNearbyByName`, `haversineMeters`, `LAT_DELTA_100M`)은 승인 시점에도 유용하므로 `PlaceChangeRequestService`로 이관 or `PlaceService`에 `checkDuplicate(name, lat, lng)` public 메서드로 남겨 승인 로직에서 호출. (선택 - 초안은 이관 생략, 중복 방어는 클라이언트 UX 안내로 커버)

**미사용 DTO 삭제**: `PlaceInsertRequest`, `CoordinateUpdateRequest`, `PlaceInfoUpdateRequest`.
단, `PlaceEntity.updateCoordinates(CoordinateUpdateRequest)`가 `CoordinateUpdateRequest`를 파라미터로 받으므로, 이걸 지우려면 시그니처를 `updateCoordinates(BigDecimal lat, BigDecimal lng)`로 바꾸거나 request record는 유지하되 Controller에서만 안 씀.

**권장**: DTO는 지우지 말고 Controller에서 노출만 제거. 향후 재활용 여지.

## 14. SecurityConfig 수정

`brd_be/src/main/java/com/bikeridediary/global/config/SecurityConfig.java`

> ⚠️ **필수: `@EnableMethodSecurity` 어노테이션 추가** ⚠️
>
> `AdminPlaceChangeRequestController`가 `@PreAuthorize("hasRole('ADMIN')")`로 어드민 전용 접근을 강제하는데,
> **`@EnableMethodSecurity`가 없으면 `@PreAuthorize`가 완전히 무시되어 어드민 API가 인증된 모든 유저에게 열립니다.**
>
> 즉, 일반 USER 계정 JWT로도 어드민 승인/거절 API를 호출할 수 있게 되는 심각한 보안 홀. 배포 전 반드시 확인.
>
> **검증 방법**: 일반 USER 계정 JWT로 `GET /api/v1/admin/place-change-requests` → **403** 이어야 함. 200 나오면 어노테이션 누락.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // ← 필수. @PreAuthorize 활성화
@RequiredArgsConstructor
public class SecurityConfig {

    private final String[] PERMIT_ALL_ENDPOINTS = {
            "/api/v1/weathers/**",
            "/api/v1/auth/**",
            "/api/v1/stations/**",
            // "/api/v1/places/**",  ← 제거 (GET만 아래로 이동)
            "/swagger-ui/**",
            "/api-docs/**",
            "/logos/**",
    };

    private final String[] GET_PERMIT_ALL_ENDPOINTS = {
            "/api/v1/courses",
            "/api/v1/bike-models/**",
            "/api/v1/places",              // 신규 - GET /places
            "/api/v1/places/search-external", // 신규 - GET /places/search-external
    };

    // ... 기존 필터 체인 그대로 ...
}
```

- `/api/v1/place-change-requests/**` — 인증 필수 (anyRequest().authenticated() 로 커버됨)
- `/api/v1/admin/**` — `@PreAuthorize("hasRole('ADMIN')")`로 클래스 레벨에서 커버 (SecurityConfig에 별도 URL 패턴 설정 불요, **단 `@EnableMethodSecurity` 필수**)

## 15. ErrorCode 추가

`brd_be/src/main/java/com/bikeridediary/global/exception/ErrorCode.java`

"장소" 섹션 하단 (`PLACE_ALREADY_EXIST` 아래)에 추가:

```java
    // 장소 변경 요청
    PLACE_REQUEST_NOT_FOUND(HttpStatus.NOT_FOUND, "PLACE_REQUEST_NOT_FOUND", "요청을 찾을 수 없습니다"),
    PLACE_REQUEST_ALREADY_PENDING(HttpStatus.CONFLICT, "PLACE_REQUEST_ALREADY_PENDING", "해당 장소에 대한 대기 중인 요청이 이미 존재합니다"),
    PLACE_REQUEST_ALREADY_REVIEWED(HttpStatus.CONFLICT, "PLACE_REQUEST_ALREADY_REVIEWED", "이미 처리된 요청입니다"),
    PLACE_REQUEST_LIMIT_EXCEEDED(HttpStatus.TOO_MANY_REQUESTS, "PLACE_REQUEST_LIMIT_EXCEEDED", "대기 중인 요청 수가 상한을 초과했습니다"),
```

문법 유의: 마지막 enum 상수의 세미콜론 위치 조정 필요.

## 16. 어드민 지정 방법 (초기 세팅)

DB 직접 UPDATE:

```sql
-- 본인 계정을 ADMIN으로
UPDATE users SET role = 'ADMIN' WHERE email = 'jyl9311@gmail.com';

-- 확인
SELECT id, email, nickname, role FROM users WHERE role = 'ADMIN';
```

**서버 인가**: role은 매 요청 `loadUserByUsername`에서 DB 조회하므로, 지정 즉시 다음 요청부터 반영됨. JWT 재발급 불필요.

**앱 UI**: 앱은 로그인 시점의 UserResponse.role을 캐시해서 어드민 메뉴 노출을 판단. 따라서 어드민 지정 후 앱에서 role을 인식하려면 **로그아웃 → 재로그인** 필요 (또는 GET /users/me 재호출로 유저 정보 새로고침). 앱 문서 섹션 12 참조.

## 주의사항

1. **`@EnableMethodSecurity` 추가 확인** — `@PreAuthorize`가 무시되면 어드민 API가 그냥 열림 (모든 인증 유저 접근 가능). Postman으로 반드시 검증: USER 계정 JWT로 `/api/v1/admin/place-change-requests` GET → 403이어야 함.
2. **JSONB 매핑 의존성** — `hypersistence-utils-hibernate-63` 못 쓰면 `String payload`로 저장하고 ObjectMapper로 직접 (de)serialize. 그러나 어드민이 JSONB로 쿼리 필터링할 여지도 있어 JSONB 권장.
3. **payload 무결성** — 유저가 clientUuid를 임의로 조작 가능 (예: 이미 존재하는 places.id로 CREATE 요청). 승인 시 `placeRepository.save`가 upsert처럼 동작하므로 기존 place가 payload 값으로 덮어쓸 위험. **`applyToPlaces` CREATE 분기에 `placeRepository.existsById(p.clientUuid())` 체크 추가 권장** (존재 시 승인 거부, 요청 자동 REJECTED로 처리).
4. **트랜잭션 격리** — 어드민 두 명이 동시에 같은 요청 승인 시 후자는 `PLACE_REQUEST_ALREADY_REVIEWED` 예외. DB 유니크 인덱스는 target 기준이라 CREATE는 unique constraint가 없어서, 애플리케이션 레이어 status 체크만으로 방어. 필요 시 optimistic locking 추가.
5. **payload에서 clientUuid 파싱** — Jackson이 `Map<String, Object>` 안 UUID 문자열을 자동으로 UUID로 변환 못 함. `objectMapper.convertValue(payload, CreatePlaceRequestPayload.class)`는 String → UUID 변환 지원 (JsonFormat 기본). 만약 실패하면 `@JsonDeserialize` 커스텀 필요.
6. **기존 PATCH 엔드포인트 사용 앱 버전** — 이 변경 이후 이전 앱 버전은 PATCH 못 씀. 앱과 백엔드 배포 시점 조율.
7. **UserResponse.role 반영 위치 확인** — `UserResponse.from()` 뿐 아니라 다른 곳에서 UserResponse를 직접 `new`로 생성하는 코드가 있으면 role 필드 채워야 컴파일 통과. IDE usages로 `new UserResponse(` 검색.
8. **어드민 자동 승인과 유저 PENDING 요청 충돌** — 유저가 place X에 UPDATE_COORDINATES 요청(PENDING)을 넣어둔 상태에서 어드민이 앱에서 같은 place X를 직접 수정하면, 어드민 요청은 즉시 승인되지만 유저의 PENDING 요청은 그대로 남아 나중에 어드민이 승인하면 유저 payload가 어드민 변경분을 덮어씀. 완벽 방어하려면 어드민 자동 승인 시 같은 target의 유저 PENDING 요청을 auto-REJECT("어드민이 직접 수정함") 처리. 이번 스코프에선 어드민 수 매우 적어 실무상 드문 케이스라 후속 개선.
9. **어드민 자동 승인 payload 무결성** — 어드민 UI에서 CREATE 요청 시에도 clientUuid를 보내는데 이미 존재하는 UUID면 place가 덮어써짐(주의사항 3과 동일 리스크). 어드민이라도 검증 우회하지 말 것 — `applyToPlaces`의 existsById 체크는 role 무관하게 적용.
