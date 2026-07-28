# JWT Stateless 리팩터 — 백엔드 구현 가이드 (backend-dev)

> 사용자가 직접 구현. 아래 스니펫은 그대로 파일에 복사 붙여넣기 가능한 완성형.
> 대상 프로젝트: `brd_be` (Spring Boot 3.x)
> 최종 갱신: 2026-07-27

---

## 목표

매 API 요청마다 발생하는 `userRepository.findByIdAndDeletedAtIsNull()` DB 조회를 제거하고,
JWT 액세스 토큰의 `role` claim에서 직접 `CustomUserDetails`를 생성하여 진정한 stateless 필터를 만든다.

---

## 현재 문제 흐름 (Before)

```
요청 → JwtAuthenticationFilter
         └─ jwtTokenProvider.extractUserId(token)          // claim 파싱
         └─ userDetailsService.loadUserByUsername(userId)  // ← DB SELECT 발생
              └─ userRepository.findByIdAndDeletedAtIsNull()  // users 테이블 hit
```

초당 N개 요청 → users 테이블 N번 read. JWT 의 stateless 이점이 사라진 상태.

---

## 목표 흐름 (After)

```
요청 → JwtAuthenticationFilter
         └─ jwtTokenProvider.extractUserId(token)   // claim 파싱
         └─ jwtTokenProvider.extractRole(token)     // claim 파싱 (신규)
         └─ new CustomUserDetails(userId, role)     // DB 조회 없음
```

`CustomUserDetailsService.loadUserByUsername`은 로그인/Refresh 흐름에서만 호출.

---

## 수정 파일 목록

| 순서 | 파일 경로 | 작업 |
|-----|-----------|------|
| 1 | `global/auth/jwt/JwtTokenProvider.java` | `generateAccessToken`에 role claim 추가, `extractRole` 신규 |
| 2 | `global/auth/jwt/JwtAuthenticationFilter.java` | DB 조회 제거, role claim 으로 CustomUserDetails 직접 생성 |
| 3 | `domain/auth/service/AuthService.java` | 4개 토큰 발급 지점 + `refresh()` 수정 |
| 4 | `global/config/SecurityConfig.java` | 필터 생성자 인자에서 `userDetailsService` 제거 |

`CustomUserDetails`, `CustomUserDetailsService`는 변경 없음.

---

## 코드 스니펫

### 1. JwtTokenProvider

파일: `src/main/java/com/bikeridediary/global/auth/jwt/JwtTokenProvider.java`

변경점:
- `generateAccessToken(UUID userId)` → `generateAccessToken(UUID userId, UserRole role)`
- `extractRole(String token)` 메서드 신규 추가
- `generateRefreshToken`, `generateGuestRefreshToken`은 role 없음 (현행 유지)

```java
package com.bikeridediary.global.auth.jwt;

import com.bikeridediary.domain.user.entity.UserRole;
import com.bikeridediary.global.exception.BusinessException;
import com.bikeridediary.global.exception.ErrorCode;
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.UUID;

// JWT 토큰 생성 및 검증 (액세스 토큰: 1시간, 리프레시 토큰: 30일)
@Slf4j
@Component
public class JwtTokenProvider {

    private static final String CLAIM_ROLE = "role"; // 액세스 토큰 role claim 키

    private final SecretKey secretKey;
    private final long accessTokenExpiry;
    private final long refreshTokenExpiry;

    public JwtTokenProvider(JwtProperties properties) {
        this.secretKey = Keys.hmacShaKeyFor(properties.secret().getBytes(StandardCharsets.UTF_8));
        this.accessTokenExpiry = properties.accessTokenExpiry() * 1000L;
        this.refreshTokenExpiry = properties.refreshTokenExpiry() * 1000L;
    }

    // 액세스 토큰 생성 — role claim 포함 (필터에서 DB 조회 없이 인증)
    public String generateAccessToken(UUID userId, UserRole role) {
        Date now = new Date();
        return Jwts.builder()
                .subject(userId.toString())
                .claim(CLAIM_ROLE, role.name())  // "USER" 또는 "ADMIN"
                .issuedAt(now)
                .expiration(new Date(now.getTime() + accessTokenExpiry))
                .signWith(secretKey)
                .compact();
    }

    // 리프레시 토큰 생성 — role 없음 (Refresh 시 DB에서 role 재조회)
    public String generateRefreshToken(UUID userId) {
        return buildToken(userId.toString(), refreshTokenExpiry);
    }

    // 게스트용 리프레시 토큰 (1년)
    public String generateGuestRefreshToken(UUID userId) {
        return buildToken(userId.toString(), 365L * 24 * 60 * 60 * 1000);
    }

    private String buildToken(String subject, long expiry) {
        Date now = new Date();
        return Jwts.builder()
                .subject(subject)
                .issuedAt(now)
                .expiration(new Date(now.getTime() + expiry))
                .signWith(secretKey)
                .compact();
    }

    // 토큰에서 사용자 ID 추출
    public UUID extractUserId(String token) {
        String subject = parseClaims(token).getSubject();
        return UUID.fromString(subject);
    }

    // 토큰에서 role 추출 (액세스 토큰 전용)
    // 리프레시 토큰에는 claim 이 없으므로 null → 기본값 USER 반환
    public UserRole extractRole(String token) {
        String roleName = parseClaims(token).get(CLAIM_ROLE, String.class);
        if (roleName == null) {
            return UserRole.USER;
        }
        try {
            return UserRole.valueOf(roleName);
        } catch (IllegalArgumentException e) {
            log.warn("Unknown role claim in token: {}", roleName);
            return UserRole.USER;
        }
    }

    // 토큰 유효성 검증
    public boolean isValid(String token) {
        try {
            parseClaims(token);
            return true;
        } catch (ExpiredJwtException e) {
            throw new BusinessException(ErrorCode.AUTH_EXPIRED_TOKEN);
        } catch (JwtException | IllegalArgumentException e) {
            throw new BusinessException(ErrorCode.AUTH_INVALID_TOKEN);
        }
    }

    private Claims parseClaims(String token) {
        return Jwts.parser()
                .verifyWith(secretKey)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```


---

### 2. JwtAuthenticationFilter

파일: `src/main/java/com/bikeridediary/global/auth/jwt/JwtAuthenticationFilter.java`

변경점:
- `UserDetailsService` 의존성 완전 제거
- role claim 추출 후 `CustomUserDetails` 직접 생성 (DB 조회 없음)

```java
package com.bikeridediary.global.auth.jwt;

import com.bikeridediary.domain.user.entity.UserRole;
import com.bikeridediary.global.auth.CustomUserDetails;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.UUID;

// JWT 인증 필터. 토큰의 claim 만으로 인증 정보를 설정한다. DB 조회 없음.
@Slf4j
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    // UserDetailsService 제거 — DB 조회 불필요

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        String token = extractToken(request);

        if (StringUtils.hasText(token) && jwtTokenProvider.isValid(token)) {
            UUID userId = jwtTokenProvider.extractUserId(token);
            UserRole role = jwtTokenProvider.extractRole(token);

            // DB 조회 없이 claim 값으로 직접 생성
            CustomUserDetails userDetails = new CustomUserDetails(userId, role);

            UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

---

### 3. SecurityConfig — 필터 생성자 수정

파일: `src/main/java/com/bikeridediary/global/config/SecurityConfig.java`

`JwtAuthenticationFilter`에서 `UserDetailsService`를 뺐으므로 필드와 필터 생성 코드도 맞춘다.

Before:
```java
private final JwtTokenProvider jwtTokenProvider;
private final UserDetailsService userDetailsService;   // ← 삭제

// filterChain() 내부
.addFilterBefore(
    new JwtAuthenticationFilter(jwtTokenProvider, userDetailsService),  // ← 인자 제거
    UsernamePasswordAuthenticationFilter.class
);
```

After:
```java
private final JwtTokenProvider jwtTokenProvider;
// userDetailsService 필드 삭제 (빈 자체 CustomUserDetailsService 는 유지)

// filterChain() 내부
.addFilterBefore(
    new JwtAuthenticationFilter(jwtTokenProvider),
    UsernamePasswordAuthenticationFilter.class
);
```

주의: `UserDetailsService` 빈(`CustomUserDetailsService`)은 삭제하지 않는다.
로그인 흐름의 이메일/비밀번호 인증에서 여전히 필요하다.
SecurityConfig 에서 의존성 필드만 제거하는 것이다.


---

### 4. AuthService — 토큰 발급 지점 수정

파일: `src/main/java/com/bikeridediary/domain/auth/service/AuthService.java`

`generateAccessToken` 시그니처가 `(UUID, UserRole)`로 바뀌었으므로 4개 호출부를 모두 수정한다.

#### 4-1. guestSignup()

```java
// Before
String accessToken = jwtTokenProvider.generateAccessToken(saved.getId());

// After
String accessToken = jwtTokenProvider.generateAccessToken(saved.getId(), saved.getRole());
```

#### 4-2. signup()

```java
// Before
String accessToken = jwtTokenProvider.generateAccessToken(id);

// After
String accessToken = jwtTokenProvider.generateAccessToken(id, savedUserEntity.getRole());
```

#### 4-3. login() — 소셜 로그인 통합 (카카오/구글/애플/네이버 4종 모두 커버)

```java
// Before
String accessToken = jwtTokenProvider.generateAccessToken(userEntity.getId());

// After
String accessToken = jwtTokenProvider.generateAccessToken(userEntity.getId(), userEntity.getRole());
```

각 provider 클래스(KakaoProvider, GoogleProvider, AppleProvider, NaverProvider)는 수정 없음.
`login()` 안의 `findOrCreateUser()`가 `UserEntity`를 반환하므로 role 을 바로 꺼낼 수 있다.

#### 4-4. loginWithEmail()

```java
// Before
String accessToken = jwtTokenProvider.generateAccessToken(id);

// After
String accessToken = jwtTokenProvider.generateAccessToken(id, userEntity.getRole());
```

#### 4-5. refresh() — DB 조회 추가 (완성형)

현재 `refresh()`는 Redis 검증만 하고 DB 를 조회하지 않는다.
새 액세스 토큰에 role 을 넣으려면 DB 조회가 필요하다.
이 지점 1회 DB 조회는 허용 범위 — Refresh 는 클라이언트가 자주 호출하지 않는다.
소프트 삭제된 유저의 Refresh Token 차단도 이 지점에서 이루어진다.

```java
@Transactional
public TokenResponse refresh(String refreshToken) {
    try {
        // Step 1: Refresh Token 에서 사용자 ID 추출
        UUID userId = jwtTokenProvider.extractUserId(refreshToken);

        // Step 2: Redis 에서 저장된 토큰과 비교하여 유효성 검증
        if (!refreshTokenRepository.isValid(userId, refreshToken)) {
            throw new BusinessException(ErrorCode.AUTH_INVALID_REFRESH_TOKEN);
        }

        // Step 3: DB 에서 유저 조회 — role 재취득 + 소프트 삭제 검사
        // 소프트 삭제된 유저의 Refresh Token 이 여기서 차단된다
        UserEntity userEntity = userRepository.findByIdAndDeletedAtIsNull(userId)
                .orElseThrow(() -> new BusinessException(ErrorCode.AUTH_INVALID_REFRESH_TOKEN));

        // Step 4: 새로운 Access Token 발급 (role claim 포함)
        String newAccessToken = jwtTokenProvider.generateAccessToken(userId, userEntity.getRole());

        // Step 5: 새로운 Refresh Token 발급 및 저장
        String newRefreshToken = jwtTokenProvider.generateRefreshToken(userId);
        refreshTokenRepository.save(userId, newRefreshToken);

        log.info("Token refreshed - userId: {}", userId);

        return new TokenResponse(newAccessToken, newRefreshToken);
    } catch (BusinessException e) {
        throw e;
    } catch (Exception e) {
        log.error("Token refresh failed", e);
        throw new BusinessException(ErrorCode.AUTH_INVALID_REFRESH_TOKEN);
    }
}
```


---

## 반영 순서

1. **`JwtTokenProvider` 수정**
   - 컴파일하면 `AuthService` 에서 기존 `generateAccessToken(UUID)` 호출 지점이 전부 에러로 표시됨
   - 컴파일러가 수정해야 할 위치를 자동으로 알려준다

2. **`AuthService` 수정**
   - 4개 `generateAccessToken` 호출에 role 인자 추가
   - `refresh()` 에 DB 조회 추가

3. **`JwtAuthenticationFilter` 수정**
   - `UserDetailsService` 의존성 제거, claim 기반으로 교체

4. **`SecurityConfig` 수정**
   - 필터 생성자에서 `userDetailsService` 인자 제거, 필드 삭제

5. **전체 빌드 + 단위 테스트 실행**

---

## 검증 방법

### 수동 테스트 시나리오

**시나리오 1 — 일반 API 호출 시 users 테이블 SELECT 사라지는지 확인**

p6spy 가 이미 활성화되어 있으므로 (`spy.properties`) IntelliJ 콘솔에서 SQL 을 바로 볼 수 있다.

1. 이메일 로그인: `POST /api/v1/auth/login/email` → accessToken 획득
2. `GET /api/v1/bikes` 호출 (Authorization: Bearer {accessToken})
3. 콘솔에서 users 테이블 SELECT 가 없는지 확인

Before 콘솔 (리팩터 전):
```
SELECT u.* FROM users WHERE u.id = ? AND u.deleted_at IS NULL
SELECT b.* FROM bikes WHERE user_entity_id = ?
```

After 콘솔 (리팩터 후):
```
SELECT b.* FROM bikes WHERE user_entity_id = ?
```

users 테이블 조회 줄이 사라지면 성공.

**시나리오 2 — 어드민 권한 엔드포인트 정상 동작**

1. DB 에서 특정 유저의 `role` 을 `ADMIN` 으로 직접 UPDATE
2. 해당 유저로 **새로 로그인** (기존 토큰에는 `USER` role 이 담겨 있으므로 새 토큰 필요)
3. `GET /api/v1/admin/places/change-requests` → 200 응답 확인
4. `USER` role 토큰으로 같은 엔드포인트 → 403 확인

**시나리오 3 — Refresh 후 role 반영 확인**

1. `USER` role 로 로그인, refreshToken 획득
2. DB 에서 해당 유저 role 을 `ADMIN` 으로 UPDATE
3. `POST /api/v1/auth/refresh` (refreshToken 전달)
4. 새 accessToken 으로 어드민 엔드포인트 → 200 확인

**시나리오 4 — 소프트 삭제 유저의 Refresh 차단**

1. 로그인 후 refreshToken 획득
2. DB 에서 해당 유저 `deleted_at = now()` 직접 UPDATE
3. 기존 accessToken 으로 API 호출 → 200 (액세스 토큰은 만료 전까지 유효, 의도된 동작)
4. `POST /api/v1/auth/refresh` 시도 → 400/401 (`findByIdAndDeletedAtIsNull` 실패)

---

## 알려진 리스크와 다음 단계

### 트레이드오프: role 변경·소프트 삭제의 즉시 반영 불가

| 상황 | 반영 시점 | 허용 가능 여부 |
|------|----------|--------------|
| role USER → ADMIN 변경 | 다음 Refresh (기본 30분) | 허용 가능. 어드민은 직접 부여하는 권한이라 즉시성 불필요 |
| role ADMIN → USER 강등 | 다음 Refresh (기본 30분) | 수동 작업이므로 30분 유예 허용 가능 |
| 소프트 삭제 | 액세스 토큰 만료 (기본 1시간) | 일반적으로 허용 범위. 탈퇴 직후 1시간 API 접근 가능 |

### 즉시 강제 로그아웃이 필요한 경우 (이번 스코프 외)

Redis 블랙리스트 패턴:
- 강제 로그아웃 시 해당 userId 를 Redis 에 `blocked:{userId}` 키로 저장 (TTL = 액세스 토큰 만료 시간)
- `JwtAuthenticationFilter` 에서 블랙리스트 확인 후 401 반환
- 필터에 Redis 조회가 추가되므로 필요성이 확인된 뒤 도입할 것

### 액세스 토큰 만료 시간 단축으로 리스크 완화 (이번 스코프 외)

액세스 토큰 만료 시간을 5~15분으로 단축 + 클라이언트 Silent Refresh 조합.
Refresh 빈도가 높아지면 role 변경도 빠르게 반영되고, DB 조회는 Refresh 시점 1회로 유지된다.
`JwtProperties.accessTokenExpiry` 값만 줄이면 된다.

---

## 체크리스트

- [ ] `generateAccessToken(UUID, UserRole)` 시그니처 변경, `extractRole` 신규 추가
- [ ] `AuthService` 내 4개 발급 지점 모두 role 인자 추가 (guestSignup/signup/login/loginWithEmail)
- [ ] `AuthService.refresh()` — DB 조회로 role 재취득 + 소프트 삭제 차단
- [ ] `JwtAuthenticationFilter` — `UserDetailsService` 의존성 제거, claim 기반으로 교체
- [ ] `SecurityConfig` — 필터 생성자 인자 제거, `userDetailsService` 필드 삭제
- [ ] 빌드 성공 확인
- [ ] p6spy 로그에서 `users` SELECT 사라지는 것 확인 (시나리오 1)
- [ ] `@PreAuthorize("hasRole('ADMIN')")` 엔드포인트 정상 동작 확인 (시나리오 2)
- [ ] Refresh 후 role 반영 확인 (시나리오 3)
- [ ] 소프트 삭제 유저 Refresh 차단 확인 (시나리오 4)

---

## 참고: ROLE_ prefix 매핑 이상 없는 이유

`CustomUserDetails`는 변경 없음:
```java
this.authorities = List.of(new SimpleGrantedAuthority("ROLE_" + role.name()));
```

`@PreAuthorize("hasRole('ADMIN')")` → Spring Security 내부에서 `ROLE_ADMIN`으로 변환하여 비교.
기존 어드민 엔드포인트 `@PreAuthorize` 어노테이션 수정 불필요.

