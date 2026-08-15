# 06. LLM 기반 다국어 비속어 필터링

> 담당: 이성은 — 설계 및 구현 전담

<img src="../assets/screen-profanity-filter.png" alt="비속어 필터 동작 화면" width="480">

---

## 1. 문제

GlobalStep은 회원가입이 없는 서비스입니다. 누구나 UUID 하나로 쪽지와 리뷰를 남길 수 있으므로, **커뮤니티 신뢰도가 서비스의 약점**이 됩니다. 최종보고서에서도 "비인증 LBS 커뮤니티 특성상 발생할 수 있는 악성 유저의 도배 어뷰징"을 한계로 지목했고, 이를 해소하기 위해 도입한 기능입니다.

문제를 어렵게 만든 조건이 세 가지 있었습니다.

1. **다국어** — 영어와 중국어(간체·번체)를 모두 걸러야 합니다. 한국어 금칙어 라이브러리는 쓸 수 없습니다.
2. **우회 표현** — `f.u.c.k`, `fuuuck`, `f u c k`처럼 구분자·반복을 넣은 변형이 단순 문자열 매칭을 빠져나갑니다.
3. **예산** — LLM API는 호출당 비용이 발생하고, 프로젝트는 프리티어·크레딧 한도 안에서 운영해야 했습니다.

---

## 2. 설계 — 2단 방어 구조

```
                    createNote() / createReview()
                              │
                              ▼
                 ┌────────────────────────────┐
                 │  SolarModerationClient     │
                 └────────────┬───────────────┘
                              │
              ┌───────────────┴────────────────┐
              │  ① 호출 가능한가?               │
              │     enabled && apiKey 존재      │
              └───────────────┬────────────────┘
                       yes    │    no
              ┌───────────────┘    └──────────────┐
              ▼                                   │
   ┌────────────────────────┐                     │
   │ ② SolarCallGuard       │  예산/동시성 초과   │
   │   · 동시성 세마포어     │─────────────────────┤
   │   · 분당 rate limit    │                     │
   │   · 일일 budget        │                     │
   └───────────┬────────────┘                     │
               │ 통과                              │
               ▼                                   │
   ┌────────────────────────┐   호출 실패·null·예외 │
   │ ③ Solar LLM 분류       │─────────────────────┤
   │   "BLOCK" or "OK"      │                     │
   └───────────┬────────────┘                     │
               │ 판정                              ▼
               │                    ┌──────────────────────────┐
               │                    │ ④ RuleBasedProfanityFilter│
               │                    │    정규화 + 금칙어 매칭    │
               │                    └──────────────┬───────────┘
               ▼                                   ▼
                    boolean containsProfanity
                              │
                    true → BusinessException(C004)
                           "비속어 및 욕설은 사용하지 않아주세요."
```

**핵심 원칙: 어떤 경로로 실패해도 반드시 판정이 나온다.** LLM이 죽어도, 예산이 떨어져도, 키가 없어도 Rule-Based 필터가 받아냅니다. 커뮤니티 기능 자체가 멈추는 상황을 만들지 않았습니다.

---

## 3. LLM 분류기 설계

Solar(Upstage) `solar-mini`를 **분류기로만** 사용합니다. 생성이 아니라 판정이므로 출력이 짧고 결정적이어야 합니다.

```java
private static final String SYSTEM_PROMPT = """
        You are a strict content moderation classifier for a walking-course community service.
        Decide whether the user's text contains profanity, swearing, slurs, sexual harassment,
        or hateful/abusive language. English and Chinese are in scope.
        Respond with EXACTLY one word and nothing else:
        - "BLOCK" if it contains such language.
        - "OK" if it does not.
        Do not explain. Do not add punctuation.
        """;
```

| 파라미터 | 값 | 이유 |
| --- | --- | --- |
| `temperature` | `0` | 같은 입력에 같은 판정이 나와야 함. 모더레이션에 창의성은 불필요 |
| `max_tokens` | `5` | 응답은 한 단어. 토큰 상한을 조여 **비용과 지연을 동시에 통제** |
| read timeout | `4초` (설정값) | 사용자가 글을 쓰는 흐름을 막지 않는 상한 |

**LLM을 택한 이유** — 금칙어 리스트는 우회 표현과 문맥을 못 잡습니다. 반대로 LLM은 `"This trail is f***ing beautiful"` 같은 문맥까지 판단할 수 있고, **중국어 간체/번체를 별도 사전 없이 커버**합니다. 다국어가 요구사항인 이 프로젝트에서 특히 유리했습니다.

---

## 4. 호출 가드 (`SolarCallGuard`)

LLM 도입에서 가장 현실적인 위험은 **비용 폭주와 스레드 고갈**입니다. 어뷰저가 요청을 반복하면 그대로 API 호출이 됩니다. 이를 막기 위한 3중 제한을 구현했습니다.

```java
// 1) 동시성 — Semaphore
if (!concurrencyPermits.tryAcquire()) return false;   // 대기하지 않고 즉시 폴백

// 2) 일일 예산
if (properties.hasDailyBudget() && dayCount.get() >= properties.dailyBudget()) {
    concurrencyPermits.release();
    return false;
}

// 3) 분당 rate limit
if (properties.hasRateLimit() && minuteCount.get() >= properties.rateLimitPerMin()) {
    concurrencyPermits.release();
    return false;
}
```

| 제한 | 기본값 | 목적 |
| --- | --- | --- |
| `max-concurrency` | 5 | 외부 API 대기로 톰캣 스레드가 묶이는 것 방지 |
| `rate-limit-per-min` | 30 | 순간 폭주 차단 |
| `daily-budget` | 500 | 하루 비용 상한 |

**설계에서 신경 쓴 두 가지**

1. **Fail-open** — 한도에 걸리면 요청을 거절하는 게 아니라 **Rule-Based로 내려갑니다.** 모더레이션은 부가 기능이므로, 여기서 실패했다고 사용자가 글을 못 쓰게 만들면 안 됩니다. 반대로 fail-close였다면 예산 소진 시각 이후 커뮤니티 전체가 마비됩니다.
2. **윈도우 롤오버** — 분/일 카운터를 `Asia/Seoul` 기준으로 리셋합니다. 스케줄러 없이 요청 시점에 현재 분·일을 계산해 갱신하므로 별도 배치가 필요 없습니다.

```java
private void rolloverWindowsIfNeeded() {
    Instant now = Instant.now();
    long nowMinute = now.getEpochSecond() / 60L;
    long nowDay = now.atZone(ZONE).toLocalDate().toEpochDay();
    if (nowMinute != currentMinute) { currentMinute = nowMinute; minuteCount.set(0); }
    if (nowDay != currentDay)       { currentDay = nowDay;       dayCount.set(0); }
}
```

세마포어는 `try/finally`의 `finally`에서 반드시 반납해 예외 발생 시에도 permit이 누수되지 않게 했습니다.

---

## 5. Rule-Based 폴백 (`RuleBasedProfanityFilter`)

LLM이 불가능할 때 동작하는 최후 방어선입니다. 단순 `contains()`가 아니라 **정규화를 거친 뒤 매칭**합니다.

### 5.1 정규화 — 우회 표현 무력화

```java
private static String normalize(String input) {
    // 1단계: 소문자화 후 라틴 문자 / 숫자 / CJK 한자만 남기고 전부 제거
    //        → 공백, 마침표, 별표, 하이픈 같은 구분자 우회를 무력화
    // 2단계: 연속 중복 문자를 하나로 압축
    //        → "fuuuuck" → "fuck"
}
```

| 입력 | 정규화 결과 | 판정 |
| --- | --- | --- |
| `f.u.c.k` | `fuck` | 차단 |
| `f u c k` | `fuck` | 차단 |
| `FUUUUCK` | `fuck` | 차단 |
| `f-u-c-k!!!` | `fuck` | 차단 |

CJK 범위(`U+4E00–U+9FFF`, `U+3400–U+4DBF`)를 명시적으로 보존해 중국어가 정규화 과정에서 날아가지 않도록 했습니다.

### 5.2 금칙어 사전 — 오탐(false positive)을 의도적으로 줄임

중국어 사전을 만들 때 가장 신경 쓴 부분입니다.

> 모호한 단독 한 글자(`操` `幹` `滚` `逼` 등)는 **일상어 의미가 있어 오탐 위험**이 크므로 제외하고, 명백히 욕설인 결합어 위주로 수록.

예를 들어 `操`는 "체조하다/조작하다"라는 일상적 의미가 있습니다. 단독으로 차단하면 정상 리뷰가 막힙니다. 대신 `操你妈`, `草泥马` 같은 **결합어 단위**로 등록했습니다. 간체·번체를 쌍으로 넣고(`他妈的`/`他媽的`), 로마자 축약(`nmsl`, `cnm`)까지 포함해 실사용 패턴을 커버했습니다.

정규화된 금칙어 집합은 생성자에서 한 번만 만들어 `Set`으로 불변 보관하므로, 요청마다 재계산하지 않습니다.

---

## 6. 적용 지점

사용자 생성 콘텐츠가 저장되는 **모든 경로**에 동일한 검사를 적용했습니다.

```java
// NoteService.createNote()
if (solarModerationClient.containsProfanity(request.getContent())) {
    throw new BusinessException(ErrorCode.PROFANITY_DETECTED);   // C004 / 400
}

// ReviewService.createReview()
if (solarModerationClient.containsProfanity(request.getContent())) {
    throw new BusinessException(ErrorCode.PROFANITY_DETECTED);
}
```

호출자는 `containsProfanity(text)` **불리언 하나만** 신경 씁니다. LLM인지 Rule-Based인지, 예산이 남았는지는 전부 클라이언트 내부에서 캡슐화됩니다.

---

## 7. 설정 (환경변수 기반)

```yaml
solar:
  enabled:            ${SOLAR_ENABLED:false}          # 기본 비활성 → 로컬에서 키 없이 개발 가능
  api-key:            ${UPSTAGE_API_KEY:}
  base-url:           ${UPSTAGE_BASE_URL:https://api.upstage.ai/v1}
  model:              ${UPSTAGE_MODEL:solar-mini}
  timeout-seconds:    ${SOLAR_TIMEOUT_SECONDS:4}
  daily-budget:       ${LLM_DAILY_BUDGET:500}
  max-concurrency:    ${LLM_MAX_CONCURRENCY:5}
  rate-limit-per-min: ${RATE_LIMIT_PER_MIN:30}
```

`@ConfigurationProperties` 레코드(`SolarProperties`)로 타입 안전하게 바인딩하고, `isCallable()`, `hasDailyBudget()` 같은 **판단 메서드를 프로퍼티 객체에 두어** 클라이언트 코드에서 조건문이 흩어지지 않게 했습니다.

기본값이 `enabled=false`이므로 API 키 없이도 애플리케이션이 정상 기동하고 Rule-Based로 동작합니다. 신규 팀원이 키 발급을 기다리지 않고 바로 개발할 수 있게 한 선택입니다.

---

## 8. 정리 — 이 기능에서 얻은 것

| 관점 | 내용 |
| --- | --- |
| **가용성** | 외부 LLM 의존 기능을, 외부 장애가 서비스에 전파되지 않도록 설계 (fail-open + 폴백) |
| **비용 통제** | 동시성·분당·일일 3중 예산 가드로 호출량 상한을 코드에 명시 |
| **품질** | 오탐을 줄이기 위해 중국어 사전에서 다의어 단독 글자를 의도적으로 배제 |
| **확장성** | 호출자에게는 불리언 하나만 노출 — 판정 엔진을 교체해도 도메인 코드는 그대로 |
