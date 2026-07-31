# FuelingServiceTest 복구 가이드

> Phase 3 sync 리팩터로 서비스/리포지토리 시그니처가 바뀌면서 테스트가 컴파일 실패 상태.
> 아래 4가지만 반영하면 복구된다.

---

## 1. 삭제된 리포지토리 메서드

**old**: `fuelingRepository.findByBikeEntityIdAndDeletedAtIsNullOrderByFuelingDateDescMileageAtFuelingDesc(bikeId)` → `List<FuelingEntity>`
**new**: `fuelingRepository.findByBikeEntityIdAndDeletedAtIsNull(bikeId, Pageable)` → `Page<FuelingEntity>`

서비스가 private `loadAllFuelings(bikeId)`로 감싸는데, 이 헬퍼는 위의 페이징 메서드에 `Pageable.unpaged(Sort.by(DESC, "fuelingDate", "mileageAtFueling"))`을 넘긴다. 따라서 목킹도 페이징 메서드로 바꿔야 함.

**test stub 변경 패턴**

```java
// old
when(fuelingRepository.findByBikeEntityIdAndDeletedAtIsNullOrderByFuelingDateDescMileageAtFuelingDesc(bikeId))
        .thenReturn(List.of(testFueling));

// new (import: org.springframework.data.domain.PageImpl, Pageable, Sort)
when(fuelingRepository.findByBikeEntityIdAndDeletedAtIsNull(eq(bikeId), any(Pageable.class)))
        .thenReturn(new PageImpl<>(List.of(testFueling)));
```

`Pageable.unpaged(Sort)`은 정적으로 만들기 번거로우니 `any(Pageable.class)`로 넘어가는 게 실용적. Mockito 인자 검증까지 원하면 `argThat(p -> p.isUnpaged())`.

---

## 2. `getFuelings` 시그니처 변경

**old**: `List<FuelingResponse> getFuelings(UUID bikeId, UUID userId)`
**new**: `PageResponse<FuelingResponse> getFuelings(UUID bikeId, UUID userId, Pageable pageable)`

내부 조회도 `findByBikeEntityIdAndDeletedAtIsNull(bikeId, pageable)`로 바뀜.

**test 변경 패턴** (`getFuelings_Success`)

```java
@Test
@DisplayName("getFuelings - 성공 (페이징)")
void getFuelings_Success() {
    Pageable pageable = PageRequest.of(0, 20);
    when(bikeRepository.findByIdAndDeletedAtIsNull(bikeId))
            .thenReturn(Optional.of(testBike));
    when(fuelingRepository.findByBikeEntityIdAndDeletedAtIsNull(bikeId, pageable))
            .thenReturn(new PageImpl<>(List.of(testFueling), pageable, 1));

    PageResponse<FuelingResponse> result = fuelingService.getFuelings(bikeId, userId, pageable);

    assertThat(result.content()).hasSize(1);
    assertThat(result.content().get(0).fuelType()).isEqualTo(FuelType.PREMIUM);
}
```

`PageResponse`의 실제 필드명은 `content()` — 이 프로젝트의 record 정의에 맞춤 (필요하면 소스 확인).

에러 케이스(`getFuelings_BikeNotFound`, `getFuelings_AccessDenied`)는 호출부만 `getFuelings(bikeId, userId, pageable)`로 인자 하나 더 넘기면 됨.

---

## 3. `updateBikeFuelEfficiency` 신규 호출 → create/update/delete 목킹 확장

Phase 3에서 create/update/delete가 종료 직전에 `updateBikeFuelEfficiency(bike)`를 부른다. 이 헬퍼는 내부에서 `loadAllFuelings(bikeId)`를 호출한다 → 1번의 페이징 메서드 스텁이 있어야 NPE 안 남.

따라서 create/update/delete 테스트도 아래 스텁을 추가해야 함:

```java
when(fuelingRepository.findByBikeEntityIdAndDeletedAtIsNull(eq(bikeId), any(Pageable.class)))
        .thenReturn(new PageImpl<>(List.of(testFueling))); // 또는 빈 리스트
```

기존 테스트에서 `findByBikeEntityIdAndDeletedAtIsNullOrderByFuelingDateDescMileageAtFuelingDesc(bikeId)`를 스텁하던 자리 그대로 위 코드로 교체.

부수적으로 `BikeEntity.updateFuelEfficiency(latest, average)`가 호출되는데, 이건 엔티티 내부 setter라 별도 목킹 불필요.

---

## 4. `UserRepository` 필드 추가

서비스 필드에 `private final UserRepository userRepository;`가 추가됨 (sync용). `@InjectMocks` 조합에 필요하니 테스트 클래스에 mock 추가:

```java
@Mock
private UserRepository userRepository;
```

`FuelingServiceTest`가 현재 다루는 스코프(CRUD/stats)만 보면 실제 스텁은 없어도 되지만, mock 필드는 있어야 InjectMocks가 생성자를 만족한다.

---

## 5. `FuelingStatsResponse` 필드 개수 확인

record 필드가 6개 (`totalCount, totalFuelAmount, totalCost, averageFuelEfficiency, latestFuelEfficiency, averagePricePerLiter`). 기존 테스트가 4개만 검증하고 있는데 컴파일은 통과. 새로 추가할 검증이 필요하면:

```java
assertThat(result.latestFuelEfficiency()).isEqualByComparingTo(new BigDecimal("41.67"));
assertThat(result.averagePricePerLiter()).isEqualTo(1800.0);
```

---

## 요약 체크리스트

- [ ] 모든 `findByBikeEntityIdAndDeletedAtIsNullOrderByFuelingDateDescMileageAtFuelingDesc(bikeId)` → `findByBikeEntityIdAndDeletedAtIsNull(bikeId, any(Pageable.class))` + `PageImpl<>` 반환
- [ ] `getFuelings*` 테스트 시그니처 `Pageable` 인자 추가, 반환 타입 `PageResponse` 검증
- [ ] `createFueling*` / `updateFueling*` / `deleteFueling*` 성공 케이스에 위 페이징 스텁 하나씩 추가 (efficiency 재계산 경로)
- [ ] `@Mock private UserRepository userRepository;` 필드 추가
- [ ] 필요 시 `PageRequest`, `PageImpl`, `Pageable`, `PageResponse` import 추가

컴파일 통과 후 `./gradlew test --tests FuelingServiceTest`로 회귀 확인.

## sync 테스트는 별도 스코프

이 가이드는 기존 CRUD/stats 복구만 다룸. `sync()` 메서드는 로직이 상당해 (LWW, deletedAt early return, save managed 엔티티 반환값 사용, efficiency 재계산) 별도 테스트 케이스가 필요하다면 후속 사이클에서 추가 권장.
