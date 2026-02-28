---
title: "🤔 Facade? 그거 우리 회사에서 그냥 'ApiService'라고 불렀는데"
date: 2026-02-06 00:00:00 +0900
categories: [Development, Architecture]
tags: [facade, dip, tdd, layered-architecture, kotlin]
image:
  path: /assets/img/posts/2026-02-06-cover.jpg
---

> **TL;DR** 💡 실무에서 쓰던 패턴에 이름이 있었다. 용어를 알고 나니 설명이 쉬워지고, 테스트도 쉬워졌다.
{: .prompt-tip }

## 🤔 이미 하고 있었는데, 이름을 몰랐다

부트캠프 과제로 User API를 구현하면서 Facade 패턴을 적용했다. 그런데 코드를 짜다 보니 이상하게 익숙했다.

> **Facade**는 프랑스어로 "건물의 정면/외관"이라는 뜻이다. 건물 앞면은 깔끔해 보이지만, 뒤에는 복잡한 구조가 숨어있다. 디자인 패턴에서도 같은 의미로, 복잡한 서브시스템을 단순한 인터페이스로 감싸는 역할을 한다.
{: .prompt-info }

"어? 이거 회사에서 맨날 하던 건데?" 😅

회사에서는 그냥 "서비스 조합하는 클래스" 정도로 불렀다. `OrderFacade` 같은 이름 대신 `OrderApiService`, `BuyerApiService` 이런 식으로 `XxxApiService`라고 불렀다. 하는 일은 똑같았다:

- 여러 Service를 조합
- 트랜잭션 경계 관리
- DTO 변환

이전 회사에서는 Controller → Service → Repository로 단순하게 썼는데, 지금 회사에서 `XxxApiService` 구조를 처음 접했다. 1년 가까이 이 구조로 코드를 짰는데, "Facade 패턴"이라는 이름이 있는 줄 몰랐다.

## ✨ 용어를 알고 나니 달라진 것

### 1. 🗣️ 설명이 쉬워졌다

예전에는 이런 식으로 설명했다:

> "이 클래스는 Service들을 조합해서 Controller에 전달하는 역할이에요"

지금은:

> "이건 Facade예요. 복잡한 서브시스템을 단순한 인터페이스로 감싸는 거죠"
{: .prompt-info }

한 단어로 의도가 전달된다.

### 2. 🏗️ 구조가 명확해졌다

```text
Controller → Facade → Service → Repository
```

Facade가 뭘 하는 레이어인지 이름만 봐도 알 수 있다.

### 3. 🧪 테스트 경계가 보였다

Facade를 알고 나니 "여기서 뭘 테스트해야 하지?"가 명확해졌다.

- **Service 테스트**: 비즈니스 로직 (Fake Repository로 단위 테스트)
- **Facade 테스트**: Service 조합, DTO 변환 (필요하면)
- **E2E 테스트**: HTTP → DB 전체 흐름

## 🔄 DIP도 마찬가지였다

Repository 인터페이스를 Domain 레이어에 두는 구조도 회사에서 쓰던 방식이었다.

```kotlin
// Domain Layer
interface UserRepository {
    fun findByLoginId(loginId: String): User?
    fun save(user: User): User
}
```
{: file="domain/UserRepository.kt" }

```kotlin
// Infrastructure Layer
class UserRepositoryImpl(
    private val jpaRepository: UserJpaRepository
) : UserRepository {
    override fun findByLoginId(loginId: String): User? =
        jpaRepository.findByLoginId(loginId)
    // ...
}
```
{: file="infra/UserRepositoryImpl.kt" }

예전에는 "JPA 의존성 분리하려고 이렇게 하는 거야"라고 설명했다. 지금은 "DIP(의존성 역전 원칙) 적용한 거야"라고 말할 수 있다.

그리고 이 구조 덕분에 테스트할 때 `FakeUserRepository`를 쉽게 끼워넣을 수 있었다:

```kotlin
// 테스트용 Fake 구현체
class FakeUserRepository : UserRepository {
    private val storage = mutableMapOf<Long, User>()
    private var sequence = 1L

    override fun save(user: User): User {
        val id = sequence++
        val savedUser = user.copy(id = id)  // 실제 저장처럼 id 채번
        storage[id] = savedUser
        return savedUser
    }
    // ...
}
```
{: file="test/FakeUserRepository.kt" }

```kotlin
// Mock 방식
val userRepo = mock<UserRepository>()
whenever(userRepo.findByLoginId("test")).thenReturn(User(...))
whenever(userRepo.save(any())).thenReturn(User(...))

// Fake 방식
val userRepo = FakeUserRepository()
userRepo.save(User(...))
```

## 🔥 이번 과제에서 겪은 실제 고민

### ⚠️ 트랜잭션 경계 문제

비밀번호 변경 API를 만들면서 삽질했다. 처음 구조는 이랬다:

```kotlin
// Facade
fun changePassword(loginId: String, loginPw: String, command: ChangePasswordCommand) {
    val user = userService.authenticate(loginId, loginPw)  // readOnly 트랜잭션
    userService.changePassword(user, command)              // 다른 트랜잭션
}
```
{: file="UserFacade.kt" }

> `authenticate()`가 `@Transactional(readOnly = true)`라서, 반환된 User 엔티티가 Detached 상태가 됐다. 이후 `changePassword()`에서 수정해도 DB에 반영이 안 됐다.
{: .prompt-warning }

결국 Service 메서드 시그니처를 바꿔서 해결 ✅

```kotlin
// Service
@Transactional
fun changePassword(loginId: String, loginPw: String, command: ChangePasswordCommand) {
    val user = findAndValidate(loginId, loginPw)  // 같은 트랜잭션 내에서 조회
    user.changePassword(passwordEncoder.encode(command.newPassword))
    // Dirty Checking으로 자동 UPDATE
}
```
{: file="UserService.kt" }

처음엔 Facade에 `@Transactional`을 붙여서 전체를 하나의 트랜잭션으로 묶을까 고민했다. 하지만 Facade가 트랜잭션을 관리하면 Service 단위 테스트가 어려워질 것 같아서, Service 내부에서 해결하는 방향을 선택했다.

`findAndValidate()`는 `authenticate()`와 중복되지만, private 메서드로 추출해서 DRY 원칙은 지켰다.

> 나중에 이벤트 기반으로 확장한다면 어떻게 해야 할지는 아직 모르겠다. 트랜잭션 경계를 어디에 둘지, 이벤트 발행 시점은 언제로 할지 등 고민이 더 필요할 것 같다.
{: .prompt-info }

### 🔒 마스킹 로직 위치

이름 마스킹을 어디에 둘지도 고민이었다. "홍길동" → "홍길*" 변환하는 로직인데:

- **A안**: Controller/DTO에서 처리
- **B안**: Facade/Application Layer에서 처리
- **C안**: Entity에 메서드로 정의

결국 C안을 선택했다 👇

```kotlin
// User 엔티티
fun maskedName(): String = if (name.length <= 1) "*" else name.dropLast(1) + "*"
```
{: file="User.kt" }

```text
┌─────────────────────────────────────────────────┐
│  Presentation Layer (Controller, DTO)           │  ← A안: 여기서 마스킹?
├─────────────────────────────────────────────────┤
│  Application Layer (Facade, UserInfo)           │  ← B안: 여기서 마스킹?
├─────────────────────────────────────────────────┤
│  Domain Layer (Entity, Service)                 │  ← C안: 여기서 마스킹? ✅
├─────────────────────────────────────────────────┤
│  Infrastructure Layer (Repository, JPA)         │
└─────────────────────────────────────────────────┘
```
{: file="Layered Architecture" }

"표현 방식"이니까 위쪽 레이어에 둘 수도 있지만, "외부에 이름을 어떻게 보여줄지"는 User 도메인이 알아야 할 지식이라고 판단해서 Entity에 뒀다. 정답인지는 모르겠지만.

## 📝 배운 점

> 앎은 작은 모름에서 큰 모름으로 나아가는 과정이다.

"그냥 이렇게 하면 편하더라"로 코드를 짰다. Facade, DIP 같은 용어를 알고 나니 오히려 모르는 게 더 많아졌다. 트랜잭션 전파는 어떻게 해야 하지? 이벤트 기반으로 확장하면? 테스트 경계는 어디까지?

그래도 용어를 알고 나니:

- 다른 개발자와 소통이 쉬워졌다
- 구조를 설명할 때 근거가 생겼다
- "왜 이렇게 하는 거야?"에 대답할 수 있게 됐다

> 이미 알던 것에 이름을 붙이니, 몰랐던 것들이 보이기 시작했다.
{: .prompt-tip }

다음에 누가 "Facade가 뭐야?"라고 물으면 이렇게 대답할 것 같다:

> "여러 Service 조합해서 Controller에 전달하는 그 클래스 있잖아. 그거."
