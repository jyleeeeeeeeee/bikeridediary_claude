# 주유(Fueling) sync 엔드포인트 백엔드 가이드

Phase 3 로컬 우선 이전에 필요한 백엔드 작업. 바이크 sync(`BikeService.sync`, `POST /bikes/sync`)를 그대로 참조. 여기선 주유에 맞춘 차이점만 자세히.

앱 사이드는 이미 완성 (커밋 예정). 백엔드가 두 엔드포인트만 추가되면 온라인/오프라인 무관하게 주유 CRUD가 로컬 우선으로 동작함.

## 필요 엔드포인트

| Method | Path | 용도 |
|---|---|---|
| POST | `/api/v1/fuelings/sync` | 클라이언트 로컬 pending 레코드를 서버에 upsert |
| GET | `/api/v1/fuelings/my` | 로그인 직후 최초 pull (로컬 비어 있을 때) |

## 1. FuelingEntity 수정

**현재 문제**: `@GeneratedValue(strategy = GenerationType.UUID)`가 있어 서버가 UUID를 자동 생성. sync는 클라이언트 UUID를 그대로 수용해야 함.

**변경**: 바이크 리팩터와 동일 패턴. `@GeneratedValue` 제거 + `createWithId(UUID id, ...)` 팩토리 추가. 기존 `create()`도 UUID 명시하도록 수정.

```java
@Id
@JdbcTypeCode(SqlTypes.UUID)
private UUID id;   // @GeneratedValue 제거

public static FuelingEntity create(
        BikeEntity bikeEntity, LocalDate fuelingDate, ...
) {
    return createWithId(UUID.randomUUID(), bikeEntity, fuelingDate, ...);
}

public static FuelingEntity createWithId(
        UUID id, BikeEntity bikeEntity, LocalDate fuelingDate, Long mileageAtFueling,
        BigDecimal fuelAmount, Long pricePerLiter, Long totalCost, FuelType fuelType,
        String memo, String stationName
) {
    FuelingEntity entity = new FuelingEntity();
    entity.id = id;
    entity.bikeEntity = bikeEntity;
    entity.fuelingDate = fuelingDate;
    entity.mileageAtFueling = mileageAtFueling;
    entity.fuelAmount = fuelAmount;
    entity.pricePerLiter = pricePerLiter;
    entity.totalCost = totalCost;
    entity.fuelType = fuelType;
    entity.memo = memo;
    entity.stationName = stationName;
    return entity;
}

```

**타임스탬프 처리**: `syncTimestamps` 같은 커스텀 메서드는 **일반적으로 불필요**. JPA auditing(`@CreatedDate`/`@LastModifiedDate`)이 저장 시 서버 시각으로 자동 세팅. 클라이언트의 `createdAt`/`updatedAt`은 LWW 비교용으로만 쓰이고 DB엔 서버 시각이 들어감. 이는 바이크 sync(`BikeService.sync`)와 동일 방식.

참고: 이미 fueling에 `syncTimestamps`를 구현했다면 `@LastModifiedDate`가 save 시점에 덮어써서 실질적으로 클라이언트 updatedAt은 보존되지 않음. 다만 새 레코드의 `createdAt`은 `@CreatedDate`가 null일 때만 세팅되므로 syncTimestamps로 넣은 값이 유지됨. 문제는 아니지만 코드 없이도 잘 동작하므로 정리 권장.

## 2. FuelingSyncRequest DTO

`BikeSyncRequest`와 같은 record 패턴. 앱이 보내는 body:

```java
package com.bikeridediary.domain.fueling.dto;

import jakarta.validation.constraints.NotNull;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.UUID;

public record FuelingSyncRequest(
        @NotNull UUID id,
        @NotNull UUID bikeId,
        @NotNull LocalDate fuelingDate,
        @NotNull Long mileageAtFueling,
        @NotNull BigDecimal fuelAmount,
        Long pricePerLiter,
        Long totalCost,
        @NotNull FuelType fuelType,
        String memo,
        String stationName,
        @NotNull LocalDateTime createdAt,
        @NotNull LocalDateTime updatedAt,
        LocalDateTime deletedAt
) {}
```

## 3. FuelingController에 두 엔드포인트 추가

```java
// POST /api/v1/fuelings/sync
@PostMapping("/sync")
public ResponseEntity<ApiResponse<FuelingResponse>> syncFueling(
        @AuthenticationPrincipal CustomUserDetails userDetails,
        @Valid @RequestBody FuelingSyncRequest request
) {
    UUID userId = userDetails.getUserId();
    FuelingResponse fueling = fuelingService.sync(userId, request);
    return ResponseEntity.ok(ApiResponse.ok(fueling));
}

// GET /api/v1/fuelings/my — 초기 pull용, 유저 전체 주유 기록
@GetMapping("/my")
public ResponseEntity<ApiResponse<List<FuelingResponse>>> getMyFuelings(
        @AuthenticationPrincipal CustomUserDetails userDetails
) {
    UUID userId = userDetails.getUserId();
    return ResponseEntity.ok(ApiResponse.ok(fuelingService.getMyFuelings(userId)));
}
```

## 4. FuelingService.sync() — LWW + efficiency 계산

`BikeService.sync`와 로직 동일 + **efficiency 재계산**이 추가됨. 주유 기록은 이전 기록의 mileage와 현재 주유량으로 efficiency가 결정되므로, sync로 새 레코드 upsert 후 efficiency를 다시 계산해야 함.

```java
@Transactional
public FuelingResponse sync(UUID userId, FuelingSyncRequest req) {
    UserEntity user = userRepository.findByIdAndDeletedAtIsNull(userId)
            .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));

    BikeEntity bike = bikeRepository.findById(req.bikeId())
            .orElseThrow(() -> new BusinessException(ErrorCode.BIKE_NOT_FOUND));
    if (!bike.isOwner(userId)) {
        throw new BusinessException(ErrorCode.FUELING_ACCESS_DENIED);
    }

    Optional<FuelingEntity> existingOpt = fuelingRepository.findById(req.id());
    FuelingEntity target;

    if (existingOpt.isPresent()) {
        FuelingEntity existing = existingOpt.get();
        if (!existing.isOwner(userId)) {
            throw new BusinessException(ErrorCode.FUELING_ACCESS_DENIED);
        }

        // soft delete
        if (req.deletedAt() != null) {
            if (!existing.isDeleted()) existing.delete();  // BaseEntity.delete()
            return FuelingResponse.from(existing);
        }

        // LWW: 요청이 더 최신일 때만 반영
        if (req.updatedAt().isAfter(existing.getUpdatedAt())) {
            existing.update(
                    req.fuelingDate(), req.mileageAtFueling(), req.fuelAmount(),
                    req.pricePerLiter(), req.totalCost(), req.fuelType(),
                    req.memo(), req.stationName()
            );
            target = existing;
        } else {
            // 서버가 더 최신 — 반영 안 함
            return FuelingResponse.from(existing);
        }
    } else {
        // 신규 upsert — auditing이 createdAt/updatedAt을 서버 시각으로 세팅
        target = FuelingEntity.createWithId(
                req.id(), bike,
                req.fuelingDate(), req.mileageAtFueling(), req.fuelAmount(),
                req.pricePerLiter(), req.totalCost(), req.fuelType(),
                req.memo(), req.stationName()
        );
        fuelingRepository.save(target);
    }

    // efficiency 재계산 — 이전 주유 기록 기준
    recalculateFuelEfficiency(target);

    return FuelingResponse.from(target);
}

private void recalculateFuelEfficiency(FuelingEntity current) {
    // 같은 바이크의 이 시점 이전(fuelingDate/mileage) 가장 최근 활성 기록 조회
    Optional<FuelingEntity> prevOpt = fuelingRepository
            .findTopByBikeEntityAndFuelingDateBeforeAndDeletedAtIsNullOrderByFuelingDateDesc(
                    current.getBikeEntity(), current.getFuelingDate());
    if (prevOpt.isEmpty()) {
        current.setFuelEfficiency(null);
        return;
    }
    long deltaKm = current.getMileageAtFueling() - prevOpt.get().getMileageAtFueling();
    if (deltaKm <= 0 || current.getFuelAmount().signum() <= 0) {
        current.setFuelEfficiency(null);
        return;
    }
    BigDecimal efficiency = BigDecimal.valueOf(deltaKm)
            .divide(current.getFuelAmount(), 2, RoundingMode.HALF_UP);
    current.setFuelEfficiency(efficiency);
}

public List<FuelingResponse> getMyFuelings(UUID userId) {
    return fuelingRepository.findByBikeEntity_UserEntity_IdAndDeletedAtIsNullOrderByFuelingDateDesc(userId)
            .stream().map(FuelingResponse::from).toList();
}
```

## 5. FuelingRepository 쿼리 추가

```java
// 특정 바이크의 fuelingDate 이전 가장 최근 기록 (efficiency 재계산용)
Optional<FuelingEntity> findTopByBikeEntityAndFuelingDateBeforeAndDeletedAtIsNullOrderByFuelingDateDesc(
        BikeEntity bikeEntity, LocalDate fuelingDate);

// 유저의 모든 활성 주유 기록 (getMyFuelings용)
@Query("SELECT f FROM FuelingEntity f " +
       "JOIN FETCH f.bikeEntity b " +
       "WHERE b.userEntity.id = :userId AND f.deletedAt IS NULL " +
       "ORDER BY f.fuelingDate DESC")
List<FuelingEntity> findByBikeEntity_UserEntity_IdAndDeletedAtIsNullOrderByFuelingDateDesc(UUID userId);
```

## 6. FuelingResponse.from(FuelingEntity)

이미 존재하면 그대로. 없다면 static factory 추가. 필드 매핑은 기존 DTO 그대로.

## 7. SecurityConfig — 별도 permitAll 불필요

`/api/v1/fuelings/**`는 이미 인증 필요 경로일 것. `/sync`, `/my` 모두 같은 규칙 적용.

## 검증 방법

1. 앱 재시작 → 로그인 → 바이크 등록(이미 sync 됨) → 주유 등록
2. 서버 로그(p6spy)에서 `INSERT INTO fuelings ... ON CONFLICT ...` 또는 `MERGE`류 로그 확인. 실제로는 `SELECT` + `INSERT` 두 문장 (JPA)
3. Wi-Fi 끄고 주유 등록 → 저장 성공(로컬만) → 앱 재시작해도 유지 → Wi-Fi 켜면 자동 sync
4. 두 기기 동시 편집 시나리오는 "한 기기 전제" 원칙상 스코프 아님

## 미결 / 알아둘 것

- **타임스탬프 LWW skew**: JPA auditing이 서버 시각으로 세팅. 클라이언트 로컬 시각과 skew 있으면 LWW 오판 가능(극단적으론 클라이언트 편집이 서버 값보다 오래된 것으로 판단). 실무엔 대개 문제 안 되지만 알아둘 것.
- 초기 pull의 GET `/fuelings/my`가 유저 데이터를 다 한 번에 반환 → 대용량 유저면 부담. 페이징 필요해지면 향후 확장. MVP는 이걸로 충분.

## 수정 이력

- **2026-07-28**: `syncTimestamps` 관련 부분 정리 — auditing 그대로 사용하는 방향으로 통일. 이미 fueling에 syncTimestamps 구현했다면 그대로 두거나 정리해도 무방(auditing이 어차피 덮어씀). 정비 sync 가이드(`maintenance-sync-backend.md`)와 동일 방식.
