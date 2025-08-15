---
title: "다중 서버 환경에서 동시성 문제를 해결하기 위한 분산락 사용하기 대작전 (with Redis)"
date: 2025-08-15 00:00:00 +0900
categories: [backend, spring]
tags: [spring, redis, distributed lock, Lettuce, Redisson]
---

## 다중 서버 환경에서 발생하는 동시성 문제

최근 많은 서비스들은 고가용성을 위해 다중 서버 환경으로 운영되고 있습니다. 그렇다면, 이러한 다중 서버 환경에서 동시성을 어떻게 안정적으로 제어할 수 있을까요?

이번 글에서는 예시 상황을 통해 문제를 해결하는 과정을 보여드리겠습니다.

예를 들어, **돈을 미리 충전해 놓고 충전한 금액을 사용하는 선불전자지급수단 시스템**(이하 '충전형 결제 시스템')을 운영하고 있다고 가정해 보겠습니다.

<br>

### **충전형 결제 시스템의 결제 로직**

```kotlin
@Transactional
suspend fun pay(userId: Long, paymentKey: String, amount: Long){
    try {
        val prepaidAccount = prepaidAccountRepository.findByUserId(userId) // 충전 금액 조회
        prepaidAccount.deduct(amount = amount) // amount 금액 차감
        prepaidAccountRepository.save(prepaidAccount)
    } catch (e: InsufficientBalanceException){
        throw PaymentFailedException(message = "잔액이 부족합니다.", cause = e)
    } catch (e: Exception){
        logger.error("결제 요청 중 알 수 없는 오류 발생. (paymentKey: $paymentKey)", e)
        throw PaymentFailedException(message = "결제 처리 중 예기치 못한 오류가 발생했습니다.", cause = e)
    }
}

```

- `userId`: 회원 번호
- `paymentKey`: 회원별 고유 결제 키
- `amount`: 결제 금액

동작 순서는 다음과 같습니다.

1. 회원의 충전 계좌 정보를 조회합니다.
2. 충전 금액에서 결제할 금액(`amount`)만큼 차감합니다.
    - 이때 잔액이 부족하면 `InsufficientBalanceException` 예외가 발생합니다.

<br>

### **다중 서버로 구성되어 있는 충전형 결제 시스템**

![image.png](/assets/img/posts/charge_payment_system_consisting_of_multiple_servers.png)

충전형 결제 시스템은 **돈을 미리 충전하고 사용하는 구조**이므로, 동일한 계좌 잔액에 대해 여러 결제 요청이 동시에 처리될 수 없습니다.

예를 들어, 사용자의 잔액이 10,000원일 때 **동시에 7,000원과 5,000원의 결제 요청**이 들어온다면, 서버가 이를 순차적으로 처리하지 않고 동시에 처리할 경우 **총 12,000원의 결제가 승인**될 수 있습니다. 즉, 실제 잔액을 초과하는 결제가 발생할 수 있는 문제가 생깁니다.

<br>

### **동시성으로 인해 잔액을 초과하는 결제가 승인되는 구조**

![image.png](/assets/img/posts/payments_exceeding_the_balance_are_approved_due_to_concurrency.png)

1. **첫 번째 결제 요청**: 7,000원이 들어옵니다.
2. **두 번째 결제 요청**: 5,000원이 들어옵니다.
3. 첫 번째 결제 요청을 처리하는 서버는 잔액이 10,000원이므로, 7,000원 결제를 수행할 수 있다고 판단합니다.
4. 두 번째 결제 요청을 처리하는 서버 역시 잔액이 10,000원이므로, 5,000원 결제를 수행할 수 있다고 판단합니다.
5. 첫 번째 결제 요청이 실제로 처리되며 7,000원이 차감됩니다. (남은 잔액: 10,000원 → 3,000원)
6. 두 번째 결제 요청이 처리되며 5,000원이 차감됩니다. (남은 잔액: 10,000원 → 5,000원)

즉, **각 서버는 처리 시점의 잔액을 기준으로 결제 가능 여부를 판단**하게 되어, 실제 잔액을 초과하는 결제가 승인되는 문제가 발생합니다.

이를 테스트 코드로 확인해보겠습니다.

<br>

### **잔액 부족 상황을 만드는 중복 결제 요청 테스트 코드**

```kotlin
@Test
fun `잔액 부족 상황을 만드는 중복 결제 요청`() {
    // given
    val userId = 1L
    val paymentKey = "PAYMENT_KEY"
    prepaidAccountRepository.save(
        PrepaidAccount(userId = userId, balance = 10_000)
    )

    val startLatch = CountDownLatch(1)
    val doneLatch = CountDownLatch(2)
    val executor = Executors.newFixedThreadPool(2)

    val exceptions = mutableListOf<PaymentFailedException>()

    // 첫번째 결제 요청: 7,000원
    executor.submit {
        try {
            startLatch.await()
            chargePaymentService.pay(userId, paymentKey, 7_000)
        } catch (e: PaymentFailedException) {
            exceptions.add(e)
        } finally {
            doneLatch.countDown()
        }
    }

    // 두번째 결제 요청: 5,000원
    executor.submit {
        try {
            startLatch.await()
            chargePaymentService.pay(userId, paymentKey, 5_000)
        } catch (e: PaymentFailedException) {
            exceptions.add(e)
        } finally {
            doneLatch.countDown()
        }
    }

    // when
    startLatch.countDown()   // 두 요청을 동시에 시작
    doneLatch.await()
    executor.shutdown()

    // then
    // 최소 하나는 잔액 부족으로 결제 실패하는걸 기대
    assertThat(exceptions.size).isEqualTo(1)
}

```

- 사용자의 잔액이 10,000원인 상황에서, 7,000원과 5,000원의 결제 요청이 **동시에** 들어갑니다.
- 정상 동작이라면 최소 한 건은 **잔액 부족**으로 결제 실패가 발생해야 하지만, 동시성 문제 때문에 **둘 다 성공**하여 테스트가 실패했습니다.
- 실행 결과: **Test failed**
    
    ![image.png](/assets/img/posts/concurrency_test_failed.png)
    
    ```kotlin
    org.opentest4j.AssertionFailedError: 
    expected: 1
     but was: 0
    ```
    

이제 **위 테스트 코드를 통과시키는 것을 목표**로 문제를 해결해보겠습니다.

<br>

### 만약 단일 서버로 구성되어 있었다면?

단일 서버 환경이라면 멀티스레드 동시성만 제어하면 되므로, `synchronized`나 `Lock`과 같은 동기화 기법을 사용해 비교적 간단하게 문제를 해결할 수 있습니다.

<br>

### 다중 서버 환경에서 동시성 문제를 해결하는 방법

다중 서버 환경에서는 서로 다른 서버 인스턴스가 동시에 결제 요청을 처리할 수 있습니다. 따라서 **단일 서버에서의 멀티스레드 동시성 제어만으로는 문제를 완전히 해결할 수 없습니다.**

다중 서버 환경에서 동시성 문제를 해결하는 대표적인 방법은 다음과 같습니다.

1. **DB 레벨에서 비관적 락 또는 낙관적 락 사용**
    - **장점**: DB만 있으면 추가 인프라가 필요 없고 구현이 단순합니다.
    - **단점**: DB에 부하가 집중되어 병목이 발생할 수 있으며, 서버 수를 늘려도 DB에 부하가 집중되어 확장성이 제한됩니다.
2. **요청을 메시지 큐에 넣고 순차 처리**
    - **장점**: 동시성 충돌 가능성을 근본적으로 제거할 수 있습니다.
    - **단점**: 큐를 통해 처리하므로 실시간성이 떨어지고, 소비자 측에서 병목이 발생할 수 있습니다.
3. **분산락을 이용하여 서버 간 동시성 제어**
    - **장점**: 메모리 기반 분산락을 사용하면 락 획득과 해제가 빠르며, TTL 설정으로 Deadlock 발생 위험이 낮습니다.
    - **단점**: 락의 유효 시간(TTL) 관리가 필요합니다.

물론 이 방법들은 서로 배타적인 것이 아니며, 필요에 따라 조합해서 사용할 수도 있습니다. 예를 들어, 분산락으로 서버 간 동시성을 우선적으로 제어하고, DB 낙관적 락으로 최종 무결성을 보장하는 방식이 가능합니다.

이번 글에서는 **단일 스레드 환경에서 빠른 성능을 기대할 수 있는 Redis 기반 분산락**을 활용하여 다중 서버 환경의 동시성 문제를 해결하는 방법을 다뤄보겠습니다.

<br>

## Redis를 이용한 분산락 구현하기

분산락은 Redis, Zookeeper, MySQL 등 다양한 방법으로 구현할 수 있습니다. 그중 **Redis**는 메모리 기반으로 빠르고 TTL(Time-To-Live)을 지원하며, 비교적 간단하게 구현할 수 있어 이번 글에서는 Redis를 활용한 분산락 구현 방법을 살펴보겠습니다.

<br>

### Lettuce vs Redisson

Java/Kotlin 환경에서 Redis와 통신할 수 있는 대표적인 클라이언트로 **Lettuce**와 **Redisson**이 있습니다.

이번 글에서는 **Lettuce와 Redisson을 이용한 분산락 구현 예시**를 작성하고, 두 방법을 비교해 보겠습니다.

<br>

### Lettuce를 이용한 분산락

Lettuce는 공식적으로 분산락을 지원하지 않기 때문에, Redis의 `SETNX`와 `DEL` 명령어를 활용해 분산락을 구현해야 합니다.

- **SETNX**: 키가 없을 때만 값을 저장하며, 이미 해당 키가 존재하면 아무 작업도 하지 않습니다. 이를 통해 **경쟁 상태에서 락을 먼저 획득한 클라이언트만 성공**하게 됩니다.
- **DEL**: 키를 삭제하여 락을 해제하고, 다른 클라이언트가 락을 획득할 수 있도록 합니다.

즉, Lettuce로 분산락을 구현할 때는 **Spin Lock 방식**으로 `SETNX`와 `DEL`을 사용합니다.

<br>

**RedisDistributedLock**

`RedisDistributedLock` 클래스는 **분산락 획득과 해제**를 담당합니다.

```kotlin
@Component
class RedisDistributedLock(
    private val redisTemplate: RedisTemplate<String, String>
) {
    fun tryLock(key: String, expireMillis: Long): String? {
        val token = UUID.randomUUID().toString() // 락을 가지고 있다는 증거로 발급하는 토큰
        val success = redisTemplate
            .opsForValue()
            .setIfAbsent(key, token, Duration.ofMillis(expireMillis)) ?: false

        return if (success) token else null // 락을 획득했다면 생성한 토큰을 반환
    }

    fun unlock(key: String, token: String) {
        val luaScript = """
            if redis.call("get", KEYS[1]) == ARGV[1] then
                return redis.call("del", KEYS[1])
            else
                return 0
            end
        """.trimIndent()

        val script = DefaultRedisScript<Long>().apply {
            setScriptText(luaScript)
            resultType = Long::class.java
        }

        redisTemplate.execute(script, listOf(key), token)
    }
}
```

- **tryLock**
    - `opsForValue().setIfAbsent()`를 사용하여 Redis의 `SETNX` 기능을 활용합니다.
    - 락 획득에 성공하면 만료 시간을 설정하여 저장하고, 소유자 증명용 **토큰을 반환**합니다.
- **unlock**
    - Lua 스크립트를 사용하여 **내가 가진 락인지 확인하고 맞으면 삭제한다는 로직을 원자적으로 수행**합니다.
        - `KEYS[1]`: 락의 키
        - `ARGV[1]`: 락 소유자 식별용 토큰
    - 이를 통해 **락 소유자가 아닌 클라이언트는 락을 해제할 수 없도록** 보장합니다.

<br>

**Lettuce를 이용한 분산락을 설정한 ChargePaymentService의 pay 메서드**

```kotlin
@Transactional
fun pay(userId: Long, paymentKey: String, amount: Long){
    val lockKey = "lock:paymentKey:$paymentKey"
    val maxWaitMillis = 3000L  // 최대 대기 시간
    val retryDelayMillis = 50L // 재시도 간격
    val expireMillis = 5000L   // 락 TTL

    val startTime = System.currentTimeMillis()
    var token: String? = null

    // 스핀락으로 락 획득 반복 시도
    while (System.currentTimeMillis() - startTime < maxWaitMillis) {
        token = redisDistributedLock.tryLock(lockKey, expireMillis)
        if (token != null) break
        Thread.sleep(retryDelayMillis)
    }

    if (token == null) {
        logger.error("동시 결제 요청 오류 발생. (paymentKey: $paymentKey)")
        throw PaymentFailedException("동시 결제 요청으로 결제가 실패했습니다.")
    }

    try {
        val prepaidAccount = prepaidAccountRepository.findByUserId(userId = userId) // 충전 금액 조회
        prepaidAccount.deduct(amount = amount) // 결제 금액 차감
        prepaidAccountRepository.save(prepaidAccount)
    } catch (e: InsufficientBalanceException){
        throw PaymentFailedException(message = "잔액이 부족합니다.", cause = e)
    } catch (e: Exception){
        logger.error("결제 요청 중 알 수 없는 오류 발생. (paymentKey: $paymentKey)", e)
        throw PaymentFailedException(message = "결제 처리 중 예기치 못한 오류가 발생했습니다.", cause = e)
    } finally {
        redisDistributedLock.unlock(lockKey, token)
    }
}

```

- **스핀락으로 락 획득 시도**
    - `tryLock`을 반복 호출하며 락 획득을 시도합니다.
    - 락 획득 실패 시 `Thread.sleep(retryDelayMillis)` 후 재시도합니다.
    - 최대 대기 시간(`maxWaitMillis`)을 초과하면 예외를 발생시키고 스핀락을 종료합니다.
- **락 TTL 설정**
    - 락을 획득한 클라이언트가 무한히 락을 가지지 않도록 `expireMillis`를 설정합니다.
    - 이때, TTL보다 실제 작업이 길면 락이 먼저 해제되어 동시성 문제가 발생할 수 있습니다.
    - 또한, `maxWaitMillis`가 TTL보다 길면 이미 해제된 락을 다른 클라이언트가 획득할 가능성이 있습니다.
- **락 해제**
    - `finally` 블록에서 항상 `unlock`을 호출하여 락을 해제합니다.
    - 락 해제 시 토큰을 통해 **락 소유자를 검증**합니다.

<br>

**토큰을 통한 락 소유자 검증의 필요성**

만약 소유자 검증 없이 키 이름만 보고 락을 해제하면, 다음과 같은 문제가 발생할 수 있습니다.

1. 클라이언트 A가 `"lock:paymentKey:PAYMENT_KEY_1"`키로 락을 획득 (TTL=5초)
2. 5초 후, A의 비즈니스 로직이 끝나지 않았음에도 TTL 만료로 인해 락 해제
3. 클라이언트 B가 같은 키인 `"lock:paymentKey:PAYMENT_KEY_1"`키로 락을 획득
4. 이후 A가 락의 소유자가 아닌데도 불구하고 락 해제 호출 → B가 가진 락이 억울하게 해제되는 상황 발생

즉, **락 소유자를 확인하는 토큰을 사용해야** 여러 클라이언트가 동시에 락을 시도할 때도 안전하게 락을 관리할 수 있습니다.

<br>

**[테스트 코드](https://kkulbeeboy.github.io/posts/using-distributed-locking-to-solve-concurrency-problems-in-multi-server-with-redis/#%EC%9E%94%EC%95%A1-%EB%B6%80%EC%A1%B1-%EC%83%81%ED%99%A9%EC%9D%84-%EB%A7%8C%EB%93%9C%EB%8A%94-%EC%A4%91%EB%B3%B5-%EA%B2%B0%EC%A0%9C-%EC%9A%94%EC%B2%AD-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%BD%94%EB%93%9C) 수행 결과 - Test passed**

목표로 했던 **중복 결제 요청 테스트**가 성공했는지 확인해 보겠습니다.

- 수행 결과
    
    ![image.png](/assets/img/posts/concurrency_test_passed_with-lettuce.png)
    
- 분산락을 통해 순차적으로 처리되어 테스트가 성공적으로 수행된것을 확인할 수 있습니다.

<br>

### Redisson를 이용한 분산락

Lettuce 기반 분산락은 **Spin Lock 방식**을 사용하기 때문에, 락을 획득하지 못한 클라이언트가 락을 획득할 때까지 반복적으로 요청을 보내며 대기합니다. 이 과정에서 많은 부하가 발생할 수 있다는 단점이 있습니다.

반면, **Redisson**은 분산락 기능을 **내장**하고 있으며, **Pub/Sub 기반**으로 동작하여 부하를 줄일 수 있습니다.

이를 통해 락을 획득한 클라이언트가 락을 해제하면, 락 획득을 기다리던 다른 클라이언트에게 신호를 보내어 순차적으로 락을 획득할 수 있도록 합니다.

<br>

**Redisson를 이용한 분산락을 설정한 ChargePaymentService의 pay 메서드**

```kotlin
@Transactional
fun pay(userId: Long, paymentKey: String, amount: Long) {
    val lockKey = "lock:paymentKey:$paymentKey"
    val maxWaitMillis = 3000L // 락 대기 시간
    val expireMillis = 5000L  // 락 TTL

    val lock: RLock = redissonClient.getLock(lockKey)

    val isLocked = lock.tryLock(maxWaitMillis, expireMillis, TimeUnit.MILLISECONDS)

    if (!isLocked) {
        logger.error("동시 결제 요청 오류 발생. (paymentKey: $paymentKey)")
        throw PaymentFailedException("동시 결제 요청으로 결제가 실패했습니다.")
    }

    try {
        val prepaidAccount = prepaidAccountRepository.findByUserId(userId) // 충전 금액 조회
        prepaidAccount.deduct(amount) // 결제 금액 차감
        prepaidAccountRepository.save(prepaidAccount)
    } catch (e: InsufficientBalanceException) {
        throw PaymentFailedException("잔액이 부족합니다.", e)
    } catch (e: Exception) {
        logger.error("결제 요청 중 알 수 없는 오류 발생. (paymentKey: $paymentKey)", e)
        throw PaymentFailedException("결제 처리 중 예기치 못한 오류가 발생했습니다.", e)
    } finally {
        if (lock.isHeldByCurrentThread) {
            lock.unlock()
        }
    }
}

```

- **락 획득 및 TTL 설정**
    - `maxWaitMillis`는 최대 대기 시간, `expireMillis`는 락 TTL을 의미합니다.
    - Redisson의 `tryLock` 메서드를 통해, 지정한 최대 대기 시간 동안 락을 시도하며, TTL도 동시에 설정할 수 있습니다.
    - 최대 대기 시간 동안 락 획득 실패 시 `false`를 반환합니다.
- **락 해제**
    - `finally` 블록에서 항상 `unlock` 호출을 수행합니다.
    - `isHeldByCurrentThread`를 확인하는 이유는 **현재 스레드가 락을 소유하고 있을 때만 락을 해제**하기 위한 안전장치입니다. (다른 스레드가 락을 가지고 있는 상황에서 `unlock()` 호출을 방지)

<br>

**[테스트 코드](https://kkulbeeboy.github.io/posts/using-distributed-locking-to-solve-concurrency-problems-in-multi-server-with-redis/#%EC%9E%94%EC%95%A1-%EB%B6%80%EC%A1%B1-%EC%83%81%ED%99%A9%EC%9D%84-%EB%A7%8C%EB%93%9C%EB%8A%94-%EC%A4%91%EB%B3%B5-%EA%B2%B0%EC%A0%9C-%EC%9A%94%EC%B2%AD-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%BD%94%EB%93%9C) 수행 결과 - Test passed**

목표로 했던 **중복 결제 요청 테스트**가 성공했는지 확인해 보겠습니다.

- 수행 결과

    ![image.png](/assets/img/posts/concurrency_test_passed_with-redisson.png)

- 분산락을 통해 순차적으로 처리되어 테스트가 성공적으로 수행된것을 확인할 수 있습니다.

<br>

## 정리

- 분산락은 **다중 서버에서 서버 간 경쟁 조건을 관리하고, 안정적인 서비스 운영을 위한 방법**입니다.
- Redis 기반 분산락을 활용하면, **다중 서버 환경에서도 동시성 문제를 안전하게 제어**할 수 있습니다.
- Lettuce를 이용한 분산락은 **Spin Lock** 방식으로 동작하며, 락 획득을 위해 반복적으로 재시도해야 합니다.
- Redisson을 이용한 분산락은 **Pub/Sub 기반**으로 동작하여, 락을 해제하면 대기 중인 클라이언트에게 신호를 보내 순차적으로 락을 획득할 수 있습니다.
- 따라서, 락을 잡지 못한 클라이언트가 반복 시도를 수행하는 것이 크게 문제가 되지 않는 경우에는 **Lettuce**가 적합하고, 재시도가 필요하거나 대기 중인 클라이언트에게 효율적으로 알림을 주고 싶다면 **Redisson**이 더 적절합니다.

<br>

---
tistory, “Java의 synchronized, Lock Stripping과 Atomic Type”, [https://320hwany.tistory.com/101](https://320hwany.tistory.com/101), (참고 날짜 2025.08.15)

tistory, “레디스를 활용한 분산 락(Distrubuted Lock) feat lettuce, redisson”, [https://0soo.tistory.com/256](https://0soo.tistory.com/256), (참고 날짜 2025.08.15)

tistory, “동시성 관리하기 3탄) Redis - Lettuce, Redisson”, [https://baeji-develop.tistory.com/131](https://baeji-develop.tistory.com/131), (참고 날짜 2025.08.15)

f-lab, “Redis를 활용한 분산 락과 동시성 처리”, [https://f-lab.kr/insight/redis-distributed-lock-20240704](https://f-lab.kr/insight/redis-distributed-lock-20240704), (참고 날짜 2025.08.15)

tistory, “[시스템 디자인] 재고시스템으로 알아보는 동시성이슈 해결방법 (3/3) - 레디스 분산 락(Lock)으로 해결하기”, [https://nooblette.tistory.com/entry/시스템-디자인-재고시스템으로-알아보는-동시성이슈-해결방법-33-레디스-분산-락Lock으로-해결하기#Lettuce를_활용하여_재고감소_로직_작성하기](https://nooblette.tistory.com/entry/시스템-디자인-재고시스템으로-알아보는-동시성이슈-해결방법-33-레디스-분산-락Lock으로-해결하기#Lettuce를_활용하여_재고감소_로직_작성하기), (참고 날짜 2025.08.15)

miintto.log, “Redis 분산 락을 활용한 동시성 처리”, [https://miintto.github.io/docs/distributed-lock](https://miintto.github.io/docs/distributed-lock), (참고 날짜 2025.08.15)


