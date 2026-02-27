---
title: "지도는 아직 없다, 나침반은 생겼다 — DIP와 도메인 서비스, 멘토링 두 번의 혼란"
date: 2026-02-27 00:00:00 +0900
categories: [Development, Architecture]
tags: [dip, layered-architecture, domain-service, clean-architecture, kotlin, refactoring]
image:
  path: /assets/img/posts/compass-fog.jpg
---

> **TL;DR** 💡 Repository 분리하면 DIP 끝인 줄 알았다. 멘토링 두 번 듣고 나니 DTO 구조를 뜯어고쳤고, "도메인 서비스"의 정의부터 다시 헷갈린다. 아직 정리가 안 됐다. 그 과정을 기록한다.
{: .prompt-tip }

## 🔧 처음 구현 — "Repository 쪼개면 되는 거 아니야?"

회사에서 1년간 써온 구조가 있다.

```
Controller → ApiService → Service → Repository(JPA 직접)
```

`OrderApiService`가 여러 Service를 조합하고, `OrderManageService`가 `OrderManageDetailService`, `OrderManageSearchService`에 위임한다. 불편한 적 없었다. 잘 돌아갔다.

3주차 과제가 나왔다. **레이어드 아키텍처 + DIP 적용. DIP 미적용 시 탈락.**

처음엔 단순하게 생각했다. Repository를 인터페이스/구현체로 나누면 되는 거 아닌가?

```kotlin
// Domain Layer — 인터페이스만 정의
interface UserRepository {
    fun save(user: User): User
    fun findByLoginId(loginId: String): User?
    fun findById(id: Long): User?
}

// Infrastructure Layer — JPA로 구현
@Component
class UserRepositoryImpl(
    private val userJpaRepository: UserJpaRepository,
) : UserRepository {
    override fun save(user: User): User = userJpaRepository.save(user)
    override fun findByLoginId(loginId: String): User? =
        userJpaRepository.findByLoginId(loginId)
    override fun findById(id: Long): User? = userJpaRepository.findByIdOrNull(id)
}
```

회사에서도 `OrdersRepositoryCustom` + `OrdersRepositoryImpl`로 쿼리를 분리하고 있었다. 비슷하지 않나? 싶었는데, 회사 코드는 쿼리 분리이지 DIP가 아니었다. Domain이 여전히 JPA를 직접 의존하고 있었으니까.

```
[ 회사 ]
Service → OrdersRepository extends JpaRepository  ← Domain이 JPA에 종속

[ 부트캠프 ]
Service → UserRepository(우리가 만든 인터페이스)  ← Domain은 JPA를 모름
               ↑ implements
          UserRepositoryImpl → UserJpaRepository
```

차이점은 Domain의 import에 JPA가 있느냐 없느냐였다.

분리하고 나니 FakeRepository를 만들 수 있었다.

```kotlin
class FakeUserRepository : UserRepository {
    private val store = mutableMapOf<Long, User>()
    private var autoIncrementId = 1L

    override fun save(user: User): User {
        // 테스트용 ID 자동 채번
        val saved = if (user.id == 0L) {
            user.also { /* 리플렉션으로 id 세팅 */ }
        } else user
        store[saved.id] = saved
        return saved
    }

    override fun findByLoginId(loginId: String) =
        store.values.find { it.loginId == loginId }
    override fun findById(id: Long) = store[id]
}
```

155개 테스트가 Spring 없이 밀리초 단위로 돌아갔다. DB 연결도 없이. "이거 별거 아니네?" 싶었다.

이때까지만 해도 DIP를 안다고 생각했다.

---

## 🎓 D 멘토링 — "레이어 건너뛰고 있어"

D 멘토 멘토링을 청강하면서 충격받은 부분이 있었다.

> "Enum을 API에서 그대로 쓰면 사고야. 상태코드 바꾸면 API 버전이 달라진다. 앱 네이티브 생각해봐."

> "Request에서 Command로 직접 변환하고 있네? Interfaces가 Domain을 직접 의존하고 있잖아."

내 코드를 돌아보니 진짜 그랬다:

```kotlin
// ❌ Controller에서 Domain의 Command를 직접 import
import com.loopers.domain.product.CreateProductCommand

class ProductAdminV1Controller {
    fun createProduct(@RequestBody request: CreateRequest) {
        val command = CreateProductCommand(...)  // Interfaces → Domain 직접 의존
        productFacade.createProduct(command)
    }
}
```

의존 방향이 `Interfaces → Application → Domain`이어야 하는데, Application을 건너뛰고 있었다.

Repository 분리만 하면 끝인 줄 알았는데, DTO 흐름까지 레이어를 지켜야 했다. 이걸 몰랐다.

---

## 🔨 리팩토링 — Criteria/Result 도입, 근데 이게 맞나?

D 멘토가 보여준 구조:

```
입력: Request(interfaces) → Criteria(application) → Command(domain) → Entity
출력: Entity → Info(domain) → Result(application) → Response(interfaces)
```

각 레이어의 DTO가 다음 레이어의 DTO로만 변환된다. 건너뛰기가 없다.

| 객체 | 위치 | 역할 |
|------|------|------|
| Request / Response | interfaces | HTTP 직렬화. Enum은 String으로 래핑 |
| Criteria / Result | application | Facade 입출력. 여러 도메인 조합 가능 |
| Command / Info | domain | Domain Service 입출력 계약 |

적용했다. 레이어 건너뛰기는 사라졌다.

근데 솔직히 의문이 들었다.

Brand 도메인은 단순하다. Criteria랑 Command가 거의 똑같다.

```kotlin
// application — Criteria
data class CreateBrandCriteria(
    val name: String,
    val description: String,
)

// domain — Command. 뭐가 다른 건데?
data class CreateBrandCommand(
    val name: String,
    val description: String,
)
```

Facade에서 변환하는 코드도 사실상 pass-through다:

```kotlin
// BrandFacade
fun createBrand(criteria: CreateBrandCriteria): BrandResult {
    val command = CreateBrandCommand(
        name = criteria.name,              // 그대로 넘기기
        description = criteria.description // 그대로 넘기기
    )
    val info = brandService.create(command)
    return BrandResult.from(info)  // 이것도 거의 그대로
}
```

D 멘토도 "그대로 쓸 수 있으면 허용한다"고 했는데, 그 "쓸 수 있으면"의 기준을 모르겠다.

Order 도메인에서는 확실히 필요했다. 여러 도메인을 조합하니까. Criteria에서 productIds를 꺼내 Product를 조회하고, 여러 Info를 하나의 Result로 합치고. 레이어별 DTO가 다른 게 자연스러웠다.

근데 Brand 같은 단순 CRUD 도메인에도 전부 넣어야 하나?

> 이때 내 결론: 일관성을 위해 전부 적용하되, 단순한 도메인은 거의 pass-through라는 걸 인정한다. 확신은 없다.
{: .prompt-info }

---

## 🌀 L 멘토링 — "그거 도메인 서비스 아닌데?"

여기서 진짜 혼란이 왔다.

L 멘토 멘토링의 핵심 메시지:

> "도메인 서비스는 순수하게 객체의 역할과 책임만 가져야 한다."

> "데이터를 꺼내오고 저장하는 건 도메인의 역할이 아니다."

> "Domain Service에 Repository 주입이 없어야 한다."

L 멘토가 보여준 코드 — 계좌이체:

```java
// 애플리케이션 서비스 — 데이터를 꺼내서 도메인 서비스에 넘김
class TransferAccountApplicationService {
    void execute(...) {
        Account from = repository.findBy(...);
        Account to = repository.findBy(...);
        transferAccountService.transfer(from, to, amount);
    }
}

// 도메인 서비스 — Repository 주입 없음, 순수 객체 협력만
class TransferAccountService {
    int transfer(Account toAccount, Account fromAccount, int amount) {
        fromAccount.deduct(amount);
        toAccount.plus(amount);
    }
}
```

그리고 주문 도메인에서도:

```java
// 애플리케이션 서비스가 데이터를 꺼내서 도메인 서비스에 넘긴다
class OrderService {
    void orderSave(Long userId, Long couponId, int amount) {
        User user = userRepo.findBy(userId);
        Coupon coupon = couponRepo.findBy(couponId);
        Order order = orderDomainService.order(user, coupon, amount);  // 순수 로직
        repo.save(order);
    }
}

// 도메인 서비스 — Repository 없음
class OrderDomainService {
    Order order(User user, Coupon coupon, int amount) {
        coupon.apply(amount, user);  // 쿠폰 적용은 비즈니스 규칙
        return new Order(...);
    }
}
```

이 기준으로 내 코드를 돌아봤다.

```kotlin
class BrandService(
    private val brandRepository: BrandRepository  // ← Repository 주입되어 있음
) {
    fun create(command: CreateBrandCommand): Brand {
        return brandRepository.save(Brand.create(command))
    }

    fun update(brandId: Long, command: UpdateBrandCommand): Brand {
        val brand = brandRepository.findById(brandId) ?: throw ...
        brand.update(command)
        return brand
    }
}
```

이 `BrandService`가 하는 일을 뜯어보면:

```
create  → Brand.create() + repository.save()     ← 데이터 접근
update  → repository.find() + brand.update()      ← 데이터 접근 + 엔티티 위임
delete  → repository.find() + brand.delete()      ← 데이터 접근 + 엔티티 위임
findAll → repository.findAll()                     ← 데이터 접근
```

**비즈니스 로직이 없다.** 진짜 도메인 로직(검증, 상태 변경)은 전부 Entity 안에 있고, Service는 Repository를 감싼 래퍼다.

D 멘토 기준: "Repository Interface가 Domain에 있으니까, Domain Service가 호출해도 문제없다."

L 멘토 기준: "그건 Domain Service가 아니다. Repository를 갖고 있잖아."

같은 코드를 보고 한 명은 OK, 한 명은 NO.

---

## 🤯 같은 단어, 다른 정의

처음엔 "누가 맞는 거야?"라고 생각했다.

멘토링 끝나고 정리하면서 내 `UserService`를 다시 뜯어봤다:

```kotlin
fun signUp(command: SignUpCommand): User {
    // 1. DB 조회 (Repository 호출) ← 이건 "흐름 제어"
    if (userRepository.findByLoginId(command.loginId) != null) { ... }

    // 2. 비밀번호 검증 ← 이건 "순수 비즈니스 로직"
    PasswordValidator.validate(command.password, command.birthday)

    // 3. 비밀번호 암호화
    val encodedPassword = passwordEncoder.encode(command.password)

    // 4. DB 저장 (Repository 호출) ← 이건 "흐름 제어"
    return userRepository.save(user)
}
```

"잠깐, 이게 진짜 Domain Service 맞아?"

1번과 4번(DB 조회/저장)은 흐름 제어, 즉 오케스트레이션이다. 2번(비밀번호 검증)만 순수 비즈니스 로직이다. 하나의 Service가 두 가지 역할을 동시에 하고 있었다.

L 멘토 기준이라면 2번만 Domain Service의 일이고, 1번·4번은 Application Service(Facade)의 일이다. 그러면 쪼개야 하나?

`BrandService`를 보면 더 헷갈린다. 비즈니스 로직이 아예 없다:

```
create  → Brand.create() + repository.save()     ← 데이터 접근만
update  → repository.find() + brand.update()      ← 데이터 접근 + 엔티티 위임
findAll → repository.findAll()                     ← 데이터 접근만
```

진짜 도메인 로직(검증, 상태 변경)은 전부 Entity 안에 있고, `BrandService`는 Repository를 감싼 래퍼일 뿐이었다.

근데 이건 Brand가 단순 CRUD 도메인이라서 그런 걸 수도 있다. 비즈니스 규칙이 "이름이 비면 안 된다" 정도밖에 없으니까 Entity 하나로 충분한 거고, Domain Service가 필요 없는 게 당연한 거 아닌가?

실제로 Order 도메인에서는 달랐다. 재고 차감, 포인트 차감, 주문 아이템 조합 — 여러 객체가 협력해야 하는 로직이 있었고, 그걸 `OrderDomainService`로 분리하니까 자연스러웠다. Brand에서 "Domain Service가 없네?"라고 느낀 건 Domain Service가 필요 없는 도메인에서 Domain Service를 찾으려 한 게 아닌가?

그렇다고 `BrandService`의 정체가 풀리는 건 아니다. Domain Service가 아니라면 Application Service? 근데 Facade가 이미 Application Layer에 있는데?

억지로 정리하면 이렇다:

D 멘토와 L 멘토는 **같은 단어를 다른 의미로 쓰고 있었다.**

```
D 구조:   Facade → Service(Repo 주입 OK)
L 구조:   Facade(= Application Service) → DomainService(Repo 주입 X, 순수 로직)
```

| | D 멘토 | L 멘토 |
|---|---|---|
| **Facade** | 오케스트레이션 | = Application Service (동의어) |
| **Service (Repo 주입)** | Domain Service라고 부름 | 이건 Application Service 역할 |
| **순수 객체 협력** | 필요할 때만 등장 | 이게 진짜 Domain Service |

L 멘토도 "Application Service와 Facade는 동의어라고 생각해도 된다"고 했다. 그러면 D 구조에서 Facade와 Service 사이에 있는 게, L 구조에서는 Application Service 하나로 합쳐지는 셈이다.

용어가 달랐지, 의존 방향(`Application → Domain ← Infrastructure`)은 둘 다 같았다.

---

## 🏢 회사 코드를 다시 봤다

회사의 `OrderManageService`를 다시 보면:

```
OrderManageService
  ├─ OrderManageDetailService에 위임 (여러 Repository 조회)
  ├─ OrderManageSearchService에 위임 (읽기 전용 쿼리)
  ├─ OrderShippingService 호출
  └─ OrderChangeLogManageService 호출
```

Repository를 주입받고, 비즈니스 로직도 하고, 데이터 접근도 한다. D 스타일이다. 실무에서 가장 흔한 패턴이라고 했는데, 실제로 1년간 이렇게 쓰고 있었다.

L 스타일(순수 Domain Service)은 회사 코드에서 본 적이 없다.

하지만 이번 과제에서 Order 도메인에 `OrderDomainService`를 순수하게 만들어보니, 테스트가 확실히 깔끔해졌다:

```kotlin
@Component
class OrderDomainService {
    // Repository 주입 없음

    fun placeOrder(command: CreateOrderCommand): Order {
        val orderItems = command.products.map { product ->
            val qty = command.quantities[product.id]
                ?: throw CoreException(ErrorType.BAD_REQUEST, "주문 수량 정보가 없습니다.")
            val brand = command.brands[product.brandId]
                ?: throw CoreException(ErrorType.BAD_REQUEST, "브랜드 정보가 없습니다.")
            OrderItem(
                productId = product.id,
                quantity = qty,
                productSnapshot = ProductSnapshot.from(product, brand),
                priceSnapshot = PriceSnapshot.from(product),
            )
        }

        val order = Order(userId = command.userId)
        orderItems.forEach { order.addItem(it) }
        order.calculateTotalAmount()
        return order
    }
}
```

Fake Repository도 필요 없이, 객체만으로 테스트가 됐다. Spring 컨텍스트는커녕 메모리 Map도 필요 없다. 그냥 객체를 만들어서 넣으면 끝.

```kotlin
@Test
fun `수량 정보가 없으면 BAD_REQUEST 예외가 발생한다`() {
    val command = createOrderCommand(
        products = listOf(createProduct()),
        quantities = emptyMap(),
    )

    val result = assertThrows<CoreException> {
        orderDomainService.placeOrder(command)
    }

    assertThat(result.errorType).isEqualTo(ErrorType.BAD_REQUEST)
}
```

두 분 다 맞는 말이었다. 선을 어디에 긋느냐의 차이고, 그 선에 따라 테스트 전략이 달라진다.

---

## 🤔 아직 모르겠는 것들

3주차가 끝났는데 정리가 안 된 게 더 많다.

- 내 `BrandService`는 Domain Service인가 Application Service인가. L 멘토 기준으로는 Application Service인데, D 멘토 구조에서는 Domain Layer에 있다.

- Criteria/Result를 단순 CRUD 도메인에도 전부 넣는 게 맞는지 아직 확신이 없다. Order에서는 확실히 필요했는데 Brand에서는 pass-through.

- `UserService.signUp()`처럼 흐름 제어와 비즈니스 로직이 섞여있는 코드를 보면, 이걸 쪼개야 하는지 이대로 둬야 하는지 매번 답이 흔들린다.

- DIP를 "완벽하게" 지키려면 Entity에서 `@Entity`도 빼고 순수 객체로 써야 한다. 근데 K 멘토님 강의에서 이런 말이 나왔다:

> "Domain에서 Entity 쓰지 말고 순수 객체로 하는 게 DIP 아냐? Infrastructure에서 Entity가 맞지 않아?? **이게 함정이다!!**"

> "JPA로 구현한 리포지터리를 마이바티스로 변경한 적이 없고, RDBMS를 MongoDB로 변경한 적도 없다. 변경이 거의 없는 상황에서 변경을 미리 대비하는 것은 과하다. DIP를 완벽하게 지키면 좋겠지만 개발 편의성과 실용성을 가져가면서 구조적인 유연함은 어느정도 유지했다."
> — 최범균, 『도메인 주도 개발 시작하기』
{: .prompt-warning }

그 선을 내가 스스로 그을 수 있을까? 아직은 모르겠다.

L 멘토도 이렇게 말했다:

> "멘토링 들을수록 이리저리 흔들릴 수 있다. 그 안에서 자신의 생각을 정리해가는 게 중요하다."

> 멘토링을 두 번 듣고, 같은 단어가 다른 뜻으로 쓰이는 걸 보고, 코드를 두 번 뜯어고쳤다. 지도는 아직 없다. 근데 나침반은 하나 생겼다 — 의존 방향만 지키면, 디테일은 선택이다.
{: .prompt-tip }

