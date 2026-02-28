---
title: "👍 좋아요 버튼 누르는 건 1초, 설계하는 건 하루"
date: 2026-02-13 00:00:00 +0900
categories: [Development, Architecture]
tags: [e-commerce, like, soft-delete, idempotency, redis, kotlin]
image:
  path: /assets/img/posts/2026-02-13-cover.jpg
---

> **TL;DR** 💡 좋아요 기능을 설계하면서 삭제 전략, 중복 요청(따닥) 처리, 상품 상태 분리까지 고민이 꼬리를 물었다. '단순한 기능'은 없었다.
{: .prompt-tip }

## 👍 좋아요 하나쯤이야

이커머스 설계 과제에서 좋아요 기능을 접했다. 처음엔 `likes` 테이블 하나 만들고 INSERT/DELETE 하면 되는 거 아닌가?'라고 생각했다.

근데 막상 설계를 시작하니까 질문이 끝도 없이 나왔다.

- 취소하면 진짜 지워? 남겨둬?
- 같은 버튼 두 번 눌리면?
- 좋아요 누른 상품이 품절이면 어떻게 보여줘?

하나씩 정리하다 보니 하루가 갔다.

---

## 1. 🗑️ 지울 거야, 남길 거야

좋아요 취소를 누르면 DB에서 어떻게 처리할지. 두 가지 선택지가 있었다.

**Soft Delete** — `deleted_at`에 시간만 찍고 행은 남겨둠. 이력이 보존되니까 나중에 추천/랭킹 데이터로 활용 가능.

**Hard Delete** — 물리 삭제. 테이블이 깔끔하지만 취소 이력이 사라짐.

좋아요 특성상 등록/취소가 빈번하다. Soft Delete로 가면 UK(Unique Key) 설계가 귀찮아진다.

`UNIQUE (user_id, product_id)`로 잡으면 deleted_at이 찍힌 행 때문에 재등록이 막힌다. 그렇다고 `UNIQUE (user_id, product_id, deleted_at)`으로 잡으면? MySQL에서 NULL은 UK 중복 체크에서 빠지기 때문에 의도대로 동작하지 않을 수 있다.

거기다 모든 조회 쿼리에 `WHERE deleted_at IS NULL`을 빼먹으면 삭제된 데이터가 섞여 나온다.

요구사항에 '좋아요 데이터는 향후 추천/랭킹 기초 데이터로 활용 가능'이라는 문장이 있어서 좀 걸렸다. 이력이 없으면 행동 패턴 분석을 못 하니까.

근데 다시 생각해보니, 추천/랭킹에 필요한 건 **'지금 누가 뭘 좋아요 하고 있는지'**이지, '3일 전에 눌렀다 취소했다' 같은 이력은 아닌 것 같다. 행동 패턴 분석이 정말 필요해지면, likes 테이블을 건드리는 게 아니라 별도 이벤트 테이블을 추가하는 게 맞지 않을까.

회사 코드(`favorite_goods`)도 확인해봤는데 물리 삭제 방식이었다.

> **결론: Hard Delete.** 좋아요의 핵심 가치는 '현재 상태'이지 '이력'이 아니라고 판단했다.
{: .prompt-info }

---

## 2. 🖱️ 따닥 — 같은 버튼 두 번 눌렸을 때

좋아요 버튼을 빠르게 두 번 누르는 경우. 프론트에서 막아야 하는 거 아니냐고 할 수 있는데, 백엔드는 '프론트가 안 막았을 때'도 대비해야 한다고 생각한다.

### 일단 중복이면 뭘 반환할지

이미 좋아요한 상품에 다시 POST가 오면?

- **A: 토글** — 있으면 삭제, 없으면 생성. UX는 간단한데 POST/DELETE 의미가 모호해짐
- **B: 200 OK** — 이미 있어도 성공 반환. 멱등적이라 재시도에 안전. 근데 신규 생성이랑 구분이 안 됨
- **C: 409 CONFLICT** — 이미 존재하면 에러. HTTP 스펙에 충실

**C안(409)을 선택했다.** POST는 '생성'이니까 이미 있으면 충돌이라고 생각했다. 다만 B안(200)도 틀린 건 아닌 것 같고, 이건 좀 정답이 없는 영역이라고 느꼈다.

### 409로 끝이 아니다

문제는 타이밍이다. 거의 동시에 두 요청이 들어오면:

```
요청 A: SELECT → 없음 → INSERT ← 성공
요청 B: SELECT → 없음 → INSERT ← ???
```

요청 B가 SELECT하는 시점에 A의 INSERT가 아직 커밋 안 됐으면, B도 '없음'으로 판단하고 INSERT를 시도한다.

여기서 DB의 Unique Key가 최종 방어선 역할을 한다:

```sql
ALTER TABLE likes ADD CONSTRAINT uk_likes UNIQUE (user_id, product_id);
```

B의 INSERT가 UK 위반으로 `DataIntegrityViolationException`이 터지면, 이걸 잡아서 409 CONFLICT로 변환하면 된다.

### 그런데 매번 DB까지 가야 해?

회사에서 비슷한 문제를 Redis로 해결한 경험이 있다. `@Cooldown`이라는 커스텀 어노테이션을 만들어서 썼다.

Cooldown은 게임에서 스킬 쓴 다음 일정 시간 동안 재사용이 안 되는 그 '쿨타임'에서 따온 이름이다. 같은 요청이 짧은 시간 안에 반복되면, 쿨타임이 안 끝났으니까 무시하는 개념.

구조는 **AOP + Redis + SpEL** 조합이다:

```java
@Cooldown(key = RECENT_GOODS, value = "'target:' + #goodsId + ':buyer:' + #buyerId", ttl = 3)
public void addRecentGoods(Long goodsId, Long buyerId) {
    // ...
}
```

`@Cooldown`을 메서드에 붙이면 AOP(`CooldownAspect`)가 가로챈다. `value`에 SpEL 표현식을 쓸 수 있어서, 메서드 파라미터를 조합해 동적으로 키를 만든다. 위 예시에서는 `RECENT_GOODS:target:123:buyer:456` 같은 키가 생성된다.

핵심은 `CooldownService.acquire()`:

```java
public boolean acquire(String key, Duration ttl) {
    return Boolean.TRUE.equals(
        redisTemplate.opsForValue().setIfAbsent(key, "1", ttl)
    );
}
```

Redis의 `setIfAbsent`는 `SETNX` 명령어에 대응한다. **키가 없으면 세팅하고 `true`, 이미 있으면 아무것도 안 하고 `false`**. 이게 원자적(atomic)으로 동작하기 때문에 동시 요청이 와도 딱 하나만 통과한다. TTL을 같이 걸어서, 3초가 지나면 키가 자동 삭제되고 다시 요청할 수 있다.

AOP에서는 `acquire()`의 결과에 따라 분기한다:

```java
// CooldownAspect.java (핵심만)
boolean firstHit;
try {
    firstHit = cooldownService.acquire(key, Duration.ofSeconds(cooldown.ttl()));
} catch (Exception e) {
    log.warn("Redis 장애 발생 – 중복 요청 제어 불가. key={}", key);
    firstHit = true;  // Redis 장애 시 → 그냥 통과 (가용성 우선)
}

if (!firstHit) {
    return null;  // 쿨타임 안 끝남 → 비즈니스 로직 실행 안 함
}
return pjp.proceed();  // 첫 요청 → 정상 실행
```

Redis가 죽어도 서비스가 멈추면 안 되니까 `catch`에서 `firstHit = true`로 빠진다. 따닥 방지보다 가용성이 더 중요하다는 판단이었다.

좋아요에 적용하면 이런 3중 방어가 된다:

| 레이어 | 방어 수단 | 역할 |
|--------|----------|------|
| **1차** | Redis `SETNX like:{userId}:{productId}` TTL 3초 | 따닥(연타) 차단 — DB까지 안 감 |
| **2차** | Service에서 `SELECT EXISTS` | 정상적인 중복 요청 409 반환 |
| **3차** | DB `UNIQUE (user_id, product_id)` | 최종 방어선 — Race Condition 대비 |

1차가 없으면 따닥 요청이 전부 DB까지 내려간다. 3차가 없으면 Redis 장애 시 중복 삽입이 발생할 수 있다. 한 겹만으로는 불안하다.

---

## 3. 🏷️ 품절인데 보여줘야 해?

좋아요 누른 상품이 품절이 됐다. 좋아요 목록에서 그 상품을 어떻게 보여줘야 할까?

처음엔 '품절이면 목록에서 빼면 되지'라고 생각했다. 근데 내가 찜해둔 상품이 어느 날 갑자기 사라져 있으면? 사용자 입장에서는 당황스럽다. 그런데 쇼핑몰에서 실제로 어떻게 하는지 떠올려보면:

![무신사 품절 상품 딤 처리 예시](/assets/img/posts/musinsa-soldout-dim.png)
_무신사 — 품절 상품은 회색 오버레이 + '품절' 뱃지로 표시된다_

**품절이어도 목록에는 보여준다.** 다만 이미지가 어둡게(딤 처리) 되고 `품절` 뱃지가 붙는다.

이걸 DB 설계로 옮기면, 상품에 두 가지 의사결정이 필요하다:

- **판매 가능 여부** — 구매 버튼이 활성화되는가?
- **목록 노출 여부** — 상품 목록에 보이는가?

이 두 개를 `status` 하나로 퉁치면 '보이지만 구매 불가' 상태를 표현할 수가 없다.

```
status = ACTIVE     → 구매 가능 + 노출
status = INACTIVE   → 구매 불가 + ???
```

INACTIVE인데 보여줘야 하나, 숨겨야 하나? status만으로는 답이 안 나온다.

그래서 **status + displayYn을 분리**했다:

| status | displayYn | 목록 노출 | 구매 가능 | UI |
|--------|-----------|----------|----------|-----|
| ACTIVE | true | ✅ | ✅ | 일반 상품 카드 |
| ACTIVE | false | ❌ | ✅ (URL 직접) | 관리자가 의도적으로 숨긴 상태 |
| INACTIVE | true | ✅ | ❌ | **딤 처리 + '판매중지' 뱃지** |
| INACTIVE | false | ❌ | ❌ | 완전 미노출 |

프론트에서는 이 두 필드를 조합해서 UI를 분기할 수 있다:

```
displayYn = false              → 목록에서 제외
displayYn = true, ACTIVE       → 정상 상품 카드
displayYn = true, INACTIVE     → 딤처리 + '판매중지' 라벨
```

'이거 오버엔지니어링 아니야?'라는 생각도 들었다. 과제 범위에서 딤 처리가 실제로 필요한 건 아니니까.

근데 Shopify, Amazon, 네이버, 쿠팡 전부 찾아봤는데, 성숙한 플랫폼은 대부분 status ≠ visibility를 분리해서 관리하고 있었다. 방향성은 틀리지 않았다고 생각한다.

> displayYn 네이밍도 고민이 있었다. Kotlin에서 `isDisplay`로 하면 Jackson 직렬화 시 `display`로 변환되는 알려진 버그([jackson-module-kotlin #80](https://github.com/FasterXML/jackson-module-kotlin/issues/80))가 있다. 한국 이커머스 관례를 따라 `Yn` 접미사로 갔다.
{: .prompt-warning }

---

## 📝 돌아보면

좋아요 기능 하나에서 나온 고민들:

1. **삭제 전략** — Hard Delete vs Soft Delete → 현재 상태 vs 이력 보존
2. **중복 처리** — 409 vs 200 → HTTP 의미론 vs 클라이언트 편의성
3. **따닥 방지** — Redis SETNX → Service 체크 → DB UK 3중 방어
4. **상태 분리** — status × displayYn → '보이지만 구매 불가'를 표현

처음에 'INSERT/DELETE면 되지'라고 생각한 게 부끄러운 건 아닌데, 파고들수록 판단의 연속이었다. 각각에 '정답'이 있다기보다는 상황에 따라 트레이드오프가 달라지는 거고.

설계 주차라 코드는 안 짰지만, 이런 고민들을 먼저 해두니까 구현할 때 덜 헤맬 것 같다. 아마.

> 좋아요 버튼 누르는 건 1초. 그 뒤를 설계하는 건 하루가 넘게 걸렸다.
{: .prompt-tip }
