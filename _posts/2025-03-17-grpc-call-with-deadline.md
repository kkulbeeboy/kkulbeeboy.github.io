---
title: "gRPC 호출 시, 무한 대기에 빠지지 않도록 Deadline을 설정하자"
date: 2025-03-17 00:00:00 +0900
categories: [backend, gRPC]
tags: [gRPC, deadline]
---

## gRPC 란?

gRPC는 **RPC(Remote Procedure Call) 모델**을 기반으로 한 통신 기술로, 클라이언트가 원격 서버의 프로시저를 **마치 로컬에서 호출하듯이** 실행할 수 있도록 해줍니다.

gRPC는 **HTTP/2**를 기반으로 동작하며, 데이터를 **ProtoBuf(Protocol Buffers)** 형식으로 직렬화/역직렬화하여 **효율적인 네트워크 전송**을 지원합니다. 이로 인해 성능상의 이점을 가지며, 특히 **마이크로서비스 아키텍처(MSA)** 환경에서 서버 간 통신을 하는 데 효과적입니다.

## Deadline

gRPC에서 **Deadline**을 통해 클라이언트가 **서버 응답을 언제까지 기다릴 것인지** 지정할 수 있습니다. 이를 활용하면, 서버 이상으로 인해 응답이 예상보다 지연되는 경우에도 **클라이언트가 적절한 시점에 요청을 포기하여 리소스를 효과적으로 관리**할 수 있습니다.

클라이언트가 설정한 Deadline을 초과하면 클라이언트는 요청을 포기하고, 요청은 `DEADLINE_EXCEEDED` 오류와 함께 실패 처리됩니다. 서버 측에서는 클라이언트의 Deadline을 인식하고, 이를 초과한 요청을 자동으로 `CANCELLED` 상태로 처리합니다. 

따라서 서버가 오랫동안 작업을 수행하는 경우, 주기적으로 요청이 취소되었는지 확인하고 필요 시 프로세스를 중단해야 합니다.

## 기본적으로 gRPC는 Deadline을 설정하지 않는다?

기본적으로 gRPC는 **Deadline을 설정하지 않으므로**, 클라이언트가 응답을 **무한정 기다릴 수 있습니다.** 따라서 이를 방지하려면 **클라이언트에서 항상 명시적으로 Deadline을 설정하는 것이 중요합니다.**

적절한 Deadline 값은 gRPC 서버의 평균 응답 시간, 부하 테스트 결과, 서비스 특성 등을 고려하여 설정할 수 있습니다.

## Deadline을 설정하지 않았을 때 발생할 수 있는 상황

Deadline을 설정하지 않았을때 어떠한 문제가 발생할 수 있는지 예시 상황을 통해 알아보겠습니다.

클라이언트와 서버가 **단방향 RPC(Unary RPC)** 로 통신하는 상황에서, 서버가 응답을 반환하지 못하고 **무한 루프에 빠진 경우**를 가정해 보겠습니다.

- RPC 통신 방법 종류
    - 단방향 RPC (Unary RPC)
        - 하나의 요청에 대해 하나의 응답을 반환하는 방식
    - 서버 스트리밍 RPC (Server Streaming RPC)
        - 하나의 요청에 대해 여러 개의 응답을 스트림 방식으로 전달하는 방식
    - 클라이언트 스트리밍 RPC (Client Streaming RPC)
        - 클라이언트가 여러 개의 요청을 스트림 방식으로 보내고, 서버가 이를 처리한 후 하나의 응답을 반환하는 방식
    - 양방향 스트리밍 RPC (Bidirectional Streaming RPC)
        - 클라이언트와 서버가 각각 여러 개의 메시지를 스트림 방식으로 주고받는 방식
    

> **포인트 서버**가 회원의 멤버십 조회를 위해 **회원 서버**(gRPC 서버)로 요청을 보냈지만, 회원 서버의 문제로 무한 루프에 빠진 상황
> 

![image.png](/assets/img/posts/grpc-deadline-exceeded.png)

### gRPC 서버 (회원 서버)

```kotlin
@GrpcService
class MembershipServiceGrpcImpl(
    private val membershipService: MembershipService
) : MembershipServiceGrpcKt.MembershipServiceCoroutineImplBase() {

    override suspend fun getMembershipLevel(request: MembershipRequest): MembershipResponse {
        val memberId = request.memberId
        
        // 무한 루프에 빠진 로직
        val memberShipLevel = membershipService.getMembershipLevel(memberId = memberId)

        return MembershipResponse.newBuilder()
            .setMembershipLevel(memberShipLevel)
            .build()
    }
}
```

- `memberId`를 요청받아 해당 회원의 **멤버십 레벨을 응답**하는 gRPC 서버입니다.
- 하지만, `membershipService.getMembershipLevel()` 내부에서 문제가 발생하여 **무한 루프에 빠진 상황**을 가정합니다.

### gRPC 클라이언트 (포인트 서버): Deadline을 설정하지 않았을 때

```kotlin
@Service
class MemberServerClient {
    @GrpcClient("member-server")
    private lateinit var membershipStub: MembershipServiceGrpcKt.MembershipServiceCoroutineStub

    suspend fun getMembershipLevel(memberId: Long): String {
        val membershipRequest = MembershipRequest.newBuilder().setMemberId(memberId).build()
        
        logger.info("gRPC 서버 호출 시작. memberId: $memberId")
        
        val membershipResponse = membershipStub.getMembershipLevel(membershipRequest)
        
        logger.info("gRPC 서버 응답 완료. memberId: $memberId")
        
        return membershipResponse.membershipLevel
    }
}
```

- **포인트 서버에서 회원 서버와 통신하기 위한 gRPC 클라이언트**입니다.
- 회원 번호를 요청값으로 보내 **멤버십 레벨을 조회**합니다.
- gRPC 서버 호출 전후로 로깅을 남깁니다.
- 요청에 대한 deadline이 설정되어 있지 않습니다.

### Deadline을 설정하지 않은 클라이언트 호출 결과

```bash
gRPC 서버 호출 시작. memberId: 1000
```

- gRPC 서버 호출 후 응답에 대한 로깅이 남지 않았습니다.
- 즉, 클라이언트가 연결을 유지한채 **계속 대기하고 있는것**을 확인할 수 있습니다.

### 문제가 된 상황

위와 같은 문제는 **gRPC 서버에서 응답을 내려주지 못하고 무한루프에 빠져, 클라이언트가 무한 대기 상태에 빠진것**입니다. 

이와 유사한 문제는 서버 스트리밍 RPC에서 서버가 명시적으로 응답 종료(ex `onCompleted()`, `onError()`)를 알리지 않아 연결을 유지하고 있는 경우 등등 다양한 경우에서 클라이언트가 무한정으로 대기하는 상황이 발생할 수 있습니다.

이러한 문제를 방지하기 위해 **클라이언트에서 반드시 Deadline을 설정하는 것이 중요합니다.**

### gRPC 클라이언트 (포인트 서버): Deadline을 설정했을 때

```kotlin
@Service
class MemberServerClient {
    @GrpcClient("member-server")
    private lateinit var membershipStub: MembershipServiceGrpcKt.MembershipServiceCoroutineStub

    suspend fun getMembershipLevel(memberId: Long): String {
        val membershipRequest = MembershipRequest.newBuilder().setMemberId(memberId).build()
        
        logger.info("gRPC 서버 호출 시작. memberId: $memberId")
        
        val membershipResponse = try {
            membershipStub.withDeadlineAfter(5, TimeUnit.SECONDS).getMembershipLevel(membershipRequest)
        } catch (e: StatusException) {
            logger.info("StatusException message: ${e.message}")
            throw e
        
        }
        
        logger.info("gRPC 서버 응답 완료. memberId: $memberId")
        
        return membershipResponse.membershipLevel
    }
}
```

- `withDeadlineAfter(5, TimeUnit.SECONDS)`를 사용하여 **gRPC 요청의 Deadline을 5초로 설정**하였습니다.
- 만약 **서버가 5초 이내에 응답하지 못하면**, `StatusException`이 발생하여 요청이 실패하게 됩니다.

### Deadline을 설정한 클라이언트 호출 결과

```bash
gRPC 서버 호출 시작. memberId: 1000
StatusException message: DEADLINE_EXCEEDED: deadline exceeded after 4.917026917s. [closed=[], open=[[remote_addr=...]]]
```

- **Deadline이 설정되어 있기 때문에**, 5초 내에 응답을 받지 못해, `DEADLINE_EXCEEDED` 예외가 발생했습니다.
- 이로 인해 클라이언트는 **무한 대기 상태에 빠지지 않고, 요청을 적절히 종료**할 수 있습니다.

## Deadline 전파

클라이언트가 요청을 보낸 **gRPC 서버**가 다른 **gRPC 서버**를 호출하는 경우, 원래 클라이언트가 설정한 **Deadline**을 준수해야 합니다. 이때 **Deadline 전파**가 필요하며, 이를 통해 하위 서버들도 적절한 시간 내에 응답을 반환해야 합니다.

**Deadline 전파**는 일부 언어(ex: C++)에서는 명시적으로 설정해야 하고, 다른 일부 언어(ex: Java, Go)에서는 **기본적으로 활성화**되어 있기도 합니다.

Deadline이 전파되는 경우, 이미 경과된 시간은 **차감되어 전파**됩니다. 즉, 클라이언트가 설정한 5초의 Deadline이 있으면, 서버가 다른 서버와 통신하는 데 소요된 시간만큼 Deadline 시간이 줄어듭니다.

### 예시 상황

회원 서버가 **회원의 멤버십을 계산하기 위해 주문 서버**와 gRPC 통신을 하여 **주문 횟수**를 조회하는 상황을 가정해 보겠습니다.

- 포인트 서버가 회원 서버와 통신할 때 **Deadline은 5초**로 설정됩니다.
- 회원 서버가 3초의 지연을 겪은 후, 주문 서버와 통신하여 주문 횟수를 조회합니다.

![image.png](/assets/img/posts/grpc-deadline-delivery.png)

### gRPC 클라이언트 (포인트 서버)

```kotlin
@Service
class MemberServerClient {
    @GrpcClient("member-server")
    private lateinit var membershipStub: MembershipServiceGrpcKt.MembershipServiceCoroutineStub

    suspend fun getMembershipLevel(memberId: Long): String {
        val membershipRequest = MembershipRequest.newBuilder().setMemberId(memberId).build()
        
        logger.info("gRPC 서버(회원 서버) 호출 시작. memberId: $memberId")
        
        val membershipResponse = try {
            membershipStub.withDeadlineAfter(5, TimeUnit.SECONDS).getMembershipLevel(membershipRequest)
        } catch (e: StatusException) {
            logger.info("StatusException message: ${e.message}")
            throw e
        
        }
        
        logger.info("gRPC 서버(회원 서버) 응답 완료. memberId: $memberId")
        
        return membershipResponse.membershipLevel
    }
}
```

- **gRPC 클라이언트**에서 `withDeadlineAfter`를 사용하여 **Deadline을 5초**로 설정했습니다.
- 5초 내에 응답이 오지 않으면, `StatusException`이 발생하여 요청이 실패합니다.

### gRPC 서버 (회원 서버)

```kotlin
@GrpcService
class MembershipServiceGrpcImpl(
    private val membershipService: MembershipService
) : MembershipServiceGrpcKt.MembershipServiceCoroutineImplBase() {
    override suspend fun getMembershipLevel(request: MembershipRequest): MembershipResponse {
        logger.info("회원 서버가 받은 request deadline: ${Context.current().deadline}")
        val memberId = request.memberId
        
        // getMembershipLevel 내부에서 주문 서버와 통신을 진
        val memberShipLevel = membershipService.getMembershipLevel(memberId = memberId)

        return MembershipResponse.newBuilder()
            .setMembershipLevel(memberShipLevel)
            .build()
    }
}
```

- **회원 서버**는 요청을 받으면, 해당 요청의 **Deadline**을 로그로 남깁니다.
- `membershipService.getMembershipLevel`를 통해 회원의 멤버십 레벨을 조회합니다.

**MembershipService**

```kotlin
@Service
class MembershipService(
    private val orderServerClient: OrderServerClient
) {
    suspend fun getMembershipLevel(memberId: Long): String {
        val orderCount = orderServerClient.getOrderCount(memberId = memberId)
        return calculateMembershipLevel(orderCount = orderCount)
    }
}
```

- 주문 서버와 통신하여 회원의 **주문 횟수**를 조회하고, 이를 기반으로 **멤버십 레벨**을 계산합니다.

**OrderServerClient**

```kotlin
class OrderServerClient {
    @GrpcClient("order-server")
    private lateinit var orderServerStub: OrderServiceGrpcKt.OrderServiceCoroutineStub

    suspend fun getOrderCount(memberId: Long): Long {
        val orderCountRequest = OrderCountRequest.newBuilder().setMemberId(memberId).build()
        
        delay(3000) // 3초 지연
        
        logger.info("gRPC 서버(주문 서버) 호출 시작. memberId: $memberId")
        
        val orderCountResponse = try {
            orderServerStub.getOrderCount(orderCountRequest)
        } catch (e: StatusException) {
            logger.info("StatusException message: ${e.message}")
            throw e
        }
        
        logger.info("gRPC 서버(주문 서버) 응답 완료. memberId: $memberId")
        
        return orderCountResponse.orderCount
    }
}
```

- 주문 서버와 통신하여 **회원의 주문 횟수**를 조회합니다.
- **3초**의 지연 후에 주문 서버와 통신을 시도합니다.

### gRPC 서버 (주문 서버)

```kotlin
@GrpcService
class OrderServiceImpl(
    private val orderService: OrderService
) : OrderServiceGrpcKt.OrderServiceCoroutineImplBase() {
    override suspend fun getOrderCount(request: OrderCountRequest): OrderCountResponse {
        logger.info("주문 서버가 받은 request deadline: ${Context.current().deadline}")
        val memberId = request.memberId
        val orderCount = orderService.getOrderCount(memberId = memberId)
        return OrderCountResponse.newBuilder()
            .setOrderCount(orderCount)
            .build()
    }
}
```

- **주문 서버**는 받은 요청의 **Deadline**을 로그로 남깁니다.
- 주문 서버에서 **회원의 주문 횟수**를 조회한 후, 그 값을 응답합니다.

### 호출 결과

```bash
// 포인트 서버 로그
gRPC 서버(회원 서버) 호출 시작. memberId: 1000

// 회원 서버 로그
회원 서버가 받은 request deadline: 4.920526042s from now
gRPC 서버(주문 서버) 호출 시작. memberId: 1000

// 주문 서버 로그
주문 서버가 받은 request deadline: 1.881303875s from now

// 회원 서버 로그
gRPC 서버(주문 서버) 응답 완료. memberId: 1000

// 포인트 서버 로그
gRPC 서버(회원 서버) 응답 완료. memberId: 1000
```

- **회원 서버**에서 **주문 서버**를 호출하기 전에 **3초의 지연**이 있었기 때문에, **주문 서버**로 전달된 **Deadline**에서 이미 경과된 3초가 차감된 것을 확인할 수 있습니다.

### Deadline을 다시 설정한다면, 더 짧은 Deadline 시간이 적용된다.

gRPC에서 **Deadline은 전파**되며, 클라이언트가 설정한 Deadline은 서버로 전달됩니다. 그러나 서버가 다른 서버로 호출을 할 때, 그 전달된 Deadline을 그대로 사용하지 않고, 새로운 Deadline을 설정할 수도 있습니다. 이때, **새로 설정된 Deadline**이 **더 짧은 시간**으로 설정된다면, **더 짧은 Deadline이 우선 적용**됩니다.

위 예시에서 회원 서버가 받은 Deadline은 약 **5초**였지만, 만약 회원 서버에서 주문 서버를 호출할 때 Deadline을 **다시 설정**한다면, 회원 서버가 받은 Deadline 시간인 5초와, 다시 설정한 시간 중 **더 짧은 Deadline이 적용됩니다.**

![image.png](/assets/img/posts/apply-short-deadline.png)

<br>

**회원 서버가 받은 Deadline보다 짧은 Deadline을 설정하여 주문 서버를 호출할 때**

```kotlin
@Component
class OrderServerClient {
    @GrpcClient("order-server")
    private lateinit var orderServerStub: OrderServiceGrpcKt.OrderServiceCoroutineStub

    suspend fun getOrderCount(memberId: Long): Long {
        val orderCountRequest = OrderCountRequest.newBuilder().setMemberId(memberId).build()
        logger.info("gRPC 서버(주문 서버) 호출 시작. memberId: $memberId")
        val orderCountResponse = try {
            orderServerStub.withDeadlineAfter(2, TimeUnit.SECONDS).getOrderCount(orderCountRequest)
        } catch (e: StatusException) {
            logger.info("StatusException message: ${e.message}")
            throw e
        }
        logger.info("gRPC 서버(주문 서버) 응답 완료. memberId: $memberId")
        return orderCountResponse.orderCount
    }
}
```

- 회원 서버가 받은 Deadline 시간(5초) 보다 더 짧은 Deadline 시간(2초)을 설정했습니다.
- 실행 결과
    
    ```bash
    // 포인트 서버
    gRPC 서버(회원 서버) 호출 시작. memberId: 1000
    
    // 회원 서버
    회원 서버가 받은 request deadline: 4.916055500s from now
    gRPC 서버(주문 서버) 호출 시작. memberId: 1000
    
    // 주문 서버
    주문 서버가 받은 request deadline: 1.990590209s from now
    
    // 회원 서버
    gRPC 서버(주문 서버) 응답 완료. memberId: 1000
    
    // 포인트 서버
    gRPC 서버(회원 서버) 응답 완료. memberId: 1000
    ```
    
    - 주문 서버가 받은 deadline 시간은 회원 서버에서 더 짧게 설정한 2초(1.990590209s)인것으로 확인할 수 있습니다. 이는 **회원 서버가 받은 5초의 Deadline**보다 **짧은 Deadline**이 적용된 결과입니다.

<br>

**회원 서버가 받은 Deadline보다 긴 Deadline을 설정하여 주문 서버를 호출할 때**

```kotlin
@Component
class OrderServerClient {
    @GrpcClient("order-server")
    private lateinit var orderServerStub: OrderServiceGrpcKt.OrderServiceCoroutineStub

    suspend fun getOrderCount(memberId: Long): Long {
        val orderCountRequest = OrderCountRequest.newBuilder().setMemberId(memberId).build()
        logger.info("gRPC 서버(주문 서버) 호출 시작. memberId: $memberId")
        val orderCountResponse = try {
            orderServerStub.withDeadlineAfter(10, TimeUnit.SECONDS).getOrderCount(orderCountRequest)
        } catch (e: StatusException) {
            logger.info("StatusException message: ${e.message}")
            throw e
        }
        logger.info("gRPC 서버(주문 서버) 응답 완료. memberId: $memberId")
        return orderCountResponse.orderCount
    }
}
```

- 회원 서버가 받은 Deadline 시간(5초) 보다 더 긴 Deadline 시간(10초)을 설정했습니다.
- 실핼 결과
    
    ```bash
    // 포인트 서버
    gRPC 서버(회원 서버) 호출 시작. memberId: 1000
    
    // 회원 서버
    회원 서버가 받은 request deadline: 4.921051375s from now
    gRPC 서버(주문 서버) 호출 시작. memberId: 1000
    
    // 주문 서버
    주문 서버가 받은 request deadline: 4.907321500s from now
    
    // 회원 서버
    gRPC 서버(주문 서버) 응답 완료. memberId: 1000
    
    // 포인트 서버
    gRPC 서버(회원 서버) 응답 완료. memberId: 1000
    ```
    
    - **회원 서버**는 **5초**의 Deadline을 받았고, **주문 서버**를 호출할 때 **10초**로 새로운 Deadline을 설정했으나, **회원 서버가 받은 Deadline**이 더 짧으므로 **5초**가 최종 적용되었습니다.

## gRPC 서버에서 Deadline 상태 확인

gRPC 서버에서 클라이언트가 요청한 Deadline이 지나서 더 이상 유효하지 않거나, 클라이언트가 요청을 취소한 경우를 확인하는 것은 서버의 리소스를 절약하는 데 매우 중요합니다. 특히 리소스를 많이 소모하거나 비용이 큰 작업을 처리할 때, 더 이상 클라이언트가 응답을 기다리지 않으면 작업을 중단하거나 취소할 수 있습니다.

### gRPC 서버에서 이미 deadline이 지나서 취소되었는지 확인

```kotlin
if(Context.current().isCancelled){
    ...
}
```

클라이언트가 설정한 Deadline 시간이 지나서 서버가 응답을 못 하면 자동으로 `CANCELLED` 상태가 됩니다. 

서버가 비용이 큰 작업을 하기 전에, 이러한 상태를 확인하고 `CANCELLED` 상태라면 작업을 진행하지 않는 방식으로 리소스를 관리할 수 있습니다. 물론, `CANCELLED` 상태라고 해도 이후 동일 요청에 대해 같은 응답을 할 수 있다면, 작업을 우선 진행하고 캐시해 놓는 방법을 고려할 수도 있습니다.

## 정리

- gRPC 서버와 통신할 때, Deadline 설정은 필수적입니다
- gRPC 서버가 다른 서버와 통신할 때, Deadline을 전파할 수 있습니다.
- 전달된 Deadline을 그대로 사용하지 않고, 새로운 Deadline을 설정한다면, 전달된 Deadline과 새로운 Deadline 중 더 짧은 Deadline을 사용합니다.
- gRPC 서버는 Deadline 상태를 확인할 수 있습니다.

<br>

---

AWS, “gRPC와 REST의 차이점은 무엇인가요?”, [https://aws.amazon.com/ko/compare/the-difference-between-grpc-and-rest/](https://aws.amazon.com/ko/compare/the-difference-between-grpc-and-rest/), (참고 날짜 2025.03.17)

위키백과 “원격 프로시저 호출”, [https://ko.wikipedia.org/wiki/원격_프로시저_호출](https://ko.wikipedia.org/wiki/원격_프로시저_호출), (참고 날짜 2025.03.17)

위키백과 “gRPC”, [https://ko.wikipedia.org/wiki/GRPC](https://ko.wikipedia.org/wiki/GRPC), (참고 날짜 2025.03.17)

mircosoft, “gRPC 서비스와 HTTP API 비교”, [https://learn.microsoft.com/ko-kr/aspnet/core/grpc/comparison?view=aspnetcore-9.0](https://learn.microsoft.com/ko-kr/aspnet/core/grpc/comparison?view=aspnetcore-9.0), (참고 날짜 2025.03.17)

Eottabom’s Lab, “gRPC 를 도입하는 이유에 대해서 알아보자 (feat. HTTP 1.0 vs HTTP 1.1 vs HTTP 2.0)”, [https://eottabom.github.io/post/why-using-grpc/](https://eottabom.github.io/post/why-using-grpc/), (참고 날짜 2025.03.17)

gRPC docs, “Deadlines”, [https://grpc.io/docs/guides/deadlines/](https://grpc.io/docs/guides/deadlines/), (참고 날짜 2025.03.17)

gRPC blog, “gRPC and Deadlines”, [https://grpc.io/blog/deadlines/](https://grpc.io/blog/deadlines/), (참고 날짜 2025.03.17)

hodongman의 기술 블로그, “gRPC Stream”, [https://hodongman.github.io/2022/02/01/gRPC-stream.html](https://hodongman.github.io/2022/02/01/gRPC-stream.html), (참고 날짜 2025.03.17)

mircosoft, “최종 기한 및 취소를 사용하여 안정적인 gRPC 서비스”, [https://learn.microsoft.com/ko-kr/aspnet/core/grpc/deadlines-cancellation?view=aspnetcore-9.0](https://learn.microsoft.com/ko-kr/aspnet/core/grpc/deadlines-cancellation?view=aspnetcore-9.0), (참고 날짜 2025.03.17)