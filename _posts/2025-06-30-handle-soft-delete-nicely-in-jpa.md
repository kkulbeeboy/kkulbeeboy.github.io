---
title: "JPA에서 Soft Delete 멋지게 다루는법"
date: 2025-06-30 00:00:00 +0900
categories: [backend, spring]
tags: [spring, jpa, soft delete]
---

## Hard Delete vs Soft Delete

서비스를 운영하다 보면, 저장된 데이터를 삭제해야 하는 상황이 생기기 마련입니다. 이때 데이터를 삭제하는 방식은 크게 두 가지로 나뉩니다.

- **Hard Delete (물리적 삭제)**: `DELETE` 쿼리를 통해 데이터베이스에서 해당 데이터를 완전히 제거하는 방식입니다.
- **Soft Delete (논리적 삭제)**: 데이터에 삭제 여부를 나타내는 플래그 컬럼을 추가하고, `UPDATE` 쿼리로 해당 플래그를 변경하여 삭제된 것처럼 처리하는 방식입니다

### Hard Delete

- `DELETE` 쿼리를 통해 데이터를 **완전히 제거**합니다.
- 한 번 삭제된 데이터는 **복구하기 어렵습니다.**
- 삭제된 만큼 **물리적인 저장공간이 확보**됩니다.

### Soft Delete

- `UPDATE` 쿼리를 통해 **삭제 여부를 나타내는 플래그 값을 변경**합니다.
- 플래그 값을 원래대로 되돌리면 **데이터 복원이 가능합니다**.
- 실제로 데이터는 삭제되지 않으므로 **물리적인 저장공간은 줄어들지 않습니다**.

## Soft Delete를 사용하는 이유

앞서 설명한 것처럼 Soft Delete는 데이터를 실제로 삭제하지 않고, **삭제 상태를 표시하는 방식**입니다. 이로 인해 삭제하더라도 물리적인 저장공간이 줄어들지 않는다는 단점이 있지만, 다음과 같은 장점들이 있어 유용하게 사용됩니다.

- **삭제한 데이터 복구가 용이**합니다. 단순히 플래그 값을 되돌리면 되므로, 복구 비용이 낮습니다.
- **데이터 삭제 시점 및 이력 관리**가 가능합니다. `updated_at`과 같은 컬럼을 함께 사용하면, 삭제된 시간이나 사유를 추적할 수 있습니다.
- **참조 무결성 유지**에 유리합니다. 삭제된 데이터가 다른 테이블에서 여전히 참조되고 있어도 문제가 발생하지 않습니다.

이러한 이유들로 인해 Soft Delete는 단순한 삭제를 넘어 **데이터 관리의 유연성을 높이는 방식**으로 널리 사용됩니다.

### Soft Delete를 위해 필요한 플래그 필드

Soft Delete를 적용하려면, **데이터가 삭제되었음을 나타내는 플래그(flag)** 필드가 필요합니다. 이 필드는 DB 테이블의 컬럼으로 존재하며, 엔티티에서도 속성값으로 관리되어야 합니다.

예를 들어, 회원의 포인트를 저장하는 테이블에 Soft Delete를 적용한 경우는 다음과 같습니다.

**Soft Delete 플래그 컬럼(`deleted`)이 추가된 테이블 예시**

| id  | user_id | amount | deleted | created_at          | updated_at          |
| --- | ------- | ------ | ------- | ------------------- | ------------------- |
| 1   | 10      | 100    | 0       | 2024-12-22 01:21:21 | 2024-12-22 01:21:21 |
| 2   | 20      | 200    | 0       | 2025-06-30 23:11:45 | 2025-06-30 23:11:45 |
| 3   | 10      | 300    | 1       | 2025-06-30 23:12:40 | 2025-06-30 23:26:56 |

- `deleted` 컬럼은 Soft Delete 여부를 나타내는 **플래그 컬럼**입니다.
    - `deleted = 1`인 데이터는 **삭제된 데이터**를 의미합니다.
    - `deleted = 0`인 데이터는 **정상 상태의 데이터**입니다.
- `updated_at` 컬럼을 통해 **삭제 시점**을 파악할 수 있습니다.
- `deleted = 1`인 데이터의 updated_at 컬럼을 통해, 해당 데이터의 삭제 시점을 확인할 수 있습니다.

**Soft Delete 플래그 컬럼(`deleted`)이 추가된 엔티티 예시**

```kotlin
@Entity
@Table(name = "point")
@EntityListeners(AuditingEntityListener::class)
data class PointEntity(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,

    @Column(name = "user_id", nullable = false)
    val userId: Long,

    @Column(name = "amount", nullable = false)
    val amount: Long,

    @Column(name = "deleted", nullable = false)
    var deleted: Boolean = false
) {
    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    lateinit var createdAt: LocalDateTime

    @LastModifiedDate
    @Column(name = "updated_at")
    var updatedAt: LocalDateTime? = null
}

```

## Soft Delete의 주의할 점

## Soft Delete를 유용하게 다룰 수 있는 **Annotation**

### @SQLDelete

### @Where, @SQLRestriction

### @FilterDef 및 @Filter

### @SoftDelete

<br>

---
velog, “[Spring boot] JPA Soft Delete 구현하기”, [https://velog.io/@jsb100800/Spring-boot-JPA-Soft-Delete-구현하기](https://velog.io/@jsb100800/Spring-boot-JPA-Soft-Delete-구현하기), (참고 날짜 2025.06.30)

티스토리, “[JPA] Soft Delete: JPA에서 Soft Delete를 구현하는 방법, @SqlDelete, @Where”, [https://engineerinsight.tistory.com/172](https://engineerinsight.tistory.com/172), (참고 날짜 2025.06.30)




