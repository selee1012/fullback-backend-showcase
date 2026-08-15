# 05. 백엔드 설계 상세

> 담당: 이성은 — Spring Boot API 전반 설계·구현

---

## 1. 공간 데이터 처리

### 1.1 왜 애플리케이션이 아니라 DB에서 거리를 계산했는가

초기에는 조회한 코스 목록을 Java에서 Haversine으로 정렬하는 방식을 고려했습니다. 그러나 이 방식은 **전체 코스를 메모리로 올린 뒤 정렬**해야 하므로 데이터가 늘수록 선형으로 느려집니다.

MySQL Spatial Extension의 `ST_Distance_Sphere`를 native query로 사용하면 DB가 공간 인덱스를 활용해 필터링·정렬을 처리합니다.

```sql
-- 거리순 정렬
SELECT * FROM course c
ORDER BY ST_Distance_Sphere(c.start_location, ST_GeomFromText(:point, 4326));

-- 반경(미터) 내 코스만 (지도 마커용)
SELECT * FROM course c
WHERE ST_Distance_Sphere(c.start_location, ST_GeomFromText(:point, 4326)) <= :distance;
```

동일한 패턴을 `Place.address`, `Note.note_place`에도 적용해 반경 조회를 통일했습니다.

### 1.2 좌표 순서 — 실제로 버그를 만든 지점

| 시스템 | 순서 |
| --- | --- |
| JTS / MySQL `POINT` | `(x=경도, y=위도)` |
| Google Maps API | `(lat=위도, lng=경도)` |

두 규약이 정반대이기 때문에, 변환 책임을 `CourseDistanceService` 한 곳에 모으고 메서드 시그니처에 `originLat` / `originLng`처럼 **의미를 이름으로 못박아** 호출자가 순서를 헷갈리지 않게 했습니다.

```java
// JTS Point는 (x=경도, y=위도) → GoogleMapsClient는 (lat, lng) 순서이므로 여기서 변환
return googleMapsClient.getWalkingDistance(
        userY,        // 사용자 위도
        userX,        // 사용자 경도
        start.getY(), // 코스 시작 위도
        start.getX()  // 코스 시작 경도
);
```

### 1.3 `GeometryFactory`를 Bean으로 관리

`new GeometryFactory()`를 서비스마다 직접 만들면 SRID 설정이 어긋날 위험이 있어, `GeometryConfig`에서 SRID 4326으로 고정한 단일 Bean을 주입받도록 통일했습니다.

---

## 2. N+1 문제 대응

코스 목록 응답에는 썸네일(첫 번째 이미지)이 필요합니다. 아무 처리 없이 `findAll()` 후 `course.getImages()`를 호출하면 코스 개수만큼 추가 쿼리가 발생합니다.

| 대상 | 대응 |
| --- | --- |
| `Course` 조회 | `@EntityGraph(attributePaths = {"images"})` — 이미지 fetch join |
| `CourseReview` 조회 | `@EntityGraph(attributePaths = {"guest"})` — 응답에 작성자 UUID가 포함되므로 |
| `PlaceRecommend` 조회 | `@EntityGraph(attributePaths = {"place", "place.images"})` — 2단계 연관까지 |
| `Place` 전체/태그 조회 | JPQL `LEFT JOIN FETCH` |
| 전역 | `default_batch_fetch_size: 100` |

**남은 한계 (문서에 명시)** — native query에는 `@EntityGraph`가 적용되지 않습니다. 따라서 반경 조회(`findNearby`)는 이미지가 LAZY 상태로 남고, `open-in-view: false` 설정과 맞물려 직렬화 시점에 `LazyInitializationException` 위험이 있습니다. 현재는 마커 응답에 이미지를 포함하지 않는 방식으로 회피 중이며, 근본 해결은 fetch join 전용 메서드 추가입니다. **알면서 남겨둔 부채**로 코드 주석과 문서에 함께 기록해 두었습니다.

> `open-in-view`를 `false`로 둔 것은 의도적입니다. 기본값(true)은 뷰 렌더링까지 커넥션을 붙잡고 있어 부하 상황에서 커넥션 풀이 빠르게 고갈됩니다. 대신 지연 로딩 예외를 개발 단계에서 잡아내는 쪽을 택했습니다.

---

## 3. 다국어 스키마 설계

번역 API를 런타임에 호출하는 대신 **컬럼 이원화 + enum 인덱스 대응** 방식을 선택했습니다.

```java
public enum Level   { High, Normal, Easy }
public enum LevelCh { 困难, 适中, 轻松 }      // Level과 인덱스 1:1 대응

public enum Tag   { Landmark, Night_View, Forest_Trail, Field,
                    River, Spring_Flower, Near_Subway, Restaurant }
public enum TagCh { 地标, 夜景, 森林小道, 田野,
                    河边, 春花, 地铁附近, 美食 }
```

| 방식 | 장점 | 단점 |
| --- | --- | --- |
| 런타임 번역 API | 언어 추가가 쉬움 | 응답 지연, 비용, 고유명사 오역, 외부 장애 전파 |
| **컬럼 이원화 (채택)** | 조회 경로에 외부 의존성 0, 큐레이션 품질 보장 | 언어 추가 시 스키마 변경 필요 |

산책로 이름·설명은 **관광 콘텐츠**라서 기계 번역 품질이 그대로 서비스 품질이 됩니다. 언어가 영·중 2종으로 고정된 요구사항이었으므로 품질과 응답 속도를 택했습니다.

**필터링은 항상 영문 enum 기준(`tag_eg`)으로 수행**하고 응답 시점에만 언어를 선택합니다. 덕분에 `language` 파라미터가 바뀌어도 쿼리는 동일합니다.

---

## 4. 인증 없는 환경에서의 사용자 식별과 어뷰징 방지

로그인이 없으므로 `Guest.uuid`(`BINARY(16)`, unique)가 유일한 식별자입니다.

```
POST /api/guest  →  UUID 발급  →  이후 모든 쓰기 요청 body에 guestUuid 포함
```

**중복 행위 차단** — 좋아요는 카운트만 올리는 게 아니라 이력을 남깁니다.

```java
courseLikeHistoryRepository.findByCourseIdAndGuestId(courseId, guest.getId())
        .ifPresentOrElse(
                history -> {                       // 이미 눌렀음 → 취소
                    courseLikeHistoryRepository.delete(history);
                    course.decreaseLikeCount();
                    likedHolder[0] = false;
                },
                () -> {                            // 처음 → 추가
                    courseLikeHistoryRepository.save(CourseLikeHistory.create(course, guest));
                    course.increaseLikeCount();
                    likedHolder[0] = true;
                }
        );
```

**개선 이력** — 초기 구현은 토글 처리 후 `existsByCourseIdAndGuestId()`를 다시 호출해 `liked` 상태를 확인했습니다. 이미 분기 안에서 결과가 확정되는데 불필요한 쿼리를 한 번 더 날리는 구조였고, 앞서 대입한 값이 의미 없이 덮어써지는 버그도 있었습니다. 분기 내부에서 결과를 즉시 확정하도록 바꿔 **요청당 쿼리 1회를 제거**했습니다.

**남은 한계** — 애플리케이션 레벨 체크이므로 동일 게스트의 빠른 연타 시 race condition 가능성이 있습니다. DB 레벨 unique 제약 `(course_id, guest_id)` 추가가 근본 해결이며, 코드 주석에 후속 과제로 명시해 두었습니다.

---

## 5. 예외 처리 · 응답 규격

```
Service ── throw BusinessException(ErrorCode.COURSE_NOT_FOUND, "course id=1")
              │
              ▼
GlobalExceptionHandler ── ErrorCode → HttpStatus + { code, message } 매핑
              │
              ▼
ApiResponse.error(...)  ── { "success": false, "error": {...} }
```

- **`ErrorCode` enum 한 곳에 집중**: HTTP 상태·코드·사용자 메시지를 한 묶음으로 관리해 도메인이 늘어도 응답 규격이 흔들리지 않습니다.
- **모든 응답을 `ApiResponse<T>`로 통일**: 프론트엔드가 성공/실패 분기를 한 가지 방식으로만 처리합니다.
- **`@Valid` 1차 검증 + Service 2차 안전망**: DTO에서 `@NotBlank`, `@Pattern`, `@Size`로 형식을 거르고, Service의 `parseXxx()` 헬퍼가 enum 변환 실패를 400으로 변환합니다.

---

## 6. 트랜잭션 정책

```java
@Service
@Transactional(readOnly = true)   // 클래스 기본: 읽기 전용
public class CourseService {

    @Transactional                // 변경 메서드에만 개별 부착
    public CourseDto.LikeResponse toggleLike(...) { ... }
}
```

읽기 전용을 기본값으로 두면 Hibernate가 flush를 건너뛰고 더티 체킹 스냅샷을 만들지 않아 조회 비용이 줄어듭니다. **이 서비스는 조회가 압도적으로 많은 워크로드**(코스 목록·상세·마커·장소)이므로 기본값을 읽기 전용으로 잡는 편이 유리했습니다.

언어 변경처럼 단순한 갱신은 별도 `save()` 호출 없이 **더티 체킹**으로 반영합니다.

```java
@Transactional
public void changeLanguage(GuestDto.LanguageUpdateRequest request) {
    Guest guest = findByUuid(request.getGuestUuid());
    guest.changeLanguage(parseLanguage(request.getLanguage()));   // flush 시점에 자동 UPDATE
}
```

---

## 7. 도메인 간 의존 규칙

다른 도메인의 Repository를 직접 주입하지 않고 **Service를 경유**합니다.

```
CourseService ──▶ GuestService.findByUuid(String)   ✅
CourseService ──▶ GuestRepository                   ❌ (금지)
```

`GuestService.findByUuid(String)`가 UUID 문자열 파싱과 존재 여부 검증(`INVALID_GUEST_UUID` / `GUEST_NOT_FOUND`)까지 책임지므로, 호출하는 쪽은 UUID 형식 처리를 신경 쓸 필요가 없습니다. 검증 로직이 여러 도메인에 복제되는 것을 막는 장치입니다.

`CourseService.findEntityById()`도 같은 목적입니다 — Note·Review가 코스를 참조할 때 이 메서드를 재사용하므로 "존재하지 않는 코스" 처리가 한 곳에만 존재합니다.

---

## 8. 외부 API 장애가 서비스를 멈추지 않게 하기

외부 연동(Google Maps, Solar LLM)은 **모두 실패를 전제로** 설계했습니다.

| 장치 | 내용 |
| --- | --- |
| 타임아웃 명시 | connect 3s / read 5s — 무한 대기로 스레드가 묶이는 것 방지 |
| 폴백 경로 | Google Maps 실패 → Haversine 직선거리 × 보정계수 1.3, 도보 4.5km/h로 추정 |
| 기능 토글 | `google.maps.enabled=false` — 서버가 외부 지도 API를 호출하지 않는 것이 **기본값** |
| 호출 가드 | Solar LLM은 분당/일일 호출 예산과 동시성 제한 → [06. LLM 비속어 필터링](06-llm-moderation.md) |

**설계 전환** — 초기에는 서버가 Google Distance Matrix API로 도보 거리를 계산했습니다. 그러나 ① 외부 API 지연이 그대로 응답 지연이 되고, ② 호출량이 곧 비용이며, ③ 프론트엔드가 이미 Google Maps JS SDK로 지도를 그리고 있다는 점을 고려해 **경로 계산을 프론트엔드로 이관**했습니다. 서버는 코스 시작점 좌표만 내려주고, 추천 알고리즘 내부 계산에는 직선거리 추정만 사용합니다. 이 전환이 부하 테스트에서 평균 16.9ms 응답을 낼 수 있었던 배경 중 하나입니다.

기존 동작이 필요할 때를 대비해 `enabled=true`로 켜면 API 호출 경로가 복구되도록 호환성을 남겨 두었습니다.

---

## 9. 테스트

| 계층 | 방식 |
| --- | --- |
| Controller | `@WebMvcTest` + MockMvc — 요청 파라미터 검증, 응답 규격, 상태 코드 |
| Service | Mockito 단위 테스트 — 분기 로직(정렬·필터·토글·예외) 검증 |
| 전역 | `ApiResponseTest`, `GlobalExceptionHandlerTest` — 공통 응답/예외 매핑 |
| 외부 연동 | `GoogleMapsClientTest` — 폴백 경로 동작 검증 |
| DB | H2 인메모리 (테스트 전용 의존성) |

6개 도메인 모두 Controller·Service 테스트를 쌍으로 작성했습니다.

---

## 10. 구현 상태에 대한 정직한 기록

포트폴리오 목적상, 문서상 설계와 최종 코드가 다른 지점을 명시합니다.

| 항목 | 설계 의도 | 코드 구현 |
| --- | --- | --- |
| 위치 검증 활성화 | 상시 활성 | **시연·부하 테스트 편의를 위해 일시 비활성** (검증 블록을 주석으로 보존, 주석 해제만으로 복구 가능) |
| 태그 필터 수행 위치 | DB 레벨 | 일부 정렬 경로에서 native query 사용으로 애플리케이션 레벨 stream 필터 |

두 번째 항목은 코스 수가 100건대인 현재 규모에서는 문제되지 않지만, 데이터가 늘면 native query에 `WHERE tag_eg = ?`를 추가해 DB로 내려야 합니다. 주석에 판단 근거와 함께 남겨 두었습니다.

> 첫 번째 항목은 **임시 비활성을 코드 주석으로 처리한 것 자체가 아쉬운 선택**이었습니다. `note.location-check.enabled` 같은 설정 플래그로 뒀다면 환경별로 켜고 끌 수 있었고, 주석 해제를 잊어 운영에 반영되지 않는 위험도 없었을 것입니다. → [08. 회고](08-retrospective.md#4-아쉬운-점과-배운-것)
