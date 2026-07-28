# 정비(Maintenance) sync 엔드포인트 백엔드 가이드

Phase 3 로컬 우선 이전에 필요한 백엔드 작업. 정비 기록 + 정비 스케줄 두 리소스, 정비 기록엔 **이미지 업로드 통합**이 특이 포인트.

> **수정 이력 (2026-07-28)**: 초판을 실제 코드베이스와 맞춰 재작성. `syncTimestamps` 불필요(JPA auditing 그대로 사용), `MaintenanceResponse.from`/`MaintenanceScheduleResponse.from` 실제 시그니처 반영, 존재하지 않던 헬퍼 참조 제거, 기존 `responseMaintenance`/`buildResponse` 헬퍼 재사용.

## 필요 엔드포인트 (4개)

| Method | Path | 용도 |
|---|---|---|
| POST | `/api/v1/maintenances/sync` | 정비 기록 upsert (멀티파트: JSON + 새 이미지 파일) |
| GET | `/api/v1/maintenances/my` | 초기 pull — 유저 전체 정비 기록 |
| POST | `/api/v1/maintenance-schedules/sync` | 정비 스케줄 upsert (JSON only) |
| GET | `/api/v1/maintenance-schedules/my` | 초기 pull — 유저 전체 스케줄 |

## 1. Entity 리팩터

### MaintenanceEntity — `setImageUrls` public setter 추가

sync 흐름에서 이미지 업로드/삭제 후 최종 URL 목록만 갱신하기 위해 필요. `update()`는 imageUrls를 인자로 받지만 sync는 이미지 처리 순서상 update 이후에 최종 URL이 확정됨.

```java
public void setImageUrls(String imageUrls) {
    this.imageUrls = imageUrls;
}
```

`@GeneratedValue` 이미 없음(현재 코드는 `@Id @JdbcTypeCode(SqlTypes.UUID) private UUID id;`만 있음), `createWithId(UUID id, ...)` 이미 존재. 추가 리팩터 없음.

### MaintenanceScheduleEntity — `@GeneratedValue` 제거 + `createWithId` 추가

```java
// Before
@Id
@GeneratedValue(strategy = GenerationType.UUID)
@JdbcTypeCode(SqlTypes.UUID)
private UUID id;

// After
@Id
@JdbcTypeCode(SqlTypes.UUID)
private UUID id;

// 기존 create()는 UUID.randomUUID()로 위임
public static MaintenanceScheduleEntity create(
        BikeEntity bikeEntity, MaintenanceType maintenanceType,
        Long intervalKm, Integer intervalMonths
) {
    return createWithId(UUID.randomUUID(), bikeEntity, maintenanceType, intervalKm, intervalMonths);
}

// sync 전용 팩토리
public static MaintenanceScheduleEntity createWithId(
        UUID id, BikeEntity bikeEntity, MaintenanceType maintenanceType,
        Long intervalKm, Integer intervalMonths
) {
    MaintenanceScheduleEntity entity = new MaintenanceScheduleEntity();
    entity.id = id;
    entity.bikeEntity = bikeEntity;
    entity.maintenanceType = maintenanceType;
    entity.intervalKm = intervalKm;
    entity.intervalMonths = intervalMonths;
    return entity;
}
```

### 타임스탬프 — 별도 처리 불필요

**JPA auditing(@CreatedDate/@LastModifiedDate)이 저장 시 자동으로 세팅**하므로 클라이언트가 보낸 createdAt/updatedAt은 LWW 비교용으로만 쓰이고 실제 DB 값은 서버 시각이 됩니다. `syncTimestamps` 같은 커스텀 메서드 필요 없음(바이크 sync도 같은 방식).

## 2. Sync Request DTO

### MaintenanceSyncRequest (멀티파트 body 안의 JSON)

```java
public record MaintenanceSyncRequest(
        @NotNull UUID id,
        @NotNull UUID bikeId,
        @NotNull MaintenanceType maintenanceType,
        @NotNull LocalDate maintenanceDate,
        @NotNull Long mileageAtMaintenance,
        Long cost,
        String description,
        Long nextDueKm,
        LocalDate nextDueDate,
        /// 이미 서버에 저장된 URL 목록. 여기서 빠진 URL은 삭제 대상.
        List<String> existingImageUrls,
        @NotNull LocalDateTime createdAt,
        @NotNull LocalDateTime updatedAt,
        LocalDateTime deletedAt
) {}
```

### MaintenanceScheduleSyncRequest

```java
public record MaintenanceScheduleSyncRequest(
        @NotNull UUID id,
        @NotNull UUID bikeId,
        @NotNull MaintenanceType maintenanceType,
        Long intervalKm,
        // 엔티티는 Integer. 앱 sync JSON은 정수 그대로 오므로 문제 없음.
        Integer intervalMonths,
        @NotNull LocalDateTime createdAt,
        @NotNull LocalDateTime updatedAt,
        LocalDateTime deletedAt
) {}
```

**주의**: `intervalMonths`는 반드시 `Integer` (Long 아님). `MaintenanceScheduleEntity.intervalMonths`가 `Integer`이고 `update(Long, Integer)` 시그니처라 타입 불일치 시 컴파일 에러.

## 3. 정비 기록 sync 컨트롤러 (멀티파트)

```java
@PostMapping(value = "/sync", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<ApiResponse<MaintenanceResponse>> syncMaintenance(
        @AuthenticationPrincipal CustomUserDetails userDetails,
        @RequestPart("data") @Valid MaintenanceSyncRequest request,
        @RequestPart(value = "images", required = false) List<MultipartFile> newImages
) {
    UUID userId = userDetails.getUserId();
    MaintenanceResponse response = maintenanceService.sync(userId, request, newImages);
    return ResponseEntity.ok(ApiResponse.ok(response));
}

@GetMapping("/my")
public ResponseEntity<ApiResponse<List<MaintenanceResponse>>> getMyMaintenances(
        @AuthenticationPrincipal CustomUserDetails userDetails
) {
    UUID userId = userDetails.getUserId();
    return ResponseEntity.ok(ApiResponse.ok(maintenanceService.getMyMaintenances(userId)));
}
```

**IOException throws 불필요** — sync 서비스가 내부에서 RuntimeException으로 래핑(기존 `addImageUrls` 헬퍼와 동일 패턴).

## 4. MaintenanceService.sync() — LWW + 이미지 처리

기존 헬퍼 재사용: `responseMaintenance(entity)`, `parseStringToList(json)`, `toJson(list)`, `imageStorageService.upload(MultipartFile, String userId)`.

```java
@Transactional
public MaintenanceResponse sync(UUID userId, MaintenanceSyncRequest req,
                                List<MultipartFile> newImages) {
    userRepository.findByIdAndDeletedAtIsNull(userId)
            .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));

    BikeEntity bike = bikeRepository.findById(req.bikeId())
            .orElseThrow(() -> new BusinessException(ErrorCode.BIKE_NOT_FOUND));
    if (!bike.isOwner(userId)) {
        throw new BusinessException(ErrorCode.MAINTENANCE_ACCESS_DENIED);
    }

    Optional<MaintenanceEntity> existingOpt = maintenanceRepository.findById(req.id());
    MaintenanceEntity target;
    List<String> oldServerUrls = List.of();

    if (existingOpt.isPresent()) {
        MaintenanceEntity existing = existingOpt.get();
        if (!existing.isOwner(userId)) {
            throw new BusinessException(ErrorCode.MAINTENANCE_ACCESS_DENIED);
        }
        if (req.deletedAt() != null) {
            if (!existing.isDeleted()) existing.delete();
            // 소프트 삭제 시 이미지 파일도 스토리지에서 정리
            parseStringToList(existing.getImageUrls()).forEach(imageStorageService::delete);
            return responseMaintenance(existing);
        }
        if (!req.updatedAt().isAfter(existing.getUpdatedAt())) {
            // 서버가 더 최신 — 반영 안 함
            return responseMaintenance(existing);
        }
        oldServerUrls = parseStringToList(existing.getImageUrls());
        // update()는 imageUrls도 필수 인자 — 아래서 setImageUrls로 최종 확정하므로 임시로 기존값 전달
        existing.update(
                req.maintenanceType(), req.maintenanceDate(),
                req.mileageAtMaintenance(), req.cost(),
                req.description(), req.nextDueKm(), req.nextDueDate(),
                existing.getImageUrls()
        );
        target = existing;
    } else {
        target = MaintenanceEntity.createWithId(
                req.id(), bike,
                req.maintenanceType(), req.maintenanceDate(),
                req.mileageAtMaintenance(), req.cost(),
                req.description(), req.nextDueKm(), req.nextDueDate(),
                "[]"
        );
        maintenanceRepository.save(target);
    }

    // 이미지: existingImageUrls에 없는 old URL은 파일 삭제
    List<String> keepUrls = req.existingImageUrls() == null
            ? List.of() : req.existingImageUrls();
    for (String oldUrl : oldServerUrls) {
        if (!keepUrls.contains(oldUrl)) {
            imageStorageService.delete(oldUrl);
        }
    }

    // 새 이미지 업로드 → 최종 URL 목록 확정 후 setImageUrls
    List<String> finalUrls = new ArrayList<>(keepUrls);
    if (newImages != null) {
        for (MultipartFile file : newImages) {
            try {
                finalUrls.add(imageStorageService.upload(file, userId.toString()));
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    }
    target.setImageUrls(toJson(finalUrls));

    return responseMaintenance(target);
}

public List<MaintenanceResponse> getMyMaintenances(UUID userId) {
    return maintenanceRepository.findMyMaintenances(userId)
            .stream().map(this::responseMaintenance).toList();
}
```

**주의 포인트**:
- `MaintenanceResponse.from(entity)` **금지** — 실제 시그니처는 `from(entity, List<String> imageUrls)`. `responseMaintenance(entity)` 헬퍼가 파싱 포함해서 처리해줌.
- `existing.update(...)`는 imageUrls를 8번째 인자로 반드시 받음. sync에선 임시로 `existing.getImageUrls()` 전달하고, 이미지 처리 후 `setImageUrls(toJson(finalUrls))`로 최종 확정.
- `imageStorageService.upload(file, userId)` **금지** — 실제 시그니처는 `String userId`. `userId.toString()` 필수.
- `UserEntity user = userRepository...` **불필요** — 검증만 하고 참조 안 하므로 리턴값 안 받음.

## 5. MaintenanceRepository — `findMyMaintenances` 쿼리 추가

파생 메서드 이름(`findByBike_UserEntity_Id...`)은 필드명 관례가 애매해 `@Query`로 명시.

```java
// 초기 pull용 — 유저의 모든 활성 정비 기록
@Query(
        "SELECT m FROM MaintenanceEntity m " +
        "JOIN FETCH m.bike b " +
        "WHERE b.userEntity.id = :userId AND m.deletedAt IS NULL " +
        "ORDER BY m.maintenanceDate DESC")
List<MaintenanceEntity> findMyMaintenances(UUID userId);
```

**주의**: `MaintenanceEntity`의 바이크 필드명은 `bike` (아니라 `bikeEntity`). `MaintenanceScheduleEntity`는 `bikeEntity` — 명명 불일치 있음. JPQL 작성 시 헷갈리지 말 것.

## 6. 정비 스케줄 sync 컨트롤러 (JSON only)

```java
@PostMapping("/sync")
public ResponseEntity<ApiResponse<MaintenanceScheduleResponse>> syncSchedule(
        @AuthenticationPrincipal CustomUserDetails userDetails,
        @Valid @RequestBody MaintenanceScheduleSyncRequest request
) {
    UUID userId = userDetails.getUserId();
    MaintenanceScheduleResponse response = scheduleService.sync(userId, request);
    return ResponseEntity.ok(ApiResponse.ok(response));
}

@GetMapping("/my")
public ResponseEntity<ApiResponse<List<MaintenanceScheduleResponse>>> getMySchedules(
        @AuthenticationPrincipal CustomUserDetails userDetails
) {
    UUID userId = userDetails.getUserId();
    return ResponseEntity.ok(ApiResponse.ok(scheduleService.getMySchedules(userId)));
}
```

## 7. MaintenanceScheduleService.sync()

기존 `buildResponse(schedule, bikeId, currentMileage)` 헬퍼 재사용 — 마지막 정비 기록 조회 + overdue 판정을 이미 처리하고 있음.

```java
private final MaintenanceScheduleRepository scheduleRepository;
private final MaintenanceRepository maintenanceRepository;
private final BikeRepository bikeRepository;
private final UserRepository userRepository;  // 신규 DI

@Transactional
public MaintenanceScheduleResponse sync(UUID userId, MaintenanceScheduleSyncRequest req) {
    userRepository.findByIdAndDeletedAtIsNull(userId)
            .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));

    BikeEntity bike = bikeRepository.findById(req.bikeId())
            .orElseThrow(() -> new BusinessException(ErrorCode.BIKE_NOT_FOUND));
    if (!bike.isOwner(userId)) {
        throw new BusinessException(ErrorCode.MAINTENANCE_SCHEDULE_ACCESS_DENIED);
    }

    Optional<MaintenanceScheduleEntity> existingOpt = scheduleRepository.findById(req.id());
    MaintenanceScheduleEntity target;

    if (existingOpt.isPresent()) {
        MaintenanceScheduleEntity existing = existingOpt.get();
        if (!existing.isOwner(userId)) {
            throw new BusinessException(ErrorCode.MAINTENANCE_SCHEDULE_ACCESS_DENIED);
        }
        if (req.deletedAt() != null) {
            if (!existing.isDeleted()) existing.delete();
            return buildResponse(existing, bike.getId(), bike.getTotalMileageKm());
        }
        if (!req.updatedAt().isAfter(existing.getUpdatedAt())) {
            return buildResponse(existing, bike.getId(), bike.getTotalMileageKm());
        }
        existing.update(req.intervalKm(), req.intervalMonths());
        target = existing;
    } else {
        target = MaintenanceScheduleEntity.createWithId(
                req.id(), bike, req.maintenanceType(),
                req.intervalKm(), req.intervalMonths()
        );
        scheduleRepository.save(target);
    }

    return buildResponse(target, bike.getId(), bike.getTotalMileageKm());
}

public List<MaintenanceScheduleResponse> getMySchedules(UUID userId) {
    return scheduleRepository.findMySchedules(userId).stream()
            .map(s -> buildResponse(
                    s,
                    s.getBikeEntity().getId(),
                    s.getBikeEntity().getTotalMileageKm()))
            .toList();
}
```

**주의 포인트**:
- `MaintenanceScheduleResponse.from(entity, boolean overdue)` **존재 안 함**. 실제는 `from(schedule, currentMileage, lastMileage, lastDate)`. 직접 호출 말고 `buildResponse` 헬퍼 통해서만 사용.
- `computeOverdue` 같은 신규 헬퍼 **작성 불필요**. `buildResponse`가 내부에서 이미 처리.
- `syncTimestamps` **호출 안 함**. auditing 그대로.

## 8. MaintenanceScheduleRepository — `findMySchedules` 추가

```java
@Query(
        "SELECT s FROM MaintenanceScheduleEntity s " +
        "JOIN FETCH s.bikeEntity b " +
        "WHERE b.userEntity.id = :userId AND s.deletedAt IS NULL " +
        "ORDER BY s.createdAt DESC")
List<MaintenanceScheduleEntity> findMySchedules(UUID userId);
```

## 9. SecurityConfig — 변경 불필요

`/api/v1/maintenances/**`, `/api/v1/maintenance-schedules/**`는 이미 인증 필요. `/sync`, `/my`도 자동 적용.

## 10. ErrorCode — 기존 재사용

- `MAINTENANCE_ACCESS_DENIED`, `MAINTENANCE_SCHEDULE_ACCESS_DENIED` — 이미 존재
- 추가 신규 필요 없음

## 검증

1. 앱에서 이미지 포함 정비 등록 → 로컬 저장 → sync fire
2. 서버 로그: `POST /maintenances/sync`, 멀티파트 파싱, `imageStorageService.upload` 호출, DB INSERT
3. 응답의 `imageUrls`가 전부 서버 URL → 앱이 로컬 값 교체
4. Wi-Fi 끄고 정비 등록 → 로컬 저장 + ☁️ 배지 (오프라인 시도 스킵 적용됨) → Wi-Fi 켜면 SyncEngine이 자동 트리거 → 배지 사라짐
5. 이미지 편집(삭제): 앱이 `existingImageUrls`에서 뺀 URL을 서버가 파일 삭제하는지 확인 (스토리지 폴더 직접 확인)

## 미결 / 주의

- **참조 무결성**: bikeId가 서버에 없는 경우 400 → FAILED 마킹 → 다음 사이클에 바이크가 성공하면 정비도 성공. SyncEngine이 바이크 → 주유 → 정비 순 등록되어 있어 자연스럽게 해결.
- **이미지 부분 실패**: 5장 중 3장 성공하고 실패면 `RuntimeException` → 트랜잭션 롤백. 앱은 다음 사이클에 5장 다시 시도.
- **큰 이미지**: dio timeout, spring `spring.servlet.multipart.max-file-size`(기본 1MB) 확인.
- **초기 pull `GET /my`**: 대용량 유저는 페이징 필요. MVP는 전체 반환. 확장 시 `?after=timestamp`.
- **`updatedAt` LWW 정확도**: JPA auditing은 서버 시각으로 세팅. 클라이언트 로컬 시각과 skew 있으면 LWW 오판 가능 (극단적으론 클라이언트 편집이 서버 값보다 오래된 것으로 판단). 실무엔 대개 문제 안 되지만 알아둘 것.
- **필드명 관례 불일치**: `MaintenanceEntity.bike` vs `MaintenanceScheduleEntity.bikeEntity`. JPQL/파생 메서드 작성 시 헷갈리지 말 것.
