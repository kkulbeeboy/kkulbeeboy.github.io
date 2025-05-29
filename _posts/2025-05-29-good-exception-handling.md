---
title: "예외 처리를 잘 하는 방법에 대해 알아보자"
date: 2025-05-29 00:00:00 +0900
categories: [backend, spring]
tags: [spring, exception]
---

## 예외 처리의 중요성

서비스를 운영하다 보면 예외가 발생하는 상황을 마주하게 됩니다. 이때 예외를 어떻게 처리하느냐에 따라 사용자에게는 적절한 에러 메시지를 제공할 수 있고, 개발자에게는 문제를 빠르게 파악하고 조치할 수 있는 단서를 제공할 수 있습니다.
결과적으로 이는 서비스의 안정성을 높이는 데 큰 도움이 됩니다.

그렇다면 예외 처리를 어떻게 잘할 수 있을까요?

<br>

## 예외를 잡았을때, 이를 적절하게 처리하자

예외가 발생할 수 있는 코드를 실행할 때는 `try-catch`문을 사용하여 예외 상황을 처리할 수 있습니다.

그렇다면, `catch`에서 잡은 예외는 어떻게 처리하는 것이 좋을까요?

### 잡은 예외는 로깅하여 모니터링하자

로깅은 프로그램 실행 중 발생하는 이벤트를 기록으로 남기는 데 사용됩니다. 예외가 발생했을 때도 마찬가지로, 해당 예외 상황을 로깅해두면 예외의 **원인과 발생 위치**를 보다 쉽게 파악할 수 있습니다.

즉, **잡은 예외가 Stack Trace에서 누락되지 않도록** 주의 깊게 처리해야 합니다.

<br>

**Stack Trace가 누락되는 잘못된 예시**

```kotlin
suspend fun requestPayment(paymentRequest: PaymentRequest): PaymentResponse {
    return try {
        webClient.post()
            .uri("/api/payment")
            .bodyValue(paymentRequest)
            .retrieve()
            .bodyToMono(PaymentResponse::class.java)
            .awaitSingle()
    } catch (e: WebClientRequestException) {
        logger.error("결제 요청 실패: 네트워크 오류. (paymentId: ${paymentRequest.paymentId})")
        throw PaymentFailedException(message = "결제 서버와의 통신에 실패했습니다.")
    } catch (e: WebClientResponseException) {
        logger.error("결제 요청 실패: 응답 오류. (paymentId: ${paymentRequest.paymentId})")
        throw PaymentFailedException(message = "결제 요청이 실패했습니다.")
    } catch (e: Exception) {
        logger.error("결제 요청 중 알 수 없는 오류 발생. (paymentId: ${paymentRequest.paymentId})")
        throw PaymentFailedException(message = "결제 처리 중 예기치 못한 오류가 발생했습니다.")
    }
}
```

- 예외를 로깅할 때 `Stack Trace`를 함께 남기지 않아 모니터링에서는 예외의 원인을 추적하기 어렵습니다.
- 원래 발생한 예외(`WebClientRequestException`, `WebClientResponseException` )를 커스텀 예외인 `PaymentFailedException`으로 감싸면서 `cause`를 전달하지 않기 때문에 **원인 예외가 완전히 사라져** 디버깅이 훨씬 어려워집니다.
- PaymentFailedException
    
    ```kotlin
    class PaymentFailedException(message: String) : RuntimeException(message)
    ```
    
- 위와 같은 상황에서 로깅 결과
    
    ```bash
    결제 요청 실패: 응답 오류. (paymentId: PAYMENT_1)
    ...
    [com.blog.demo.exception.PaymentFailedException: 결제 요청이 실패했습니다.]
    ```
    
<br>

**Stack Trace를 유지하는 올바른 예시**

```kotlin
suspend fun requestPayment(paymentRequest: PaymentRequest): PaymentResponse {
    return try {
        webClient.post()
            .uri("/api/payment")
            .bodyValue(paymentRequest)
            .retrieve()
            .bodyToMono(PaymentResponse::class.java)
            .awaitSingle()
    } catch (e: WebClientRequestException) {
        logger.error("결제 요청 실패: 네트워크 오류. (paymentId: ${paymentRequest.paymentId})", e)
        throw PaymentFailedException(message = "결제 서버와의 통신에 실패했습니다.", cause = e)
    } catch (e: WebClientResponseException) {
        logger.error("결제 요청 실패: 응답 오류. (paymentId: ${paymentRequest.paymentId})", e)
        throw PaymentFailedException(message = "결제 요청이 실패했습니다.", cause = e)
    } catch (e: Exception) {
        logger.error("결제 요청 중 알 수 없는 오류 발생. (paymentId: ${paymentRequest.paymentId})", e)
        throw PaymentFailedException(message = "결제 처리 중 예기치 못한 오류가 발생했습니다.", cause = e)
    }
}
```

- 예외를 로깅할 때 `Throwable` 객체를 함께 전달하여 전체 `Stack Trace`가 로깅됩니다.
- `PaymentFailedException` 생성 시 `cause`로 원래 예외를 전달함으로써 **원래 예외를 보존**하고, 모니터링이나 로그를 통해 정확한 원인을 추적할 수 있습니다.
- PaymentFailedException
    
    ```kotlin
    class PaymentFailedException(message: String, cause: Throwable? = null) : RuntimeException(message, cause)
    ```
    
- 이 경우의 로깅 결과
    
    ```bash
    결제 요청 실패: 응답 오류. (paymentId: PAYMENT_1)
    org.springframework.web.reactive.function.client.WebClientResponseException$InternalServerError: 500 Internal Server Error from POST http://...
    ...
    [com.blog.demo.exception.PaymentFailedException: 결제 요청이 실패했습니다.] with root cause
    org.springframework.web.reactive.function.client.WebClientResponseException$InternalServerError: 500 Internal Server Error from POST http://...
    ```
    
<br>

물론 글로벌 예외 핸들러에서 로깅을 잘 처리하고 있다면, 모든 위치에서 예외를 로깅할 필요는 없습니다. 하지만 **예외가 발생한 컨텍스트에 대한 추가 정보** (ex paymentId)는 글로벌 예외 핸들러에서 알 수 없는 경우가 많기 때문에, 예외를 잡은 위치에서 해당 정보를 로깅하는 것이 더 편할 수 있습니다.

### 예외 상황을 설명하는 메시지와 식별자를 함께 로깅하자

효율적인 모니터링을 위해서는 단순히 `Exception.message`를 로깅하는 것만으로는 부족합니다. 예외의 맥락을 설명하는 **의미 있는 메시지**(ex "결제 요청 실패: 네트워크 오류" 등)와 예외를 구체적으로 식별할 수 있는 **고유 식별자**(ex paymentId)를 함께 기록하는 것이 중요합니다.

<br>

**나쁜 예시 – 단순 메시지만 로깅**

```kotlin
try {
    ...
} catch (e: WebClientRequestException) {
    logger.error(e.message, e)
    ...
} catch (e: WebClientResponseException) {
    logger.error(e.message, e)
    ...
} catch (e: Exception) {
    logger.error(e.message, e)
    ...
}
```

- 로그만 보고 **무슨 상황에서 예외가 발생했는지 파악하기 어렵습니다**.
- 특히 동일한 예외 타입이 여러 곳에서 발생할 수 있기 때문에, **문맥 없이 message만 로깅하는 것은 문제를 추적하기 어렵습니다.**

<br>

**좋은 예시 – 상황 메시지와 식별자 포함**

```kotlin
try {
    ...
} catch (e: WebClientRequestException) {
    logger.error("결제 요청 실패: 네트워크 오류. (paymentId: ${paymentRequest.paymentId})", e)
    ...
} catch (e: WebClientResponseException) {
    logger.error("결제 요청 실패: 응답 오류. (paymentId: ${paymentRequest.paymentId})", e)
    ...
} catch (e: Exception) {
    logger.error("결제 요청 중 알 수 없는 오류 발생. (paymentId: ${paymentRequest.paymentId})", e)
    ...
}
```

- **예외가 발생한 원인**과 **어떤 요청에서 발생했는지 식별 가능한 정보**가 함께 로그에 남기 때문에, 빠르게 원인을 파악할 수 있습니다.
- 이는 Datadog과 같은 모니터링 시스템에서 예외를 수집할 때도 매우 유용합니다.

### 잡은 예외에 대해 책임을 지자

`catch`를 통해 예외를 잡았다면, 그 예외에 대한 **책임도 함께 져야** 합니다. 그저 로깅만 한 뒤 **예외를 그대로 다시 던지는 행위는 일반적으로는 지양하는 것이 좋습니다.**

한 번 예외를 `catch`로 잡았다는 것은 "이 예외를 내가 처리하겠다"는 의사 표시와 같으며, 아무런 조치 없이 `throw e`로 해당 예외를 다시 던지는 것은 책임을 회피하는 것과 같습니다.

<br>

**지양해야 할 코드: 예외를 잡고 그대로 다시 던지는 코드**

```kotlin
try {
    ...
} catch (e: Exception) {
    logger.error("알 수 없는 오류 발생")
    throw e
}
```

이런 코드는 단순히 Stack Trace만 길어지고, 예외 흐름을 불필요하게 복잡하게 만들 수 있습니다.

그렇다면 예외를 잡은 후에는 어떤 방식으로 책임 있게 처리할 수 있을까요?

### 잡은 예외를 처리하는 방법

**예외 흡수**

```kotlin
try {
    ...
} catch (e: TimeoutException) {
    logger.warn("일시적 타임아웃 발생. 무시하고 계속 진행.")
}

```

- 발생한 예외가 이후 로직에 영향을 주지 않는 경우, **로깅만 하고 흐름을 유지할 수 있습니다**.
- 이처럼 **의도적으로 예외를 무시하는 상황**에는 `error` 레벨보다는 `warn` 레벨을 사용하는 것이 적절합니다.
- ex 간헐적인 타임아웃, 재시도 전에 발생한 일시적 예외 등

<br>

**예외 전환**

```kotlin
try {
    ...
} catch (e: WebClientRequestException) {
    logger.error("결제 요청 실패: 네트워크 오류. (paymentId: ${paymentRequest.paymentId})", e)
    throw PaymentFailedException(message = "결제 서버와의 통신에 실패했습니다.", cause = e)
} catch (e: WebClientResponseException) {
    logger.error("결제 요청 실패: 응답 오류. (paymentId: ${paymentRequest.paymentId})", e)
    throw PaymentFailedException(message = "결제 요청이 실패했습니다.", cause = e)
} catch (e: Exception) {
    logger.error("결제 요청 중 알 수 없는 오류 발생. (paymentId: ${paymentRequest.paymentId})", e)
    throw PaymentFailedException(message = "결제 처리 중 예기치 못한 오류가 발생했습니다.", cause = e)
}
```

- 발생한 예외를 **도메인에 맞는 의미 있는 예외로 변환**합니다.
- 이 방식의 장점은 **상위 계층에서 도메인 중심의 예외만 다루면** 된다는 점입니다.
    - `PaymentFailedException`은 "결제 실패"라는 도메인 관점의 의미를 전달합니다.
- 이때 반드시 **원래 예외를 `cause`로 전달**해 예외의 상세 정보(원인, 위치)를 보존합니다.
- 예외 메시지는 **사용자에게 보여질 수 있는 메시지**라는 점을 고려해 작성합니다.
    - 로깅은 개발자를 위한 것이고, 예외 메시지는 사용자를 위한 것이라고 생각합니다.

<br>

## 비즈니스 로직에서는 커스텀 예외를 사용하자

비즈니스 로직에서 커스텀 예외를 사용하면, 해당 예외가 어떤 의미를 가지는지 **도메인 관점에서 명확하게 표현**할 수 있고, 이로 인해 유지보수성과 예외 처리의 일관성이 향상됩니다.

### 왜 표준 예외만으로는 부족할까?

만약 비즈니스 로직에서 발생한 예외임에도 불구하고, `IllegalStateException`, `IllegalArgumentException` 같은 **표준 예외만을 사용**한다면 가장 큰 문제는 **프레임워크(Spring)**나 **언어(Java/Kotlin)** 자체에서 발생한 예외와 **구분이 어렵다는점** 입니다.

따라서 이 예외가 내 비즈니스 로직에서 발생한 건지, 시스템 내부에서 발생한 건지 판단하기가 어렵고, 그에 따라 적절한 후속 조치(예외 복구, 재시도, 보상 처리 등)를 적용하기 힘들어집니다.

### 커스텀 예외의 장점

1. **의미 부여**
    - 예외가 어떤 상황에서 발생했는지 명확하게 표현할 수 있습니다.
2. **구분 용이**
    - 비즈니스 로직에서 발생한 예외인지, 시스템 또는 프레임워크 예외인지 명확히 구분할 수 있습니다.
3. **핸들링 편의성**
    - 커스텀 예외를 기준으로 글로벌 예외 처리기에서 특정 예외들을 별도로 핸들링하기가 쉬워집니다.

### 커스텀 예외를 계층적으로 관리하자

비즈니스 로직에서 발생할 수 있는 다양한 예외 상황에 대해 커스텀 예외를 정의하다 보면, 정의된 예외 클래스가 점점 많아지고 관리가 어려워질 수 있습니다.

이럴 때는 **예외를 분류하여 계층적으로 설계**함으로써, 예외를 **그룹화하고 일관되게 관리**할 수 있습니다.

**예외 계층 구조의 장점**

- 전역 예외 처리에서 **상위 예외만 핸들링**하면, 하위 예외들을 모두 포괄할 수 있습니다.
- 예외의 **도메인별 책임을 명확하게 구분**할 수 있습니다.
    - ex 결제 관련 예외는 `PaymentException`, 주문 관련 예외는 `OrderException` 등
- 유지보수 시 어떤 예외들이 어떤 흐름에서 발생하는지 **파악하기 쉬워집니다.**

### 커스텀 예외를 계층적으로 관리하자

비즈니스 로직에서 발생할 수 있는 상황에 따른 커스텀 예외를 만들다 보면, 꽤 많은 커스텀 예외들이 정의될 수 있습니다.

이때, 예외들을 분류하고 상위 예외로 관리하여 예외를 계층적으로 관리할 수 있습니다.

그러면, 전역 예외 핸들러에서 모든 비즈니스 예외를 핸들링하는 대신, 상위 비즈니스 예외(PaymentException)만 핸들링하도록 관리할 수 도 있습니다.

<br>

**계층 구조를 가지고 있는 커스텀 예외**

```kotlin
abstract class PaymentException(
    message: String, 
    cause: Throwable? = null
) : RuntimeException(message, cause)

class PaymentFailedException(
    message: String, 
    cause: Throwable? = null
) : PaymentException(message, cause)
```

- `PaymentFailedException`은 결제 실패에 대한 하위 예외이며, `PaymentException`을 상속받아 결제 도메인 예외라는 그룹 안에 속합니다.
- 즉 글로벌 예외 핸들러에서 하위 예외(PaymentFailedException, PaymentTimeoutException 등)를 개별적으로 핸들링하지 않아도, 상위 예외인 PaymentException 하나로 결제 도메인 예외를 포괄해서 처리할 수 있습니다.

### 커스텀 예외에 세부 정보를 넣어서 활용하자

커스텀 예외는 단순히 메시지와 원인만 담는 용도에서 그치지 않고, **HTTP 상태 코드(HttpStatus), 로깅 레벨(LogLevel) 등** 추가적인 속성값을 함께 관리함으로써 **예외 처리의 확장성을 높일 수 있습니다.**

<br>

**HttpStatus 속성을 가진 커스텀 예외**

```kotlin
abstract class PaymentException(
    message: String,
    cause: Throwable? = null,
    val httpStatus: HttpStatus
) : RuntimeException(message, cause)

class PaymentFailedException(
    message: String,
    cause: Throwable? = null,
    httpStatus: HttpStatus
) : PaymentException(message, cause, httpStatus)
```

- `PaymentFailedException`은 httpStatus를 명시적으로 포함하고 있어, 예외 처리 시 어떤 응답 코드로 반환할지 판단하기가 쉬워집니다.
- 이 방식은 HttpStatus뿐만 아니라 로깅 레벨, 식별값 등 다양한 예외 속성들을 함께 포함시켜 관리할 수 도 있습니다.
- 해당 속성값을 이용해 글로벌 예외 핸들러에서 속성값에 따른 각각의 처리를 수행할 수 있습니다.

<br>

## 글로벌 예외 핸들러를 이용하자

모든 예외를 일일이 `try-catch`로 처리하는 것은 현실적으로 어렵습니다. 특히 `NullPointerException`, `IllegalStateException` 같은 예외는 **깊은 호출 스택에서 예기치 않게 발생**할 수 있습니다.

이런 예외들을 놓치면 사용자는 500 에러 페이지를 마주하고, 개발자는 로그에 아무것도 남지 않아 디버깅조차 할 수 없는 상황에 놓이게 됩니다.

그러므로 글로벌 예외 핸들러를 통해 **예기치 못한 예외라도** **예외 응답 형태를 표준화**하고, **로그에 남길수 있는 환경**을 제공해야 합니다.

### try-catch로 잡지 못하는 예외라도 최소한 로깅은 하자

예상하지 못한 예외에 대비해서, **최상위 예외 핸들러를 등록**하면 예외가 어딘가에서 터져도 **최소한 로깅과 표준 응답**은 할 수 있습니다.

<br>

**최상위 예외(Exception)를 포착하여 예상치 못한 예외에 대해 공통으로 로깅과 기본적인 응답처리를 하는 예시 코드**

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(Exception::class)
    fun handleException(ex: Exception): ResponseEntity<ErrorResponse> {
        logger.error("예상하지 못한 예외 발생: message=${ex.message}", ex)
        val response = ErrorResponse(
            code = ErrorCode.SERVER_ERROR,
            message = "서버 오류가 발생했습니다. 잠시 후 다시 시도해주세요."
        )
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response)
    }
    
    ...
}
```

- `@RestControllerAdvice`는 모든 컨트롤러에서 발생하는 예외를 가로채 처리합니다.
- `@ExceptionHandler(Exception::class)`는 모든 예외를 포착합니다.
- `logger.error(...)`를 통해 Stack Trace를 남겨 디버깅이 가능하도록 합니다.
- 실제 로깅 예시
    
    ```bash
    예상하지 못한 예외 발생: message=UNKNOWN ERROR
    java.lang.Exception: UNKNOWN ERROR
    	...
    ```
    
- 사용자에게 일관된 에러 응답 형태로 전달합니다.
    
    ```json
    {
        "code": "SERVER_ERROR",
        "message": "서버 오류가 발생했습니다. 잠시 후 다시 시도해주세요."
    }
    ```
    

### 포착한 예외에 따른 개별 처리

글로벌 예외 핸들러에서는 **발생한 예외의 타입에 따라 개별 처리를 적용할 수 있습니다.** 이를 통해 클라이언트에게 더 명확한 HTTP 응답을 전달하고, 로그도 상황에 맞게 남길 수 있습니다.

<br>

**예외 유형에 따른 HttpStatus 응답 예시**

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(PaymentException::class)
    fun handlePaymentException(ex: PaymentException): ResponseEntity<ErrorResponse> {
        logger.error("결제 예외 발생: message=${ex.message}", ex)
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse(ErrorCode.PAYMENT_ERROR, ex.message ?: "결제 오류가 발생했습니다"))
    }

    @ExceptionHandler(PaymentBadRequestException::class)
    fun handlePaymentBadRequestException(ex: PaymentBadRequestException): ResponseEntity<ErrorResponse> {
        logger.error("잘못된 결제 요청 예외 발생: message=${ex.message}", ex)
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(ErrorResponse(ErrorCode.BAD_REQUEST, ex.message ?: "결제 요청값이 유효하지 않습니다"))
    }
    
    ...
}
```

하지만, 예외별로 개별 처리를 진행하기 위해 커스텀 예외를 만든다면, 예외 수 만큼 `@ExceptionHandler`가 늘어나고, 유지 보수가 어려워진다는 한계가 있습니다.

앞서 소개한 것처럼, 커스텀 예외 클래스에 개별 처리에 필요한 속성값(ex HttpStatus)을 포함하면 **예외마다 따로 매핑하지 않아도** 적절한 응답을 반환할 수 있습니다. 이를 이용하면, 개별 처리에 필요한 속성값을 매핑하기 위한 커스텀 예외를 따로 만들지 않아도 됩니다

<br>

**예외의 HttpStatus 속성값에 따른 HttpStatus 응답 예시**

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(PaymentException::class)
    fun handlePaymentException(ex: PaymentException): ResponseEntity<ErrorResponse> {
        logger.error("결제 예외 발생: message=${ex.message}", ex)
        return ResponseEntity.status(ex.httpStatus)
            .body(ErrorResponse(ErrorCode.PAYMENT_ERROR, ex.message ?: "결제 오류가 발생했습니다"))
    }
    
    ...
}
```

- 위 예시처럼 커스텀 예외 클래스에 HttpStatus 값을 포함시키면, 예외 타입 자체에 정의된 HttpStatus로 응답을 만들 수 있습니다.

<br>

## 정리

- 예외가 발생했다면 예외 상황을 잘 파악할 수 있도록 로깅을 해야한다.
- 비즈니스 로직에서는 표준 예외 대신 도메인에 맞는 커스텀 예외를 사용하자.
- 커스텀 예외는 계층적으로 관리하고, HttpStatus 등의 속성값을 포함하면 예외 처리를 확장성 있게 할 수 있다.
- 글로벌 예외 핸들러를 통해 처리하지 못한 예외도 포착하고, 로그와 응답을 표준화할 수 있다.

<br>

---

티스토리, “좋은 예외(Exception) 처리”, [https://jojoldu.tistory.com/734](https://jojoldu.tistory.com/734), (참고 날짜 2025.05.29)

우아한 기술블로그, “IllegalArgumentException은 400 Bad Request인가?”, [https://techblog.woowahan.com/21686/](https://techblog.woowahan.com/21686/), (참고 날짜 2025.05.29)

F-Lab, “자바에서의 예외 처리와 로깅의 중요성”, [https://f-lab.kr/insight/exception-handling-and-logging-in-java](https://f-lab.kr/insight/exception-handling-and-logging-in-java), (참고 날짜 2025.05.29)

F-Lab, “코틀린에서의 예외 처리와 안전한 코드 작성”, [https://f-lab.kr/insight/exception-handling-and-writing-safe-code-in-kotlin-20240818](https://f-lab.kr/insight/exception-handling-and-writing-safe-code-in-kotlin-20240818), (참고 날짜 2025.05.29)
