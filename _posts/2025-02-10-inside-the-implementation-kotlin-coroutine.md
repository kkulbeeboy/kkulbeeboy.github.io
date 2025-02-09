---
title: "코틀린 코루틴의 구현 방식, 그 내부로 들어가 보자"
date: 2025-02-10 00:00:00 +0900
categories: [language, kotlin]
tags: [kotlin, coroutine, suspend, continuation, cps]
---

이전에, **코틀린의 중단(suspend) 함수가 어떻게 중단되고 재개될 수 있는지**를 살펴보면서, 이러한 동작이 **Continuation 객체**와 밀접한 관계가 있다는 점을 확인했습니다. ([제목은 코틀린 중단(suspend) 동작으로 하겠습니다. 근데 이제 Continuation 객체를 곁들인](https://kkulbeeboy.github.io/posts/suspend-operation-with-continuation/))

이번에는 코루틴이 내부적으로 어떤 방식으로 구현되고 동작하는지 더 깊이 파헤쳐 보겠습니다.

## CPS (Continuation Passing style)

코틀린에서는 중단(suspend) 함수를 **Continuation 전달 방식**, 즉 **CPS**(Continuation Passing Style) 을 사용하여 구현합니다. CPS란 **함수를 호출할 때, Continuation 객체를 함께 전달하는 방식**을 의미합니다.

그렇다면 Continuation 객체란 무엇일까요? Continuation 객체는 **코루틴의 현재 상태와 다음에 수행할 작업을 저장하는 확장된 콜백(callback) 역할**을 합니다. 즉, **중단(suspend)** 된 이후에도 이전의 컨텍스트를 유지하면서 **재개(resume)** 될 수 있도록 돕는 핵심 요소입니다.

이를 이해하기 위해 간단한 중단 함수를 살펴보겠습니다.

### **코틀린 중단 함수의 형태 예시**

```kotlin
suspend fun getUser(): User? {
    ...
}

suspend fun setUser(user: User) {
    ...
}

suspend fun checkUser(user: User, userId: Long): Boolean {
    ...
}
```

위와 같은 중단 함수들은 내부적으로 어떻게 변환될까요?

### **코틀린 중단 함수의 내부 구현 형태 예시**

```kotlin
fun getUser(continuation: Continuation<*>): Any? {
    ...
}

fun setUser(user: User, continuation: Continuation<*>): Any {
    ...
}

fun checkUser(user: User, userId: Long, continuation: Continuation<*>): Any {
    ...
}
```

코드를 비교해 보면, 중단 함수는 내부적으로 다음과 같은 변화가 생깁니다.

1. **파라미터 타입에 Continuation 객체 추가**
    - 함수가 실행을 중단하고 다시 재개할 수 있어야 하므로, 이를 관리할 **Continuation 객체를 마지막 파라미터로 추가합니다.**
2. **반환 타입이 ‘Any’ 또는 ‘Any?’로 변경**
    - 중단 함수는 중단이 일어날 경우, 기존의 반환 타입이 아닌 **`COROUTINE_SUSPENDED`** 값을 반환할 수 있습니다.
    - 따라서, 반환 타입이 원래 명시한 타입(`User?`, `Boolean` 등)과 일치하지 않을 수 있게 되고, 명시된 반환 타입과 가장 가까운 슈퍼타입인 **`Any` 또는 `Any?`로 변환됩니다.**

이제 몇 가지 함수 예시를 통해 이를 더 자세히 살펴보겠습니다.

<br>

## 간단한 중단 함수

### **중단이 일어나기 전후를 출력하는 중단 함수: `exampleFunctionOne`**

```kotlin
suspend fun exampleFunctionOne() {
    println("Before")
    delay(1000) // 중단 함수
    println("After")
}
```

위 함수에서 `delay` 또한 중단(suspend) 함수입니다. 즉, `exampleFunctionOne`은 `delay`를 호출하기 전에 `"Before"`를 출력하고, 중단 후 `"After"`를 출력하는 간단한 구조입니다.

이 함수는 내부적으로 어떻게 변환될까요?

### **`exampleFunctionOne`의 내부 구현 (간략화된 형태)**

```kotlin
fun exampleFunctionOne(continuation: Continuation<Unit>): Any {
    val continuation = continuation as? ExampleContinuationImpl ?: ExampleContinuationImpl(continuation)

    if(continuation.label == 0){
        println("Before")
        continuation.label = 1 // 이를 통해 다음에 재개할때는 label 1로 재개합니다.
        if(delay(1000, continuation) == COROUTINE_SUSPENDED) {
            return COROUTINE_SUSPENDED
        }
    }
    if(continuation.label == 1){
        println("After")
        return Unit
    }
    error("Impossible")
}
```

이제 위 코드를 부분별로 분석해 보겠습니다.

### **1. Continuation 객체 래핑**

중단 함수는 상태를 저장하기 위해 **자신만의 Continuation 구현체**를 사용합니다. 여기서는 이를 `ExampleContinuationImpl`이라고 부르겠습니다. (실제로는 이름이 없는 객체의 표현식이지만, 설명을 위해 이름을 붙였습니다.)

아래 코드에서, 기존에 `ExampleContinuationImpl`로 래핑 되어 있지 않다면 새로 래핑 합니다.

```kotlin
val continuation = continuation as? ExampleContinuationImpl ?: ExampleContinuationImpl(continuation)
```

이미 래핑된 경우에는 **그대로 사용해야 중단 이후에도 올바르게 재개**될 수 있습니다. 만약 코루틴이 재실행되고 있으면, Continuation 객체는 이미 `ExampleContinuationImpl` 타입으로  래핑되어 있을테니, 이때는 파라미터로 받은 Continuation 객체를 그대로 두어야 하기 때문입니다.

### **2. `label` 필드를 통한 상태 관리**

중단 함수는 재개 시 어느 부분부터 실행해야 하는지 알아야 합니다. 여기서는 함수가 시작되는 지점이 함수의 시작점과, 중단 이후 재개 시점으로 총 두 곳입니다.

이를 위해 **`label` 필드**를 사용하여 실행 위치를 저장하여, 어디로 돌아와야 할지에 대한 정보를 저장합니다.

```kotlin
if(continuation.label == 0){
    ...
    continuation.label = 1
    ...
}
if(continuation.label == 1){
    ...
}
```

- 처음 실행될 때 `label == 0` 상태에서 `"Before"`를 출력합니다.
- 이후 중단되게 전에 `label = 1`로 변경하여, 재개될 때 `label == 1`에서 `"After"`를 출력합니다.

### **3. `COROUTINE_SUSPENDED`를 활용한 중단 처리**

```kotlin
if (delay(1000, continuation) == COROUTINE_SUSPENDED) {
    return COROUTINE_SUSPENDED
}
```

- `delay`가 실행되면 **중단(suspend)** 될 수 있습니다.
- 이 경우, `COROUTINE_SUSPENDED`를 반환하여 **현재 함수가 중단되었음을 나타냅니다.**
- `exampleFunctionOne`을 호출한 함수도 `COROUTINE_SUSPENDED`를 반환하며, 콜 스택에 있는 모든 함수도 마찬가지로 `COROUTINE_SUSPENDED`를 반환합니다. 즉, 최종적으로 **현재 실행 중인 코루틴이 중단**됩니다.
- `COROUTINE_SUSPENDED` 을 반환함으로써, 함수가 끝난 것 같은 효과를 줍니다.
- 이를 통해 **실행 중이던 스레드를 차단하지 않고 다른 작업을 수행할 수 있도록 만듭니다.**

<br>

## 상태를 가진 중단 함수

중단된 이후에도 실행될 때 다시 사용하기 위해 **유지해야 하는 상태(예: 지역 변수, 파라미터)가 있다면**, 해당 값을 **Continuation 객체에 저장해야 합니다.**

이를 살펴보기 위해 다음 예제를 보겠습니다.

### **상태를 가진 중단 함수: `exampleFunctionTwo`**

```kotlin
suspend fun exampleFunctionTwo() {
    println("Before")
    var counter = 0
    delay(1000) // 중단 함수
    counter++
    println("Counter: $counter")
    println("After")
}
```

여기서 `counter`는 `++` 연산을 통해 값이 변경되므로, **중단되기 전에 저장하고, 재개될 때 복원해야 합니다.**

### **`exampleFunctionTwo`의 내부 구현 (간략화된 형태)**

```kotlin
fun exampleFunctionTwo(continuation: Continuation<Unit>): Any {
    val continuation = continuation as? ExampleContinuationImpl ?: ExampleContinuationImpl(continuation)

    var counter = continuation.counter
    
    if(continuation.label == 0){
        println("Before")
        counter = 0
        continuation.counter = counter // 상태 저장
        continuation.label = 1
        if(delay(1000, continuation) == COROUTINE_SUSPENDED) {
            return COROUTINE_SUSPENDED
        }
    }
    if(continuation.label == 1){
        counter = (counter as Int) + 1
        println("Counter: $counter")
        println("After")
        return Unit
    }
    error("Impossible")
}

```

여기서는 **`counter` 값을 `continuation`에 저장하여 상태를 유지**합니다. 이를 통해 중단 후 재개될 때 **저장된 `counter` 값을 불러와 이어서 실행**할 수 있습니다.

<br>

## 값을 받아 재개되는 중단 함수

이제 중단된 후 **외부로부터 값을 받아야 하는 경우**를 살펴보겠습니다.

### **값을 받아 재개되는 중단 함수: `exampleFunctionThree`**

```kotlin
suspend fun exampleFunctionThree(token: String) {
    println("Before")
    val userId = getUserId(token = token) // 중단 함수
    println("User id: $userId")
    val point = getPoint(userId = userId) // 중단 함수
    println("User point: $point")
    println("After")
}
```

- `getUserId`는 `token`을 받아 **회원 ID를 반환하는 중단 함수**입니다.
- `getPoint`는 `userId`를 받아 **포인트를 반환하는 중단 함수**입니다.
- 이처럼 **중단된 후 값을 반환받아야 하는 경우**, 값을 `Continuation` 객체에 저장해야 합니다.

### **`exampleFunctionThree`의 내부 구현 (간략화된 형태)**

```kotlin
fun exampleFunctionThree(token: String, continuation: Continuation<Unit>): Any {
    val continuation = continuation as? ExampleContinuationImpl ?: ExampleContinuationImpl(continuation, token)

    if(continuation.label == 0) {
        println("Before")
        continuation.label = 1 // 라벨을 1로 저장
        val userId = getUserId(token = token, continuation = continuation)
        if(userId == COROUTINE_SUSPENDED) {
            return COROUTINE_SUSPENDED
        }
        continuation.result = Result.success(userId)
    }
    if(continuation.label == 1) {
        val userId = continuation.result!!.getOrThrow() as Long
        println("User id: $userId")
        continuation.label = 2 // 라벨을 2로 저장
        val point = getPoint(userId = userId, continuation = continuation)
        if(point == COROUTINE_SUSPENDED) {
            return COROUTINE_SUSPENDED
        }
        continuation.result = Result.success(point)
    }
    if(continuation.label == 2) {
        val point = continuation.result!!.getOrThrow() as Long
        println("User point: $point")
        println("After")
        return Unit
    }
    error("Impossible")
}

```

여기서 **중단 함수가 값을 반환하면 `Result.success(value)`로 저장하고, 해당 값을 얻어 사용하게 됩니다**. 만약 **예외가 발생하면 `Result.failure(value)`를 사용해 예외를 던질 수 있습니다.**

<br>

## 콜 스택을 대신하는 **Continuation**

함수가 다른 함수를 호출하면 해당 정보는 콜 스택에 저장됩니다. 그러나 코루틴은 중단되면 스레드를 반환하므로, 콜 스택의 정보가 사라집니다. 이 때문에 코루틴을 재개할 때 기존의 콜 스택 정보를 사용할 수 없습니다.

이를 해결하기 위해 **Continuation 객체가 콜 스택의 역할을 대신합니다.**

Continuation 객체는 중단될 때의 상태(label), 지역 변수와 파라미터, 그리고 중단 함수를 호출한 함수가 재개될 위치 정보를 가지고 있습니다. 또한, Continuation 객체는 다른 Continuation 객체를 참조할 수 있어, 콜 스택이 수행하는 역할을 대체할 수 있습니다.

예를 들어, 중단 함수 `aFunction`이 중단 함수 `bFunction`을 호출하고, `bFunction`이 중단 함수 `cFunction`을 호출하는 상황을 살펴보겠습니다.

### **중단 함수 호출 예시**

```kotlin
suspend fun aFunction() {
    bFunction()
    bFunction()
}

suspend fun bFunction() {
    for (i in 1..10) {
        cFunction(i)
    }
}

suspend fun cFunction(i: Int) {
    delay(i * 100L)
    println("Hello")
}
```

### **해당 호출 과정에서 Continuation 구조 예시**

```kotlin
cFunctionContinuation(
    i = 9,
    label = 1,
    completion = bFunctionContinuation(
        i = 9,
        label = 1,
        completion = aFunctionContinuation(
            label = 2,
            completion = ...
        )
    )
)
```

위 상황에서 "Hello"는 몇 번 출력 되었는지 추측해 볼까요?

Continuation 구조에서 `aFunctionContinuation`의 `label`이 2이므로, `aFunction`에서 호출하는 첫 번째 `bFunction` 호출은 완료된 상태입니다. 따라서 **첫 번째 `bFunction`에서 "Hello"가 10번 출력되었고,** 진행 중인 두 번째 `bFunction` 에서 i는 9이므로 **두 번째 `bFunction`에서 "Hello"가 8번 출력된 상황입니다.** 

**따라서 총 18번 "Hello"가 출력 된 상황으로 볼 수 있습니다.**

<br>

## Continuation의 재개 순서

Continuation 객체가 재개될 때, **각 Continuation 객체는 자신이 담당하는 함수를 실행한 후, 자신을 호출한 함수의 Continuation을 재개합니다.** 이 과정은 스택이 끝날 때까지 반복됩니다.

### **Continuation을 재개하는 `resumeWith` 예시**

```kotlin
override fun resumeWith(result: Result<String>) {
    this.result = result
    val res = try {
        var outcome = invokeSuspendFunction(...)
        if (outcome == COROUTINE_SUSPENDED) {
	          return
        }
        Result.success(outcome)
    } catch (e: Throwable) {
        Result.failure(e)
    }
    completion.resumeWith(res)
}
```

- `invokeSuspendFunction(...)`을 호출하여 중단 함수의 실행을 시도합니다.
- 결과가 `COROUTINE_SUSPENDED`라면, 즉 코루틴이 아직 완료되지 않았다면 아무 작업도 수행하지 않고 종료합니다.
    - 이후 `resumeWith`이 다시 호출될 때, 다시 실행을 합니다.
- 예외가 발생하면 `Result.failure`로 감싸서 전달합니다.
- 마지막으로 `completion.resumeWith(res)`을 호출하여 이전 Continuation을 재개합니다.

### **Continuation 객체를 통해 재개될 때의 호출 순서**

> 중단 함수 a(`aFunction`)가 중단 함수 b(`bFunction`)를 호출하고, 중단 함수 b가 중단 함수 c(`cFunction`)를 호출한 상황에서, 중단된 상황
>

- `aFunction`의 Continuation 객체: `aFunctionContinuation`
- `bFunction`의 Continuation 객체: `bFunctionContinuation`
- `cFunction`의 Continuation 객체: `cFunctionContinuation`

1. 가장 마지막에 호출된 중단 함수 `cFunction`의 Continuation 객체가 재개되어 `cFunction`을 실행합니다.
2. `cFunction`이 완료되면, `cFunctionContinuation`이 `bFunctionContinuation`을 재개합니다.
3. `bFunctionContinuation`이 재개되면서 중단 함수 `bFunction`을 실행합니다.
4. `bFunction`이 완료되면, `bFunctionContinuation`이 `aFunctionContinuation`을 재개합니다.
5. 마지막으로 `aFunctionContinuation`이 재개되며 중단 함수 `aFunction`을 실행합니다.

즉, 중단된 함수가 재개될 때는 일반적인 콜 스택처럼 LIFO(Last In, First Out) 방식으로 동작합니다.

<br>

**일반적인 콜 스택 호출 순서와 Continuation 객체를 통해 재개될 때의 호출 순서 비교**

![image.png](/assets/img/posts/comparison-normal-call-stack-and-resume-continuation.png)

<br>

## 중단 함수의 성능

코루틴의 구현 방식을 보면 복잡해 보일 수 있지만, 성능에 대한 걱정은 크게 하지 않아도 됩니다.

- Continuation 객체에 상태를 저장하고, 상태에 따라 분기하는 작업은 간단한 작업이므로 성능 부담이 크지 않습니다.
- 지역 변수를 복사하는 대신 특정 메모리 위치를 참조하는 방식이므로 메모리 사용량도 크게 증가하지 않습니다.
- Continuation 객체를 생성하는 비용도 큰 비용이 아닙니다.

결론적으로, 코루틴은 성능에 대한 큰 걱정 없이, 효과적으로 비동기 작업을 처리할 수 있습니다.

<br>

## 정리

코루틴의 동작 원리를 살펴보았습니다. 내부 구현을 몰라도 코루틴을 사용하는 데 문제는 없지만, 이를 이해하면 더 깊이 있는 활용이 가능하다고 생각합니다.

흔히 코루틴을 "경량 스레드"라고 표현하기도 하는데, 실제로는 여러 작업을 효율적으로 관리하는 메커니즘에 가깝다고 느꼈습니다. 즉, **코루틴은 하나의 스레드에서 여러 작업을 효과적으로 실행할 수 있도록 도와주는 기술**입니다.

<br>

---

Marcin Moskała, ⌜코틀린 코루틴 Kotlin Coroutines : Deep Dive⌟, 신성열 옮김, 도서출판 인사이트, 2023, p.35-51

우아한테크, "[10분 테코톡] 빙티의 코틀린 코루틴의 동작 방식", [https://www.youtube.com/watch?v=iy-ouIRGDOY](https://www.youtube.com/watch?v=iy-ouIRGDOY), (참고 날짜 2025.02.09)

asdf코딩, "코틀린 코루틴.. 어떻게 동작함?", [https://www.youtube.com/watch?v=IhXXbwjXvyI](https://www.youtube.com/watch?v=IhXXbwjXvyI), (참고 날짜 2025.02.09)