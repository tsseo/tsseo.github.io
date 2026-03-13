---
title: "넣을 때는 마음대로였겠지만, 꺼낼 때는 아니란다 — Redis JSON 직렬화 삽질기"
date: 2026-03-13 00:00:00 +0900
categories: [Development, Architecture]
tags: [redis, cache, serialization, json, kotlin, spring-boot, jackson]
image:
  path: /assets/img/posts/2026-03-13-cover.jpg
---

> **TL;DR** 💡 Redis에 `@class` 없는 순수 JSON을 저장하고 싶었다. 저장은 됐는데 꺼내니까 `LinkedHashMap`이 나왔다. ThreadLocal로 해결하고, ZonedDateTime에 당하고, Page 역직렬화까지 — 전부 꺼내는 쪽 문제였다.
{: .prompt-tip }

## 📦 직렬화 — 한 줄로 펴는 것

직렬화(Serialization)라는 단어를 처음 접했을 때 뭔 소린가 싶었다.

Serialize의 어원은 serial — '연속적인', '직렬의'. 메모리에 흩어져 있는 객체의 필드들을 **한 줄로 나열**해서 바이트 스트림으로 만드는 행위다. 네트워크로 보내든, 파일에 저장하든, Redis에 넣든 — 결국 어딘가에 보관하려면 한 줄로 펴야 한다.

역직렬화(Deserialization)는 그 반대. 한 줄로 펴놓은 바이트를 다시 객체로 복원하는 것.

문제는 **어떤 형태로** 한 줄로 펴느냐다.

---

## 🔍 바이너리 vs JSON — 눈으로 읽을 수 있느냐

회사에서 Redis 캐시 디버깅을 하다가 겪은 일이 있다. 캐시가 이상하게 동작해서 Redis CLI로 직접 값을 확인하려 했는데:

```
> GET cache:product:123
"\xac\xed\x00\x05sr\x00\x1ecom.example.ProductInfo..."
```

바이너리 직렬화(JDK Serialization)로 저장된 값이었다. 사람이 읽을 수가 없다. 캐시에 뭐가 들어있는지, 역직렬화가 왜 실패하는지 — 확인할 방법이 없었다.

회사 캐시는 바이너리, 헥사, JSON이 섞여 있었다. 캐시마다 직렬화 방식이 달라서, 문제가 생기면 "이 캐시는 뭘로 저장했더라?"부터 찾아야 했다.

이번 과제에서 캐시를 구현하면서 하나는 확실히 하고 싶었다.

> **Redis에 저장되는 값은 전부 JSON으로 통일한다.** 눈으로 읽을 수 있어야 디버깅이 된다.
{: .prompt-info }

JSON이면 Redis CLI에서 바로 확인할 수 있다:

```json
{
  "id": 1,
  "name": "상품A",
  "price": 15000,
  "brandName": "브랜드X",
  "createdAt": "2026-03-11T15:30:27+09:00"
}
```

역직렬화가 실패하면 저장된 JSON과 DTO 필드를 비교해서 어디가 틀렸는지 바로 찾을 수 있다. 바이너리였으면 불가능한 일이다.

---

## 🏷️ @class가 왜 붙는 건데

Spring에서 Redis JSON 직렬화를 검색하면 열에 아홉은 `GenericJackson2JsonRedisSerializer`가 나온다. 쓰기는 간단한데, 저장된 JSON을 열어보면 이런 게 붙어있다:

```json
{
  "@class": "com.loopers.domain.product.ProductInfo",
  "id": 1,
  "name": "상품A",
  "price": 15000,
  "brandName": "브랜드X",
  "createdAt": ["java.time.ZonedDateTime", "2026-03-11T15:30:27+09:00"]
}
```

`@class`는 Jackson의 `activateDefaultTyping`이 삽입하는 타입 메타데이터다. 역직렬화할 때 "이 JSON이 어떤 클래스인지" 알려주는 힌트.

왜 필요하냐면, `GenericJackson2JsonRedisSerializer`는 이름 그대로 **Generic** — 아무 타입이나 받을 수 있는 범용 직렬화기다. `ProductInfo`를 넣든 `BrandInfo`를 넣든 `RedisSerializer<Object>`로 처리한다. 역직렬화 시 원본 타입을 모르니까, JSON 안에 클래스 이름을 박아둬야 복원이 된다.

문제는:

1. **클래스 패키지를 옮기면 깨진다.** `com.loopers.domain.product.ProductInfo`를 `com.loopers.application.product.ProductResult`로 리팩토링하면? 기존 캐시 전부 역직렬화 실패.

2. **Java 종속 정보가 Redis에 노출된다.** `@class`는 Java/Kotlin 패키지 구조다. Redis에 저장된 데이터가 특정 언어의 내부 구조에 묶인다.

3. **페이로드가 비대해진다.** 모든 필드마다 타입 정보가 붙으니까 실 데이터보다 메타데이터가 클 때도 있다.

순수 JSON을 원했다. `@class` 없이, 사람이 읽어도 바로 이해할 수 있는 깨끗한 JSON.

---

## 💥 @class를 지웠더니

`activateDefaultTyping`을 빼고 일반 `ObjectMapper`로 직렬화기를 구성했다.

```kotlin
// @class 없는 순수 ObjectMapper
val redisObjectMapper = ObjectMapper().apply {
    findAndRegisterModules()
    disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
    // activateDefaultTyping 없음 → @class 삽입 안 함
}
val jsonSerializer = TypeAwareRedisSerializer(redisObjectMapper)
```
{: file="CacheConfig.kt" }

저장은 잘 됐다. Redis CLI로 확인하니 깨끗한 JSON이 들어가 있었다.

꺼냈더니 터졌다.

```
ClassCastException: LinkedHashMap cannot be cast to ProductInfo
```

`@class`가 없으니까 Jackson이 "이 JSON을 뭘로 만들어야 하지?"를 모른다. 타입 힌트 없이 JSON을 파싱하면 Jackson의 기본 동작은 `LinkedHashMap`으로 변환하는 것이다. `Map`은 `ProductInfo`가 아니니까 캐스팅에서 터진다.

`Jackson2JsonRedisSerializer`라는 대안도 있다. 생성 시 타입을 고정할 수 있다:

```kotlin
Jackson2JsonRedisSerializer(ProductInfo::class.java)
```

근데 이러면 `ProductInfo` 전용 직렬화기가 된다. 캐시가 여러 개인데(`ProductInfo`, `BrandInfo`, `Page<ProductResult>`...) 캐시마다 직렬화기를 따로 만들어야 한다. 현실적이지 않다.

---

## 🧵 AOP는 타입을 알고 있다

`LinkedHashMap`으로 계속 터지길래 AOP 코드를 다시 보다가 단서를 잡았다.

```kotlin
@Cached(cacheName = CacheNames.PRODUCT_INFO)
fun getProductInfo(productId: Long): ProductInfo {
    // ...
}
```

`@Cached`를 처리하는 `CachedAspect`는 AOP다. 메서드를 가로채는 시점에 **리턴 타입을 이미 알고 있다**. `getProductInfo`의 리턴 타입은 `ProductInfo`라는 정보가 리플렉션으로 추출 가능하다.

문제는 이 타입 정보를 `RedisSerializer`까지 어떻게 전달하느냐.

Spring Cache의 호출 흐름은 이렇다:

```
CachedAspect → CacheManager → Cache.get(key) → RedisSerializer.deserialize(bytes)
```

`CachedAspect`와 `RedisSerializer` 사이에 CacheManager, Cache, RedisTemplate이 끼어있다. 메서드 파라미터로 타입을 넘길 수 있는 구간이 없다.

**ThreadLocal**을 썼다.

```kotlin
class TypeAwareRedisSerializer(
    private val objectMapper: ObjectMapper,
) : RedisSerializer<Any> {

    companion object {
        private val expectedType = ThreadLocal<JavaType>()

        fun setExpectedType(type: JavaType) { expectedType.set(type) }
        fun clearExpectedType() { expectedType.remove() }
    }

    override fun deserialize(bytes: ByteArray?): Any? {
        if (bytes == null || bytes.isEmpty()) return null
        val type = expectedType.get()
        return if (type != null) {
            objectMapper.readValue(bytes, type)  // 타입 힌트가 있으면 정확한 타입으로 복원
        } else {
            objectMapper.readValue(bytes, Any::class.java)  // 없으면 LinkedHashMap (fallback)
        }
    }
}
```
{: file="TypeAwareRedisSerializer.kt" }

`CachedAspect`에서는 캐시 조회 직전에 타입을 설정하고, `finally`에서 정리한다:

```kotlin
private fun getCacheValue(cache: Cache, key: String, returnType: JavaType): Any? {
    return try {
        TypeAwareRedisSerializer.setExpectedType(returnType)  // 타입 힌트 설정
        cache.get(key)?.get()
    } catch (e: Exception) {
        log.warn("캐시 조회 실패, DB 조회 진행 [key={}]", key, e)
        null
    } finally {
        TypeAwareRedisSerializer.clearExpectedType()  // ThreadLocal 정리 — 누수 방지
    }
}
```
{: file="CachedAspect.kt" }

같은 스레드 안에서 AOP → CacheManager → RedisSerializer로 흐르니까, ThreadLocal이 타입 정보를 실어나르는 파이프 역할을 한다.

`@class` 없는 순수 JSON이 저장되고, 꺼낼 때는 AOP가 알려준 타입으로 정확히 복원된다.

---

## 🕐 됐다고 생각했는데 — ZonedDateTime

E2E 테스트를 돌렸다. 상품 상세 조회를 두 번 호출하면 두 번째는 캐시 히트가 되어야 하는데, 매번 DB를 찍고 있었다.

로그를 보니:

```
WARN  캐시 저장 실패 [L2, cacheName=product-info, key=1]
```

L1(Caffeine)은 정상인데 L2(Redis)만 실패. Caffeine은 JVM 힙에 **객체 참조를 그대로** 저장한다. 직렬화를 안 한다. Redis는 네트워크를 타니까 JSON으로 변환해야 하고, 그 과정에서 터진 거다.

원인은 `ZonedDateTime`. `ObjectMapper`에 `JavaTimeModule`이 없으면 `ZonedDateTime`을 직렬화할 수 없다. Spring의 기본 `ObjectMapper`에는 `JavaTimeModule`이 등록되어 있지만, Redis용으로 **별도 생성**한 `ObjectMapper`에는 없었다.

```kotlin
val redisObjectMapper = ObjectMapper().apply {
    findAndRegisterModules()  // ← 이걸로 JavaTimeModule, KotlinModule 자동 등록
    disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)  // ISO-8601 문자열로 저장
}
```
{: file="CacheConfig.kt" }

`findAndRegisterModules()`가 ServiceLoader를 통해 클래스패스에 있는 모듈을 자동 탐지한다. 이걸 빼먹으면 `JavaTimeModule`이 없어서 날짜/시간 필드에서 터진다.

L1은 직렬화를 안 하니까 문제가 없었고, L2만 터지니까 발견하기도 늦었다.

---

## 📄 이번엔 Page가 — 제네릭 역직렬화

`ProductInfo` 캐시는 해결됐다. 다음은 상품 목록 — `Page<ProductResult>`.

Spring Data의 `PageModule`은 `Page`를 직렬화할 때 이런 구조로 만든다:

```json
{
  "content": [
    { "id": 1, "name": "상품A", "price": 15000 },
    { "id": 2, "name": "상품B", "price": 23000 }
  ],
  "page": {
    "size": 20,
    "number": 0,
    "totalElements": 100,
    "totalPages": 5
  }
}
```

깔끔한 구조인데, 문제는 `PageModule`이 **역직렬화기를 제공하지 않는다**는 것. `Page`는 인터페이스라서 Jackson이 어떤 구현체로 만들어야 하는지 모른다.

ThreadLocal로 `Page<ProductResult>` 타입이 전달되더라도, Jackson은 `Page` 인터페이스를 인스턴스화할 수 없다. 수동 복원이 필요했다.

```kotlin
private fun deserializePage(bytes: ByteArray, pageType: JavaType): Page<*> {
    val tree = objectMapper.readTree(bytes)

    // content의 제네릭 타입 추출 (Page<ProductResult> → ProductResult)
    val contentType = pageType.containedType(0)
    val listType = objectMapper.typeFactory
        .constructCollectionType(List::class.java, contentType)
    val content: List<Any> = objectMapper.readValue(
        objectMapper.treeAsTokens(tree.get("content")), listType,
    )

    // page 메타데이터 추출
    val pageNode = tree.get("page")
    val number = pageNode.get("number").asInt()
    val size = pageNode.get("size").asInt()
    val totalElements = pageNode.get("totalElements").asLong()

    return PageImpl(content, PageRequest.of(number, size), totalElements)
}
```
{: file="TypeAwareRedisSerializer.kt" }

`pageType.containedType(0)`으로 `Page<ProductResult>`에서 `ProductResult`를 꺼낸다. 이게 ThreadLocal로 전달된 `JavaType`이니까 가능한 것이다. 타입 정보가 없었으면 `content` 안의 각 원소를 뭘로 역직렬화해야 하는지 알 수 없다.

---

## 🧪 결과 — Redis에 저장된 실제 JSON

상품 상세 캐시:

```json
{
  "id": 1,
  "name": "무선 키보드",
  "description": "인체공학 설계",
  "price": 49000,
  "stockQuantity": 150,
  "likeCount": 42,
  "brandName": "로지텍",
  "status": "ACTIVE",
  "createdAt": "2026-03-11T15:30:27.123+09:00"
}
```

`@class` 없다. Java 패키지 구조도 없다. 사람이 읽을 수 있는 깨끗한 JSON.

클래스를 리팩토링해도 캐시가 깨지지 않는다. 필드명만 같으면 된다.

---

## 🤔 남은 고민

### ThreadLocal 방식

동작은 하지만, AOP와 Serializer 사이를 ThreadLocal이라는 암묵적 채널로 연결하는 구조다. `finally`에서 `clearExpectedType()`을 빼먹으면 다른 요청에 타입이 새어나갈 수 있다. 코루틴 같은 비동기 환경에서는 ThreadLocal 자체가 위험하기도 하다.

캐시 값에 타입 힌트 헤더를 붙이는 방식(예: 첫 바이트에 타입 인덱스를 넣는다든가)도 생각해봤는데, 그러면 Redis CLI에서 깨끗한 JSON으로 안 보이니까 원래 목적이 훼손된다.

### DTO 필드 변경 시 캐시 호환성

`ObjectMapper`에 `FAIL_ON_UNKNOWN_PROPERTIES = false`를 걸어두면 모르는 필드는 무시하고 역직렬화한다. 필드가 추가되는 건 괜찮다. 근데 기존 필드의 타입이 바뀌거나 삭제되면 여전히 깨진다.

캐시 키에 버전 접미사(`product-info-v2`)를 붙이면 확실하지만, 배포할 때마다 버전을 올려야 하는 운영 부담이 생긴다. 당장은 `FAIL_ON_UNKNOWN_PROPERTIES`로 가되, 필드 타입 변경이 잦아지면 버전 접미사를 도입할 생각이다.

---

## 📝 돌아보면

```
1일차: JSON으로 저장하자 → ObjectMapper 구성 → 저장 성공 ✅
       역직렬화 → LinkedHashMap → ClassCastException 💥

2일차: ThreadLocal로 타입 전달 → 역직렬화 성공 ✅
       E2E 테스트 → L2만 실패 → ZonedDateTime 💥
       JavaTimeModule 등록 → 성공 ✅
       Page 캐시 → 역직렬화 실패 → Page 인터페이스 💥
       수동 복원 로직 → 성공 ✅
```

돌이켜보면 직렬화(넣기)에서 막힌 적은 없다. 타입 전달, 날짜 처리, 제네릭 복원 — 전부 역직렬화(꺼내기) 쪽 문제였다.

Serialize의 어원으로 다시 돌아가면, 객체를 한 줄로 펴는 건 기계적인 작업이다. 어려운 건 **펴놓은 걸 원래 모양으로 되돌리는 것**. 원래 모양을 기억하고 있어야 하니까.

> 넣을 때는 마음대로였다. 꺼낼 때는 아니었다.
{: .prompt-tip }
