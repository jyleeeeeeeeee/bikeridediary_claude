# 유저 제보(UserReport) 백엔드 정정 가이드

작성일: 2026-08-18
담당: backend-dev (사용자 직접 반영)
근거: 사용자가 `user_reports` 테이블 + Entity/Repository/Service/Controller 스켈레톤 작성 완료. Claude 리뷰에서 critical/high 이슈 확인 → 정정 필요.

---

## 1. 현재 상태 & 문제 요약

| # | 항목 | 우선순위 | 파일 |
|---|------|--------|------|
| 1 | DDL의 `no` 컬럼이 `users_no_seq` 참조 (users 시퀀스 오염) | 🔴 필수 | DDL / schema.sql |
| 2 | Entity에 `@EntityListeners(AuditingEntityListener.class)` 누락 → `@CreatedDate/@LastModifiedDate` 안 채워짐 | 🔴 필수 | `UserReportEntity` |
| 3 | 팩토리에서 `report.id = UUID.randomUUID()` 수동 세팅 (`@GeneratedValue`와 중복 — Hibernate가 채우므로 불필요) | 🟡 정리 | `UserReportEntity` |
| 4 | Validation 메시지 오타 (`"credential은 필수입니다"` 복붙) | 🟡 High | `UserReportRequest` |
| 5 | `reportType`/`status` enum화 (값 고정) | 🟡 High | 신규 enum + Entity |
| 6 | `user_id` 인덱스 없음 | 🟡 High | Entity `@Table indexes` |
| 7 | SecurityConfig 매칭 확인 (`@AuthenticationPrincipal` null 방지) | 🟡 High | `SecurityConfig` |
| 8 | `content` 컬럼 세만틱 오버로딩 (PLACE_DELETE 시 place_id) | 🟡 High | `target_place_id` 별도 컬럼 검토 |
| 9 | 불필요 import (`@Slf4j` 미사용, `RequestMapping` in Service, `UserReportResponse` in Controller) | 🟢 Low | 3파일 |
| 10 | Entity 주석 오배치 (reply 위 status 주석), endedAt 주석 없음 | 🟢 Low | `UserReportEntity` |
| 11 | `UserReportResponse` 빈 record | 🟢 Low | 조회 붙일 때 확장 |

---

## 2. schema.sql 반영 (신규)

현재 schema.sql에는 user_reports 관련 정의가 없음 (사용자가 pgAdmin에서 직접 DDL 실행한 상태). 다른 환경/CI 재현 위해 반드시 추가.

### 2.1 시퀀스 추가

`schema.sql` 상단 시퀀스 블록에 추가:

```sql
CREATE SEQUENCE IF NOT EXISTS user_reports_no_seq;
```

### 2.2 테이블 정의 추가

`14. api_call_logs` 다음, `15. place_change_requests` 이전 위치에 삽입:

```sql
-- ============================================================
-- 14.5 user_reports (유저 제보 — 버그/장소 삭제/기타)
-- ============================================================
CREATE TABLE IF NOT EXISTS user_reports (
    no               BIGINT       UNIQUE DEFAULT nextval('user_reports_no_seq'),  -- 조회용 친숙 번호
    id               UUID         DEFAULT gen_random_uuid() PRIMARY KEY,
    title            VARCHAR(100) NOT NULL,
    -- 요청 종류: PLACE_DELETE / BUG_REPORT / ETC (enum 매핑)
    report_type      VARCHAR(30)  NOT NULL,
    -- 요청 내용. reportType=PLACE_DELETE면 이유 문자열 (target_place_id 컬럼 별도 사용 권장)
    content          TEXT         NOT NULL,
    -- 처리 상태: REPORTED / PROCEEDING / DONE / REJECT
    status           VARCHAR(20)  NOT NULL DEFAULT 'REPORTED',
    -- 관리자 처리 응답 (반려 사유 or 처리 결과)
    reply            TEXT,
    -- 대상 place (report_type=PLACE_DELETE 등에서 사용, 그 외 NULL)
    -- ON DELETE SET NULL: place 삭제돼도 제보 히스토리는 유지
    target_place_id  UUID         REFERENCES places(id) ON DELETE SET NULL,
    -- 제보자. 유저 탈퇴 시 제보 히스토리는 사라져도 무방
    user_id          UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    -- 처리 완료 시각 (status DONE/REJECT 시 세팅)
    ended_at         TIMESTAMP,
    created_at       TIMESTAMP    NOT NULL DEFAULT now(),
    updated_at       TIMESTAMP,
    -- BaseEntity 상속이라 필요. 실제 소프트 삭제 UX는 없지만 스키마 검증 통과용
    deleted_at       TIMESTAMP,

    CONSTRAINT chk_user_reports_type   CHECK (report_type IN ('PLACE_DELETE', 'BUG_REPORT', 'ETC')),
    CONSTRAINT chk_user_reports_status CHECK (status IN ('REPORTED', 'PROCEEDING', 'DONE', 'REJECT'))
);

-- 유저별 제보 내역 조회용
CREATE INDEX IF NOT EXISTS idx_user_reports_user_id
    ON user_reports (user_id);

-- 관리자 큐 조회용 (status + 최신순)
CREATE INDEX IF NOT EXISTS idx_user_reports_status_created_at
    ON user_reports (status, created_at DESC);
```

### 2.3 ALTER 섹션 (기존 DB용)

이미 pgAdmin으로 만든 로컬 DB의 `no` 컬럼 default를 정정 (users → user_reports 시퀀스):

```sql
-- ALTER 섹션에 추가
ALTER TABLE user_reports ADD COLUMN IF NOT EXISTS no BIGINT;
ALTER TABLE user_reports ALTER COLUMN no SET DEFAULT nextval('user_reports_no_seq');

-- target_place_id 신규 컬럼 (기존 DDL엔 없었음)
ALTER TABLE user_reports ADD COLUMN IF NOT EXISTS target_place_id UUID
    REFERENCES places(id) ON DELETE SET NULL;

-- BaseEntity 상속 대응 (기존 DDL엔 없었음)
ALTER TABLE user_reports ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMP;

-- CHECK 제약 추가 (기존 데이터에 잘못된 값 있으면 실패)
-- 필요 시 먼저 UPDATE로 정규화 후 실행
ALTER TABLE user_reports DROP CONSTRAINT IF EXISTS chk_user_reports_type;
ALTER TABLE user_reports ADD CONSTRAINT chk_user_reports_type
    CHECK (report_type IN ('PLACE_DELETE', 'BUG_REPORT', 'ETC'));

ALTER TABLE user_reports DROP CONSTRAINT IF EXISTS chk_user_reports_status;
ALTER TABLE user_reports ADD CONSTRAINT chk_user_reports_status
    CHECK (status IN ('REPORTED', 'PROCEEDING', 'DONE', 'REJECT'));
```

**⚠️ 시퀀스 정정 후 확인**: `nextval('user_reports_no_seq')`의 현재 값이 `users` 테이블 상황과 무관한 지점에서 시작하는지 확인. 기존에 잘못 채워진 `no` 값이 있어도 새 시퀀스는 처음부터 시작하므로 중복 UNIQUE 위반 가능 → 필요 시 `SELECT setval('user_reports_no_seq', (SELECT COALESCE(MAX(no), 0) FROM user_reports) + 1);` 실행.

---

## 3. Enum 신설 (2개)

### 3.1 `ReportType`

`src/main/java/com/bikeridediary/domain/user_report/entity/ReportType.java`

```java
package com.bikeridediary.domain.user_report.entity;

public enum ReportType {
    // 장소 삭제 제보 (target_place_id 세팅 필요)
    PLACE_DELETE,
    // 버그 리포트
    BUG_REPORT,
    // 기타 문의/개발자에게 할 말
    ETC
}
```

### 3.2 `ReportStatus`

`src/main/java/com/bikeridediary/domain/user_report/entity/ReportStatus.java`

```java
package com.bikeridediary.domain.user_report.entity;

public enum ReportStatus {
    // 접수 (기본)
    REPORTED,
    // 관리자 처리 중
    PROCEEDING,
    // 처리 완료
    DONE,
    // 반려
    REJECT
}
```

---

## 4. `UserReportEntity` 재작성

`src/main/java/com/bikeridediary/domain/user_report/entity/UserReportEntity.java`

**전체 교체**. 주요 변경:
- `@EntityListeners(AuditingEntityListener.class)` 추가
- `@GeneratedValue(strategy = GenerationType.UUID)` **유지** — UserReport는 서버 전용 엔티티(sync/시드 없음)라 Hibernate 자동 생성이 자연스러움. `UserEntity`와 동일 패턴. 팩토리에서 UUID 수동 세팅은 제거
- `reportType` / `status` enum + `@Enumerated(EnumType.STRING)`
- `target_place_id` FK 추가
- `@Table indexes` 명시
- 주석 정리

```java
package com.bikeridediary.domain.user_report.entity;

import com.bikeridediary.domain.common.entity.BaseEntity;
import com.bikeridediary.domain.place.entity.PlaceEntity;
import com.bikeridediary.domain.user.entity.UserEntity;
import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.Generated;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.generator.EventType;
import org.hibernate.type.SqlTypes;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;
import java.util.UUID;

// 유저 제보(버그/장소 삭제/기타) 엔티티.
@Entity
@Table(
        name = "user_reports",
        indexes = {
                @Index(name = "idx_user_reports_user_id", columnList = "user_id"),
                @Index(name = "idx_user_reports_status_created_at", columnList = "status, created_at DESC")
        }
)
@EntityListeners(AuditingEntityListener.class)
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class UserReportEntity extends BaseEntity {

    // 제보 ID (UUID) — Hibernate가 자동 생성 (UserEntity와 동일 패턴)
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    @Column(name = "id")
    @JdbcTypeCode(SqlTypes.UUID)
    private UUID id;

    // 조회용 친숙 번호 (자동 증가, DB DEFAULT nextval('user_reports_no_seq'))
    @Column(name = "no", insertable = false, updatable = false)
    @Generated(event = EventType.INSERT)
    private Long no;

    // 요청 제목
    @Column(name = "title", nullable = false, length = 100)
    private String title;

    // 요청 종류
    @Enumerated(EnumType.STRING)
    @Column(name = "report_type", nullable = false, length = 30)
    private ReportType reportType;

    // 요청 내용 (PLACE_DELETE 시 이유 문자열, 대상 place는 target_place_id 참조)
    @Column(name = "content", nullable = false, columnDefinition = "TEXT")
    private String content;

    // 대상 place (report_type=PLACE_DELETE 등에서 사용, 그 외 null)
    // ON DELETE SET NULL: place 삭제돼도 제보 히스토리 유지
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "target_place_id")
    private PlaceEntity targetPlace;

    // 처리 상태 (기본 REPORTED — 스키마 DEFAULT와 일치)
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private ReportStatus status = ReportStatus.REPORTED;

    // 관리자 처리 응답 (반려 사유 or 처리 결과)
    @Column(name = "reply", columnDefinition = "TEXT")
    private String reply;

    // 제보자
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private UserEntity userEntity;

    // 처리 완료 시각 (status DONE/REJECT 시 세팅)
    @Column(name = "ended_at")
    private LocalDateTime endedAt;

    // 신규 제보 생성. id는 @GeneratedValue, status는 필드 기본값(REPORTED) 사용.
    public static UserReportEntity create(
            String title,
            ReportType reportType,
            String content,
            PlaceEntity targetPlace,   // PLACE_DELETE 아니면 null
            UserEntity userEntity
    ) {
        UserReportEntity report = new UserReportEntity();
        report.title = title;
        report.reportType = reportType;
        report.content = content;
        report.targetPlace = targetPlace;
        report.userEntity = userEntity;
        return report;
    }

    // 관리자 상태 변경
    public void updateStatus(ReportStatus status, String reply) {
        this.status = status;
        this.reply = reply;
        if (status == ReportStatus.DONE || status == ReportStatus.REJECT) {
            this.endedAt = LocalDateTime.now();
        }
    }
}
```

---

## 5. `UserReportRequest` 정정

`src/main/java/com/bikeridediary/domain/user_report/dto/UserReportRequest.java`

Validation 메시지 필드별로 정정 + `targetPlaceId` 옵셔널 필드 추가.

```java
package com.bikeridediary.domain.user_report.dto;

import com.bikeridediary.domain.user_report.entity.ReportType;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import org.hibernate.validator.constraints.Length;

import java.util.UUID;

public record UserReportRequest(
        @NotBlank(message = "제목은 필수입니다")
        @Length(max = 100, message = "제목은 100자 이내로 작성해주세요")
        String title,

        @NotNull(message = "요청 종류는 필수입니다")
        ReportType reportType,

        @NotBlank(message = "내용은 필수입니다")
        String content,

        // reportType=PLACE_DELETE일 때만 필요, 그 외 null 허용
        UUID targetPlaceId
) {}
```

**주의**: `ReportType` enum 필드로 받으면 잘못된 값은 400 자동 리턴됨. 별도 검증 로직 불필요.

---

## 6. `UserReportService` 재작성

`src/main/java/com/bikeridediary/domain/user_report/service/UserReportService.java`

주요 변경: PLACE_DELETE 분기 처리 (targetPlaceId 필수 + place 존재 검증) + `PlaceRepository` 주입 + 불필요 import 제거.

```java
package com.bikeridediary.domain.user_report.service;

import com.bikeridediary.domain.place.entity.PlaceEntity;
import com.bikeridediary.domain.place.repository.PlaceRepository;
import com.bikeridediary.domain.user.entity.UserEntity;
import com.bikeridediary.domain.user.repository.UserRepository;
import com.bikeridediary.domain.user_report.dto.UserReportRequest;
import com.bikeridediary.domain.user_report.entity.ReportType;
import com.bikeridediary.domain.user_report.entity.UserReportEntity;
import com.bikeridediary.domain.user_report.repository.UserReportRepository;
import com.bikeridediary.global.exception.BusinessException;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

import static com.bikeridediary.global.exception.ErrorCode.PLACE_NOT_FOUND;
import static com.bikeridediary.global.exception.ErrorCode.USER_NOT_FOUND;

@Service
@RequiredArgsConstructor
public class UserReportService {

    private final UserReportRepository userReportRepository;
    private final UserRepository userRepository;
    private final PlaceRepository placeRepository;

    // 유저 제보 생성. PLACE_DELETE면 targetPlaceId 필수 + 존재 검증.
    @Transactional
    public void report(UserReportRequest request, UUID userId) {
        UserEntity user = userRepository.findByIdAndDeletedAtIsNull(userId)
                .orElseThrow(() -> new BusinessException(USER_NOT_FOUND));

        PlaceEntity targetPlace = null;
        if (request.reportType() == ReportType.PLACE_DELETE) {
            if (request.targetPlaceId() == null) {
                throw new IllegalArgumentException("PLACE_DELETE 제보에는 targetPlaceId가 필요합니다.");
            }
            targetPlace = placeRepository.findByIdAndDeletedAtIsNull(request.targetPlaceId())
                    .orElseThrow(() -> new BusinessException(PLACE_NOT_FOUND));
        }

        UserReportEntity report = UserReportEntity.create(
                request.title(),
                request.reportType(),
                request.content(),
                targetPlace,
                user
        );
        userReportRepository.save(report); // 응답 DTO 반환 없음 → 반환값 사용 불필요
    }
}
```

**참고**: 응답 DTO 만들 때 `createdAt` 필요하면 `saved = userReportRepository.save(report)` 반환값 사용. `@GeneratedValue` + `@PrePersist` 조합이라 sync 엔티티(BikeSync 등)에서 발생하던 merge 경로 이슈는 이 엔티티에선 발생하지 않음.

---

## 7. `UserReportController` 정정

`src/main/java/com/bikeridediary/domain/user_report/controller/UserReportController.java`

주요 변경: `@Tag` description 추가, `@Operation(summary)` 추가, 불필요 import(`UserReportResponse`) 제거.

```java
package com.bikeridediary.domain.user_report.controller;

import com.bikeridediary.domain.user_report.dto.UserReportRequest;
import com.bikeridediary.domain.user_report.service.UserReportService;
import com.bikeridediary.global.auth.CustomUserDetails;
import com.bikeridediary.global.response.ApiResponse;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Tag(name = "유저 제보", description = "버그/장소 삭제/기타 제보")
@RestController
@RequestMapping("/api/v1/user-reports")
@RequiredArgsConstructor
public class UserReportController {

    private final UserReportService userReportService;

    @Operation(summary = "제보 생성")
    @PostMapping
    public ResponseEntity<ApiResponse<Void>> report(
            @AuthenticationPrincipal CustomUserDetails userDetails,
            @RequestBody @Valid UserReportRequest request
    ) {
        userReportService.report(request, userDetails.getUserId());
        return ResponseEntity.ok(ApiResponse.ok());
    }
}
```

---

## 8. `UserReportRepository` 유지 + 관리자 조회용 추가 (선택)

현재 파일 그대로 두되, 향후 관리자/내 목록 조회용 페이징 메서드 추가 가능:

```java
public interface UserReportRepository extends JpaRepository<UserReportEntity, UUID> {

    // 유저별 제보 내역 (내 목록)
    Page<UserReportEntity> findByUserEntity_IdOrderByCreatedAtDesc(UUID userId, Pageable pageable);

    // 관리자 큐 (상태별 최신순)
    Page<UserReportEntity> findByStatusOrderByCreatedAtDesc(ReportStatus status, Pageable pageable);

    // 제보 내역 상세 조회 (내 것만 조회 가능하도록 소유권 필터 포함)
    Optional<UserReportEntity> findByIdAndUserEntity_Id(UUID reportId, UUID userId);
}
```

---

## 9. SecurityConfig 확인

`/api/v1/user-reports/**`는 인증 필수. `SecurityConfig`의 `PERMIT_ALL_ENDPOINTS`/`GET_PERMIT_ALL_ENDPOINTS`에 해당 경로가 없으면 `anyRequest().authenticated()`로 자동 인증됨 → 별도 매칭 불필요. 다만 관리자 조회/처리 엔드포인트 붙이면 `@PreAuthorize("hasRole('ADMIN')")` 사용 (place change request 어드민 컨트롤러와 동일).

---

## 10. `UserReportResponse` 정의

향후 조회 붙일 때 참고:

```java
public record UserReportResponse(
        UUID id,
        String title,
        ReportType reportType,
        String content,
        ReportStatus status,
        String reply,
        UUID targetPlaceId,
        LocalDateTime createdAt,
        LocalDateTime endedAt
) {
    public static UserReportResponse from(UserReportEntity entity) {
        return new UserReportResponse(
                entity.getId(),
                entity.getTitle(),
                entity.getReportType(),
                entity.getContent(),
                entity.getStatus(),
                entity.getReply(),
                entity.getTargetPlace() == null ? null : entity.getTargetPlace().getId(),
                entity.getCreatedAt(),
                entity.getEndedAt()
        );
    }
}
```

---

## 11. 반영 순서 권장

1. `schema.sql` 시퀀스 + 테이블 + 인덱스 추가 (§2.1, §2.2)
2. 기존 로컬 DB에 ALTER (§2.3) — pgAdmin에서 실행
3. `ReportType`, `ReportStatus` enum 신설 (§3)
4. `UserReportEntity` 전체 교체 (§4)
5. `UserReportRequest` 정정 (§5)
6. `UserReportService` 재작성 (§6)
7. `UserReportController` 불필요 import 제거 (§7)
8. 부팅 확인 (@ConfigurationPropertiesScan/EntityScan 자동 등록되므로 별도 설정 없음)
9. Swagger에서 `POST /api/v1/user-reports` 시나리오 4종 확인:
   - BUG_REPORT (targetPlaceId 없음, 성공)
   - PLACE_DELETE + 유효한 targetPlaceId (성공)
   - PLACE_DELETE + targetPlaceId null (400)
   - ETC (targetPlaceId 없음, 성공)

---

## 12. 체크리스트

### DDL / DB
- [ ] `user_reports_no_seq` 시퀀스 생성
- [ ] 테이블 정의 schema.sql에 추가 (CHECK 제약 포함, `deleted_at` 포함)
- [ ] 인덱스 2개 (`user_id`, `status+created_at`)
- [ ] 기존 로컬 DB의 `no` 컬럼 default를 `user_reports_no_seq`로 정정
- [ ] `target_place_id` 컬럼 추가 (기존 DDL엔 없음)
- [ ] `deleted_at` 컬럼 추가 (BaseEntity 상속 대응, 기존 DDL엔 없음)

### Entity
- [ ] `@EntityListeners(AuditingEntityListener.class)` 추가
- [ ] `@GeneratedValue(strategy = GenerationType.UUID)` 유지, 팩토리에서 UUID 수동 세팅 코드는 제거
- [ ] `reportType` / `status` `@Enumerated(EnumType.STRING)`
- [ ] `target_place_id` `@ManyToOne PlaceEntity`
- [ ] `@Table indexes` 명시
- [ ] 주석 정정 (reply 위 status 주석 제거, endedAt 주석 추가)

### DTO
- [ ] Validation 메시지 필드별로 정정
- [ ] `reportType`을 `ReportType` enum으로
- [ ] `targetPlaceId` 옵셔널 필드 추가

### Service
- [ ] `PlaceRepository` 주입 (PLACE_DELETE 시 존재 검증)
- [ ] PLACE_DELETE에 `targetPlaceId` 필수 검증
- [ ] 불필요 import 제거 (`RequestMapping`, `ErrorCode`(static과 중복), `@Slf4j`)

### Controller
- [ ] `@Tag` description 명시
- [ ] `@Operation(summary)` 추가
- [ ] 불필요 import 제거 (`UserReportResponse`)

### 확인
- [ ] 부팅 성공 (schema.sql 실행 + Hibernate 검증 통과)
- [ ] Swagger에서 4개 시나리오 통과
- [ ] DB에 저장 시 `no` 값이 `user_reports_no_seq` 기준으로 증가하는지 (users 시퀀스와 무관)
- [ ] `created_at`/`updated_at`이 auditing으로 자동 채워지는지

---

## 13. 별도 스코프 (이번 사이클 아님)

- **내 제보 목록 조회** (`GET /api/v1/user-reports/me`) — Repository 페이징 메서드는 준비됨
- **관리자 큐 조회/처리** (`GET /admin/user-reports`, `PATCH /admin/user-reports/{id}/status`) — `@PreAuthorize("hasRole('ADMIN')")` + `updateStatus` 활용
- **앱 UI** — 설정 화면에서 "제보하기" 진입 + PLACE_DELETE 흐름은 지도 상세 시트에서 딥링크
