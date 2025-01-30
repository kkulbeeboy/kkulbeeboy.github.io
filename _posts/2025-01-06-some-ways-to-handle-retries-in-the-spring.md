---
title: "스프링에서 재시도를 처리하는 몇 가지 방법 (@Retryable, RetryTemplate)"
date: 2025-01-06 00:00:00 +0900
categories: [backend, spring]
tags: [spring, retry, spring-retry, Retryable, Recover, RetryTemplate]
---

## 재시도 처리가 필요한 상황

서비스를 운영하면 외부 서버와 통신할 일이 빈번하고, 특히 MSA 구조를 채택한 시스템에서는 서버 간의 통신이 더욱 잦아졌습니다.

이러한 서비스 간 통신에서 주의해야 될 점은 **장애가 전파될 수 있다는** 점입니다. 예를 들어, A 서비스가 B 서비스를 호출하는 상황에서 B 서비스에 장애가 발생하면, 그 장애가 A 서비스로 전파될 수 있습니다. 따라서 서버 간 통신에서 장애 전파를 처리하는 것은 서비스 운영에 중요한 요소입니다.

그런데 만약 호출 받은 서버에서 발생한 오류가 **일시적인 오류**라면, 즉, 오류가 잠시 발생한 것일 뿐, 다시 시도하면 정상적인 응답을 받을 수 있는 경우라면 이를 해결하는 가장 간단하고 효과적인 방법은 무엇일까요? 아마도 **재시도 처리를 통해** 일시적인 오류를 극복하고 정상적인 처리를 하는 방법일 것입니다.

그렇다면 **Spring에서 재시도를 어떻게 효과적으로 처리할 수 있을지** 알아보겠습니다.

<br>

## Spring 에서 재시도를 처리하는 방법

스프링에서는 재시도 처리를 위한 여러 방법을 지원하며, 대표적인 방법으로는 다음과 같습니다.

- **어노테이션 기반의 재시도 처리**
    - `@Retryable` 어노테이션을 사용하여 재시도 로직을 선언적으로 처리합니다.
- **RetryTemplate을 이용한 재시도 처리**
    - `RetryTemplate` 클래스를 사용하여 코드 내에서 재시도 로직을 명시적으로 처리합니다.

이제, 예시 상황을 들어 각각의 재시도 처리 방법을 코드를 통해 알아보고, 비교해 보겠습니다.

<br>

## 예시 상황

> 회원에게 구매한 금액만큼의 포인트를 지급하는데, 멤버십 구독에 가입되어 있는 회원에게는 구매한 금액의 2배 포인트를 지급합니다.
> 

> 이때 회원이 멤버십 구독에 가입되어 있는 회원인지 확인하기 위해 회원 서버와 통신을 진행합니다.
> 

<br>

### **코드 설계**

**UserState**

```kotlin
enum class UserState {
    NORMAL,
    SUBSCRIPTION;

    companion object {
        fun fromString(userState: String): UserState {
            return values().find { it.name.equals(userState, ignoreCase = true) } ?: NORMAL
        }
    }
}
```

- 회원 상태를 나타내는 enum 클래스입니다.
- NORMAL은 일반회원, SUBSCRIPTION은 멤버십 구독회원 상태를 나타냅니다.
- `fromString`를 통해 문자열 값을 **UserState** enum 값으로 변환할 수 있습니다.

<br>

**UserService**

```kotlin
@Service
class UserService(
    private val userClient: UserClient,
) {
    fun getUserState(id: Long): UserState {
        val userState = userClient.getUser(id = id).state
        return UserState.fromString(userState = userState)
    }
}
```

- `getUserState`를 통해 **회원 상태(멤버십 구독 여부)를 조회합니다.**
- **UserClient**의 `getUser`는 **회원 서버와 통신을 통해** 회원 정보(id, 이름, 상태)를 조회합니다.
    - 일시적 통신 오류가 발생하면, **TransientCommunicationException**가 발생합니다.
    - 존재하지 않는 회원이면, **UserNotFoundException**가 발생합니다.

<br>

**PointService**

```kotlin
@Service
class PointService(
    private val userService: UserService,
) {

    fun calculatePoint(userId: Long, purchaseAmount: Long): Long {
        // 회원 상태 조회
        val userState = userService.getUserState(id = userId)

        // 회원 상태 확인 후, 보상 포인트 계산
        val rewardPoint = when(userState) {
            UserState.SUBSCRIPTION -> {
                purchaseAmount * 2
            }
            else -> {
                purchaseAmount
            }
        }

        return rewardPoint
    }
}
```

- `calculatePoint`를 통해 **지급할 포인트를 계산합니다.**
- userId는 회원번호, purchaseAmount는 구매 금액을 의미합니다.
- **회원 상태를 조회하여 이를 기반으로 지급할 포인트를 계산합니다.**

<br>

### **코드 구조**

전체적인 구조는 다음과 같이 표현할 수 있습니다.

![image.png](/assets/img/posts/spring-retry-example-code-design-structure.png)

<br>

### **요구 사항**

위와 같은 구조에서 재시도 처리에 대한 요구사항은 다음과 같습니다.

- 회원 서버와 통신할 때 일시적 장애로 인해 발생하는 `TransientCommunicationException` 발생 시에 **재시도를 통해 다시 호출을 수행합니다.**
    - 이때, **재시도 처리는 최대 2번**만 수행하며, **재시도 간의 간격은 0.5초(500ms)**로 처리합니다.
    - 재시도를 2회 수행해도 모두 실패하면, **실패한 회원 번호를 알림 서비스로 전달합니다.**
- 단, 회원 서버와 통신할 때 존재하지 않는 회원 조회로 인해 발생하는 `UserNotFoundException` 발생 시에는 **재시도 처리를 진행하지 않습니다.**

<br>

### **테스트 코드**

재시도 처리 요구사항을 만족하는지 확인하기 위한 테스트 코드는 다음과 같습니다.

<br>

1. **회원 서버와 첫 번째 호출에서 TransientCommunicationException이 발생하고, 두 번째 호출에서 성공적으로 응답을 가져오는 상황의 테스트 코드**
    
    ```kotlin
    @Test
    fun `should retry on TransientCommunicationException and eventually succeed`() {
        val userId = 1L
    
        // given
        // 첫 번째 호출에서 일시적 통신 오류가 발생하고, 두 번째 호출에서 성공하는 상황
        whenever(userClient.getUser(userId))
            .thenThrow(TransientCommunicationException("Temporary issue"))
            .thenReturn(UserResponse(id = userId, name = "jude", state = "subscription"))
    
        // when
        val userState = userService.getUserState(userId)
    
        // then
        // 결과가 성공적으로 반환되는지 확인
        assertEquals(UserState.SUBSCRIPTION, userState)
        // userClient.getUser()가 2번 호출되었는지 확인
        verify(userClient, times(2)).getUser(userId)
    }
    ```
    
    - 회원 서버와 두 번째 호출에서 성공했으므로, **성공적으로 응답을 가져왔는지 확인합니다.**
    - 회원 서버와 두 번째 호출에서 성공했으므로, **총 두 번 재시도를 수행했는지 확인합니다.**

2. **회원 서버와 첫 번째 호출에서 UserNotFoundException이 발생하면, 재시도를 수행하지 않는 상황의 테스트 코드**
    
    ```kotlin
    @Test
    fun `should not retry on UserNotFoundException`() {
        val userId = 1L
    
        // given
        // UserNotFoundException이 발생하도록 설정
        whenever(userClient.getUser(userId)).thenThrow(UserNotFoundException("User not found (userId: $userId)"))
    
        // when
        // 예외가 발생하는지 확인
        assertThrows<UserNotFoundException> {
            userService.getUserState(userId)
        }
    
        // then
        // userClient.getUser()가 1번만 호출되었는지 확인
        verify(userClient, times(1)).getUser(userId)
    }
    ```
    
    - 회원 서버와의 통신에서 UserNotFoundException이 발생했으므로, **재시도를 수행하지 않고, 호출을 1번만 수행했는지 확인합니다.**

3. **재시도 2번을 모두 수행해도 성공하지 못하면, 알림 서비스를 호출하는 상황의 테스트 코드**
    
    ```kotlin
    @Test
    fun `should call notifyService after max retry failures`() {
        val userId = 1L
    
        // given
        // 항상 TransientCommunicationException을 던지도록 설정
        whenever(userClient.getUser(userId))
            .thenThrow(TransientCommunicationException("Temporary issue"))
    
        // when
        assertThrows<TransientCommunicationException> {
            userService.getUserState(userId)
        }
    
        // then
        // 최대 재시도 횟수인 2번 호출되었는지 확인
        verify(userClient, times(2)).getUser(userId)
        // 최대 재시도 횟수만큼 재시도 처리 이후, 알림 서비스를 호출했는지 확인
        verify(notifyService, times(1)).send(userId)
    }
    ```
    
    - **최대 재시도 횟수만큼 재시도를 수행하고, 알림 서비스를 호출했는지 확인합니다.**

<br>

이제부터 **위 3가지 테스트 코드를 모두 통과하도록 재시도 처리를 하는 코드를 작성해 보겠습니다.**

<br>

## 어노테이션 기반의 재시도 처리 (@Retryable)

메서드에 어노테이션을 선언하여, 실패한 메서드를 자동으로 재시도 처리하는 방법입니다.

### build.gradle dependencies 추가

```kotlin
dependencies {
	...
	implementation("org.springframework.retry:spring-retry")
	implementation("org.springframework:spring-aspects")
	...
}
```

- 재시도 및 복구를 위해 필요한 어노테이션(**@Retryable**, **@Recover**)을 사용하기 위해 Spring Retry가 필요합니다
- **@Retryable**은 **AOP** 기반으로 동작하기 때문에 Spring Aspects가 필요합니다.

### Spring Retry 활성화

Spring Retry를 활성화하기 위해 설정 클래스(@Configuration 클래스)에 `@EnableRetry` 어노테이션을 추가해야 합니다.

```kotlin
@EnableRetry
@Configuration
class ServerConfiguration {
	...
}
```

### 재시도 메서드 설정: @Retryable

재시도를 수행할 메서드에 `@Retryable`을 선언하여 재시도 처리 및 설정을 할 수 있습니다.

```kotlin
@Service
class UserService(
    private val userClient: UserClient,
) {
    @Retryable(
        include = [TransientCommunicationException::class], // 재시도할 예외
        exclude = [UserNotFoundException::class], // 재시도를 수행하지 않을 예외
        maxAttempts = 2, // 최대 재시도 횟수
        backoff = Backoff(delay = 500) // 재시도 간격 (500ms)
    )
    fun getUserState(id: Long): UserState {
        val userState = userClient.getUser(id = id).state
        return UserState.fromString(userState = userState)
    }
}
```

- **include**
    - TransientCommunicationException를 재시도 처리 예외로 지정합니다.
    - default: 모든 예외를 재시도 처리합니다.
- **exclude**
    - UserNotFoundException를 재시도 처리하지 않을 예외로 지정합니다.
    - default: 모든 예외를 재시도 처리합니다.
- **maxAttempts**
    - 최대 재시도 횟수를 2회로 지정합니다.
    - default: 3회
- **backoff**
    - 재시도 간격을 500ms로 지정합니다.
    - default: 1초 (1000ms)

### 복구 메서드 설정: @Recover

`@Recover`를 선언하여, **최대 횟수만큼 재시도 처리 후에도 실패한 경우 복구 메서드를 수행할 수 있습니다.**

단, 복구 메서드의 파라미터에는 몇 가지 지켜야 될 특징이 있습니다. **첫 번째 파라미터에는 복구하려는 예외 유형(Throwable 유형)을 가지고 있어야 되며**, **다음 파라미터로는 재시도를 수행한 메서드의 파라미터 목록을 동일한 순서로 가지고 있어야 됩니다.** 

또한 리턴 타입에도 지켜야 될 특징이 있는데, **재시도를 수행한 메서드와 동일한 리턴 타입을 가져야 됩니다.**

@Recover를 통해 복구 메서드를 설정을 하지 않고, 최대 재시도 횟수만큼 재시도를 수행하고 실패하면 `ExhaustedRetryException`이 발생합니다.

```kotlin
@Recover
fun recoverGetUserState(e: Exception, id: Long): UserState {
    logger.info("Retry failed. (id: $id, message: ${e.localizedMessage})")
    notifyService.send(userId = id)
    throw e
}
```

- **첫 번째 파라미터**는 TransientCommunicationException 예외 유형을 포괄하는 `e: Exception`이며, **두 번째 파라미터**는 getUserState의 파라미터인 `id: Long`입니다.
- **리턴 타입**은 getUserState의 리턴 타입인, `UserState`입니다.
- 만약 최대 재시도 횟수만큼 재시도를 수행해도 실패를 했을 때, 조회하는 회원의 상태를 특정 상태(NORMAL or SUBSCRIPTION)로 처리하고 싶다면 복구 로직 반환값에 해당 상태를 반환하도록 하면 됩니다.

### 테스트 코드 수행 결과

그렇다면, 앞서 작성해 두었던 테스트 코드가 잘 동작하는지 확인해 보겠습니다.

![image.png](/assets/img/posts/spring-retry-annotation-test-result.png)

모든 테스트가 성공적으로 수행되어 재시도 처리 요구사항을 만족한 것을 확인할 수 있습니다.

<br>

## RetryTemplate을 이용한 재시도 처리

RetryTemplate을 이용하면 어노테이션 기반으로 재시도를 처리했던 것과 달리, 코드 내에서 재시도 정책과 동작 등을 명시적으로 제어하여 처리할 수 있습니다.

### build.gradle dependencies 추가

```kotlin
dependencies {
	...
	implementation("org.springframework.retry:spring-retry")
	...
}
```

- **RetryTemplate**을 이용한 재시도 처리는 Annotation 기반의 재시도와 달리, AOP 기반으로 동작하지 않습니다. 따라서 spring-retry 의존성만으로 충분합니다.

### RetryTemplate

Spring Retry는 다음과 같이 `execute` 메서드가 있는 **RetryOperations 인터페이스를 제공합니다.**

<br>

**RetryOperations**

```java
public interface RetryOperations {
  /**
   * 설정된 재시도 동작을 사용하여 제공된 {@link RetryCallback}을 실행합니다.  
   * 구성에 대한 자세한 내용은 구현 클래스를 참조하세요.
   *
   * @param <T> 반환값의 타입
   * @param retryCallback 실행할 {@link RetryCallback}
   * @param <E> 발생할 수 있는 예외 타입
   * @return {@link RetryCallback}이 성공적으로 실행되었을 때 반환된 값
   * @throws E {@link RetryCallback}이 재시도 과정에서 발생시킨 예외
   * @throws E 발생한 예외
   */
	<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback) throws E;
	...
}
```

여기서 execute 메서드의 매개변수인 **RetryCallback**은 **실패 시 재시도를 수행할 로직을 삽입할 수 있는 인터페이스입니다.**

<br>

**RetryCallback**

```java
public interface RetryCallback<T, E extends Throwable> {
  /**
   * 재시도 동작을 적용하여 작업을 실행합니다.  
   * 일반적으로 작업은 멱등성을 가져야 하지만, 구현에 따라 재시도 시 보상(Compensation) 로직을 적용할 수도 있습니다.
   *
   * @param context 현재 재시도 컨텍스트
   * @return 성공적으로 실행된 작업의 결과
   * @throws E 처리 실패 시 발생하는 예외
   */
	T doWithRetry(RetryContext context) throws E;
	...
}
```

<br>

**RetryTemplate**

**RetryTemplate** 은 **RetryOperations를 구현한 구현체**로, 설정 클래스(@Configuration)에서 RetryTemplate Bean을 구현해 보겠습니다.

```kotlin
@EnableRetry
@Configuration
class ServerConfiguration {

    @Bean
    fun retryTemplate(): RetryTemplate {
        val retryTemplate = RetryTemplate()
        
        // 백오프 정책 설정
        val fixedBackOffPolicy = FixedBackOffPolicy().apply {
            backOffPeriod = 500 // 재시도 간격 (500ms)
        }
        retryTemplate.setBackOffPolicy(fixedBackOffPolicy)

        // 재시도 정책 설정
        val retryPolicy: RetryPolicy = SimpleRetryPolicy(
            2, // 최대 2번까지 재시도
            mapOf(
                TransientCommunicationException::class.java to true, // // TransientCommunicationException 예외는 재시도 수행
                UserNotFoundException::class.java to false // UserNotFoundException 예외는 재시도 하지 않음
            ),
        )
        retryTemplate.setRetryPolicy(retryPolicy)

        return retryTemplate
    }
}
```

- **BackOffPolicy 설정**
    - `FixedBackOffPolicy`를 통해 재시도 간격을 고정 시간인 500ms로 설정합니다.
    - <details>
        <summary>BackOffPolicy 구현체 종류</summary>
        <div markdown="1">
        `BackOffPolicy`의 주요 구현체들은 재시도할 때 다음 시도를 얼마나 기다릴지(지연 시간)에 대한 정책을 정의합니다.
        <br> <br>
        `NoBackOffPolicy`: 지연 없이 즉시 재시도 <br>
        `FixedBackOffPolicy`: 일정한 시간 간격으로 재시도 <br>
        `ExponentialBackOffPolicy`: 재시도 간격을 지수적으로 증가 <br>
        `ExponentialRandomBackOffPolicy`: 지수 증가 + 랜덤 요소 추가 <br>
        `UniformRandomBackOffPolicy`: 일정 범위 내에서 랜덤한 지연 적용 <br>
        </div>
      </details> 
- **RetryPolicy 설정**
    - `SimpleRetryPolicy`를 통해 재시도 정책을 설정하여, 재시도 최대 횟수를 2회로 설정하였으며, **TransientCommunicationException** 예외가 발생하면 재시도를 수행하고, **UserNotFoundException** 예외가 발생하면 재시도를 하지 않도록 설정했습니다.
     - <details>
        <summary>RetryPolicy 구현체 종류</summary>
        <div markdown="1">
        `RetryPolicy`의 주요 구현체들은 각기 다른 재시도 정책을 정의하며, 특정 상황에서 작업을 재시도할지 여부를 결정합니다.
        <br> <br>
        `SimpleRetryPolicy`: 정해진 횟수만큼 재시도 <br>
        `AlwaysRetryPolicy`: 실패 시 무한 재시도 <br>
        `NeverRetryPolicy`: 절대 재시도하지 않음 <br>
        `CircuitBreakerRetryPolicy`: 일정 횟수 이상 실패하면 재시도를 차단 <br>
        `ExceptionClassifierRetryPolicy`: 예외 유형별로 다른 재시도 정책 적용 <br>
        `TimeoutRetryPolicy`: 특정 시간 내에서만 재시도 허용 <br>
        `ExpressionRetryPolicy`: SpEL (Spring Expression Language) 표현식을 사용하여 재시도 여부 결정 <br>
        `CompositeRetryPolicy`: 여러 개의 재시도 정책(RetryPolicy)을 조합하여 동작 <br>
        </div>
      </details>     

### RetryTemplate을 이용한 재시도 처리 및 복구

```kotlin
@Service
class UserService(
    private val userClient: UserClient,
    private val notifyService: NotifyService,
    private val retryTemplate: RetryTemplate
) {
    fun getUserState(id: Long): UserState {
        return try {
            retryTemplate.execute<UserState, RuntimeException> { context ->
                val userState = userClient.getUser(id = id).state
                UserState.fromString(userState)
            }
        } catch (e: TransientCommunicationException) {
            recoverGetUserState(e = e, id = id)
        }
    }
    
    fun recoverGetUserState(e: Exception, id: Long): UserState {
        notifyService.send(userId = id)
        throw e
    }
}
```

- 재시도를 수행할 callback을 RetryTemplate에 넘겨줍니다.
- `retryTemplate.execute`를 통해 재시도 로직을 수행하도록 합니다.
- 만약 최대 재시도 횟수만큼 재시도를 수행하고 실패하면 `catch` 블록으로 넘어가 `recoverGetUserState` 메서드를 호출하여 복구 로직을 수행합니다.

<br>

### 테스트 코드 수행 결과

마찬가지로 앞서 작성해 두었던 테스트 코드가 잘 동작하는지 확인해 보겠습니다.

![image.png](/assets/img/posts/spring-retry-retry-template-test-result.png)

모든 테스트가 성공적으로 수행되어 재시도 처리 요구사항을 만족한것을 확인할 수 있습니다.

<br>

## @Retryable vs RetryTemplate

두 가지 재시도 처리 방법을 이용하여, 요구사항을 만족하는 코드를 작성했습니다. 그렇다면, 두 방법의 장단점을 비교해 보겠습니다.

### @Retryable

**장점**

- 어노테이션을 통해 재시도 정책을 정의하여, 간단한 방법으로 재시도 로직을 수행할 수 있다.
- 재시도에 실패한 경우 @Recover 메서드를 호출하여 복구 로직을 수행할 수 있다.
- 기존 코드에 큰 변경 없이 사용이 가능하다.

**단점**

- 복잡한 재시도 정책, 백오프 정책을 수립하기 어렵다.
- AOP에 종속적이다.
    - AOP 설정이 올바르지 않으면 동작하지 않을 수 있습니다.
    - AOP는 기본적으로 public 메서드만 지원하므로 private 메서드에는 적용되지 않습니다.

### RetryTemplate

**장점**

- 재시도 로직을 코드로 작성하여, 복잡한 조건에 따른 정책을 쉽게 구현할 수 있다.
    - 복잡한 백오프 정책을 이용한 제어가 가능합니다. (ex 재시도 간격을 지수적으로 증가)
    - 복잡한 재시도 정책을 이용한 제어가 가능합니다. (ex 특정 시간 내에서만 재시도 허용)
- AOP 없이도 작동하며, 메서드의 접근 제한자(public, private 등)에 구애받지 않습니다.
- RetryTemplate 인스턴스를 여러 곳에서 재사용할 수 있어, 비슷한 재시도 로직이 많은 경우 코드 중복을 줄일 수 있습니다.

**단점**

- 재시도 로직이 코드에 명시되어, 어노테이션 방식에 비해 코드가 길어지고 복잡해질 수 있습니다.
- 간단한 재시도 로직을 구현할 때에도 불필요하게 코드가 복잡해질 수 있습니다.

<br>

## 정리

스프링에서 재시도를 수행하는 몇 가지 방법을 살펴봤습니다. 각 방법의 특징을 기반으로, 간단한 재시도 로직을 수행할 때는 `@Retryable`, 복잡한 재시도 로직을 수행하는 환경에서는 `RetryTemplate`을 사용하는 게 적절하다고 생각합니다.

<br>

---

baeldung, "Guide to Spring Retry", [https://www.baeldung.com/spring-retry](https://www.baeldung.com/spring-retry), (참고 날짜 2025.01.05)

hvho.github.io, "[ Spring Framework ] spring retry 로 간편하게 재수행 로직 추가하기", [https://hvho.github.io/2021-12-05/spring-retry](https://hvho.github.io/2021-12-05/spring-retry), (참고 날짜 2025.01.05)

Daddy programmer, "Spring Retry Review", [https://www.daddyprogrammer.org/post/12091/spring-retry-review/](https://www.daddyprogrammer.org/post/12091/spring-retry-review/), (참고 날짜 2025.01.05)

티스토리, "[Spring] spring-retry 재시도 및 백오프 정책 정리", [https://namocom.tistory.com/847](https://namocom.tistory.com/847), (참고 날짜 2025.01.30)