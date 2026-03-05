---
title: "FOR UPDATE는 하나도 안 썼다 — 재고, 좋아요, 쿠폰의 3색 동시성 제어"
date: 2026-03-06 00:00:00 +0900
categories: [Development, Architecture]
tags: [concurrency, optimistic-lock, atomic-update, kotlin, e-commerce, jpa]
image:
  path: /assets/img/posts/2026-03-06-cover.jpg
---

> **TL;DR** 💡 4주차 과제: 동시성 제어. 처음엔 전부 비관적 락으로 통일하려 했다. 재고 차감, 좋아요 수, 쿠폰 사용 — 세 가지 동시성 포인트를 분석하고 나니 각각 다른 전략이 필요했다. 동시성 제어는 목적이 아니라 수단이었다.
{: .prompt-tip }

## 🔒 처음엔 다 잠그려고 했다

4주차 과제: 주문 처리에 동시성 제어를 적용하라. `SELECT ... FOR UPDATE` 세 군데 걸고 끝내려던 참에 D 멘토링에서 브레이크가 걸렸다.

> "아토믹 업데이트가 가장 좋다 > 낙관적 > 비관적 > 분산락. 아토믹 업데이트로 해결되지 않는 상황이 뭔지를 먼저 생각해봐."

K 멘토님도 각 전략의 선택 기준을 깔끔하게 잘라줬다:

| 전략 | 언제 쓰는가 |
|------|------------|
| 비관적 락 | 충돌이 많고, 락 잡은 시점에 데이터를 직접 보고 판단해야 할 때 |
| 낙관적 락 | 충돌이 적고, 동시 요청 중 하나만 성공해도 충분할 때 |
| Atomic Update | 동시 요청 중 일부 성공/일부 실패가 자연스러울 때. DB가 알아서 해줌 |

비관적 락은 마지막 수단이었다. 처음부터 그걸 꺼내든 건 과했다.

---

## 📦 재고 차감 — DB한테 시키면 된다

100명이 동시에 같은 상품을 주문한다. 재고는 10개.

만약 `@Version` 낙관적 락을 쓰면?

```
100명이 동시에 findById → stock=10, version=0

1명 성공:  UPDATE SET stock=9, version=1 WHERE version=0
99명 실패: OptimisticLockingFailureException

→ 99명 재시도 → 또 98명 실패 → 재시도 폭주
```

플래시세일이면 재시도만으로 서버가 터진다.

Atomic Update는 다르다:

```sql
UPDATE product
SET stock_quantity = stock_quantity - 1
WHERE id = :id AND stock_quantity >= 1
```

DB가 현재값 기준으로 계산한다. 100명이 동시에 들어와도 DB가 row-level lock으로 한 줄씩 처리한다. 재고 있으면 차감, 없으면 `affected=0`으로 깔끔하게 실패. 재시도도 필요 없다.

```kotlin
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("""
    UPDATE Product p
    SET p.stockQuantity = p.stockQuantity - :quantity
    WHERE p.id = :productId
      AND p.stockQuantity >= :quantity
      AND p.deletedAt IS NULL
""")
fun decreaseStock(productId: Long, quantity: Int): Int
```

다중 상품 주문 시에는 데드락 방지도 신경 썼다. D 멘토님이 "여러 row에 락 걸 때 1,2,3 순서랑 3,2,1 순서가 서로 물릴 수 있다"고 했다.

```kotlin
// ID 오름차순 정렬 → 모든 트랜잭션이 같은 순서로 잠금
products.sortedBy { it.id }.forEach { product ->
    val qty = quantities[product.id]!!
    productService.decreaseStock(product.id, qty)
}
```

작은 ID부터 잠그면 서로 물리는 일이 없다.

---

## 👍 좋아요 수 — Product에 @Version을 걸면 안 되는 이유

좋아요 수도 Atomic Update다. 재고와 같은 논리.

```sql
UPDATE product SET like_count = like_count + 1 WHERE id = :id
```

근데 여기서 중요한 건 "왜 Product에 `@Version`을 안 걸었느냐"다.

Product 엔티티에는 재고도 있고, 좋아요 수도 있고, 상품명/가격도 있다. `@Version`을 걸면 이 모든 필드의 변경이 같은 version을 공유한다.

```
Admin이 상품명 수정  → version 5 → 6
User가 좋아요 클릭   → version 5로 읽었는데 이미 6 → 실패!
```

전혀 관계없는 작업끼리 충돌한다. 이게 false contention이다.

JPA의 `@Version`은 엔티티 단위로 동작한다. Product 엔티티에 `@Version`을 걸면 `stockQuantity`, `likeCount`, `name`, `price` 등 어떤 필드가 바뀌든 version이 올라간다.

문제는 이 필드들이 전혀 다른 맥락에서 변경된다는 거다:

```
시나리오: 인기 상품에 Admin 수정과 User 좋아요가 동시에 발생

TX A (Admin): findById → version=5 → product.updateName("신상품")
TX B (User):  findById → version=5 → product.increaseLikeCount()

TX A COMMIT → version 5→6 ✅
TX B COMMIT → WHERE version=5 → 이미 6 → OptimisticLockingFailureException ❌
```

Admin이 상품명을 고쳤을 뿐인데 User의 좋아요가 실패한다. 상품명과 좋아요 수는 비즈니스적으로 아무 관계가 없는데, 같은 `version` 필드를 공유하기 때문에 서로를 방해하는 거다.

인기 상품일수록 좋아요·재고·상품 수정이 겹칠 확률이 높아서, `@Version`을 걸면 false contention이 실제 장애로 이어질 수 있다.

Atomic Update는 `like_count` 컬럼만 건드린다. 다른 필드가 바뀌든 말든 상관없다. 좋아요 눌렀는데 "동시 요청으로 인해 처리에 실패했습니다"가 뜨면 사용자는 황당할 수밖에 없다.

> 좋아요 수는 비정규화된 캐시 값이다. 원본은 Like 테이블의 UK(`user_id + product_id`)이고, Product의 `likeCount`는 조회 성능을 위한 집계 필드다.
>
> Atomic Update 중 어딘가에서 카운트가 어긋나더라도, 새벽 배치에서 Like 테이블을 `COUNT GROUP BY product_id`로 집계해서 Product 테이블에 다시 써주면 정합성이 복구된다.
>
> 정합성이 돈만큼 중요하진 않고, 복구 수단도 있으니까 Atomic Update로 충분했다.
{: .prompt-info }

---

## 🎫 쿠폰 사용 — 비관적 → Atomic → @Version, 세 번 바꿨다

이게 제일 고민이 많았다. 전략을 세 번 바꿨다.

### 1차: 비관적 락

처음엔 "쿠폰은 돈이니까 비관적 락"이라고 생각했다. D 멘토님도 "정합성이 중요한 곳은 비관적 락"이라고 했다.

근데 한 발 물러서 생각해봤다. "내 쿠폰을 내가 쓰는 건데, 누구랑 경쟁할 일이 있나?"

K 멘토님도 멘토링에서 "쿠폰 발급과 쿠폰 사용을 분리하면 뭐가 좋을까?"라는 질문을 던졌는데, 답은 같았다. 발급 테이블이 분리되면 경쟁 범위가 유저 단위로 좁아진다.

CouponIssue는 특정 유저에게 발급된 것이다. 다른 유저가 내 쿠폰을 쓸 수 없다.

유일한 동시성 시나리오는 같은 유저의 더블클릭. `FOR UPDATE`는 "여러 주체가 같은 자원을 경쟁"할 때 쓰는 건데, 여기선 과하다.

### 2차: Atomic Update

그럼 재고처럼 Atomic Update?

```sql
UPDATE coupon_issues
SET status = 'USED', used_at = NOW()
WHERE id = ? AND status = 'AVAILABLE'
```

동작은 한다. 근데 문제가 있다.

쿠폰 사용에는 소유권·만료·상태 전이 같은 검증이 들어간다. 이걸 Entity 메서드 `couponIssue.use()`에 담아뒀다.

Atomic Update로 바꾸면 이 메서드가 의미 없어지고, 비즈니스 규칙이 Repository의 SQL로 내려간다.

거기다 `@Modifying @Query`는 영속성 컨텍스트를 우회한다. JPQL이 SQL로 변환되어 DB에 바로 실행되기 때문에, 1차 캐시에 올라와 있는 엔티티는 변경 전 상태 그대로 남는다:

```
1. findById(1) → 영속성 컨텍스트에 올림 (status = AVAILABLE)
2. @Modifying UPDATE status = 'USED' → DB 반영
3. 메모리: status = AVAILABLE (스테일!)
4. 이후 couponIssue.status 읽으면 → AVAILABLE 반환 (버그)
```

해결 방법이 없진 않다. 재고 차감에서 실제로 쓰고 있는 조합이 있다:

```kotlin
// 재고 차감 — Atomic Update + 영속성 컨텍스트 동기화
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("""
    UPDATE Product p
    SET p.stockQuantity = p.stockQuantity - :quantity
    WHERE p.id = :productId
      AND p.stockQuantity >= :quantity
      AND p.deletedAt IS NULL
""")
fun decreaseStock(productId: Long, quantity: Int): Int
```

이 두 옵션이 하는 일을 풀어보면:

```
flushAutomatically = true
→ @Modifying 쿼리 실행 "전"에 영속성 컨텍스트의 변경을 먼저 DB에 flush
→ 아직 커밋 안 된 dirty entity가 있으면 먼저 밀어넣고 나서 UPDATE 실행
→ 없으면 flush할 게 없으니 그냥 넘어감

clearAutomatically = true
→ @Modifying 쿼리 실행 "후"에 영속성 컨텍스트를 통째로 비움 (em.clear())
→ 이후 같은 엔티티를 조회하면 DB에서 새로 읽어옴 → 스테일 해소
→ 단, 다른 dirty entity도 같이 날아감 → flushAutomatically와 세트로 써야 안전
```

```
실행 순서:
1. flushAutomatically → 영속성 컨텍스트 flush (dirty entity DB 반영)
2. @Modifying 쿼리 실행 → DB 직접 UPDATE
3. clearAutomatically → 영속성 컨텍스트 clear (메모리 비움)
4. 이후 findById 하면 → DB에서 최신 값으로 다시 로드
```

재고 차감에서는 이 조합이 자연스럽다. `decreaseStock()` 후에 재고 값을 다시 읽을 일이 거의 없고, 설령 읽더라도 clear 덕분에 DB에서 새로 가져온다.

근데 쿠폰에까지 이걸 쓸 필요가 있나? `couponIssue.use()`라는 엔티티 메서드를 포기하고, 영속성 컨텍스트 flush/clear까지 신경 써야 한다. 경합이 낮은 상황에서 이 복잡성을 안을 이유가 없다.

### 3차: @Version 낙관적 락 (최종)

```kotlin
@Entity
class CouponIssue(
    // ...
    @Version
    @Column(nullable = false)
    val version: Long = 0,
) {
    fun use() {
        require(status == CouponIssueStatus.AVAILABLE) {
            "이미 사용된 쿠폰입니다."
        }
        status = CouponIssueStatus.USED
        usedAt = LocalDateTime.now()
    }
}
```

왜 이게 맞는지:

- CouponIssue는 독립 엔티티 → Product처럼 false contention이 없다
- 경합이 극히 낮다 (같은 유저 더블클릭뿐) → 재시도 폭주 걱정 없음
- JPA dirty checking과 자연스럽다 — `couponIssue.use()` 그대로 사용
- 충돌 시 `OptimisticLockingFailureException` → 409 CONFLICT 응답

"돈이니까 비관적 락"이라는 처음 생각이 틀렸던 건, 경합 빈도를 안 따져봤기 때문이었다.

---

## 🧠 동시성 제어는 목적이 아니었다

4주차 과제가 "동시성"이라는 프레이밍이어서, 처음엔 동시성을 목적으로 코드를 짜려 했다. 어디에 락을 걸지부터 고민했다.

근데 실제 흐름은 이랬다:

```
1. 비즈니스 로직 구현 (단일 스레드에서 정상 동작)
2. "이거 동시에 들어오면?" 위험 지점 식별
3. 각 지점의 특성에 맞는 최소한의 전략 적용
4. 동시성 테스트로 검증
```

재고 차감 Atomic Update는 3주차에 이미 적용해 있었다. "동시성 제어를 하자"라고 의식한 게 아니라, `stock = stock - 1`이 자연스러워서 쓴 것이었다.

> 동시성 제어는 이미 만든 코드를 보호하는 수단이지, 코드를 짤 때의 출발점이 아니었다. 과제 프레이밍에 끌려가서 목적과 수단을 뒤집을 뻔했다.
{: .prompt-info }

---

## 🔄 재시도는 필요 없었다

전략을 정하고 나서 재시도(Retry) 로직이 필요한지 따져봤다.

결론: 세 가지 모두 필요 없다.

| 전략 | 실패 원인 | 재시도하면? |
|------|-----------|------------|
| Atomic Update (재고) | 재고 부족 (`affected=0`) | 재고 없는데 다시 해봤자 없다 |
| Atomic Update (좋아요) | 실패 자체가 없음 | — |
| @Version (쿠폰) | 이미 다른 요청이 사용 완료 | 재시도해도 `status=USED` → 비즈니스 예외 |

핵심은 "충돌 실패"와 "비즈니스 실패"의 구분이다.

- **충돌 실패**: 좌석 예약에서 A, B가 동시에 배치도를 수정 → A 성공 → B 충돌 → B가 다시 읽고 수정하면 성공 가능
- **비즈니스 실패**: 재고가 0인데 주문 → 다시 시도해도 재고는 0

우리 프로젝트의 세 가지 실패는 전부 비즈니스 실패다. 재고가 바닥났거나 쿠폰이 이미 사용됐거나. 재시도가 의미 없다.

Java 동시성 강의에서 낙관적 락에 `while(true)` 재시도를 넣는 코드를 본 적 있다. 그건 "재고 차감"에 낙관적 락을 적용한 케이스였다. 충돌 실패(version mismatch)를 재시도로 복구할 수 있으니까 의미가 있었다.

우리 프로젝트에서는 재고를 Atomic Update로 처리했기 때문에 그런 재시도가 불필요했다.

---

## 🧪 100명이 동시에 — CountDownLatch로 테스트

말로만 "동시성 안전"이라고 하면 안 된다. 실제 동시 요청을 시뮬레이션했다.

```kotlin
@Test
fun `20명이 동시에 같은 쿠폰을 사용하면 1명만 성공한다`() {
    val threadCount = 20
    val startLatch = CountDownLatch(1)   // 동시 출발용
    val endLatch = CountDownLatch(threadCount)
    val executor = Executors.newFixedThreadPool(threadCount)
    val successCount = AtomicInteger(0)
    val failCount = AtomicInteger(0)

    repeat(threadCount) {
        executor.submit {
            startLatch.await()  // 모든 스레드 대기
            try {
                couponService.useCouponForOrder(
                    couponIssueId, userId, orderAmount
                )
                successCount.incrementAndGet()
            } catch (e: Exception) {
                failCount.incrementAndGet()
            } finally {
                endLatch.countDown()
            }
        }
    }

    startLatch.countDown()  // 동시 출발
    endLatch.await()

    assertThat(successCount.get()).isEqualTo(1)
    assertThat(failCount.get()).isEqualTo(19)
}
```

`startLatch`를 쓴 이유가 있다. CompletableFuture는 스레드 생성 타이밍이 제각각이라 동시 출발을 보장 못 한다.

`startLatch(1)`로 모든 스레드를 대기시켜 놓고, `countDown()` 한 방에 출발시키는 게 더 정확하다.

재고 차감 테스트도 같은 패턴이다:

```kotlin
@Test
fun `100명이 동시에 재고 1개씩 차감하면 Lost Update가 발생하지 않는다`() {
    // stock = 100인 상품 준비
    val threadCount = 100
    // ... CountDownLatch + ExecutorService 동일 패턴

    val product = productRepository.findById(productId).get()
    assertThat(product.stockQuantity).isEqualTo(0)  // 정확히 0
}
```

동시성 테스트 5개, 단위 테스트 55개, E2E 테스트 18개. 총 78개 테스트가 통과했다.

---

## 🏢 FK를 안 쓴다고?

K 멘토님 멘토링에서 이런 질문을 했다.

회사에서 자식 테이블에 대량 INSERT 배치가 돌 때, InnoDB가 FK 검증을 위해 부모 row에 S-lock을 걸었다. 이용자 API에서 같은 부모 row를 UPDATE하려면 X-lock이 필요한데, 배치의 S-lock이 안 풀려서 타임아웃이 터졌다.

배치를 청크 단위로 쪼개서 해결하긴 했는데, 근본적으로 FK를 아예 안 거는 회사도 있다고 들어서 물어봤다.

> "FK를 빼면 DB 레벨의 정합성 보장이 없어지고, 애플리케이션에서 정합성을 챙겨야 하는데... 개발자 실수를 최종 방어하는 게 DB 아닌가? 이 트레이드오프를 실무에서 어떻게 판단하시나요?"

K 멘토님이 멘토링 때 해준 말이 인상적이었다.

> "FK를 써본 적이 없다. 정합성은 DB 혼자하는 게 아니라, 애플리케이션의 비중이 높다."

예전에는 DB 프로시저·펑션에 비즈니스를 넣었는데, 요즘은 애플리케이션에서 비즈니스를 구성하는 게 트렌드라고. 이유는 두 가지였다:

1. DB는 애플리케이션 외부에 있어서 코드의 형상관리나 유지보수가 어렵다
2. DB는 확장이 어렵다. 특히 마스터는 거의 불가능하다

주문처럼 정합성이 중요한 도메인도 마찬가지라고 했다. FK 제약조건으로 DB가 지켜주는 게 아니라, 애플리케이션 레이어에서 트랜잭션과 락 전략으로 보호하는 거라고.

회사에서도 재고 차감은 `stock = stock - 1` 패턴을 쓰고 있었다. 좋아요 따닥 방지에 Redis `SETNX`를 쓰는 것도 [2주차 글](/posts/like-button-one-second-design-one-day/)에서 정리한 적 있다.

---

## 🤔 정리하며

전략 선택의 판단 흐름을 하나로 정리하면:

```
"이 연산이 단순 산술인가?" (증감, 차감)
  ├─ YES → Atomic Update
  └─ NO (상태 전이, 비즈니스 규칙 검증 필요)
       → "경합이 높은가?"
            ├─ 낮음 → @Version 낙관적 락
            └─ 높음 → 비관적 락 (FOR UPDATE)
```

세 줄짜리 판단 기준이지만, 이걸 체감하기까지 쿠폰 전략을 세 번 바꿨다.

| 대상 | 전략 | 핵심 이유 |
|------|------|-----------|
| 재고 차감 | Atomic Update | 단순 산술, 고경합, 재시도 비용 큼 |
| 좋아요 수 | Atomic Update | 단순 증감, Product `@Version` 시 false contention |
| 쿠폰 사용 | @Version | 상태 전이, 엔티티 메서드 필요, 저경합 |

> `FOR UPDATE` 세 개 걸고 끝내려던 과제에서, 결국 `FOR UPDATE`는 하나도 안 썼다.
{: .prompt-tip }

## 참고
