# 04. REST API 명세

> Spring Boot 백엔드가 제공하는 전체 엔드포인트입니다. 모든 컨트롤러는 springdoc-openapi로 문서화되어 `/swagger-ui.html`에서 확인할 수 있습니다.

---

## 1. 공통 응답 규격

모든 응답은 제네릭 래퍼 `ApiResponse<T>`로 감쌉니다. 프론트엔드가 성공/실패 분기를 한 가지 방식으로만 처리하도록 통일한 규격입니다.

**성공**
```json
{
  "success": true,
  "data": { }
}
```

**실패**
```json
{
  "success": false,
  "error": {
    "code": "CO001",
    "message": "존재하지 않는 산책로입니다."
  }
}
```

---

## 2. 엔드포인트 목록

### Guest — 비인증 사용자

| Method | Path | 설명 |
| --- | --- | --- |
| `POST` | `/api/guest` | 게스트 생성 (UUID 발급) — **201 Created** |
| `PUT` | `/api/guest/language` | 게스트 응답 언어 변경 |

<details>
<summary>요청/응답 예시</summary>

```http
POST /api/guest
{ "language": "ENG" }

201 Created
{
  "success": true,
  "data": {
    "guestId": 1,
    "uuid": "550e8400-e29b-41d4-a716-446655440000",
    "setLanguage": "ENG"
  }
}
```

```http
PUT /api/guest/language
{ "guestUuid": "550e8400-...", "language": "CHI" }
```
</details>

---

### Course — 산책로

| Method | Path | 설명 |
| --- | --- | --- |
| `GET` | `/api/courses` | 코스 목록 (정렬 + 태그 필터) |
| `GET` | `/api/course/{courseId}` | 코스 상세 |
| `GET` | `/api/course/{courseId}/images` | 코스 이미지 목록 |
| `GET` | `/api/course/{courseId}/reviews` | 코스별 리뷰 목록 |
| `PUT` | `/api/course/{courseId}/like` | 좋아요 토글 |

**`GET /api/courses` 쿼리 파라미터**

| 파라미터 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `user_x` | Double | - | 사용자 경도 (`sort=distance` 시 필수) |
| `user_y` | Double | - | 사용자 위도 (`sort=distance` 시 필수) |
| `sort` | String | `popular` | `popular`(좋아요순) / `distance`(가까운순) |
| `tag` | String | - | `Landmark` `Night_View` `Forest_Trail` `Field` `River` `Spring_Flower` `Near_Subway` `Restaurant` |
| `language` | String | `ENG` | `ENG` / `CHI` |

<details>
<summary>응답 예시</summary>

```json
// GET /api/courses?sort=distance&user_x=126.978&user_y=37.5665&language=ENG
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name": "Bukhansan·Seongbuk Course",
      "level": "Normal",
      "hours": "02:50",
      "distance": 6.0,
      "likeCount": 12,
      "startLocation": { "x": 127.0142, "y": 37.6205 },
      "courseImg": "https://.../thumbnail.jpg"
    }
  ]
}
```

```json
// GET /api/course/1?language=ENG  — 상세는 경로(route)와 접근 안내를 추가로 포함
{
  "success": true,
  "data": {
    "id": 1,
    "name": "Bukhansan·Seongbuk Course",
    "level": "Normal",
    "hours": "02:50",
    "distance": 6.0,
    "description": "A quiet forest trail for contemplation ...",
    "likeCount": 12,
    "startLocation": { "x": 127.0142, "y": 37.6205 },
    "route": "LINESTRING(127.0142 37.6205, ...)",
    "images": ["https://.../1.jpg", "https://.../2.jpg"],
    "startAccess": "Hyeongjebong entrance",
    "endAccess": "Hwagyesa entrance",
    "reviewsUrl": "/api/course/1/reviews",
    "createReviewUrl": "/api/review"
  }
}
```

```json
// PUT /api/course/1/like   body: { "guestUuid": "550e8400-..." }
{ "success": true, "data": { "liked": true, "likeCount": 13 } }
```
</details>

> **좋아요 토글 설계** — 요청마다 상태를 뒤집습니다. `CourseLikeHistory`에 `(course_id, guest_id)` 이력이 있으면 삭제 후 카운트 감소, 없으면 생성 후 증가. 결과(`liked`)를 분기 내부에서 즉시 확정하여 토글 후 재조회 쿼리를 제거했습니다.

---

### Place — POI (맛집 / 랜드마크)

| Method | Path | 설명 |
| --- | --- | --- |
| `GET` | `/api/place` | 장소 목록 (다중 필터) |
| `GET` | `/api/place/{placeId}/images` | 장소 이미지 목록 |

**검색 우선순위** — 파라미터 조합에 따라 3가지 조회 전략으로 분기합니다.

| 순위 | 조건 | 동작 |
| --- | --- | --- |
| 1 | `course_id` 있음 | 해당 코스에 큐레이션된 추천 장소 (`PlaceRecommend`) |
| 2 | `center_x` + `center_y` + `distance` 있음 | 반경 내 장소 (`ST_Distance_Sphere`) |
| 3 | 둘 다 없음 | 전체 장소 |

이후 `tag` 필터를 공통 적용하고, `language`(`ENG`/`CHI`)로 응답 언어를 결정합니다.

---

### Note — 위치 기반 쪽지 (LBS 커뮤니티)

| Method | Path | 설명 |
| --- | --- | --- |
| `POST` | `/api/note` | 쪽지 작성 — **201 Created** |
| `GET` | `/api/notes?x=&y=&distance=` | 반경 내 쪽지 목록 (지도 핀용, **본문 미포함**) |
| `GET` | `/api/note/{noteId}` | 쪽지 상세 (본문 포함) |

<details>
<summary>요청 예시</summary>

```http
POST /api/note
{
  "courseId": 1,
  "guestUuid": "550e8400-...",
  "content": "The forest section after the second bridge is beautiful at sunset.",
  "imageUrl": null,
  "notePlace": { "x": 127.0142, "y": 37.6205 },
  "type": "Normal"
}
```
</details>

**설계 포인트**
- 목록 API는 **본문을 내려주지 않습니다.** 지도에는 핀 위치·타입·작성시각만 표시되고, 본문은 상세 조회에서만 노출됩니다. "그 장소에 가야 읽을 수 있다"는 서비스 컨셉을 API 레벨에서 강제한 설계입니다.
- 작성 시 `SolarModerationClient`로 비속어 검사를 먼저 통과해야 저장됩니다 → [06. LLM 비속어 필터링](06-llm-moderation.md)
- 거리 계산은 `ST_Distance_Sphere`로 DB에서 수행합니다. JTS의 `Point.distance()`는 평면 거리라 위경도에서 미터 단위로 동작하지 않기 때문입니다.

---

### Review — 코스 리뷰

| Method | Path | 설명 |
| --- | --- | --- |
| `GET` | `/api/review/{reviewId}` | 리뷰 상세 |
| `POST` | `/api/review` | 리뷰 작성 (1~500자) — **201 Created** |
| `PUT` | `/api/review/{reviewId}/like` | 리뷰 좋아요 토글 |
| `GET` | `/api/course/{courseId}/reviews?sort=` | 코스별 목록 — `latest`(기본) / `popular` |

리뷰 작성 역시 비속어 검사를 통과해야 저장됩니다.

---

### Map — 지도

| Method | Path | 설명 |
| --- | --- | --- |
| `GET` | `/api/map/config` | 지도 초기 설정 (중심 좌표, 줌, 스타일) |
| `GET` | `/api/map/markers?center_x=&center_y=&distance=` | 반경 내 코스 시작점 마커 (기본 3000m) |

**정책** — 첫 지도 화면에는 **코스 마커만** 노출합니다. Place / Note 마커는 코스 상세 진입 이후 별도 API로 조회하도록 분리했는데, 초기 화면에서 세 종류 마커를 한꺼번에 내려주면 응답 크기와 렌더링 비용이 커지고 사용자도 무엇을 봐야 할지 알기 어렵기 때문입니다.

`/api/map/config`의 중심 좌표·줌 값은 하드코딩 대신 `@ConfigurationProperties`(`map.config.*`)로 분리해 환경별로 바꿀 수 있게 했습니다.

---

## 3. 에러 코드 체계

`[도메인 약자][3자리 숫자]` 형식으로 전역 관리합니다. Service 계층은 `BusinessException(ErrorCode, detail)`만 던지고, `GlobalExceptionHandler`가 HTTP 상태와 응답 본문으로 변환합니다.

| 코드 | HTTP | 메시지 |
| --- | --- | --- |
| `C001` | 400 | 잘못된 입력값입니다. |
| `C002` | 400 | 허용되지 않는 값입니다. |
| `C003` | 405 | 허용되지 않는 HTTP 메서드입니다. |
| `C004` | 400 | 비속어 및 욕설은 사용하지 않아주세요. |
| `C999` | 500 | 서버 오류가 발생했습니다. |
| `G001` | 404 | 존재하지 않는 게스트입니다. |
| `G002` | 400 | 올바르지 않은 게스트 UUID 형식입니다. |
| `G003` | 400 | 지원하지 않는 언어입니다. (ENG, CHI) |
| `CO001` | 404 | 존재하지 않는 산책로입니다. |
| `CO002` | 400 | 허용되지 않는 코스 태그입니다. |
| `CO003` | 400 | 허용되지 않는 정렬 기준입니다. |
| `P001` | 404 | 존재하지 않는 장소입니다. |
| `P002` | 400 | 허용되지 않는 장소 태그입니다. |
| `R001` | 404 | 존재하지 않는 리뷰입니다. |
| `N001` | 404 | 존재하지 않는 쪽지입니다. |
| `N002` | 403 | 쪽지를 읽을 수 있는 위치가 아닙니다. |
| `N003` | 400 | 허용되지 않는 쪽지 타입입니다. |

**왜 이렇게 했는가** — 초기 구현에서는 잘못된 `tag`/`language`/`sort` 문자열이 들어오면 `enum.valueOf()`가 `IllegalArgumentException`을 던져 **500 Internal Server Error**로 응답했습니다. 클라이언트 입력 오류가 서버 장애로 보고되는 문제였습니다. 모든 enum 파싱을 `parseXxx()` 헬퍼로 감싸 `BusinessException`으로 변환하고, **400 Bad Request + 어떤 값이 잘못됐는지 명시**하도록 정리했습니다.
