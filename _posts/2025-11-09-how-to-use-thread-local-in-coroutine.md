---
title: "스레드 로컬을 코루틴 환경에서 적절하게 사용하는 방법"
date: 2025-11-09 00:00:00 +0900
categories: [language, kotlin]
tags: [kotlin, coroutine, threadLocal, CoroutineContext, asContextElement]
---

## 스레드 로컬과 코루틴

코틀린에서 코루틴은 비동기 프로그래밍을 단순화 해주는 도구입니다. 하지만 코루틴은 스레드 기반의 동작과는 조금 다르게 동작하기 때문에, 기존 자바 개발 환경에서 익숙하게 사용하던 **스레드 로컬 같은 기능이 코루틴 환경에서는 제대로 동작하지 않을 수 있습니다.**

이번 글에서는 스레드 로컬이 어떻게 코루틴 친화적인 방식으로 사용할 수 있는지 알아보겠습니다.

<br>

### 스레드 로컬이란?

스레드 로컬(ThreadLocal)은 스레드 단위로 지역적인 데이터를 저장하는 방법입니다. 즉, 여러 스레드가 하나의 값을 공유하는게 아닌, 각 스레드마다 독립된 값을 보유할 수 있도록 합니다.

<br>

**스레드 로컬 예시**

```kotlin
    val threadLocal = ThreadLocal<String>()

    @Test
    fun `스레드마다 독립적인 값을 보유하는 ThreadLocal`() {
        threadLocal.set("threadLocal from main thread")

        val thread = Thread {
            logger.info(threadLocal.get()) // null (다른 스레드이기 때문)
            threadLocal.set("threadLocal from new thread")
            logger.info(threadLocal.get()) // threadLocal from new thread
        }

        thread.start()
        thread.join()

        logger.info(threadLocal.get()) // threadLocal from main thread
    }
```

- 실행 결과
    
    ```bash
    [Thread-3] INFO logger -- null
    [Thread-3] INFO logger -- threadLocal from new thread
    [Test worker] INFO logger -- threadLocal from main thread
    ```
    
- 위 예시처럼 스레드 로컬은 같은 키를 쓰더라도 **스레드마다 독립적인 저장 공간을 가지는 구조입니다.**
- 이는 스레드 풀 환경에서 인증 정보, 트랜잭션 ID, 로깅 컨텍스트(MDC) 같은 값을 저장할 때 유용하게 쓰입니다.

<br>

### 코루틴에서 스레드 로컬을 바로 사용할 수 없는 이유

**코루틴은 스레드 위에서 동작하지만, 스레드와 일대일 대응하지 않습니다.** 즉, 하나의 코루틴은 실행 도중 여러 스레드로 옮겨 다닐 수 있습니다. (코루틴이 suspend 지점에서 일시 중단되었다가 재개될 때, 재개되는 스레드는 이전과 다른 스레드일 수 있습니다.)

반면에 스레드 로컬 값은 스레드에 붙어있기 때문에, 코루틴이 다른 스레드에서 재개되면 값이 유실된것처럼 동작할 수 있게 됩니다. 즉, 코루틴은 스레드에 독립적인 실행 단위이므로, 스레드 로컬은 자연스럽게 동작하지 않습니다.

<br>

## 코루틴 컨텍스트(CoroutineContext)란?

이전에 코루틴 동작 과정에서 `Continuation` 객체를 이용한다는 이야기를 했었습니다. ([제목은 코틀린 중단(suspend) 동작으로 하겠습니다. 근데 이제 Continuation 객체를 곁들인](https://kkulbeeboy.github.io/posts/suspend-operation-with-continuation/), [코틀린 코루틴의 구현 방식, 그 내부로 들어가 보자](https://kkulbeeboy.github.io/posts/inside-the-implementation-kotlin-coroutine/)) 이 `Continuation`은 코루틴 실행에 필요한 모든 정보를 담는 **코루틴 컨텍스트(CoroutineContext)를 포함하고 있습니다.**

<br>

**코루틴 컨텍스트(CoroutineContext)를 포함하고 있는 Continuation**

```kotlin
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```

CoroutineContext에는 대표적으로 다음과 같은 정보들이 들어있습니다.

- **Job**
    - 코루틴의 생명주기 관리
- **Dispatcher**
    - 코루틴이 어떤 스레드에서 실행될지 결정
- **CoroutineName**
    
    코루틴 이름
    

즉, CoroutineContext는 **맵(Map)처럼 구성**되어 있고, **여러 Element들이 모여 만들어집니다.**

이야기들 들어보면, 코루틴 컨텍스트에 여러 정보들을 넣어서 관리할 수 있는것 처럼 보입니다. 이를 통해 **스레드 로컬을 코루틴 환경에서 사용하는것에 대한 핵심**은 **스레드 로컬을 코루틴 컨텍스트에 통합하여 관리하는 방법**이라는걸 알 수 있습니다.

즉, 정리하자면 **스레드 로컬을 어떻게 코루틴 컨텍스트에 잘 관리할 수 있는지**가 핵심입니다.

<br>

### CoroutineContext.Element란?

CoroutineContext는 여러 Element들이 모여 만들어진다고 이야기 했습니다. 여기서 말하는 Element들이, **CoroutineContext.Element**로써, 하나의 코루틴 컨텍스트 요소를 의미합니다.

각 요소는 키(`Key<E>`)를 통해 식별되고, 서로 결합(`+`)되어 하나의 코루틴 컨텍스트를 형성합니다.

<br>

**CoroutineContext의 Element 예시: Job**

```kotlin
public interface Job : CoroutineContext.Element
```

- 이처럼 Job은 CoroutineContext.Element를 구현한 코루틴 컨텍스트의 요소이므로 코루틴 컨텍스트에서 관리할 수 있게 되는것 입니다.
- 다른 값도 코루틴 컨텍스트의 원소로 추가하고 싶으면, CoroutineContext.Element를 구현하면 됩니다.

<br>

## 코루틴에서 스레드 로컬을 사용하는 방법: asContextElement

코루틴에서는 스레드 로컬을 직접 사용하는 대신, `asContextElement()` 확장 함수를 이용해 **ThreadLocal을 CoroutineContext에 통합할 수 있습니다.** 

<br>

**코루틴에서 정의한 ThreadLocal의 확장함수 asContextElement()**

```kotlin
public fun <T> ThreadLocal<T>.asContextElement(value: T = get()): ThreadContextElement<T> = ThreadLocalElement(value, this)
```

`ThreadLocal.asContextElement()`은 내부적으로 `ThreadContextElement` 인터페이스를 구현한 하나의 `CoroutineContext.Element`이므로 CoroutineContext에 통합하여 관리할 수 있게되는것 입니다.

→ **ThreadContextElement**에 대해서는 아래에서 더 자세히 확인해 보겠습니다.

즉, **asContextElement는 이 ThreadLocal을 CoroutineContext의 일부로 등록하여, 코루틴이 재개될 때마다 해당 값이 자동으로 복원되도록 하는 역할을 합니다.**

<br>

**asContextElement()을 통해 ThreadLocal을 CoroutineContext에 통합**

```kotlin
    val threadLocal = ThreadLocal<String>()

    @Test
    fun `ThreadLocal을 CoroutineContext에 통합`() = runBlocking {
        threadLocal.set("main")

        launch(Dispatchers.Default + threadLocal.asContextElement(value = "coroutine")) {
            logger.info("Started in ${Thread.currentThread().name}, value=${threadLocal.get()}")
            delay(1000)
            logger.info("Resumed in ${Thread.currentThread().name}, value=${threadLocal.get()}")
        }

        logger.info("After launch: ${Thread.currentThread().name}, value=${threadLocal.get()}")
    }
```

- 출력 결과
    
    ```bash
    [DefaultDispatcher-worker-1 @coroutine#2] INFO logger -- Started in DefaultDispatcher-worker-1 @coroutine#2, value=coroutine
    [Test worker @coroutine#1] INFO logger -- After launch: Test worker @coroutine#1, value=main
    // 1초 후 코루틴 재개
    [DefaultDispatcher-worker-1 @coroutine#2] INFO logger -- Resumed in DefaultDispatcher-worker-1 @coroutine#2, value=coroutine
    ```
    
- 이처럼 코루틴이 중단되었다가 다른 스레드에서 재개되어도 스레드 로컬 값이 코루틴 컨텍스트를 통해 함께 전달됩니다.

<br>

### 코루틴이 재개될 때와 중단될 때 ThreadLocal 상황

정리하면, 코루틴을 생성할 때 `threadLocal.asContextElement()` 를 사용해 `ThreadLocal` 값을 `CoroutineContext`의 요소로 등록하면, 코루틴의 **재개** 시점에는 `ThreadLocal`에 해당 컨텍스트 값이 설정되고, **중단** 시점에는 기존 `ThreadLocal` 값으로 다시 복원됩니다.

즉, 코루틴의 실행 흐름에 따라 `ThreadLocal` 값이 안전하게 교체/복구되도록 관리되는 것입니다.

**코루틴이 재개될 때 (resume)**

- 코루틴이 어떤 스레드에서 실행될지 모르는 상황
- 실행 직전에 `updateThreadContext()` 호출
- 해당 코루틴 컨텍스트에 보관된 값이 **ThreadLocal.set(value)** 로 설정
    - 즉, 그 스레드는 이제 그 코루틴의 ThreadLocal 값을 사용
    

**코루틴이 중단될 때 (suspend)**

- 코루틴이 실행되던 스레드에서 빠져나갈 수도 있음
- suspend 직전에 `restoreThreadContext()` 호출
- 원래 그 스레드가 가지고 있던 ThreadLocal 값으로 복원

이를 통해 스레드가 바뀌는 환경의 코루틴에서, **어떤 시점에 특정 ThreadLocal 값이 보장되어 있어야 하는 상황에 대응**할 수 있게 됩니다.

즉, **재개 시점에 그 코루틴이 가져야 할 ThreadLocal 값을 덮어쓰고**, **중단 시점에 이전 스레드의 상태를 돌려주는 방식의** 구조를 가지게 되었습니다.

<br>

## ThreadContextElement 인터페이스

`ThreadContextElement` 인터페이스는 ThreadLocal과 CoroutineContext 간의 연동을 위한 핵심 인터페이스 입니다.

더 나아가 `ThreadContextElement` 인터페이스 코드를 파악해 보겠습니다.

<br>

**ThreadContextElement 인터페이스**

```kotlin
public interface ThreadContextElement<S> : kotlin.coroutines.CoroutineContext.Element {
    public abstract fun restoreThreadContext(context: kotlin.coroutines.CoroutineContext, oldState: S): kotlin.Unit

    public abstract fun updateThreadContext(context: kotlin.coroutines.CoroutineContext): S
}
```

`ThreadContextElement`는 `CoroutineContext.Element`를 구현(extend)한 인터페이스 입니다.

이 인터페이스는 두 가지 메서드(updateThreadContext(), restoreThreadContext()) 를 구현해야 된다는것을 알 수 있습니다.

- `updateThreadContext()`
    - 호출되는 시점
        - 코루틴이 **다시 실행될 때(resume)**
        - 즉, 코루틴이 어떤 스레드에서 실행될지 확정되는 순간
    - 역할
        - 코루틴 컨텍스트의 값을 ThreadLocal에 실제로 세팅하는 작업
        - 스레드 실행 전에 ThreadLocal 값을 코루틴이 갖고 있어야 하는 값으로 설정
    - 반환값 `S`
        - ThreadLocal에 원래 들어 있던 값이며, 중단(suspend) 시점에 복원하기 위해 저장되는 값(oldState)
- `restoreThreadContext()`
    - 호출되는 시점
        - 코루틴이 **중단(suspend)** 되기 직전
    - 역할
        - 재개(resume) 전에 ThreadLocal을 수정했으므로 suspend 시에는 **원래 ThreadLocal에 존재하던 값(oldState)을 돌려놓는 역할**

즉, 재개(resume) 시점에는 ThreadLocal의 값을 코루틴의 값으로 설정한다는것을 알 수 있고, 중단(suspend) 시점에는 ThreadLocal의 값을 스레드 원래의 값(oldState)로 설정한다는것을 알 수 있습니다.

<br>

### ThreadContextElement 커스텀 예시

`ThreadContextElement`는 코루틴이 스레드를 바꿀 때 어떤 Thread 상태를 가져오고 복원할지를 정의하는 인터페이스이므로, 이를 직접 구현하면 커스텀하게 ThreadLocal 동작을 만들 수 있습니다.

예를 들어 아래와 같은 상황을 가정해 보겠습니다.

> 분산 시스템에서 전체 트랜잭션을 추적하기 위해 각각의 요청마다 고유 식별자인 traceId를 전달하는데, 이를 코루틴에서도 전달하고 싶은 상황
> 

<br>

**직접 ThreadContextElement를 구현한 커스텀 CoroutineContext.Element → TraceIdContext**

```kotlin
class TraceIdContext (
    private val threadLocal: ThreadLocal<String>,
    private val traceId: String
) : ThreadContextElement<String> {

    companion object Key : CoroutineContext.Key<TraceIdContext>

    override val key: CoroutineContext.Key<TraceIdContext> get() = Key

    override fun updateThreadContext(context: CoroutineContext): String {
        val oldValue = threadLocal.get()
        threadLocal.set(traceId)
        return oldValue
    }

    override fun restoreThreadContext(context: CoroutineContext, oldState: String) {
        threadLocal.set(oldState)
    }
}
```

- `ThreadContextElement`를 구현한 커스텀 `CoroutineContext.Element`입니다.
- 코루틴이 실행(또는 재개)될 때 특정 `ThreadLocal`에 traceId를 넣고, 중단되면 원래 값으로 복원합니다.
- **생성자**
    - `private val threadLocal: ThreadLocal<String>`
        - 변경/복원 대상이 되는 `ThreadLocal` 객체
    - `private val requestId: String`
        - 코루틴이 실행 중에 추적할 trace ID
    - 예시에서는 생성자 파라미터 모두 non-null로 가정했습니다.
- **companion object Key 와 key 속성값**
    - `CoroutineContext`에서 요소(Element)는 고유 키로 식별됩니다.
    - 여기서 `companion object` 자체를 `Key`로 사용하여 `context[RequestIdContext.Key]`로 요소를 조회할 수 있습니다.
    - `override val key`는 **`CoroutineContext.Element` 인터페이스 요구사항**으로, 이 요소의 키를 반환합니다.

<br>

**TraceIdContext를 사용하는 테스트 코드**

```kotlin
    @Test
    fun `TraceIdContext를 사용하여 코루틴내에서 traceId 추적`() = runBlocking {
        val traceIdThreadLocal = object : ThreadLocal<String>() {
            override fun initialValue(): String = "main" // ThreadLocal 초기값을 "main"으로 설정
        }

        launch(Dispatchers.Default + TraceIdContext(traceIdThreadLocal, "trace-1234")) {
            logger.info("Start: ${Thread.currentThread().name}, id=${traceIdThreadLocal.get()}")
            delay(100)
            logger.info("Resume: ${Thread.currentThread().name}, id=${traceIdThreadLocal.get()}")
        }

        logger.info("After: ${Thread.currentThread().name}, id=${traceIdThreadLocal.get()}")
    }
```

- 실행 결과
    
    ```bash
    22:43:14.853 Start: DefaultDispatcher-worker-1 @coroutine#2, id=trace-1234
    22:43:14.853 After: Test worker @coroutine#1, id=main
    22:43:14.962 Resume: DefaultDispatcher-worker-1 @coroutine#2, id=trace-1234
    ```
    
- 코루틴이 실행되는 스레드(Dispatchers.Default)에서 ThreadLocal 값이 trace-1234로 설정
    - 코루틴 실행 직전에 `updateThreadContext()`가 호출되어 기존 ThreadLocal 값(main)을 저장하고 새 값(trace-1234) 설정
- 코루틴 실행 후, main 스레드의 ThreadLocal 값은 원래 값(main)으로 복원
    - suspend 시점에서 `restoreThreadContext()`가 호출되어 main 스레드의 ThreadLocal 값이 원래 상태로 복원
- delay 이후 코루틴이 재개될 때, 다시 `updateThreadContext()`가 호출되고, ThreadLocal에 trace-1234가 다시 설정

<br>

## 정리

- 코루틴은 스레드를 자유롭게 전환하므로 단순히 스레드 로컬(ThreadLocal)을 그대로 사용하면 데이터가 사라지거나 잘못된 값이 참조될 수 있습니다.
- 코루틴 컨텍스트(CoroutineContext)란 코루틴 실행에 필요한 모든 정보를 담고 있습니다.
- CoroutineContext는 여러 CoroutineContext.Element들이 모여 만들어 집니다.
- asContextElement()는 ThreadLocal을 코루틴 컨텍스트로 감싸주는 간단한 방법으로, 스레드 로컬을 코루틴 환경에서 안전하게 사용할 수 있도록 합니다.

<br>

---

티스토리, “[Kotlin] 코루틴 - CoroutineContext 이해하기!”, [https://jaehoney.tistory.com/423](https://jaehoney.tistory.com/423), (참고 날짜 2025.11.09)

티스토리, “2장 코루틴 라이브러리 - CoroutineContext”, [https://androidpangyo.tistory.com/207](https://androidpangyo.tistory.com/207), (참고 날짜 2025.11.09)

Kotlin, “Coroutine context and dispatchers﻿”, [https://kotlinlang.org/docs/coroutine-context-and-dispatchers.htm](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html), (참고 날짜 2025.11.09)

Kotlin, “asContextElement”, [https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/as-context-element.html](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/as-context-element.html), (참고 날짜 2025.11.09) 

medium, “코루틴 공식 가이드 자세히 읽기 — Part 5 — Dive 3”, [https://myungpyo.medium.com/코루틴-공식-가이드-자세히-읽기-part-5-dive-3-3c82eb80245c](https://myungpyo.medium.com/%EC%BD%94%EB%A3%A8%ED%8B%B4-%EA%B3%B5%EC%8B%9D-%EA%B0%80%EC%9D%B4%EB%93%9C-%EC%9E%90%EC%84%B8%ED%9E%88-%EC%9D%BD%EA%B8%B0-part-5-dive-3-3c82eb80245c), (참고 날짜 2025.11.26)

Kotlin, “Element”, [https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.coroutines/-coroutine-context/-element/](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.coroutines/-coroutine-context/-element/), (참고 날짜 2025.11.26)