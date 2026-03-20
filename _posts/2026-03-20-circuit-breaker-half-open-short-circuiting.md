---
title: "Claude도 Gemini도 틀렸다 — 서킷브레이커 HALF_OPEN 삽질기"
date: 2026-03-20 00:00:00 +0900
categories: [Development, Architecture]
tags: [payment, resilience4j, circuit-breaker, retry, rate-limiter, kotlin, spring-boot, e-commerce]
image:
  path: /assets/img/posts/2026-03-20-cover.jpg
  alt: "신의 한 수는 Actuator 새로고침이었습니다"
---

> **TL;DR** 💡 6주차 과제: PG 연동 결제 시스템에 Resilience4j 적용. 결제는 "성공/실패"가 아니라 "모르는 상태"가 존재하는 도메인이었다. TIMEOUT 상태를 만들고, 서킷브레이커를 설정하고, 직접 테스트하다가 문서에 없는 동작을 발견했다. HALF_OPEN에서 permitted 5건을 다 안 채웠는데 3건에서 서킷이 열렸다. 소스코드까지 추적해서 원인을 확인한 기록.
{: .prompt-tip }

## 💳 결제는 "모르는 상태"가 있다

6주차 과제는 PG 연동 결제에 Resilience4j를 적용하는 것이었다. A 멘토님 발제에서 첫 질문이 날아왔다.

> "결제에서 타임아웃이 나면 그게 실패야? PG에서는 실제로 결제가 됐을 수도 있어. 그 상태에서 주문을 취소하면?"

결제됐는데 주문이 취소된다. 사용자 돈은 빠져나갔는데 상품은 안 온다.

> 결제 실패와 "결제 결과를 모르는 상태"는 다르다. 이 구분이 안 되면 설계가 처음부터 틀어진다.
{: .prompt-warning }

그래서 Payment 상태에 `TIMEOUT`을 추가했다. "PG가 거절한 게 아니라 응답을 못 받은 것"을 명시적으로 표현하는 상태다.

```kotlin
enum class PaymentStatus {
    REQUESTED,   // PG에 결제 요청을 보낸 상태
    APPROVED,    // PG에서 결제 승인 완료
    REJECTED,    // PG에서 결제 거절
    TIMEOUT,     // PG 요청 타임아웃 (실제 처리됐을 수 있어 폴링 확인 필요)
}
```

`TIMEOUT`에서 `APPROVED`로 갈 수 있다는 게 핵심이다. 타임아웃 났지만 PG에서 확인해보니 결제 성공이었다 — 이런 케이스를 복구할 수 있어야 한다. 복구 경로는 두 가지: PG가 결제 결과를 웹훅으로 전달하는 **콜백**(빠른 경로), 스케줄러가 60초마다 PG에 상태를 조회하는 **폴링**(안전망).

Order 상태도 리팩토링했다. `PAID`를 `CONFIRMED`으로 바꾸고 `PAYMENT_FAILED`를 삭제했다. 결제가 실패해도 주문 자체는 실패가 아니다. `ORDERED` 상태로 남아서 다른 카드로 재결제할 수 있어야 한다. 결제 상태는 Payment 엔티티가 따로 관리한다.

상태를 나눴으니, 다음은 PG 호출을 트랜잭션에서 어떻게 분리하느냐였다.

---

## ✂️ 트랜잭션을 세 토막 냈다

결제 요청에서 가장 고민한 건 트랜잭션 경계였다. 처음엔 `@Transactional` 하나로 전부 묶었다.

L 멘토님 멘토링에서 트랜잭션 경계 질문이 나왔을 때 핵심이 명확해졌다. PG 호출이 3초 걸리면 트랜잭션이 3초 동안 열려있고, 그 동안 DB 커넥션 하나가 잡혀있다. 동시 요청 100개 들어오면 커넥션 풀이 마르고, 결제만 죽는 게 아니라 서비스 전체가 멈춘다.

```kotlin
fun requestPayment(userId: Long, criteria: RequestPaymentCriteria): PaymentResult {
    // ── 트랜잭션 1: 주문 검증 + Payment 생성 ──
    val payment = paymentService.createPayment(command)

    // ── PG 호출 (트랜잭션 밖) ──
    try {
        val pgResponse = pgPaymentClient.requestPayment(request)
        // ── 트랜잭션 2: transactionKey 저장 ──
        paymentService.updateTransactionKey(payment.id, pgResponse.transactionKey)
    } catch (ex: CoreException) {
        // PG 호출 실패 → REJECTED가 아니라 TIMEOUT
        paymentService.markTimeout(payment.id)
    }
}
```

PG 호출이 트랜잭션 밖으로 나갔다. catch 블록에서 REJECTED가 아니라 **TIMEOUT**으로 마킹하는 게 포인트. "거절된 게 아니라 모르는 거니까."

결제 따닥 방지는 [2주차 좋아요 글](/posts/like-button-one-second-design-one-day/)에서 만든 Redis SETNX 기반 중복 방지를 `@RateLimit` 어노테이션으로 발전시켜서 적용했다. 좋아요에서는 따닥을 조용히 무시(`throwOnDuplicate = false`)했지만, 결제에서는 `throwOnDuplicate = true`로 예외를 던져서 사용자에게 "이미 진행 중"이라고 명확히 알려준다. `@Order(Ordered.HIGHEST_PRECEDENCE)`로 트랜잭션보다 먼저 실행되니까 DB 부하도 없다.

PG 호출이 트랜잭션 밖으로 나왔다. 근데 PG가 아예 죽어버리면? 트랜잭션 분리만으로는 장애 전파를 막을 수 없다.

---

## 🛡️ 서킷브레이커 — 장애 전파를 차단하다

PG가 죽으면 모든 결제 요청이 `readTimeout`(1초)까지 기다린다. 100명이 동시에 결제하면 100개 스레드가 블로킹된다. 톰캣 스레드 풀이 소진되면 상품 조회, 장바구니 같은 다른 API까지 전부 멈춘다. PG 하나 죽었는데 우리 서비스 전체가 죽는 **장애 전파**.

Resilience4j 서킷브레이커를 적용했다. 설계 결정이 몇 가지 있었다.

### 서킷 인스턴스를 둘로 분리

```yaml
pg-request:   # 결제 요청 (POST) — failureRate 15%
pg-query:     # 결제 조회 (GET) — failureRate 20%
```

A 멘토님 멘토링 청강에서 "인스턴스를 분리해야 한다"는 얘기가 나왔다. 결제 요청(POST)이 터져서 서킷이 열리면, 복구 스케줄러가 PG 상태를 조회(GET)하는 것도 같이 막힌다. 장애 복구 경로가 장애에 의해 차단되는 거다. 분리하면 결제 서킷이 열려도 스케줄러는 독립적으로 복구를 시도할 수 있다.

### Retry는 멱등한 호출에만

결제 요청(POST)을 재시도하면 PG에서 새로운 결제가 하나 더 생길 수 있다. 시뮬레이터 코드를 직접 확인했는데 멱등 처리를 보장하지 않았다. PG가 멱등키를 지원한다면 POST에도 Retry를 걸 수 있겠지만, 이번 시뮬레이터에서는 아니었다. A 멘토님 청강에서도, L 멘토님 멘토링에서도 같은 결론이었다:

> "멱등성이 보장되지 않으면 리트라이 하지 않는 것이 안전하다."

### AOP 프록시 문제 — 빈 분리

서킷브레이커(Outer)와 리트라이(Inner)를 같은 클래스에 두면 Spring AOP 프록시가 내부 호출을 가로채지 못한다. `PGPaymentClientImpl`(서킷+레이트리미터)과 `PGPaymentApiCaller`(리트라이+HTTP 호출)로 분리했다.

```
요청 → RateLimiter → CircuitBreaker → [PGPaymentApiCaller] → Retry → RestClient → PG
```

설정은 끝났다. 근데 이게 실제로 설정대로 동작하는지는 별개의 문제였다.

---

## 🧪 "직접 테스트해봐라"

서킷브레이커 설정을 마치고 L 멘토님 멘토링에서 테스트 결과를 공유했다. L 멘토님이 이렇게 말했다:

> "작성해 주셨던 '실제로 동작하는지' — 이게 중요합니다. 대부분 설정만 해놓고 실제 동작을 안 해봐요. 대부분 실패하는 경우들이 많습니다."

흔한 오해 하나를 짚어줬다:

> "착각하는 것 중에 하나가 Window Size가 10이면 이 10을 다 채우고 판단할 거라고 생각하는데, 그렇지 않습니다."

설정값을 하나씩 바꿔가면서 actuator 엔드포인트를 새로고침하며 상태 전이를 확인하고 있었다. 테스트용으로 일부러 윈도우와 임계치를 낮게 잡았다. 눈으로 빠르게 상태 전이를 확인하려고.

```yaml
pg-request:
  sliding-window-size: 20
  minimum-number-of-calls: 3
  failure-rate-threshold: 15
  permitted-number-of-calls-in-half-open-state: 5
  wait-duration-in-open-state: 10s
```

CLOSED → OPEN → HALF_OPEN까지는 예상대로 동작했다. 문제는 HALF_OPEN에서 일어났다.

---

## 💥 5건 중 3건에서 서킷이 열렸다

HALF_OPEN 상태. `permitted`가 5건이니까 5건 호출한 다음에 판단할 거라고 생각했다.

그런데 3건째에서 바로 OPEN이 됐다.

```json
{
  "pg-request": {
    "state": "OPEN",
    "bufferedCalls": 3,
    "failedCalls": 1
  }
}
```

5건인데 왜 3건에서?

Claude Code한테 물어봤더니 "HALF_OPEN에서는 permitted 건수를 다 채운 후에 판단합니다"라고 했다. 근데 3건에서 열렸으니까 이건 맞지 않다. Gemini한테도 물어봤다. "조기 차단(short-circuiting)입니다. 실패가 발생하면 수학적으로 복구 불가능할 때 즉시 OPEN됩니다." 방향은 맞았는데, "1건 실패로 즉시"라고 해서 정확하진 않았다. 실제로는 1건째가 아니라 3건째에서 열렸으니까.

호출 순서대로 추적했다:

```
  permitted = 5건                     threshold = 15%
  minimum   = 3건

  ┌─────┬─────┬─────┬─────┬─────┐
  │  1  │  2  │  3  │  4  │  5  │   ← permitted 슬롯
  │ ✅  │ ✅  │ ❌  │     │     │
  └─────┴─────┴──┬──┴─────┴─────┘
                  │
          minimum(3) 충족 → 실패율 계산
          failureRate = 1/3 = 33%  >  threshold 15%
                  │
                  ▼
     ┌─────────────────────────┐
     │  남은 2건이 전부 성공해도  │
     │  1/5 = 20% > 15%        │
     │  → 복구 불가능 → 즉시 OPEN │
     └─────────────────────────┘
```

5건을 기다리지도 않았고, 1건 실패로 즉시 판단하지도 않았다. `minimumNumberOfCalls`(3건)을 채운 시점부터 매 호출마다 실패율을 계산하고 있었다. 그리고 남은 호출이 전부 성공해도 threshold를 넘길 수밖에 없으면, 나머지를 기다리지 않고 바로 OPEN으로 전이한다.

---

## 🔍 소스코드까지 추적

gradle 캐시에서 `resilience4j-circuitbreaker-2.2.0-sources.jar`를 꺼내서 뜯어봤다.

HALF_OPEN 상태에서 `onSuccess()`나 `onError()`가 호출될 때마다 이 메서드가 실행된다:

```java
private void checkIfThresholdsExceeded(Result result) {
    if (Result.hasExceededThresholds(result)) {
        transitionToOpenState();      // permitted 안 기다리고 바로 OPEN
    }
    if (result == BELOW_THRESHOLDS) {
        transitionToClosedState();    // 바로 CLOSED
    }
}
```
{: file="CircuitBreakerStateMachine.java:1168" }

매 호출마다 판단한다. permitted 건수를 다 채울 때까지 기다리는 로직이 아니다.

`Result`를 결정하는 실패율 계산 부분:

```java
private float getFailureRate(Snapshot snapshot) {
    int bufferedCalls = snapshot.getTotalNumberOfCalls();
    if (bufferedCalls == 0 || bufferedCalls < minimumNumberOfCalls) {
        return -1.0f;  // minimum 미달이면 판단 보류
    }
    return snapshot.getFailureRate();
}
```
{: file="CircuitBreakerMetrics.java:165" }

`-1.0f`이 반환되면 `BELOW_MINIMUM_CALLS_THRESHOLD`로 처리돼서 상태 전이가 일어나지 않는다. minimum을 채우기 전까지는 판단 자체를 하지 않는 구조.

HALF_OPEN의 실제 판단 흐름:

```
                    매 호출 후
                        │
            ┌───────────┴───────────┐
            ▼                       ▼
  bufferedCalls < minimum     bufferedCalls >= minimum
            │                       │
            ▼                       ▼
       판단 보류               실패율 계산
                          ┌─────┼─────────────┐
                          ▼     ▼             ▼
                     threshold  threshold    threshold 이하
                       초과     이하 +        + permitted
                          │     permitted      안 채움
                          │     다 채움          │
                          ▼        ▼            ▼
                      즉시 OPEN   CLOSED     HALF_OPEN
                     (조기 차단)   복귀         유지
```

---

## 🔄 threshold가 50%면 어떻게 되나

같은 상황에서 threshold만 50%로 바꿔보면 결과가 달라진다.

```
  permitted = 5건                     threshold = 50%
  minimum   = 3건

  ┌─────┬─────┬─────┬─────┬─────┐
  │  1  │  2  │  3  │  4  │  5  │   ← permitted 슬롯
  │ ❌  │ ✅  │ ✅  │     │     │
  └─────┴─────┴──┬──┴─────┴─────┘
                  │
          minimum(3) 충족 → 실패율 계산
          failureRate = 1/3 = 33%  <  threshold 50%
                  │
                  ▼
     ┌─────────────────────────┐
     │  남은 2건이 전부 성공하면  │
     │  1/5 = 20% < 50%        │
     │  → 복구 가능 → 대기       │
     └─────────────────────────┘

남은 2건 전부 성공하면 1/5 = 20% < 50% → 복구 가능
→ 조기 차단 안 함, 5건 다 채울 때까지 대기
```

수학적으로 복구 가능한 상황이니까 기다린다. 조기 차단은 "남은 호출이 전부 성공해도 threshold를 넘을 때"만 발동한다.

테스트 코드로 두 케이스를 재현했다:

```kotlin
@Test
@DisplayName("permitted=5, minimum=3, threshold=15% → 3건 중 1건 실패로 즉시 OPEN")
fun shortCircuitsWhenRecoveryImpossible() {
    val cb = createCircuitBreaker(
        minimumNumberOfCalls = 3,
        failureRateThreshold = 15f,
        permittedNumberOfCallsInHalfOpenState = 5,
    )
    // CLOSED → OPEN
    callFailure(cb); callFailure(cb); callSuccess(cb)
    assertThat(cb.state).isEqualTo(CircuitBreaker.State.OPEN)
    Thread.sleep(200)

    // HALF_OPEN에서 3건 호출 (1건 실패, 2건 성공)
    callFailure(cb)
    callSuccess(cb)
    callSuccess(cb)

    // 5건 안 채웠는데 3건에서 바로 OPEN
    assertThat(cb.state).isEqualTo(CircuitBreaker.State.OPEN)
    assertThat(cb.metrics.numberOfBufferedCalls).isLessThan(5)
}

@Test
@DisplayName("permitted=5, minimum=3, threshold=50% → 1건 실패해도 HALF_OPEN 유지")
fun doesNotShortCircuitWhenRecoveryStillPossible() {
    val cb = createCircuitBreaker(
        minimumNumberOfCalls = 3,
        failureRateThreshold = 50f,
        permittedNumberOfCallsInHalfOpenState = 5,
    )
    callFailure(cb); callFailure(cb); callSuccess(cb) // OPEN
    Thread.sleep(200)

    callFailure(cb) // HALF_OPEN 트리거 + 실패

    // 복구 가능하니까 HALF_OPEN 유지
    assertThat(cb.state).isEqualTo(CircuitBreaker.State.HALF_OPEN)
}
```

> `failure-rate-threshold`를 15%로 민감하게 잡아놨기 때문에 발견한 동작이다. 50%로 놓고 테스트했으면 5건 다 채운 뒤에 판단했을 거고, 조기 차단은 발견 못 했을 것이다. 설정값의 조합이 동작을 바꾸니까, 테스트할 때 운영 설정 그대로 돌려봐야 한다.
{: .prompt-info }

---

## 📋 공식 문서와의 차이

[Resilience4j 공식 문서](https://resilience4j.readme.io/docs/circuitbreaker#create-and-configure-a-circuitbreaker)의 `minimumNumberOfCalls` 설명은 CLOSED 상태 기준으로만 되어 있다. HALF_OPEN에서도 동일하게 적용된다는 건 언급하지 않고, 조기 차단이라는 동작 자체도 문서에 나오지 않는다.

| 항목 | 공식 문서 | 실제 동작 |
|------|-----------|-----------|
| HALF_OPEN 판단 시점 | permitted 건수 후 판단 (뉘앙스) | 매 호출마다 판단 |
| minimumNumberOfCalls | CLOSED 기준으로만 설명 | HALF_OPEN에서도 적용 |
| 조기 차단 | 언급 없음 | 복구 불가능하면 즉시 전이 |

---

## 🤔 남은 고민

### 콜백과 폴링의 동시 도착

PG 콜백(외부 → 서버)과 폴링 스케줄러(서버 내부)가 같은 결제건을 동시에 처리하는 케이스가 있다. `markApproved`에 멱등성을 넣어서 논리적으로는 안전하지만, 동시에 실행되면 두 트랜잭션이 같은 row를 UPDATE하게 된다.

A 멘토님 멘토링 청강에서 실무에서는 **Single Flight 패턴**을 쓴다는 얘기가 나왔다. 특정 키로 첫 요청이 들어오면 마킹하고, 같은 키의 후속 요청은 대기시킨 뒤 첫 요청의 결과를 공유하는 방식이다. 생각해보면 이미 만들어둔 `@RateLimit`이 Redis SETNX 기반이니까, transactionKey 기준으로 같은 메커니즘을 적용하면 될 것 같다.

### PG 멀티 벤더

L 멘토님이 공유해준 실무 사례가 인상적이었다. 문자 발송에서 NHN(Toast) → 엔클라우드 → LG CNS 순으로 벤더를 바꿔가며 재시도하고, 최종 실패하면 Redis에 저장한 뒤 개발자에게 알림을 보내는 구조. 결제도 같은 패턴이 적용된다고.

`PGPaymentClient` 인터페이스를 DIP로 분리해둔 게 여기서 빛을 발할 것 같다.

---

## 📝 돌아보면

결제 도메인의 코드를 세어보면 정상 플로우보다 실패 처리가 더 많다. TIMEOUT 상태, 콜백 멱등성, 폴링 복구, 서킷브레이커, 따닥 방지, 예외 분류 — 결제 시스템에서 코드의 대부분은 "안 되면 어쩌지?"에 대한 답이다.

그리고 서킷브레이커를 테스트하면서 문서에도, AI에도 없는 동작을 발견했다. L 멘토님이 말한 대로 "설정값을 외우는 것보다 실제 동작을 실험하고 체감하는 것이 훨씬 중요"했다.

문서 → AI → 테스트 → 소스코드. 이 순서로 파고드니까 표면적인 이해가 아니라 구조를 이해하게 됐다.

> 결제는 "됐다/안됐다"가 아니라 "됐다/안됐다/모른다"의 세 상태를 다루는 도메인이다. 서킷브레이커도 마찬가지. 문서에 적힌 동작과 실제 동작 사이에 "문서에 안 적힌 영역"이 있다. 둘 다 직접 확인해야 안다.
{: .prompt-tip }

## 참고

- [Resilience4j 공식 문서 — CircuitBreaker](https://resilience4j.readme.io/docs/circuitbreaker)
- [Resilience4j GitHub — CircuitBreakerStateMachine.java](https://github.com/resilience4j/resilience4j)
- [Martin Fowler — CircuitBreaker](https://martinfowler.com/bliki/CircuitBreaker.html)
