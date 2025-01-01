---
title: "DB 라우팅이 안되는 이유! 원인은 OSIV?"
date: 2024-12-22 00:00:00 +0900
categories: [backend, spring]
tags: [spring, jpa, db, osiv]
---

## 문제 상황

많은 개발자들이 DB에 접근할 때, **Replication**을 이용하여 DB를 수직적인 구조로 나누어 분리하고, **읽기 연산만 하는 트랜잭션에는 DB 복제본인 읽기 전용 DB를 접근**하도록하고, **쓰기 연산이 필요한 트랜잭션 로직에만 쓰기 전용 DB를 접근**하도록 하여, DB 접근의 부하를 분산하는 방법을 사용하고 있습니다.

저 또한 마찬가지로 라우팅을 통하여,  `readOnly = true` 설정인 트랜잭션은 읽기 전용 DB, 그 외 트랜잭션은 쓰기 전용 DB로 라우팅하여 부하를 분산하고 있었습니다.

스프링 부트를 사용하고, JPA를 이용해 데이터베이스를 다루고 있었으며, 데이터 정합성을 위해 읽기 전용 DB에는 직접적인 쓰기 연산이 불가능하도록 트리거를 통해 제한을 걸어 둔 상황입니다.

그런데, 개발을 진행하던 중 **쓰기 작업이 이루어지지 않는다는 것**을 발견했습니다.

<br>

## 문제가 발생한 로직

문제가 발생한 로직을 조금 더 명확하게 설명하기 위해, 예시 상황으로 설명을 진행해 보겠습니다.

> “*이벤트를 진행하여 회원에게 포인트를 지급하는데, 이때 20세 이상인 회원만 포인트를 받을 수 있다*”
> 

그렇다면 전체적인 로직은 다음과 같이 생각할 수 있습니다.

1. 회원 번호와, 지급할 포인트 금액을 받는다.
2. 회원 번호로 회원의 정보를 조회해서 회원이 20세 이상인지 확인한다.
    - 이때 만약, 회원이 20세 미만이라면 이후 로직은 진행할 수 없다.
3. 회원에게 포인트를 지급한다.

이와 같은 로직을 수행하는 API를 **서비스 계층**과 **컨트롤러 계층**으로 나누어 보겠습니다.

<br>

### 서비스 계층

**UserService**

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository
) {
    @Transactional(readOnly = true)
    fun getUserById(id: Long): User {
        return userRepository.findById(id).orElseThrow {
            UserNotFoundException("Failed to find user by id (id=$id)")
        }
    }
}
```

- 회원 관련 로직을 담당하는 서비스 계층입니다.
- `getUserById`는 회원 번호로 회원 정보를 조회하며, readOnly 설정이 되어 있으므로, **읽기 전용 DB로 라우팅이 됩니다.**

<br>

**PointService**

```kotlin
@Service
class PointService(
    private val pointRepository: PointRepository
) {
    @Transactional
    fun savePoint(userId: Long, amount: Long): Point {
        val point = Point(userId = userId, amount = amount)
        return pointRepository.save(point)
    }
}
```

- 포인트 관련 로직을 담당하는 서비스 계층입니다.
- `savePoint`는 회원에게 포인트를 지급하며, readOnly 설정이 없으므로, **쓰기 전용 DB로 라우팅이 됩니다.**

<br>

**EventService**

```kotlin
@Service
class EventService (
    private val userService: UserService,
    private val pointService: PointService
) {
    fun givePoint(userId: Long, amount: Long): PointResponse {
        // 회원 조회
        val user = userService.getUserById(id = userId)

        // 나이 제한 확인
        if (user.age < 20) {
            throw NotEligibleForEventException("User under the age of 20 is not eligible for this event.")
        }

        // 포인트 적재
        val savedPoint = pointService.savePoint(userId = userId, amount = amount)
        return PointResponse.fromEntity(savedPoint)
    }
```

- 이벤트 관련 로직을 담당하는 서비스 계층입니다.
- 회원에게 포인트를 지급하는 이벤트 로직은 다음과 같습니다.
    1. 회원 정보를 조회한다.
    2. 조회한 회원의 나이가 20세 이상인지 검사한다.
    3. 회원에게 포인트를 부여한다.

<br>

### 컨트롤러 계층

**EventController**

```kotlin
@RestController
@RequestMapping("/api/v1/events")
class EventController(
    private val eventService: EventService
) {

    @PostMapping("/users/{userId}")
    fun givePoint(
        @PathVariable userId: Long,
        @RequestBody eventRequest: EventRequest
    ): ResponseEntity<PointResponse> {
        val pointResponse = eventService.givePoint(
            userId = userId, amount = eventRequest.amount
        )
        return ResponseEntity.ok(pointResponse)
    }
}
```

- 이벤트를 담당하는 컨트롤러 계층입니다.

현재 회원 테이블에는 다음과 같이 저장되어 있습니다.

| **id** | **name** | **age** |
| ------ | -------- | ------- |
| 1      | jude     | 25      |

회원번호 1번인 jude 회원은 나이가 25세라서 이벤트를 진행한다면, 나이 제한에 걸리지 않고 의도한 대로 포인트가 지급이 되어야 합니다. 그렇다면, 위 API를 호출하여 회원에게 포인트 지급이 잘 되는지 확인해 보겠습니다.

- **URI**: `api/v1/events/users/1`
- **HTTP method**: `POST`
- **request**
    
    ```json
    {
        "amount": 100
    }
    ```

<br>

### 그런데?!

```bash
java.sql.SQLException: INSERT operations are not allowed on this database
```

다음과 같이 읽기 전용 DB에 쓰기 작업이 이루어져, INSERT 작업을 막는 트리거가 동작한 것을 확인했습니다.

여기서, 회원의 포인트를 적재하는 로직이 읽기 전용 DB에서 수행되었다는 것을 확인할 수 있으며, 라우팅이 의도한 대로 잘 수행되지 않았다는 것을 알 수 있습니다.

<br>

## 원인은 OSIV!

문제 원인은 JPA의 **OSIV**(Open Session In View)에 있다는 것을 알 수 있었습니다. 

OSIV는 **영속성 컨텍스트를 뷰까지 열어둔다는 의미**입니다. 즉, 뷰까지 엔티티를 영속 상태로 유지하여 뷰에서도 지연 로딩이 가능하게 할 수 있습니다. 그렇다면 스프링은 OSIV를 어떠한 방식으로 사용하고 있을까요?

### 스프링의 OSIV: 비즈니스 계층에서 트랜잭션을 사용하는 OSIV

스프링 프레임워크에서 제공하는 OSIV는 **비즈니스 계층에서 트랜잭션을 사용하는 OSIV** 입니다. 즉, OSIV를 사용하지만 트랜잭션은 비즈니스 계층에서만 사용한다는 뜻입니다.

**스프링 OSIV: 비즈니스 계층 트랜잭션**
![스프링 OSIV: 비즈니스 계층 트랜잭션](/assets/img/posts/spring-osiv-business-layer-transaction.png)

1. 클라이언트의 요청이 들어오면 서블릿 필터나, 스프링 인터셉터에서 영속성 컨텍스트를 생성하지만, **이때 트랜잭션은 시작하지 않는다.**
2. 서비스 계층에서 `@Transactional` 로 트랜잭션을 시작할 때 1번에서 미리 생성해둔 영속성 컨텍스트를 찾아와서 트랜잭션을 시작한다.
3. 서비스 계층이 끝나면 트랜잭션을 커밋하고 영속성 컨텍스트를 플러시 한다. **이때 트랜잭션은 끝내지만 영속성 컨텍스트는 종료하지 않는다.**
4. 컨트롤러와 뷰까지 영속성 컨텍스트가 유지되므로 **조회한 엔티티는 영속 상태를 유지한다.** 
    - 즉, 이때 **트랜잭션은 없으므로 엔티티를 수정할 수는 없지만, 영속성 컨텍스트는 유지되므로 지연 로딩 기능은 사용할 수 있다.**
5. 서블릿 필터나, 스프링 인터셉터로 **요청이 돌아오면 영속성 컨텍스트를 종료한다.** 이때 플러시를 호출하지 않고 바로 종료한다.

<br>

### 영속성 컨텍스트가 획득한 DB 커넥션

그렇다면 어떠한 특징 때문에, 위와 같은 문제의 원인이 OSIV라고 말했던 것일까요? 그것은 바로 **OSIV에서 영속성 컨텍스트가 획득한 DB 커넥션** 때문입니다. 

영속성 컨텍스트는 `EntityManager`를 생성할 때마다 하나가 생성되며, 이 `EntityManager`는 **DB 연결이 필요한 시점에 DataSource를 사용하여 DB 커넥션 풀에서 DB 커넥션을 가져옵니다.**

그런데, 앞서 **OSIV에서 영속성 컨텍스트는 뷰까지 유지**가 된다고 했습니다. 이로 인해, EntityManager는 획득한 DB 커넥션을 계속 유지하게 되는겁니다. 

즉, 위 문제 상황에서 **회원 정보를 조회하기 위해 사용했던 읽기 전용 DB의 커넥션을 회원의 포인트를 적재하는 로직에서도 사용하여 문제가 된것**이었습니다.

**스프링 OSIV에서 같은 DB 커넥션을 사용하게 되는 상황**
![스프링 OSIV에서 같은 DB 커넥션을 사용하게 되는 상황](/assets/img/posts/spring-osiv-same-db-connection.png)

- 회원 정보를 조회하는 `UserService`의 `getUserById`를 사용할 때와, 회원의 포인트를 적재하는 `PointService`의 `savePoint`를 사용할 때 **모두 같은 영속성 컨텍스트를 공유하게 되므로, DB 커넥션도 공유하게 됩니다.**
- 즉, `UserService`의 `getUserById`를 사용할 때 획득한 읽기 전용 DB 커넥션을 `PointService`의 `savePoint`를 수행할 때도 사용하게 되어, **읽기 전용 DB에 쓰기 작업이 수행되어 문제가 발생했던것 입니다.**

<br>

### 해결 방법

이를 해결하는 가장 간단한 방법은 **OSIV를 비활성화하는 것**입니다. OSIV를 비활성화하면, **영속성 컨텍스트의 생명주기는 트랜잭션 시작부터 종료까지로 좁혀지게 됩니다.** 이를 통해 `@Transactional` 가 설정된 로직을 수행할 때마다, **영속성 컨텍스트가 새롭게 생성되어 DB 커넥션도 함께 새로 획득하게 됩니다.**

**OSIV를 비활성화한 application.yaml 파일**

```yaml
spring:
  jpa:
    open-in-view: false
```

<br>

**OSIV를 비활성화했을 때 영속성 컨텍스트 생존 범위**
![OSIV를 비활성화했을 때 영속성 컨텍스트 생존 범위](/assets/img/posts/spring-osiv-false-persistence-context-scope.png)

<br>

**서비스 로직 별로 다른 DB 커넥션을 사용하게 되는 상황**
![서비스 로직 별로 다른 DB 커넥션을 사용하게 되는 상황](/assets/img/posts/spring-osiv-different-db-connection.png)

- 회원 정보를 조회하는 `UserService`의 `getUserById`를 사용할 때와, 회원의 포인트를 적재하는 `PointService`의 `savePoint`를 사용할 때 **모두 각각 다른 영속성 컨텍스트를 생성하여 사용하므로, DB 커넥션 또한 각각 획득하여 사용합니다.**
- 즉, `UserService`의 `getUserById`를 수행할 때는 읽기 전용 DB 커넥션,  `PointService`의 `savePoint`를 수행할 때는 쓰기 전용 DB 커넥션을 사용하여, 문제없이 로직이 수행될 수 있습니다.

<br>

### 어? 나는 OSIV를 설정하지 않았는데?

**OSIV의 기본 설정은 true**로, OSIV를 따로 설정하지 않았더라도, OSIV가 동작하여 위와 같은 문제가 발생할 수 있습니다.

## 정리
- DB 라우팅이 잘되지 않는 문제 상황을 확인하며 OSIV 동작 방식을 배울 수 있는 계기가 되었습니다.
- OSIV가 활성화된 상태라면, 의도한 대로 동작하고 있는지 확인해 볼 필요가 있습니다.
  - OSIV 기본 설정은 활성 상태이므로 OSIV가 필요하지 않은 상황이라면, 의도치 않은 동작을 제어하기 위해 비활성화하는 게 좋다고 생각합니다.

<br>

---

김영한, ⌜자바 ORM 표준 JPA 프로그래밍⌟, 에이콘출판사, 2022, p.593-607

hudi.blog, “대체 왜 DataSource 라우팅이 안되는거야!? (feat. OSIV)”, [https://hudi.blog/multi-datasource-issue-with-osiv/](https://hudi.blog/multi-datasource-issue-with-osiv/), (참고 날짜 2024.12.22)

haon.blog, “DB 레플리케이션 환경에서 DataSource 라우팅이 안되는 이슈 해결기 😤 (feat. JPA OSIV)”, [https://haon.blog/database/replication-osiv-issue/](https://haon.blog/database/replication-osiv-issue/), (참고 날짜 2024.12.22)

티스토리, “DataSource 라우팅이 안되는 이유. OSIV”, [https://eastc.tistory.com/entry/DataSource-라우팅이-안되는-이유-OSIV](https://eastc.tistory.com/entry/DataSource-라우팅이-안되는-이유-OSIV), (참고 날짜 2024.12.22)