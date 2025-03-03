---
title: "WebFlux에서 코루틴 사용하기"
date: 2025-03-03 00:00:00 +0900
categories: [backend, spring]
tags: [spring, kotlin, coroutine, webflux]
---

운영 중인 서비스가 많은 트래픽을 처리해야 한다면, 어떤 방법을 고려할 수 있을까요?

서버 리소스를 늘리는 것도 하나의 방법입니다. 그러나 비용 문제로 인해 무작정 리소스를 늘릴 수는 없습니다. 이럴 때 고려할 수 있는 것이 **Reactive Stack**이며, 이를 지원하는 프레임워크가 바로 **Spring WebFlux**입니다.

하지만 **WebFlux**를 사용하려면 **Reactor**의 `Mono`, `Flux` 타입을 다뤄야 합니다. 또한, 명령형 방식이 아닌 **선언형 방식**으로 코드를 작성해야 하므로, **이는 개발자에게 익숙하지 않은 구조일 수 있습니다.**

그런데 **코틀린의 코루틴을 WebFlux와 함께 사용하면, Spring MVC 스타일과 유사한 방식으로 Reactive Stack을 구현할 수 있습니다.** 이제 코루틴과 WebFlux를 어떻게 조합하여 사용할 수 있는지 살펴보겠습니다.

먼저, **Reactive Stack**과 이를 구성하는 핵심 기술을 간략히 정리해보겠습니다.

## Reactive Stack

**Reactive Stack**은 전통적인 Servlet Stack과 달리, **Non-Blocking** 서버(예: Netty, Undertow, Servlet 3.1+ 컨테이너)에서 동작하는 **Reactive Streams** 기반의 기술 스택입니다.

이를 통해 많은 요청을 적은 스레드로 처리할 수 있어, 한정된 리소스로도 안정적인 서비스를 구축하는 데 효과적입니다.

Reactive Stack의 주요 구성 요소에는 **Reactive Streams**, **Reactor**, **WebFlux** 등이 있습니다.

### **Reactive Streams**

**Reactive Streams**는 데이터 스트림을 **비동기적**이고 **Non-Blocking**한 방식으로 처리하기 위한 **표준 사양**입니다. 즉, 리액티브 라이브러리를 어떻게 구현할지 정의해 놓은 표준 사양이기도 합니다.

Reactive Streams는 `Publisher`, `Subscriber`, `Subscription`, `Processor`와 같은 핵심 컴포넌트로 구성됩니다.

- **Reactive Streams를 구현한 대표적인 라이브러리**
    - RxJava, **Reactor**, Akka Streams, Java 9 Flow API 등

### **Reactor (Project Reactor)**

**Reactor**는 **Reactive Streams를 구현한 라이브러리 중 하나**로, Spring WebFlux 기반의 개발을 위한 핵심 역할을 담당합니다.

### **WebFlux**

**Spring WebFlux**는 **리액티브 웹 애플리케이션 구현을 위한 웹 프레임워크**로, Reactor의 `Publisher`(`Mono`, `Flux`)를 사용하여 **Non-Blocking** 통신을 지원합니다.

→ Reactor Core 라이브러리는 Spring WebFlux 프레임워크에 포함되어 있습니다.

Spring MVC는 서블릿 기반의 Blocking 방식이지만, Spring WebFlux는 Non-Blocking 방식으로 적은 스레드로도 대량의 트래픽을 안정적으로 처리할 수 있습니다.

<br>

## WebFlux를 이용한 코드

WebFlux를 이용한 코드 구조를 예제를 통해 확인해 보겠습니다.

**예시 상황**

> 회원의 주문 상세 정보를 조회하는 API를 개발한다.
> 

**통신 구조**

1. **주문 서버**에서 주문 내역 조회
2. **결제 서버**에서 결제 상태 조회
3. **배송 서버**에서 배송 상태 조회
4. 각 조회 데이터를 조합하여 하나의 응답으로 반환

![image.png](/assets/img/posts/webflux-sample-communication-structure.png)

### **종속성 추가**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
}
```

- Spring WebFlux를 사용하기 위해 종속성을 추가합니다.

<br>

### **Controller 계층**

**OrderController**

```kotlin
@RestController
@RequestMapping("/orders")
class OrderController(
    private val orderService: OrderService
) {
    @GetMapping("/{orderId}")
    fun getOrderInfo(@PathVariable orderId: Long): Mono<OrderInfoResponse> {
        return orderService.getOrderDetails(orderId)
    }
}
```

- WebFlux에서는 비동기 처리를 위해 `Mono` 또는 `Flux` 타입을 사용합니다.
- `getOrderInfo()` 메서드를 통해 주문 상세 정보를 조회합니다.
- OrderInfoResponse 응답 예시
    
    ```json
    {
        "orderId": 10,
        "userId": 100,
        "orderItems": [
            {
                "productId": 1001,
                "quantity": 1
            },
            {
                "productId": 1002,
                "quantity": 2
            }
        ],
        "paymentStatus": "COMPLETED",
        "deliveryStatus": "IN_TRANSIT"
    }
    ```
    
<br>

### **Service 계층**

**OrderService**

```kotlin
@Service
class OrderService(
    private val orderClient: OrderClient,
    private val paymentClient: PaymentClient,
    private val deliveryClient: DeliveryClient
) {
    fun getOrderDetails(orderId: Long): Mono<OrderInfoResponse> {
        return Mono.zip(
                orderClient.getOrder(orderId), // 주문 정보 조회 (Mono 반환)
                paymentClient.getPaymentStatus(orderId), // 결제 상태 조회 (Mono 반환)
                deliveryClient.getDeliveryStatus(orderId) // 배송 상태 조회 (Mono 반환)
            )
            .map { tuple ->
                OrderInfoResponse(
                    orderId = tuple.t1.orderId,
                    userId = tuple.t1.userId,
                    orderItems = tuple.t1.orderItems,
                    paymentStatus = tuple.t2.paymentStatus,
                    deliveryStatus = tuple.t3.deliveryStatus
                )
            }
    }
}
```

- 각 클라이언트(`OrderClient`, `PaymentClient`, `DeliveryClient`)에서 데이터를 `Mono`로 반환합니다.
- `Mono.zip()`을 사용하여 여러 서버의 데이터를 **동시에** 조회합니다
- `zip`의 결과로 `tuple`을 반환하면, 이를 `OrderInfoResponse`객체로 변환합니다.

<br>

### **Client 계층**

WebFlux의 리액티브 웹 클라이언트인 **WebClient**를 사용하여 **Non-Blocking HTTP 요청**을 처리했습니다.

WebClient는 내부적으로 HTTP 요청을 기본 HTTP 클라이언트 라이브러리에게 위임하는데, 기본 HTTP 클라이언트 라이브러리가 **Reactor Netty 입니다.**

<br>

**OrderClient**

```kotlin
@Component
class OrderClient(private val orderClient: WebClient) {
    fun getOrder(orderId: Long): Mono<OrderResponse> {
        return orderClient.get()
            .uri("/api/orders/$orderId")
            .retrieve()
            .bodyToMono(OrderResponse::class.java)
    }
}
```

- 주문 서버에서 주문 정보를 가져옵니다.
- 응답을 `Mono<OrderResponse>` 타입으로 반환합니다.
- OrderResponse 응답 예시
    
    ```json
    {
      "orderId": 10,
      "userId": 100,
      "orderItems": [
        { "productId": 1001, "quantity": 1 },
        { "productId": 1002, "quantity": 2 }
      ]
    }
    ```

<br>    

**PaymentClient**

```kotlin
@Component
class PaymentClient(private val paymentClient: WebClient) {
    fun getPaymentStatus(orderId: Long): Mono<PaymentStatusResponse> {
        return webClient.get()
            .uri("/api/payments/$orderId/status")
            .retrieve()
            .bodyToMono(PaymentStatusResponse::class.java)
    }
}
```

- 결제 서버에서 주문의 결제 상태를 조회합니다.
- 응답을 `Mono<PaymentStatusResponse>` 타입으로 반환합니다.
- PaymentStatusResponse 응답 예시
    
    ```json
    {
      "paymentStatus": "COMPLETED"
    }
    ```
    
<br>

**DeliveryClient**

```kotlin
@Component
class DeliveryClient(private val deliveryClient: WebClient) {
    fun getDeliveryStatus(orderId: Long): Mono<DeliveryStatusResponse> {
        return webClient.get()
            .uri("/api/deliveries/$orderId/status")
            .retrieve()
            .bodyToMono(DeliveryStatusResponse::class.java)
    }
}
```

- 배송 서버에서 주문의 배송 상태를 조회합니다.
- 응답을 `Mono<DeliveryStatusResponse>` 타입으로 반환합니다.
- DeliveryStatusResponse 응답 예시
    
    ```json
    {
      "deliveryStatus": "IN_TRANSIT"
    }
    ```
    

<br>

이처럼 WebFlux 기반의 구조에서는 **Reactor를 사용하여 Mono, Flux 타입으로 데이터를 반환해야 하며, 명령형 프로그래밍이 아닌 선언형 프로그래밍 방식**으로 코드를 작성해야 합니다.

이는 기존 **Spring MVC 방식에 익숙한 개발자들에게 다소 낯선 코드 스타일**로 느껴질 수 있습니다.

즉, WebFlux를 사용하면 **적은 리소스로 많은 요청을 효율적으로 처리할 수 있는 장점**이 있지만, 반대로 **Reactor 기반의 비동기 코드 스타일에 대한 학습이 필요하다는 단점**도 존재합니다.

특히, 요구사항이 점점 복잡해질수록 이러한 단점이 더 부각될 수 있습니다

<br>

## WebFlux와 코루틴의 조합

Spring WebFlux 5.2 이상부터 **코틀린의 코루틴을 Reactive Streams에 대응하여 사용할 수 있도록 지원**합니다.

즉, 앞에서 **Reactor를 사용해 작성했던 비동기 코드를, 코루틴을 활용하여 더 익숙한 스타일의 코드로 변환할 수 있습니다.**

이를 통해 기존 Spring MVC와 유사한 코드 스타일을 유지하면서도 WebFlux의 Reactive Stack을 활용할 수 있습니다.

### **종속성 추가**

```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor")
}
```

- **kotlinx-coroutines-core**
    - 코틀린 코루틴의 기본 기능을 제공합니다.
- **kotlinx-coroutines-reactor**
    - 코틀린 코루틴과 Reactor를 연동할 수 있는 기능을 제공합니다.
    - ex) `Mono<T>` → `suspend fun` 변환, `Flux<T>` → `Flow<T>` 변환

<br>

### **Controller 계층**

**OrderController**

```kotlin
@RestController
@RequestMapping("/orders")
class OrderController(
    private val orderService: OrderService
) {
    @GetMapping("/{orderId}")
    suspend fun getOrderInfo(@PathVariable orderId: Long): OrderInfoResponse {
        return orderService.getOrderDetails(orderId)
    }
}
```

- `Mono<T>`를 반환하는 함수는 `suspend` 함수로 변환하여 코루틴을 사용할 수 있습니다.

<br>

### **Service 계층**

**OrderService**

```kotlin
@Service
class OrderService(
    private val orderClient: OrderClient,
    private val paymentClient: PaymentClient,
    private val deliveryClient: DeliveryClient
) {
    suspend fun getOrderDetails(orderId: Long): OrderInfoResponse = coroutineScope {
        val orderDeferred = async { orderClient.getOrder(orderId) }
        val paymentDeferred = async { paymentClient.getPaymentStatus(orderId) }
        val deliveryDeferred = async { deliveryClient.getDeliveryStatus(orderId) }

        val order = orderDeferred.await()
        val payment = paymentDeferred.await()
        val delivery = deliveryDeferred.await()

        OrderInfoResponse(
            orderId = order.orderId,
            userId = order.userId,
            orderItems = order.orderItems,
            paymentStatus = payment.paymentStatus,
            deliveryStatus = delivery.deliveryStatus
        )
    }
}
```

1. `Mono<T>`를 반환하는 함수를 `suspend` 함수로 변환하여 코루틴을 사용합니다.
- `async`를 사용하여 비동기적으로 병렬 요청을 보냅니다.
- `coroutineScope` 내에서 실행하여 구조적 동시성을 보장합니다.
    - 하나의 코루틴 Scope 안에서 실행되기 때문에, 하나라도 실패하면 다른 비동기 작업들이 취소됩니다.
- `await()`를 사용하여 각 비동기 호출이 완료될 때까지 기다립니다.
- 모든 요청이 완료된 후, 받은 데이터를 조합하여 반환합니다.

<br>

### **Client 계층**

- `Mono<T>`를 반환하는 함수를 `suspend` 함수로 변환하여 코루틴을 사용합니다.
- `awaitSingle()`을 사용하여 WebClient 응답을 `suspend`로 변환하여, WebFlux의 비동기 호출을 코루틴 방식으로 처리합니다.

<br>

**OrderClient**

```kotlin
@Component
class OrderClient(private val orderClient: WebClient) {
    suspend fun getOrder(orderId: Long): OrderResponse {
        return orderClient.get()
            .uri("api/orders/$orderId")
            .retrieve()
            .bodyToMono(OrderResponse::class.java)
            .awaitSingle()
    }
}
```

<br>

**PaymentClient**

```kotlin
@Component
class PaymentClient(private val paymentClient: WebClient) {
    suspend fun getPaymentStatus(orderId: Long): PaymentStatusResponse {
        return paymentClient.get()
            .uri("api/payments/$orderId/status")
            .retrieve()
            .bodyToMono(PaymentStatusResponse::class.java)
            .awaitSingle()
    }
}
```

<br>

**DeliveryClient**

```kotlin
@Component
class DeliveryClient(private val deliveryClient: WebClient) {
    suspend fun getDeliveryStatus(orderId: Long): DeliveryStatusResponse {
        return deliveryClient.get()
            .uri("api/deliveries/$orderId/status")
            .retrieve()
            .bodyToMono(DeliveryStatusResponse::class.java)
            .awaitSingle()
    }
}
```

<br>

### **코루틴을 활용한 WebFlux**

이전에 Reactor를 사용한 WebFlux 코드와 비교하여, 코루틴을 사용한 방식은 상대적으로 더 익숙한 코드 구조로 변환되었습니다. 이를 통해 Spring MVC의 코드 스타일과 비슷한 방식으로 작성하지만, 내부적으로는 리액티브 스트림으로 동작할 수 있게 되었습니다.

하지만 코루틴을 적용할 때는 중요한 점이 있습니다. 연계되는 모든 코드가 **Non-Blocking 방식**으로 동작할 수 있도록 보장해야 합니다. 중간에 **Blocking 코드**가 포함되면, 전체적인 처리 흐름이 예기치 않게 **Blocking**으로 동작할 수 있습니다. 따라서 **Blocking 코드의 존재 여부**를 철저히 점검하고, 이를 적절히 처리하는 것이 중요합니다.

<br>

## Reactor vs Coroutine

지금까지 리액티브 스트림를 사용하는데, Reactor를 이용한 방법과 코루틴을 이용한 방법을 비교해 봤습니다. 그렇다면 각 방법을 어떠한 경우에 사용하면 좋을지 장점과 단점을 비교해 보겠습니다.

<br>

### **Reactor의 장점**

1. **Reactor의 고급 기능 (배압, 시간 연산자)**
    - Reactor는 **배압(backpressure)** 처리를 위한 기능을 제공합니다. 이는 대량의 데이터를 효율적으로 처리하는 데 매우 유용합니다. 예를 들어, 이벤트의 양이 많거나, 데이터를 과도하게 요청하는 경우에 적절하게 제어할 수 있습니다.
    - **시간 연산자**에 대해서도 풍부한 기능을 제공하여, 일정 시간 동안 신호를 모은 후 한 번에 처리하는 방식 등 다양한 고급 처리가 가능합니다. 이러한 기능은 대량의 이벤트를 다룰 때 유리하게 작용합니다.
2. **대량의 이벤트 처리에 최적화**
    - Reactor는 대량의 이벤트를 **효율적으로 수신하고 처리**하는 데 적합합니다. 예를 들어, 10초 동안 또는 1000개의 이벤트가 모였을 때 한 번에 처리하는 방식(`bufferTimeout`)을 사용하면, 많은 요청을 효율적으로 처리할 수 있습니다.

### **Reactor의 단점**

1. **코드 복잡성**
    - Reactor는 비동기 프로그래밍을 위해 **복잡한 코드 구조**를 요구합니다. `Mono`, `Flux`와 같은 타입을 사용하고, 명령형이 아닌 선언형 방식으로 코드를 작성해야 하기 때문에 익숙하지 않은 개발자에게는 **학습 곡선**이 있을 수 있습니다.

<br>

### **Coroutine의 장점**

1. **단순한 코드 구조**
    - **Coroutine**을 사용하면 비동기 처리가 필요해도, **동기적 코드 스타일**을 유지할 수 있어 코드가 훨씬 **간결하고 읽기 쉬운** 구조를 가질 수 있습니다. Reactor에서 발생할 수 있는 복잡한 선언형 코드 대신, 코루틴을 사용하면 `suspend` 함수를 이용하여 명령형 스타일로 작성할 수 있습니다.
2. **개발자 친화적**
    - 코틀린 코루틴은 **Spring MVC와 유사한 코드 스타일**을 사용할 수 있게 해 주기 때문에, 기존에 동기 방식으로 작업하던 개발자들이 더 쉽게 **리액티브 프로그래밍**을 도입할 수 있게 합니다.

### **Coroutine의 단점**

1. **배압 처리 불가**
    - 코루틴의 **Flow**는 시그널을 전달하고 처리하지만, 단순 푸시 방식으로 동작하여 **배압(backpressure)** 처리가 불가능합니다.
2. **고급 연산자 부족**
    - **시간 연산자**나 고급 **비동기 처리** 기능에 있어서는 **Reactor**가 더 풍부한 기능을 제공하기 때문에, 복잡한 데이터 흐름 처리나 시간 지연 처리가 필요한 경우에는 Coroutine보다는 Reactor가 유리할 수 있습니다.
3. **제한적인 사용처**
    - **Coroutine**은 주로 단일 요청/결과 반환을 처리하는 데 적합하며, **대량의 이벤트 처리**나 **시간 요소**가 중요한 작업에서는 **Reactor**가 더 좋은 선택이 될 수 있습니다.

<br>

### **정리**
- **Reactor**는 **배압** 처리와 **고급 비동기 연산자**가 필요한 경우, 대량의 이벤트 처리에서 뛰어난 성능을 발휘합니다. 그러나 코드 구조가 복잡하고, 배압이 필요하지 않은 경우 과도한 복잡성을 초래할 수 있습니다.
- **Coroutine**은 더 간단하고 직관적인 코드 구조를 제공하지만, 대량의 이벤트 처리나 배압이 필요한 상황에서는 적합하지 않습니다. 단일 요청에 단일 결과를 반환하는 경우에는 더 효율적이고 개발자 친화적입니다.
- 따라서 **Reactor**와 **Coroutine**은 상황에 따라 각각 장점이 있기 때문에, 사용하는 시스템의 요구 사항에 맞춰 적절한 선택을 할 필요가 있습니다.

<br>

---

황정식, ⌜스프링으로 시작하는 리액티브 프로그래밍⌟, 비제이퍼블릭, 2023, p.39-40 p.117-124 p.419-422, p.498

우아한 기술블로그, “배민광고리스팅 개발기(feat. 코프링과 DSL 그리고 코루틴)”, [https://techblog.woowahan.com/7349/](https://techblog.woowahan.com/7349/), (참고 날짜 2025.03.03)

Spring blog, “Going Reactive with Spring, Coroutines and Kotlin Flow”, [https://spring.io/blog/2019/04/12/going-reactive-with-spring-coroutines-and-kotlin-flow](https://spring.io/blog/2019/04/12/going-reactive-with-spring-coroutines-and-kotlin-flow), (참고 날짜 2025.03.03)

kakaopay tech, “WebFlux와 코루틴으로 BFF(Backend For Frontend) 구현하기”, [https://tech.kakaopay.com/post/bff_webflux_coroutine/](https://tech.kakaopay.com/post/bff_webflux_coroutine/), (참고 날짜 2025.03.03)

Spring docs, “Web on Reactive Stack”, [https://docs.spring.io/spring-framework/reference/web-reactive.html](https://docs.spring.io/spring-framework/reference/web-reactive.html), (참고 날짜 2025.03.03)

티스토리, “Spring reactive Stack VS Servlet Stack”, [https://codenme.tistory.com/122](https://codenme.tistory.com/122), (참고 날짜 2025.03.03)

폭간의 기술블로그, “Web on Reactive Stack(Spring WebFlux 개요)”, [https://sejoung.github.io/2019/04/2019-04-02-spring-web-reactive1/](https://sejoung.github.io/2019/04/2019-04-02-spring-web-reactive1/), (참고 날짜 2025.03.03)