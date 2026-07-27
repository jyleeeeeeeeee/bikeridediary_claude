# 장소 승인 워크플로 — Flutter 앱 구현 가이드 (flutter-dev)

> Claude가 직접 구현. 아래는 flutter-dev 에이전트가 수행할 작업 명세.
> 대상 프로젝트: `brd_app`
> 관련: `place-approval-workflow.md`, `place-approval-schema.md`, `place-approval-backend.md`

---

## 요약

- 신규 장소 등록 흐름을 "로컬 저장 → 서버 공개 요청" 두 단계로 분리
- 서버 place 수정도 즉시 반영 폐기, 요청 생성으로 통합
- 어드민 유저에게만 노출되는 관리 화면 추가
- 유저 role은 서버 JWT의 응답 시 UserResponse.role 필드로 실려 옴 (백엔드가 응답에 role 포함 예정)

## 작업 순서

1. `UserResponse`에 `role` 필드 추가
2. `AppDatabase` 버전 3 마이그레이션: `local_places` 테이블
3. `LocalPlaceRepository` (SQLite CRUD)
4. `local_place.dart` 모델 (앱 도메인)
5. `PlaceChangeRequest` 관련 모델 + repository
6. `CourseMapScreen`: 로컬 place 마커 합쳐 렌더링 (반투명 스타일)
7. `PlaceSearchAddScreen`: 저장 로직을 로컬 저장으로 교체
8. `LocalPlaceDetailScreen` (신규): 로컬 pin 상세 + "공개 등록 요청" 버튼
9. `place_info_edit_screen`, `place_coordinate_edit_screen`: 저장 로직을 "요청 생성"으로 교체 (서버 place 대상일 때만 사용)
10. `MyPlaceRequestsScreen` (신규): 내 요청 목록
11. `AdminPlaceRequestsScreen`, `AdminPlaceRequestDetailScreen` (신규): 어드민 화면
12. 설정 화면에서 role=ADMIN이면 어드민 카드 노출, 로컬 게스트가 아니면 "내 장소 요청" 노출
13. `main_shell` 확장 메뉴 or 다른 진입점 결정 (원래 4버튼 유지)
14. 라우트 신규: `/local-places/:id`, `/my-place-requests`, `/admin/place-requests`, `/admin/place-requests/:id`
15. Auth logout 시 `AppDatabase.clearAll()`이 이미 있으므로 그 안에 `DELETE FROM local_places` 추가

## 세부 명세

### 1. UserResponse에 role 필드

`brd_app/lib/features/auth/data/model/user_response.dart`

```dart
@JsonSerializable()
class UserResponse {
  final String id;
  final String? provider;
  final String? nickname;
  final String? email;
  final String? profileImageUrl;
  final String createdAt;
  final String role;  // USER | ADMIN. 기본 USER (백엔드 미포함 시 fallback)

  UserResponse({
    required this.id,
    this.provider,
    this.nickname,
    this.email,
    this.profileImageUrl,
    required this.createdAt,
    this.role = 'USER',
  });

  bool get isAdmin => role == 'ADMIN';

  factory UserResponse.fromJson(Map<String, dynamic> json) =>
      _$UserResponseFromJson(json);
}
```

`.g.dart` 재생성 필요: `flutter pub run build_runner build --delete-conflicting-outputs`

**백엔드 대응 (완료 예정)**: backend-dev 문서 섹션 5에서 `UserResponse` record에 `UserRole role` 필드를 추가하고 `from(UserEntity)` 팩토리에서 반환하도록 명시됨. 백엔드 배포 이전 앱은 role 필드가 응답에 없어 fallback으로 `'USER'` 사용.

### 2. AppDatabase v3

`brd_app/lib/core/local/app_database.dart`

```dart
class AppDatabase {
  static Database? _db;
  static const _dbName = 'brd_local.db';
  static const _currentVersion = 3;  // 2 → 3

  // ... instance(), _onCreate, _onUpgrade 그대로 ...

  static final Map<int, Future<void> Function(Database)> _migrations = {
    1: (db) async {},
    2: (db) async {
      // 기존 bikes 테이블 (변경 없음)
      await db.execute('''
        CREATE TABLE bikes ( ... 기존 그대로 ... )
      ''');
      await db.execute('CREATE INDEX idx_bikes_sync_state ON bikes(sync_state)');
      await db.execute('CREATE INDEX idx_bikes_deleted ON bikes(deleted_at)');
    },
    3: (db) async {
      // 로컬 place pin — 유저가 개인적으로 저장한 장소.
      // request_id/request_status가 null이면 순수 로컬 pin.
      await db.execute('''
        CREATE TABLE local_places (
          id TEXT PRIMARY KEY,
          place_name TEXT NOT NULL,
          category_code TEXT NOT NULL,
          latitude REAL NOT NULL,
          longitude REAL NOT NULL,
          address TEXT,
          road_address TEXT,
          description TEXT,
          phone TEXT,
          photo_url TEXT,
          created_at INTEGER NOT NULL,
          updated_at INTEGER NOT NULL,
          request_id TEXT,
          request_status TEXT,
          request_synced_at INTEGER,
          deleted_at INTEGER
        )
      ''');
      await db.execute(
        'CREATE INDEX idx_local_places_request_status ON local_places(request_status)',
      );
      await db.execute(
        'CREATE INDEX idx_local_places_deleted ON local_places(deleted_at)',
      );
    },
  };

  static Future<void> clearAll() async {
    final db = await instance();
    await db.transaction((txn) async {
      await txn.delete('bikes');
      await txn.delete('local_places');  // 신규
    });
  }
}
```

### 3. LocalPlaceRepository

`brd_app/lib/features/place/data/local/local_place_repository.dart`

```dart
import 'package:sqflite/sqflite.dart';
import '../../../../core/local/app_database.dart';
import '../model/local_place.dart';

/// 로컬 place SQLite CRUD.
class LocalPlaceRepository {
  Future<Database> get _db => AppDatabase.instance();

  // 활성(soft delete 안 된) 로컬 pin 전체
  Future<List<LocalPlace>> listActive() async {
    final db = await _db;
    final rows = await db.query(
      'local_places',
      where: 'deleted_at IS NULL',
      orderBy: 'created_at DESC',
    );
    return rows.map(LocalPlace.fromRow).toList();
  }

  Future<LocalPlace?> findById(String id) async {
    final db = await _db;
    final rows = await db.query(
      'local_places',
      where: 'id = ?',
      whereArgs: [id],
      limit: 1,
    );
    if (rows.isEmpty) return null;
    return LocalPlace.fromRow(rows.first);
  }

  Future<void> insert(LocalPlace p) async {
    final db = await _db;
    await db.insert('local_places', p.toRow(),
        conflictAlgorithm: ConflictAlgorithm.replace);
  }

  Future<void> update(LocalPlace p) async {
    final db = await _db;
    await db.update('local_places', p.toRow(),
        where: 'id = ?', whereArgs: [p.id]);
  }

  Future<void> softDelete(String id) async {
    final db = await _db;
    await db.update(
      'local_places',
      {'deleted_at': DateTime.now().millisecondsSinceEpoch},
      where: 'id = ?',
      whereArgs: [id],
    );
  }

  // 승인 완료 → 서버 places로 이관되었으니 로컬에서 hard delete
  Future<void> hardDelete(String id) async {
    final db = await _db;
    await db.delete('local_places', where: 'id = ?', whereArgs: [id]);
  }

  // 요청 제출 후 request_id/status 갱신
  Future<void> attachRequest(String id, String requestId) async {
    final db = await _db;
    await db.update(
      'local_places',
      {
        'request_id': requestId,
        'request_status': 'PENDING',
        'request_synced_at': DateTime.now().millisecondsSinceEpoch,
      },
      where: 'id = ?',
      whereArgs: [id],
    );
  }

  // 서버에서 요청 상태 재조회 후 반영
  Future<void> syncRequestStatus(String id, String status) async {
    final db = await _db;
    await db.update(
      'local_places',
      {
        'request_status': status,
        'request_synced_at': DateTime.now().millisecondsSinceEpoch,
      },
      where: 'id = ?',
      whereArgs: [id],
    );
  }
}

final localPlaceRepositoryProvider =
    Provider<LocalPlaceRepository>((ref) => LocalPlaceRepository());
```

### 4. local_place.dart 모델

`brd_app/lib/features/place/data/model/local_place.dart`

```dart
import 'place_category.dart';

/// 로컬 pin 모델. 서버 PlaceResponse와 필드 구성 유사하나 request_* 트래킹 컬럼 추가.
class LocalPlace {
  final String id;                    // 클라이언트 UUID v4
  final String name;
  final PlaceCategory category;
  final double latitude;
  final double longitude;
  final String? address;
  final String? roadAddress;
  final String? description;
  final String? phone;
  final String? photoUrl;
  final int createdAt;                // millis
  final int updatedAt;

  final String? requestId;            // 서버 요청 ID (null이면 요청 안 한 순수 로컬)
  final String? requestStatus;        // null | PENDING | APPROVED | REJECTED
  final int? requestSyncedAt;

  const LocalPlace({
    required this.id,
    required this.name,
    required this.category,
    required this.latitude,
    required this.longitude,
    this.address,
    this.roadAddress,
    this.description,
    this.phone,
    this.photoUrl,
    required this.createdAt,
    required this.updatedAt,
    this.requestId,
    this.requestStatus,
    this.requestSyncedAt,
  });

  Map<String, dynamic> toRow() => {
        'id': id,
        'place_name': name,
        'category_code': category.serverCode,
        'latitude': latitude,
        'longitude': longitude,
        'address': address,
        'road_address': roadAddress,
        'description': description,
        'phone': phone,
        'photo_url': photoUrl,
        'created_at': createdAt,
        'updated_at': updatedAt,
        'request_id': requestId,
        'request_status': requestStatus,
        'request_synced_at': requestSyncedAt,
      };

  factory LocalPlace.fromRow(Map<String, Object?> row) {
    return LocalPlace(
      id: row['id'] as String,
      name: row['place_name'] as String,
      category: PlaceCategory.fromServer(row['category_code'] as String) ??
          PlaceCategory.other,
      latitude: (row['latitude'] as num).toDouble(),
      longitude: (row['longitude'] as num).toDouble(),
      address: row['address'] as String?,
      roadAddress: row['road_address'] as String?,
      description: row['description'] as String?,
      phone: row['phone'] as String?,
      photoUrl: row['photo_url'] as String?,
      createdAt: row['created_at'] as int,
      updatedAt: row['updated_at'] as int,
      requestId: row['request_id'] as String?,
      requestStatus: row['request_status'] as String?,
      requestSyncedAt: row['request_synced_at'] as int?,
    );
  }
}
```

### 5. PlaceChangeRequest 관련 모델

`brd_app/lib/features/place/data/model/place_change_request.dart`

```dart
enum PlaceChangeRequestType { create, updateCoordinates, updateInfo;

  String get serverCode {
    switch (this) {
      case PlaceChangeRequestType.create: return 'CREATE';
      case PlaceChangeRequestType.updateCoordinates: return 'UPDATE_COORDINATES';
      case PlaceChangeRequestType.updateInfo: return 'UPDATE_INFO';
    }
  }

  static PlaceChangeRequestType fromServer(String s) {
    switch (s) {
      case 'CREATE': return PlaceChangeRequestType.create;
      case 'UPDATE_COORDINATES': return PlaceChangeRequestType.updateCoordinates;
      case 'UPDATE_INFO': return PlaceChangeRequestType.updateInfo;
      default: throw ArgumentError('unknown type: $s');
    }
  }
}

enum PlaceChangeRequestStatus { pending, approved, rejected;

  String get label {
    switch (this) {
      case PlaceChangeRequestStatus.pending: return '대기 중';
      case PlaceChangeRequestStatus.approved: return '승인됨';
      case PlaceChangeRequestStatus.rejected: return '거절됨';
    }
  }

  static PlaceChangeRequestStatus fromServer(String s) {
    switch (s) {
      case 'PENDING': return PlaceChangeRequestStatus.pending;
      case 'APPROVED': return PlaceChangeRequestStatus.approved;
      case 'REJECTED': return PlaceChangeRequestStatus.rejected;
      default: throw ArgumentError('unknown status: $s');
    }
  }
}

class PlaceChangeRequest {
  final String id;
  final PlaceChangeRequestType type;
  final String? targetPlaceId;
  final String? targetPlaceName;
  final Map<String, dynamic> payload;
  final PlaceChangeRequestStatus status;
  final String? reviewNote;
  final DateTime? reviewedAt;
  final DateTime createdAt;

  const PlaceChangeRequest({
    required this.id,
    required this.type,
    this.targetPlaceId,
    this.targetPlaceName,
    required this.payload,
    required this.status,
    this.reviewNote,
    this.reviewedAt,
    required this.createdAt,
  });

  factory PlaceChangeRequest.fromJson(Map<String, dynamic> json) {
    return PlaceChangeRequest(
      id: json['id'] as String,
      type: PlaceChangeRequestType.fromServer(json['type'] as String),
      targetPlaceId: json['targetPlaceId'] as String?,
      targetPlaceName: json['targetPlaceName'] as String?,
      payload: Map<String, dynamic>.from(json['payload'] as Map),
      status: PlaceChangeRequestStatus.fromServer(json['status'] as String),
      reviewNote: json['reviewNote'] as String?,
      reviewedAt: json['reviewedAt'] != null
          ? DateTime.parse(json['reviewedAt'] as String) : null,
      createdAt: DateTime.parse(json['createdAt'] as String),
    );
  }
}

/// 어드민 목록/상세 응답용 (요청자 정보 포함)
class AdminPlaceChangeRequest {
  final String id;
  final PlaceChangeRequestType type;
  final String? targetPlaceId;
  final String? targetPlaceName;
  final String requesterId;
  final String? requesterNickname;
  final Map<String, dynamic> payload;
  final PlaceChangeRequestStatus status;
  final String? reviewNote;
  final DateTime? reviewedAt;
  final DateTime createdAt;

  const AdminPlaceChangeRequest({
    required this.id,
    required this.type,
    this.targetPlaceId,
    this.targetPlaceName,
    required this.requesterId,
    this.requesterNickname,
    required this.payload,
    required this.status,
    this.reviewNote,
    this.reviewedAt,
    required this.createdAt,
  });

  factory AdminPlaceChangeRequest.fromJson(Map<String, dynamic> json) {
    return AdminPlaceChangeRequest(
      id: json['id'] as String,
      type: PlaceChangeRequestType.fromServer(json['type'] as String),
      targetPlaceId: json['targetPlaceId'] as String?,
      targetPlaceName: json['targetPlaceName'] as String?,
      requesterId: json['requesterId'] as String,
      requesterNickname: json['requesterNickname'] as String?,
      payload: Map<String, dynamic>.from(json['payload'] as Map),
      status: PlaceChangeRequestStatus.fromServer(json['status'] as String),
      reviewNote: json['reviewNote'] as String?,
      reviewedAt: json['reviewedAt'] != null
          ? DateTime.parse(json['reviewedAt'] as String) : null,
      createdAt: DateTime.parse(json['createdAt'] as String),
    );
  }
}
```

`brd_app/lib/features/place/data/repository/place_change_request_repository.dart`

```dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../../../core/network/dio_client.dart';
import '../model/place_change_request.dart';

final placeChangeRequestRepositoryProvider =
    Provider<PlaceChangeRequestRepository>((ref) {
  return PlaceChangeRequestRepository(ref.watch(dioProvider));
});

class PlaceChangeRequestRepository {
  final Dio _dio;
  PlaceChangeRequestRepository(this._dio);

  /// 요청 생성. type별 payload 규격은 백엔드 계약 참조.
  Future<PlaceChangeRequest> create({
    required PlaceChangeRequestType type,
    String? targetPlaceId,
    required Map<String, dynamic> payload,
  }) async {
    final res = await _dio.post(
      '/place-change-requests',
      data: {
        'type': type.serverCode,
        if (targetPlaceId != null) 'targetPlaceId': targetPlaceId,
        'payload': payload,
      },
    );
    return PlaceChangeRequest.fromJson(res.data['data'] as Map<String, dynamic>);
  }

  Future<List<PlaceChangeRequest>> listMine() async {
    final res = await _dio.get('/place-change-requests/mine');
    final list = res.data['data'] as List;
    return list
        .map((e) => PlaceChangeRequest.fromJson(e as Map<String, dynamic>))
        .toList();
  }

  // ============================================================
  // Admin
  // ============================================================

  Future<List<AdminPlaceChangeRequest>> adminList({String status = 'PENDING'}) async {
    final res = await _dio.get(
      '/admin/place-change-requests',
      queryParameters: {'status': status},
    );
    final list = res.data['data'] as List;
    return list
        .map((e) => AdminPlaceChangeRequest.fromJson(e as Map<String, dynamic>))
        .toList();
  }

  Future<AdminPlaceChangeRequest> adminApprove(String id, {String? note}) async {
    final res = await _dio.post(
      '/admin/place-change-requests/$id/approve',
      data: {if (note != null) 'note': note},
    );
    return AdminPlaceChangeRequest.fromJson(
        res.data['data'] as Map<String, dynamic>);
  }

  Future<AdminPlaceChangeRequest> adminReject(String id, {String? note}) async {
    final res = await _dio.post(
      '/admin/place-change-requests/$id/reject',
      data: {if (note != null) 'note': note},
    );
    return AdminPlaceChangeRequest.fromJson(
        res.data['data'] as Map<String, dynamic>);
  }
}
```

### 6. CourseMapScreen — 로컬 place 마커 합치기

`brd_app/lib/features/course/presentation/course_map_screen.dart` (817줄)

수정 포인트:

- `allPlacesProvider` 옆에 `localPlacesProvider` 신설 (FutureProvider<List<LocalPlace>>)
- `_syncMarkers` 로직: 서버 place 마커 그린 후 로컬 place 마커도 그림. 로컬은 아이콘/색 다르게 (예: 반투명 마커 or 뱃지 아이콘)
- 로컬 마커 탭 시 하단 시트에 "내가 저장한 장소" 뱃지 + "공개 등록 요청" or "요청 상태(PENDING/REJECTED)" 표시
- 마커 id 규칙: 서버는 `place-{uuid}`, 로컬은 `local-{uuid}`로 prefix 구분

`brd_app/lib/features/place/domain/local_place_provider.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../data/local/local_place_repository.dart';
import '../data/model/local_place.dart';

final localPlacesProvider = FutureProvider<List<LocalPlace>>((ref) async {
  return ref.watch(localPlaceRepositoryProvider).listActive();
});
```

### 7. PlaceSearchAddScreen — 로컬 저장 전환

`brd_app/lib/features/place/presentation/place_search_add_screen.dart`

`_ConfirmStepState._save()` 로직 교체:

```dart
Future<void> _save() async {
  final name = _nameController.text.trim();
  if (name.isEmpty) { /* 기존 안내 */ return; }
  if (_saving) return;
  setState(() => _saving = true);
  try {
    final now = DateTime.now().millisecondsSinceEpoch;
    final local = LocalPlace(
      id: const Uuid().v4(),
      name: name,
      category: _category,
      latitude: _center.latitude,
      longitude: _center.longitude,
      address: widget.candidate.address,
      roadAddress: widget.candidate.roadAddress,
      createdAt: now,
      updatedAt: now,
    );
    await ref.read(localPlaceRepositoryProvider).insert(local);
    ref.invalidate(localPlacesProvider);
    if (!mounted) return;
    Navigator.of(context).pop(true);
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('내 지도에 저장했어요. 공개 등록은 상세에서.')),
    );
  } finally {
    if (mounted) setState(() => _saving = false);
  }
}
```

`uuid: ^4.0.0` 이미 있음 (Phase 1에서 추가됨).

### 8. LocalPlaceDetailScreen (신규)

`brd_app/lib/features/place/presentation/local_place_detail_screen.dart`

라우트: `/local-places/:id`

구성:
- AppBar: 장소명
- 상단: 지도 미니뷰(마커 하나) or 좌표/카테고리 텍스트
- 정보: 이름, 카테고리, 주소, 설명
- 액션 버튼:
  - **요청 상태 없음**: "공개 등록 요청" (primary), "삭제" (destructive)
  - **PENDING**: 뱃지 "검토 대기 중" (회색), "요청 상태 새로고침"
  - **REJECTED**: 뱃지 "거절됨" (빨강) + reviewNote 표시, "다시 요청" (새 request 생성), "삭제"
  - **APPROVED**: 뱃지 "승인됨 (공개 지도에 등록됨)", "로컬에서 제거" (hardDelete)

"공개 등록 요청" 로직:
```dart
final req = await ref.read(placeChangeRequestRepositoryProvider).create(
  type: PlaceChangeRequestType.create,
  payload: {
    'clientUuid': local.id,
    'placeName': local.name,
    'category': local.category.serverCode,
    'latitude': local.latitude,
    'longitude': local.longitude,
    'address': local.address,
    'roadAddress': local.roadAddress,
    'description': local.description,
    'phone': local.phone,
    'photoUrl': local.photoUrl,
  },
);
await ref.read(localPlaceRepositoryProvider).attachRequest(local.id, req.id);
ref.invalidate(localPlacesProvider);
```

에러 처리:
- 429 (`PLACE_REQUEST_LIMIT_EXCEEDED`) → "대기 중인 요청이 많아요. 처리 후 다시 시도해주세요."
- 그 외 → 일반 에러 스낵바

### 9. place_info_edit_screen / place_coordinate_edit_screen — 요청 생성으로

기존 `PlaceInfoEditScreen`, `PlaceCoordinateEditScreen`은 서버 place 대상. `_save()` 로직 교체:

**PlaceCoordinateEditScreen**:
```dart
Future<void> _save() async {
  if (_saving) return;
  setState(() => _saving = true);
  try {
    final req = await ref.read(placeChangeRequestRepositoryProvider).create(
      type: PlaceChangeRequestType.updateCoordinates,
      targetPlaceId: widget.place.id,
      payload: {
        'latitude': _center.latitude,
        'longitude': _center.longitude,
      },
    );
    if (!mounted) return;
    // 어드민 요청은 서버에서 즉시 자동 승인 → 응답 status로 스낵바 분기
    final isApproved = req.status == PlaceChangeRequestStatus.approved;
    if (isApproved) ref.invalidate(allPlacesProvider);
    Navigator.of(context).pop(true);
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text(isApproved
            ? '좌표가 즉시 반영되었어요.'
            : '좌표 수정 요청을 보냈어요. 어드민 승인 후 반영됩니다.'),
      ),
    );
  } on DioException catch (e) {
    if (!mounted) return;
    final status = e.response?.statusCode;
    String msg;
    if (status == 409) {
      msg = '이 장소에 대한 대기 중인 요청이 이미 있어요.';
    } else {
      msg = '요청 실패: ${status ?? '네트워크 오류'}';
    }
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(msg), backgroundColor: _originalColor),
    );
  } finally {
    if (mounted) setState(() => _saving = false);
  }
}
```

버튼 라벨: "이 위치로 저장" → "좌표 수정 요청"

**PlaceInfoEditScreen** 동일 패턴 (type: updateInfo, payload: {placeName, category}).

### 10. MyPlaceRequestsScreen (신규)

`brd_app/lib/features/place/presentation/my_place_requests_screen.dart`

라우트: `/my-place-requests`

- FutureBuilder 또는 FutureProvider로 `listMine` 호출
- 리스트 아이템: type 아이콘(+/📍/✏️) + 대상 place명(UPDATE_*) or "새 장소: {name}"(CREATE) + 상태 뱃지 + 생성 시각
- 상태별 색: PENDING 회색, APPROVED 초록, REJECTED 빨강
- 탭 시 payload 요약 다이얼로그 + reviewNote (있으면)

### 11. Admin 화면

`brd_app/lib/features/admin/presentation/admin_place_requests_screen.dart`

라우트: `/admin/place-requests`

- 상단 세그먼트 컨트롤: 대기중 / 승인됨 / 거절됨
- 리스트: type / requester nickname / target 이름 or 새 장소명 / 생성 시각
- 각 아이템 탭 → 상세 화면

`brd_app/lib/features/admin/presentation/admin_place_request_detail_screen.dart`

라우트: `/admin/place-requests/:id`

- payload 시각화:
  - CREATE: 지도 마커 + 정보 카드
  - UPDATE_COORDINATES: 지도에 before(빨강) / after(파랑) 두 마커 + 이동 거리 표시
  - UPDATE_INFO: before/after 이름·카테고리 대비 표
- 하단 액션: "승인" (primary) / "거절" (destructive)
- 거절 시 Cupertino 다이얼로그로 사유 입력 (선택이지만 강력 권장)

### 12. 설정 화면 노출 조건

`brd_app/lib/features/settings/presentation/settings_screen.dart` (경로는 실제 확인)

로그인 유저(local guest 아님)일 때:
- "내 장소 요청" 카드 상시 노출 → `/my-place-requests`

`user.isAdmin`일 때:
- "요청 관리 (N건 대기)" 카드 노출 → `/admin/place-requests`
- N건은 optionally FutureProvider로 pre-fetch (부담되면 그냥 라벨만)

### 13. main_shell 확장 메뉴 (변경 여부 결정)

기존 4버튼: 찾아보기 / 라이딩 코스 / 내 바이크 / 뱅킹각.
어드민 진입은 설정 화면에 두는 게 자연스러움 (main_shell은 4버튼 유지).

### 14. 라우트 추가

`brd_app/lib/core/router/app_router.dart`

shell 밖 라우트로 추가:
```dart
GoRoute(path: '/local-places/:id', builder: (_, s) => LocalPlaceDetailScreen(id: s.pathParameters['id']!)),
GoRoute(path: '/my-place-requests', builder: (_, __) => const MyPlaceRequestsScreen()),
GoRoute(path: '/admin/place-requests', builder: (_, __) => const AdminPlaceRequestsScreen()),
GoRoute(path: '/admin/place-requests/:id', builder: (_, s) => AdminPlaceRequestDetailScreen(id: s.pathParameters['id']!)),
```

어드민 라우트 가드: `redirect`로 `!user.isAdmin`이면 홈으로. `authProvider`에서 role 읽어서 판단.

### 15. Auth logout 반영

이미 `AppDatabase.clearAll()`이 `bikes`만 지우고 있음. 위 2번 마이그레이션 스텝에서 `local_places` 삭제 추가.

## 상태별 뱃지 스타일 제안

| 상태 | 색 | 아이콘 | 문구 |
|-----|----|--------|------|
| PENDING | `Color(0xFF8E8E93)` (회색) | `Icons.hourglass_top` | "검토 대기 중" |
| APPROVED | `Color(0xFF34C759)` (iOS 초록) | `Icons.check_circle` | "승인됨" |
| REJECTED | `Color(0xFFFF3B30)` (iOS 빨강) | `Icons.cancel` | "거절됨" |

로컬 마커 (요청 없음): 반투명 처리 (alpha 0.6) + 우상단에 작은 흰 원 뱃지 "내"

## 주의사항

1. **role 반영 타이밍**: 백엔드 UserResponse.role은 매 요청 DB lookup 방식 (JWT claim 아님)이라 서버 인가는 즉시 반영. 그러나 앱은 로그인 시점의 role을 캐시(authProvider)해서 어드민 메뉴 노출을 판단하므로, 어드민 지정 후 앱에서 인식하려면 **로그아웃 → 재로그인** 필요. 대안: 설정 화면 진입 시 GET /users/me 재호출해서 authProvider의 user만 갱신.
2. **로컬 pin 승인 후 서버 pull**: 승인되면 `places` 테이블에 같은 UUID로 뜬다. 앱은 다음 `allPlacesProvider` invalidate 시 서버 것으로 그림. 로컬 것은 `request_status='APPROVED'`로 유지 → 사용자가 상세에서 "로컬에서 제거" 액션 시 hardDelete. 자동 정리하고 싶으면 pull 후 로컬에서 APPROVED인 것 자동 삭제 로직 추가 (부담 낮은 편).
3. **낙관적 UI 없음**: 요청 생성은 서버 통신 필수. 오프라인이면 요청 실패 → 안내. 로컬 pin 저장은 오프라인 OK.
4. **UPDATE 요청 중 대상 place 지워짐**: 백엔드는 CASCADE라 요청 자체가 사라짐. 앱은 상세 재조회 시 404 → "요청이 만료되었습니다" 안내.
5. **어드민 화면 오프라인**: 어드민 모든 액션은 서버 필수. 오프라인이면 로딩 실패 표시.
6. **로컬 pin 카테고리 편집**: 최소 스코프에서는 편집 UI 생략(삭제 후 재등록). 필요 시 LocalPlaceDetailScreen에 편집 액션 추가.
7. **RequestStatus 새로고침**: MyPlaceRequestsScreen 진입 시 항상 서버 재조회 + 로컬 pin의 request_status도 syncRequestStatus로 반영. Pull-to-refresh 지원.
8. **어드민 자동 승인 응답 처리**: 어드민이 편집/등록하면 서버가 요청 생성 후 같은 트랜잭션에서 자동 승인해 응답 `status=APPROVED`로 내려줌. 앱은 응답 status로 스낵바 분기("즉시 반영" vs "승인 대기") + APPROVED면 `allPlacesProvider` invalidate로 지도 즉시 갱신. 3개 편집 화면(place_coordinate_edit / place_info_edit / local_place_detail의 공개 등록) 모두 동일 패턴.
