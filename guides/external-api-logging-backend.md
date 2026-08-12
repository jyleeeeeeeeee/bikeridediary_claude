# 외부 API 호출 로깅 — 백엔드 구현 가이드

작성일: 2026-08-12
담당: backend-dev
근거 계획: `plans/external-api-logging.md` (D1~D7 확정)
DB 스키마: `guides/external-api-logging-schema.md`

## 사전 확인 결과

- `build.gradle`에 `spring-boot-starter-aop` **없음** → 추가 필요
- `@EnableScheduling`은 `BikeRideDiaryApplication`에 이미 있음
- `hypersistence-utils-hibernate-63:3.15.4` 이미 있음 → `@Type(JsonType.class)` 바로 사용
- `SecurityConfig`에 `@EnableMethodSecurity` 이미 활성화 → `@PreAuthorize` 바로 사용
- `/api/v1/admin/**`는 `anyRequest().authenticated()`로 인증 필수 → role 검증은 컨트롤러의 `@PreAuthorize`

## 로깅 대상 클라이언트 (실제 코드 확인 결과)

| 클라이언트 | 경로 | 대상 메서드 | ApiName |
|-----------|------|------------|---------|
| NaverMapsClient | `infra/naver/maps/NaverMapsClient.java` | `search(String)` | NAVER_GEOCODING |
| NaverMapsClient | 위와 동일 | `reverseGeocode(BigDecimal, BigDecimal)` | NAVER_REVERSE_GEOCODING |
| NaverMapsClient | 위와 동일 | `directions(...)` | NAVER_DIRECTIONS |
| NaverSearchClient | `infra/naver/search/NaverSearchClient.java` | `search(String, int)` | NAVER_SEARCH |
| OpinetClient | `infra/opinet/OpinetClient.java` | `areaCode2()`, `areaCode4()` | OPINET |
| WeatherService | `domain/weather/service/WeatherService.java` | `getCurrentConditions(double, double)` | OPENWEATHER |
| KakaoLocalClient | 미존재 (`_disabled_features` 확인 필요) | — | 어노테이션 보류 |

**NaverGeocodingClient는 별도 클래스 없음** — `NaverMapsClient.search()`가 Geocoding 역할.

## 파일 생성/수정 목록

| # | 경로 | 신규/수정 |
|---|------|----------|
| 1 | `build.gradle` | 수정 (AOP starter) |
| 2 | `global/logging/ApiNames.java` | 신규 — String 상수 클래스 (enum 아님) |
| 3 | `global/logging/LogExternalApi.java` | 신규 |
| 4 | `global/logging/SensitiveParamsFilter.java` | 신규 |
| 5 | `global/logging/ExternalApiLoggingAspect.java` | 신규 (핵심) |
| 6 | `domain/apicalllog/entity/ApiCallLogEntity.java` | 신규 |
| 7 | `domain/apicalllog/repository/ApiCallLogRepository.java` | 신규 |
| 8 | `domain/apicalllog/dto/ApiCallLogResponse.java` | 신규 |
| 9 | `domain/apicalllog/service/ApiCallLogService.java` | 신규 (REQUIRES_NEW 핵심) |
| 10 | `domain/apicalllog/controller/AdminApiCallLogController.java` | 신규 |
| 11 | `domain/apicalllog/scheduler/ApiCallLogRetentionScheduler.java` | 신규 |
| 12 | `infra/naver/maps/NaverMapsClient.java` | 수정 (어노테이션 3개) |
| 13 | `infra/naver/search/NaverSearchClient.java` | 수정 (어노테이션 1개) |
| 14 | `infra/opinet/OpinetClient.java` | 수정 (어노테이션 2개) |
| 15 | `domain/weather/service/WeatherService.java` | 수정 (어노테이션 1개) |

## ApiName은 enum 대신 String 상수 (2026-08-12 결정)

**이유**:
- 새 외부 API 추가 시 enum 값 추가·재컴파일 불필요 (어노테이션에 임의 String 넣으면 됨)
- DTO 직렬화/JPA 매핑 단순 (Enumerated 처리 불필요)
- 어드민 API 필터 파라미터도 String으로 자연스레 처리

**컴파일 타임 안전 유지**: 자주 쓰는 값은 `ApiNames` 클래스의 `public static final String` 상수로 노출. 오타 방지 + IDE 자동완성. 상수에 없는 값도 어노테이션에 String literal로 삽입 가능.

## 코드 스니펫

### 1. build.gradle

```groovy
// dependencies 블록 내 Spring Boot core 섹션 아래에 추가
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

`@EnableAspectJAutoProxy`는 Spring Boot AOP starter 있으면 자동. 별도 설정 불필요.

### 2. ApiNames.java (String 상수 클래스)

`src/main/java/com/bikeridediary/global/logging/ApiNames.java`

```java
package com.bikeridediary.global.logging;

/**
 * 외부 API 식별자 String 상수.
 * enum이 아닌 이유: 새 API 추가 시 재컴파일 없이 어노테이션에 String literal로 삽입 가능.
 * api_call_logs.api_name 컬럼 저장 값이므로 한 번 정한 이름 변경 금지 (기존 로그와 불일치).
 * 어노테이션에 상수를 참조하지 않고 String literal("NAVER_DIRECTIONS")을 직접 써도 무방.
 */
public final class ApiNames {

    private ApiNames() {}

    public static final String NAVER_GEOCODING = "NAVER_GEOCODING";
    public static final String NAVER_REVERSE_GEOCODING = "NAVER_REVERSE_GEOCODING";
    public static final String NAVER_DIRECTIONS = "NAVER_DIRECTIONS";
    public static final String NAVER_SEARCH = "NAVER_SEARCH";
    public static final String KAKAO_LOCAL = "KAKAO_LOCAL";
    public static final String OPINET = "OPINET";
    public static final String OPENWEATHER = "OPENWEATHER";
}
```

### 3. LogExternalApi.java

`src/main/java/com/bikeridediary/global/logging/LogExternalApi.java`

```java
package com.bikeridediary.global.logging;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 외부 API 호출 로깅 어노테이션.
 * 이 어노테이션이 붙은 메서드는 ExternalApiLoggingAspect가 인터셉트하여
 * api_call_logs 테이블에 호출 이력을 기록한다.
 *
 * apiName은 String — ApiNames 상수 사용 권장 (오타 방지), 신규 API는 literal도 허용.
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogExternalApi {

    /** 호출하는 외부 API 식별자 (예: ApiNames.NAVER_DIRECTIONS 또는 "NAVER_DIRECTIONS") */
    String apiName();

    /**
     * 요청 파라미터 저장 여부.
     * true(기본)이면 파라미터를 Map으로 변환 후 민감 필드 제거하여 저장.
     */
    boolean logParams() default true;
}
```

### 4. SensitiveParamsFilter.java

`src/main/java/com/bikeridediary/global/logging/SensitiveParamsFilter.java`

```java
package com.bikeridediary.global.logging;

import java.util.HashMap;
import java.util.Map;
import java.util.Set;

/**
 * 요청 파라미터 Map에서 민감 필드 제거.
 * 헤더에 담긴 API Key/Secret은 파라미터에 넣지 않으므로 방어적 성격이지만,
 * 향후 누군가 실수로 파라미터에 시크릿을 담는 경우를 대비.
 */
public final class SensitiveParamsFilter {

    private SensitiveParamsFilter() {}

    /** 제거 대상 키 (소문자 비교) */
    private static final Set<String> BLOCKED_KEYS = Set.of(
            "apikey", "api_key", "api-key",
            "clientid", "client_id", "client-id",
            "clientsecret", "client_secret", "client-secret",
            "appid",   // OpenWeather
            "x-ncp-apigw-api-key", "x-ncp-apigw-api-key-id"
    );

    public static Map<String, Object> filter(Map<String, Object> params) {
        if (params == null || params.isEmpty()) return Map.of();
        Map<String, Object> result = new HashMap<>(params.size());
        for (Map.Entry<String, Object> entry : params.entrySet()) {
            String keyLower = entry.getKey() == null ? "" : entry.getKey().toLowerCase();
            if (!BLOCKED_KEYS.contains(keyLower)) {
                result.put(entry.getKey(), entry.getValue());
            }
        }
        return result;
    }
}
```

### 5. ExternalApiLoggingAspect.java (핵심)

`src/main/java/com/bikeridediary/global/logging/ExternalApiLoggingAspect.java`

전략:
- `@Around`로 시작 시각 기록 → `proceed()` → 성공/실패 모두 로그 저장
- 로그 저장은 `ApiCallLogService.saveLog()`를 호출, 해당 메서드가 `REQUIRES_NEW`로 별도 트랜잭션
- AOP 자체에서 try/catch로 감싸지 않고, `saveLog()` 내부에서 예외 catch 후 warn log만
- endpoint는 메서드 첫 String 파라미터에서 URL 추출 시도, 없으면 `클래스명.메서드명`
- userId는 `SecurityContextHolder`에서 추출, 미인증이면 null

```java
package com.bikeridediary.global.logging;

import com.bikeridediary.domain.apicalllog.service.ApiCallLogService;
import com.bikeridediary.global.auth.CustomUserDetails;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.core.annotation.Order;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;

import java.lang.reflect.Parameter;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.UUID;

@Slf4j
@Aspect
@Component
@Order(10)  // 트랜잭션 AOP(Integer.MAX_VALUE)보다 먼저 실행
@RequiredArgsConstructor
public class ExternalApiLoggingAspect {

    private final ApiCallLogService apiCallLogService;

    @Around("@annotation(logExternalApi)")
    public Object logApiCall(ProceedingJoinPoint joinPoint, LogExternalApi logExternalApi)
            throws Throwable {

        long startMs = System.currentTimeMillis();
        UUID userId = extractUserId();
        String endpoint = extractEndpoint(joinPoint);
        String httpMethod = "GET";  // 확장 필요 시 어노테이션 파라미터로

        Throwable caughtException = null;
        Integer statusCode = null;

        try {
            Object result = joinPoint.proceed();
            statusCode = 200;
            return result;
        } catch (Throwable ex) {
            caughtException = ex;
            throw ex;
        } finally {
            long responseTimeMs = System.currentTimeMillis() - startMs;

            Map<String, Object> params = null;
            if (logExternalApi.logParams()) {
                params = extractParams(joinPoint);
            }

            String errorMessage = caughtException != null
                    ? caughtException.getClass().getSimpleName() + ": " + caughtException.getMessage()
                    : null;

            apiCallLogService.saveLog(
                    userId, logExternalApi.apiName(),
                    endpoint, httpMethod,
                    statusCode, (int) responseTimeMs,
                    params, errorMessage
            );
        }
    }

    private UUID extractUserId() {
        try {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth == null || !auth.isAuthenticated()) return null;
            Object principal = auth.getPrincipal();
            if (principal instanceof CustomUserDetails userDetails) {
                return userDetails.getUserId();
            }
        } catch (Exception e) {
            log.debug("[ExternalApiLogging] userId 추출 실패 (무시): {}", e.getMessage());
        }
        return null;
    }

    private String extractEndpoint(ProceedingJoinPoint joinPoint) {
        try {
            Object[] args = joinPoint.getArgs();
            if (args != null) {
                for (Object arg : args) {
                    if (arg instanceof String str && str.startsWith("http")) {
                        int queryStart = str.indexOf('?');
                        return queryStart >= 0 ? str.substring(0, queryStart) : str;
                    }
                }
            }
        } catch (Exception e) {
            log.debug("[ExternalApiLogging] endpoint 추출 실패 (무시): {}", e.getMessage());
        }
        MethodSignature sig = (MethodSignature) joinPoint.getSignature();
        return sig.getDeclaringType().getSimpleName() + "." + sig.getName();
    }

    private Map<String, Object> extractParams(ProceedingJoinPoint joinPoint) {
        try {
            MethodSignature sig = (MethodSignature) joinPoint.getSignature();
            Parameter[] parameters = sig.getMethod().getParameters();
            Object[] args = joinPoint.getArgs();

            Map<String, Object> raw = new LinkedHashMap<>();
            for (int i = 0; i < parameters.length; i++) {
                Object value = (args != null && i < args.length) ? args[i] : null;
                raw.put(parameters[i].getName(), value != null ? value.toString() : null);
            }
            return SensitiveParamsFilter.filter(raw);
        } catch (Exception e) {
            log.debug("[ExternalApiLogging] params 추출 실패 (무시): {}", e.getMessage());
            return Map.of();
        }
    }
}
```

**주의**: `NaverMapsClient` 공개 메서드는 URL 문자열을 파라미터로 받지 않고 내부에서 조립. 이 경우 `extractEndpoint`는 폴백인 `NaverMapsClient.directions` 형태로 저장. URL 조립 결과를 저장하고 싶다면 어노테이션에 `endpoint()` 속성 추가 or client 시그니처 리팩터 필요. 현재 목적(사용량/에러 모니터링)에는 `클래스명.메서드명`으로도 충분.

### 6. ApiCallLogEntity.java

`src/main/java/com/bikeridediary/domain/apicalllog/entity/ApiCallLogEntity.java`

```java
package com.bikeridediary.domain.apicalllog.entity;

import com.bikeridediary.global.logging.ApiNames;
import io.hypersistence.utils.hibernate.type.json.JsonType;
import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.Generated;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.annotations.Type;
import org.hibernate.generator.EventType;
import org.hibernate.type.SqlTypes;

import java.time.LocalDateTime;
import java.util.Map;
import java.util.UUID;

// 외부 API 호출 이력 (사용량 모니터링·이상 탐지·유저별 차단 근거용)
// BaseEntity 상속 안 함 — updated_at/deleted_at 불필요한 불변 이력 로그
@Entity
@Table(name = "api_call_logs")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class ApiCallLogEntity {

    // 조회용 친숙 번호 (자동 증가, DB DEFAULT nextval)
    @Column(name = "no", insertable = false, updatable = false)
    @Generated(event = EventType.INSERT)
    private Long no;

    // 로그 고유 ID (UUID)
    @Id
    @Column(name = "id")
    @JdbcTypeCode(SqlTypes.UUID)
    private UUID id;

    // 호출한 유저 ID (미인증 호출 시 null. FK → users ON DELETE SET NULL)
    @Column(name = "user_id")
    @JdbcTypeCode(SqlTypes.UUID)
    private UUID userId;

    // 외부 API 식별자 (String — enum 아님. ApiNames 상수 참고)
    @Column(name = "api_name", nullable = false, length = 50)
    private String apiName;

    // 실제 호출 URL path (쿼리스트링 제외. URL 파악 안 되면 클래스명.메서드명)
    @Column(name = "endpoint", nullable = false, length = 200)
    private String endpoint;

    // HTTP 메서드
    @Column(name = "http_method", nullable = false, length = 10)
    private String httpMethod;

    // 응답 HTTP status (예외 발생 시 null)
    @Column(name = "status_code")
    private Integer statusCode;

    // 응답 소요 시간 (밀리초)
    @Column(name = "response_time_ms", nullable = false)
    private Integer responseTimeMs;

    // 마스킹된 요청 파라미터 (JSONB. API Key/Secret 제거됨)
    @Type(JsonType.class)
    @Column(name = "request_params", columnDefinition = "jsonb")
    private Map<String, Object> requestParams;

    // 예외 발생 시 에러 메시지
    @Column(name = "error_message", columnDefinition = "TEXT")
    private String errorMessage;

    // 호출 시각
    @Column(name = "called_at", nullable = false)
    private LocalDateTime calledAt;

    public static ApiCallLogEntity create(
            UUID userId,
            String apiName,
            String endpoint,
            String httpMethod,
            Integer statusCode,
            int responseTimeMs,
            Map<String, Object> requestParams,
            String errorMessage
    ) {
        ApiCallLogEntity e = new ApiCallLogEntity();
        e.id = UUID.randomUUID();
        e.userId = userId;
        e.apiName = apiName;
        e.endpoint = truncate(endpoint, 200);
        e.httpMethod = httpMethod;
        e.statusCode = statusCode;
        e.responseTimeMs = responseTimeMs;
        e.requestParams = requestParams;
        e.errorMessage = errorMessage;
        e.calledAt = LocalDateTime.now();
        return e;
    }

    private static String truncate(String value, int maxLength) {
        if (value == null) return "unknown";
        return value.length() <= maxLength ? value : value.substring(0, maxLength);
    }
}
```

**JPA 주의**: `id`를 `UUID.randomUUID()`로 세팅하면 `isNew()=false`로 판단해 `merge()` 경로. Service에서 반드시 `repository.save(entity)`의 반환값 사용 (Phase 3 교훈).

### 7. ApiCallLogRepository.java

`src/main/java/com/bikeridediary/domain/apicalllog/repository/ApiCallLogRepository.java`

```java
package com.bikeridediary.domain.apicalllog.repository;

import com.bikeridediary.domain.apicalllog.entity.ApiCallLogEntity;
import com.bikeridediary.global.logging.ApiNames;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.LocalDateTime;
import java.util.UUID;

public interface ApiCallLogRepository extends JpaRepository<ApiCallLogEntity, UUID> {

    @Query("""
            SELECT a FROM ApiCallLogEntity a
            WHERE (:apiName IS NULL OR a.apiName = :apiName)
              AND (:userId IS NULL OR a.userId = :userId)
              AND (:from IS NULL OR a.calledAt >= :from)
              AND (:to IS NULL OR a.calledAt <= :to)
            ORDER BY a.calledAt DESC
            """)
    Page<ApiCallLogEntity> search(
            @Param("apiName") String apiName,
            @Param("userId") UUID userId,
            @Param("from") LocalDateTime from,
            @Param("to") LocalDateTime to,
            Pageable pageable
    );

    @Modifying(clearAutomatically = true)
    @Query("DELETE FROM ApiCallLogEntity a WHERE a.calledAt < :threshold")
    int deleteByCalledAtBefore(@Param("threshold") LocalDateTime threshold);
}
```

### 8. ApiCallLogResponse.java

```java
package com.bikeridediary.domain.apicalllog.dto;

import com.bikeridediary.domain.apicalllog.entity.ApiCallLogEntity;
import com.bikeridediary.global.logging.ApiNames;

import java.time.LocalDateTime;
import java.util.Map;
import java.util.UUID;

public record ApiCallLogResponse(
        Long no,
        UUID id,
        UUID userId,
        String apiName,
        String endpoint,
        String httpMethod,
        Integer statusCode,
        Integer responseTimeMs,
        Map<String, Object> requestParams,
        String errorMessage,
        LocalDateTime calledAt
) {
    public static ApiCallLogResponse from(ApiCallLogEntity entity) {
        return new ApiCallLogResponse(
                entity.getNo(), entity.getId(), entity.getUserId(),
                entity.getApiName(), entity.getEndpoint(), entity.getHttpMethod(),
                entity.getStatusCode(), entity.getResponseTimeMs(),
                entity.getRequestParams(), entity.getErrorMessage(),
                entity.getCalledAt()
        );
    }
}
```

### 9. ApiCallLogService.java

```java
package com.bikeridediary.domain.apicalllog.service;

import com.bikeridediary.domain.apicalllog.dto.ApiCallLogResponse;
import com.bikeridediary.domain.apicalllog.entity.ApiCallLogEntity;
import com.bikeridediary.domain.apicalllog.repository.ApiCallLogRepository;
import com.bikeridediary.global.logging.ApiNames;
import com.bikeridediary.global.response.PageResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.Map;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class ApiCallLogService {

    private final ApiCallLogRepository apiCallLogRepository;

    /**
     * 외부 API 호출 로그 저장.
     * REQUIRES_NEW: 호출자의 트랜잭션과 완전히 분리.
     * 로그 저장 실패 시 원 API 호출 결과에 영향 없도록 예외 catch → warn log만.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveLog(
            UUID userId,
            String apiName,
            String endpoint,
            String httpMethod,
            Integer statusCode,
            int responseTimeMs,
            Map<String, Object> requestParams,
            String errorMessage
    ) {
        try {
            ApiCallLogEntity entity = ApiCallLogEntity.create(
                    userId, apiName, endpoint, httpMethod,
                    statusCode, responseTimeMs, requestParams, errorMessage
            );
            apiCallLogRepository.save(entity);
        } catch (Exception e) {
            log.warn("[ApiCallLog] 로그 저장 실패 (무시): apiName={}, endpoint={}, error={}",
                    apiName, endpoint, e.getMessage());
        }
    }

    @Transactional(readOnly = true)
    public PageResponse<ApiCallLogResponse> search(
            String apiName,
            UUID userId,
            LocalDateTime from,
            LocalDateTime to,
            Pageable pageable
    ) {
        return PageResponse.of(
                apiCallLogRepository.search(apiName, userId, from, to, pageable),
                ApiCallLogResponse::from
        );
    }
}
```

### 10. AdminApiCallLogController.java

```java
package com.bikeridediary.domain.apicalllog.controller;

import com.bikeridediary.domain.apicalllog.dto.ApiCallLogResponse;
import com.bikeridediary.domain.apicalllog.service.ApiCallLogService;
import com.bikeridediary.global.logging.ApiNames;
import com.bikeridediary.global.response.ApiResponse;
import com.bikeridediary.global.response.PageResponse;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.http.ResponseEntity;
import org.springframework.lang.Nullable;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.time.LocalDateTime;
import java.util.UUID;

@Tag(name = "어드민 - 외부 API 호출 로그", description = "어드민만 접근 가능. 외부 API 사용량 모니터링.")
@RestController
@RequestMapping("/api/v1/admin/api-logs")
@RequiredArgsConstructor
@PreAuthorize("hasRole('ADMIN')")
public class AdminApiCallLogController {

    private final ApiCallLogService apiCallLogService;

    @Operation(summary = "외부 API 호출 로그 목록 (필터, 페이징)")
    @GetMapping
    public ResponseEntity<ApiResponse<PageResponse<ApiCallLogResponse>>> list(
            @Nullable @RequestParam(required = false) String apiName,
            @Nullable @RequestParam(required = false) UUID userId,
            @Nullable @RequestParam(required = false)
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime from,
            @Nullable @RequestParam(required = false)
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime to,
            @PageableDefault(size = 20, sort = "calledAt", direction = Sort.Direction.DESC)
            Pageable pageable
    ) {
        return ResponseEntity.ok(ApiResponse.ok(
                apiCallLogService.search(apiName, userId, from, to, pageable)
        ));
    }
}
```

### 11. ApiCallLogRetentionScheduler.java

```java
package com.bikeridediary.domain.apicalllog.scheduler;

import com.bikeridediary.domain.apicalllog.repository.ApiCallLogRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Slf4j
@Component
@RequiredArgsConstructor
public class ApiCallLogRetentionScheduler {

    private static final int RETENTION_DAYS = 90;

    private final ApiCallLogRepository apiCallLogRepository;

    /**
     * 매일 새벽 3시 실행.
     * cron: 초 분 시 일 월 요일
     */
    @Scheduled(cron = "0 0 3 * * *")
    @Transactional
    public void deleteExpiredLogs() {
        LocalDateTime threshold = LocalDateTime.now().minusDays(RETENTION_DAYS);
        int deletedCount = apiCallLogRepository.deleteByCalledAtBefore(threshold);
        log.info("[ApiCallLogRetention] {}일 경과 로그 {}건 삭제 (기준: {})",
                RETENTION_DAYS, deletedCount, threshold);
    }
}
```

### 12. 각 클라이언트 어노테이션 부착

#### 12-1. NaverMapsClient.java

import 추가:
```java
import com.bikeridediary.global.logging.ApiNames;
import com.bikeridediary.global.logging.LogExternalApi;
```

각 메서드 위:
```java
@LogExternalApi(apiName = ApiName.NAVER_GEOCODING)
public NaverGeocodeResponse search(String query) { ... }

@LogExternalApi(apiName = ApiName.NAVER_REVERSE_GEOCODING)
public NaverReverseGeocodeResponse reverseGeocode(BigDecimal latitude, BigDecimal longitude) { ... }

@LogExternalApi(apiName = ApiName.NAVER_DIRECTIONS)
public NaverDirectionsResponse directions(...) { ... }
```

#### 12-2. NaverSearchClient.java

```java
@LogExternalApi(apiName = ApiName.NAVER_SEARCH)
public NaverLocalSearchResponse search(String query, int start) { ... }
```

#### 12-3. OpinetClient.java

```java
@LogExternalApi(apiName = ApiName.OPINET)
public ResponseEntity<Map> areaCode2() { ... }

@LogExternalApi(apiName = ApiName.OPINET)
public void areaCode4() { ... }
```

#### 12-4. WeatherService.java

```java
@LogExternalApi(apiName = ApiName.OPENWEATHER)
public Object getCurrentConditions(double lat, double lng) { ... }
```

`WeatherService`는 내부에서 예외를 삼키고 null 반환하고 있음. AOP 입장에서 예외가 던져지지 않으므로 `statusCode=200`, `errorMessage=null`로 로그 남음. 정확한 실패 추적을 원하면 catch 블록 리팩터 필요.

## 구현 순서 권장

1. `build.gradle`에 AOP starter 추가 후 `./gradlew compileJava` 확인
2. DDL 반영 (schema.sql — dba 가이드 참조)
3. `ApiName` → `LogExternalApi` → `SensitiveParamsFilter` 순 생성
4. `ApiCallLogEntity` → `Repository` → `Response` → `Service` 순
5. `ExternalApiLoggingAspect` 생성 (Service 의존)
6. `AdminApiCallLogController` + `Scheduler` 생성
7. 클라이언트 4곳 어노테이션 부착
8. 서버 기동 후 Swagger로 Directions 호출 → `api_call_logs` DB 조회 확인
9. 어드민 계정으로 `GET /api/v1/admin/api-logs` 호출 확인

## 체크리스트

- [ ] `ApiCallLogEntity` 필드 전체 한글 주석 (CLAUDE.md 컨벤션)
- [ ] `BaseEntity` 상속 안 함 (의도적)
- [ ] `@JdbcTypeCode(SqlTypes.UUID)` on `id`, `userId`
- [ ] `no` 컬럼 `insertable=false, updatable=false` + `@Generated(event=EventType.INSERT)`
- [ ] `saveLog()`에 `Propagation.REQUIRES_NEW` 명시
- [ ] `saveLog()` 내부 try/catch — 예외 삼키고 warn log만
- [ ] `search()`에 `readOnly = true`
- [ ] 스케줄러 `@Transactional` (쓰기)
- [ ] `@PreAuthorize("hasRole('ADMIN')")` 컨트롤러 클래스 레벨
- [ ] `ExternalApiLoggingAspect`에 `@Order(10)` 명시
- [ ] 클라이언트 4곳 어노테이션 import 추가

## 예상 함정

**함정 1 — AOP 동작 안 함**:
- `@Component` 스프링 빈에만 동작. 현재 코드는 모두 스프링 빈이라 무관
- Self-invocation(같은 클래스 내부 호출)은 AOP 프록시 우회. `NaverMapsClient.directions()`가 내부 `executeGetRequest()` 호출하는 구조 — 외부에서 직접 호출되는 public 메서드에만 어노테이션 붙임

**함정 2 — `REQUIRES_NEW` 트랜잭션 데드락**:
- `api_call_logs`는 FK가 `user_id`(SET NULL)뿐이고 INSERT 전용이라 데드락 가능성 없음

**함정 3 — 같은 클래스 내 `REQUIRES_NEW` 호출**:
- `saveLog()`와 `search()`가 같은 서비스 안에 있지만, 외부(Aspect)에서 호출하는 구조라 무관

**함정 4 — endpoint URL에 API Key 포함**:
- `WeatherService`가 `?appid=<key>&lat=...` 형태 URL 조립. `extractEndpoint`가 `?` 이전만 저장하므로 API Key는 저장 안 됨. `requestParams`는 Java 파라미터(lat, lng)만 기록 → 이중 안전

**함정 5 — `@Query` JPQL enum null 비교**:
- `:apiName IS NULL` 조건은 JPQL enum에도 동작. 일부 Hibernate 버전에서 미묘한 경우 있으면 Specification 전환 or Service에서 null 분기

## 확인용 curl

```bash
# 1. 어드민 로그인
TOKEN=$(curl -s -X POST http://localhost:8081/api/v1/auth/login/email \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@test.com","password":"..."}' \
  | jq -r '.data.accessToken')

# 2. 어드민 API 로그 조회
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8081/api/v1/admin/api-logs?page=0&size=10"

# 3. 필터
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8081/api/v1/admin/api-logs?apiName=NAVER_DIRECTIONS"

# 4. 401 확인 (인증 없이)
curl -v "http://localhost:8081/api/v1/admin/api-logs"

# 5. 403 확인 (일반 유저)
USER_TOKEN=$(curl -s -X POST http://localhost:8081/api/v1/auth/login/email \
  -H "Content-Type: application/json" \
  -d '{"email":"user@test.com","password":"..."}' \
  | jq -r '.data.accessToken')
curl -H "Authorization: Bearer $USER_TOKEN" \
  "http://localhost:8081/api/v1/admin/api-logs"

# 6. DB 직접
psql -d your_db -c "SELECT no, api_name, endpoint, response_time_ms, called_at
                    FROM api_call_logs ORDER BY called_at DESC LIMIT 10;"
```

## 앱(flutter-dev) 변경

없음. 백엔드 내부 및 어드민 전용.
