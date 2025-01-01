---
title: "제목은 코틀린 중단(suspend) 동작으로 하겠습니다. 근데 이제 Continuation 객체를 곁들인"
date: 2024-11-30 00:00:00 +0900
categories: [language, kotlin]
tags: [kotlin, coroutine, suspend, continuation]
---

## 코틀린 코루틴의 중단

코틀린 코루틴의 핵심은 코루틴을 특정 지점에서 멈춘 뒤, 이후에 다시 재개할 수 있다는 것입니다. 

예를 들어, 스레드에서 API를 호출하여 외부에서 데이터를 얻어올 때 잠시 코루틴을 중단시키지만, **스레드는 블로킹되지 않고 다른 코루틴을 실행시키는 것**과 같은 작업이 가능합니다.

즉, 많은 동작을 하기 위해, 많은 스레드를 생성하는 것보다 코루틴을 사용하는 게 메모리 및 프로세스 비용적으로 이점이 있으며, 이러한 중단은 코루틴의 핵심입니다.

<br>

## 코루틴이 중단되었을 때 반환하는 Continuation 객체

코루틴을 중단하는 것은 코루틴 실행을 중간에 멈추는 것을 의미하며, 이후에 중단한 코루틴의 실행이 필요할 때 재개할 수 있습니다.

그렇다면, 어떻게 중단이 된 곳에서 코루틴 재개를 할 수 있는 것일까요? 이는 **코루틴이 중단되었을때 `Continuation` 객체를 반환하기 때문입니다.**

### 중단된 코루틴이 재개되는 원리

중단된 코루틴이 재개되는 원리를 에제 코드를 통해 확인해 보겠습니다. 예제 코드에서는 코루틴을 재개하기 위해, 중단(suspend) 함수를 사용해서 코루틴을 중단할 수 있는 환경을 만들었습니다.

<br>

**main 함수를 중단 가능한 함수로 사용하여, 코루틴을 중단할 수 있는 환경을 제공**

```kotlin
suspend fun main() {
    println("Before")
    println("After")
}
```

- **실행 결과**
    
    ```kotlin
    Before
    After
    ```
    
<br>

이때, 함수 내부에서 `suspendCoroutine` 함수를 이용하여, "Before"와 "After"를 출력하는 출력문 사이를 중단 지점으로 하여, 중단해 보겠습니다.

<br>

**suspendCoroutine 함수를 사용한 중단**

```kotlin
suspend fun main() {
    println("Before")

    suspendCoroutine<Unit> {  }

    println("After")
}
```

- suspendCoroutine 함수란?
    
    ```kotlin
    suspend inline fun <T> suspendCoroutine(crossinline block: (Continuation<T>) -> Unit): T
    ```
    
    suspendCoroutine 람다 표현식(`{ }`)에 인자로 들어가는 람다 함수는 **중단이 되기 전에 실행**되는데, **해당 함수가 Continuation 객체를 인자로 받습니다.**
    
- **실행 결과**
    
    ```kotlin
    Before
    ```
    
    - "After"는 출력되지 않으며 실행이 끝나지 않은 상태로 유지됩니다.
    - 이는 "Before"가 출력된 이후 프로그램이 멈추고, **재개되지 않았기 때문입니다.**

<br>

위 예시에서 중단된 프로그램을 재개하는 방법의 핵심은 위에서 언급한, 코루틴이 중단되었을 때 반환하는 **`Continuation` 객체에 있습니다.**

이때, 람다 함수는 Continuation 객체를 저장한 뒤 코루틴을 다시 실행할 시점을 결정하기 위해 사용됩니다.

<br>

**Continuation 객체를 이용해 코루틴을 중단 한 뒤 곧바로 실행**

```kotlin
suspend fun main() {
    println("Before")

    suspendCoroutine<Unit> { continuation ->
        continuation.resume(Unit)
    }

    println("After")
}
```

- **실행 결과**
    
    ```kotlin
    Before
    After
    ```
    
- `suspendCoroutine`가 호출된 뒤에는 이미 중단되어 Continuation 객체를 사용할 수 없기 때문에, **Continuation 객체를 사용하기 위해서는 suspendCoroutine 함수의 인자로 들어가 람다 함수에서 중단되기 전에 사용할 수 있습니다.**
- suspendCoroutine의 **코루틴이 중단되기 전에 Continuation 객체의 `resume`을 호출하여, 중단된 코루틴이 재개**되어 "After"가 출력되고 프로그램 실행이 종료될 수 있었습니다.
    - 하지만, 실제로는 중단된 코루틴이 곧바로 재개될 경우 최적화로 인해 아예 중단되지 않습니다.

<br>

**suspendCoroutine에서 1초 뒤 재개되는 다른 스레드를 실행**

```kotlin
fun continueAfterSecond(continuation: Continuation<Unit>) {
    thread {
        Thread.sleep(1000) // 다른 스레드에서 1초 동안 정지
        continuation.resume(Unit)
    }
}

suspend fun main() {
    println("Before")

    suspendCoroutine<Unit> { continuation ->
        continueAfterSecond(continuation)
    }

    println("After")
}
```

- **실행 결과**
    
    ```kotlin
    Before
    After
    ```
    
    - “Before” 출력 후 1초 뒤 “After”를 출력합니다.
- 호출된 다른 스레드에서 1초 동안 정지된 뒤, 다시 코루틴을 재개하였습니다.

<br>

## 중단된 코루틴을 값으로 재개

위 예시에서 suspendCoroutine 타입의 인자로 `Unit`을 사용하고, resume 함수에 `Unit` 인자를 넣은 이유는, **Unit이 중단 함수의 리턴 타입이고, Continuation의 제네릭 타입 인자**이기 때문입니다.

```kotlin
val result: Unit = suspendCoroutine<Unit> { continuation: Continuation<Unit> -> 
    continuation.resume(Unit)
}
```

suspendCoroutine을 호출할 때 Continuation 객체로 반환될 값의 타입을 지정할 수 있으며, resume을 통해 반환되는 값은 **반드시 지정된 타입과 같은 타입**이어야 합니다.

코루틴에서 **값으로 중단된 코루틴을 재개하는 상황**은 **외부 API를 호출해 특정 데이터를 기다리려고 중단하는 상황**을 예시로 들 수 있습니다.

코루틴이 없다면 스레드는 응답을 기다리고 있을 수밖에 없지만, 코루틴을 통해 중단함과 동시에 `“데이터를 받고 나면, 받은 데이터를 resume 함수를 통해 보내줘”`라고 Continuation 객체를 통해 전달하면 **스레드는 다른 일을 할 수 있습니다.** 그리고 **데이터가 도착하면 스레드는 코루틴이 중단된 지점에서 재개**하게 됩니다.

```kotlin
suspend fun requestUser(name: String): User {
    ...
}

suspend fun main() {
    println("Before")

    val user = requestUser(name = "jude")
    println(user)

    println("After")
}

```

- **실행 결과**
    
    ```kotlin
    Before
    User(name=jude)
    After
    ```
    
<br>

## 중단된 코루틴을 예외로 재개하기

외부 API를 통해 데이터를 가져오려고 할 때, 예외가 발생하면 데이터를 가져올 수 없으므로, 코루틴이 중단된 곳에서 예외를 발생시켜야 합니다.

이때, `resumeWithException`이 호출되면 중단된 지점에서 인자로 넣어준 예외를 던집니다.

<br>

**resumeWithException를 사용해 중단된 지점에서 예외 발생 예시 코드**

```kotlin
class MyException: Throwable("Just an exception")

suspend fun main() {
    try {
        suspendCoroutine<Unit> { continuation ->
            continuation.resumeWithException(MyException())
        }
    } catch (e: MyException) {
        println("Caught!")
    }
}
```

<br>

## 중단 함수는 함수를 중단시키는게 아닌, 코루틴을 중단시킨다

중단 함수는 코루틴이 아니고, **단지 코루틴을 중단할 수 있는 함수**라고 할 수 있습니다. 

→ 단, 중단 가능한 main 함수(`suspend fun main`)와 같은 경우는 특별한 경우로, 코틀린 컴파일러가 main 함수를 코루틴으로 실행시킵니다.

<br>

**변수에 Continuation 객체를 저장하고, 함수를 호출한 다음에 재개하는 상황**

```kotlin
var varContinuation: Continuation<Unit>? = null

suspend fun suspendAndSetContinuation() {
    suspendCoroutine<Unit> { continuation ->
        varContinuation = continuation
    }
}

suspend fun main() {
    println("Before")

    suspendAndSetContinuation()
    varContinuation?.resume(Unit)

    println("After")
}
```

- **실행 결과**
    
    ```kotlin
    Before
    ```
    
    - 실행이 끝나지 않고, 실행된 상태로 유지됩니다.
- 위 코드의 실행은, 다른 스레드나 다른 코루틴을 재개하지 않으면 계속 실행된 상태로 유지하게 됩니다.
- 즉, `continuation?.resume(Unit)`가 호출되는 시점에서 코루틴이 이미 종료 상태이므로, 프로그램이 정지 상태에 빠지게 됩니다.

## 정리
효과적인 동작을 위해 코틀린 중단(suspend)을 사용하는데, 이러한 코틀린 중단이 어떻게 중단되고 재개될 수 있는지에 대해 알아보며, Continuation 객체와 밀접한 관계가 있다는 것을 알 수 있었습니다.

<br>

---

Marcin Moskała, ⌜코틀린 코루틴 Kotlin Coroutines : Deep Dive⌟, 신성열 옮김, 도서출판 인사이트, 2023, p.10, p.23-34