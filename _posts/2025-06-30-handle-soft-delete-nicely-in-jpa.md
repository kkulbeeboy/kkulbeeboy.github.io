---
title: "JPA에서 Soft Delete 멋지게 다루는법"
date: 2025-06-30 00:00:00 +0900
categories: [backend, spring]
tags: [spring, jpa, soft delete]
---

## Hard Delete vs Soft Delete

서비스를 운영하다 보면, 저장된 데이터를 삭제해야 되는 상황이 생기기 마련입니다. 이때 데이터를 삭제하는 방식은 크게 두 가지로 나뉩니다.

- **Hard Delete (물리적 삭제)**: `DELETE` 쿼리를 사용하여 데이터베이스에서 해당 데이터를 **완전히 제거**하는 방식입니다.
- **Soft Delete (논리적 삭제)**: 삭제 여부를 나타내는 **플래그 컬럼**(ex deleted, is_deleted)을 두고, `UPDATE` 쿼리로 **플래그를 변경하여 삭제된 것처럼** 처리하는 방식입니다

### Hard Delete

- `DELETE` 쿼리를 통해 데이터를 **완전히 제거**합니다.
- 한 번 삭제된 데이터는 **복구하기 어렵습니다.**
- 삭제된 만큼 **물리적인 저장공간이 확보**됩니다.

### Soft Delete

- `UPDATE` 쿼리를 통해 **삭제 여부를 나타내는 플래그 값을 변경**합니다.
- 플래그 값을 원래대로 되돌리면 **데이터 복원이 가능합니다**.
- 실제로 데이터는 삭제되지 않으므로 **물리적인 저장공간은 줄어들지 않습니다**.

<br>

## Soft Delete를 사용하는 이유

앞서 설명한 것처럼 Soft Delete는 데이터를 실제로 삭제하지 않고, **삭제 상태를 표시하는 방식**입니다. 이로 인해 삭제하더라도 물리적인 저장공간이 줄어들지 않는다는 단점이 있지만, 다음과 같은 장점들이 있어 유용하게 사용됩니다.

- **삭제한 데이터 복구가 용이**합니다. 단순히 플래그 값을 되돌리면 되므로, 복구 비용이 낮습니다.
- **데이터 삭제 시점 및 이력 관리**가 가능합니다. `updated_at`(또는 deleted_at)과 같은 컬럼을 함께 사용하면, 삭제된 시간이나 사유를 추적할 수 있습니다.
- **참조 무결성 유지**에 유리합니다. 삭제된 데이터를 참조 중인 다른 테이블이 있더라도, 물리적으로 제거되지 않기 때문에 참조 제약 조건을 위반하지 않습니다.

이러한 이유들로 인해 Soft Delete는 **데이터 관리의 유연성을 높이는 방식**으로 널리 사용됩니다.

### Soft Delete를 위해 필요한 플래그 컬럼

Soft Delete를 적용하려면, **데이터가 삭제되었음을 나타내는 플래그(flag)** 컬럼이 필요합니다. 이 컬럼은 데이터베이스 테이블의 컬럼으로 존재하며, JPA 엔티티에서도 속성값으로 관리되어야 합니다.

예를 들어, 회원의 포인트를 저장하는 테이블에 Soft Delete를 적용하기 위해 플래그 컬럼인 `deleted` 컬럼을 추가한 경우는 다음과 같습니다.

<br>

**Soft Delete 플래그 컬럼(`deleted`)이 추가된 테이블 예시**

| id  | user_id | amount | deleted | created_at          | updated_at          |
| --- | ------- | ------ | ------- | ------------------- | ------------------- |
| 1   | 10      | 100    | 0       | 2024-12-22 01:21:21 | 2024-12-22 01:21:21 |
| 2   | 20      | 200    | 0       | 2025-06-30 23:11:45 | 2025-06-30 23:11:45 |
| 3   | 10      | 300    | 1       | 2025-06-30 23:12:40 | 2025-06-30 23:26:56 |

- `deleted` 컬럼은 Soft Delete 여부를 나타내는 **플래그 컬럼**입니다.
    - `deleted = 1`인 데이터는 **삭제된 데이터**를 의미합니다.
    - `deleted = 0`인 데이터는 **정상 상태의 데이터**입니다.
- `deleted = 1`인 데이터의 `updated_at` 컬럼을 통해, 해당 데이터의 삭제 시점을 확인할 수 있습니다.
    - updated_at이 아닌, 따로 deleted_at 컬럼을 따로 사용하여 삭제된 시점을 관리할 수도 있습니다.

<br>

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

- 삭제 플래그 필드인 `deleted`값을 엔티티의 속성값으로 관리하고 있습니다.

<br>

## Soft Delete의 주의할 점

Soft Delete는 **물리적으로 데이터를 삭제하지 않기 때문에**, 몇 가지 주의할 점이 있습니다. 이를 간과하면 의도하지 않은 동작이나 데이터 오류가 발생할 수 있습니다.

1. 삭제 시 `DELETE` 쿼리가 아닌 `UPDATE` 쿼리가 수행되어야 한다는 것을 주의하고 인지해야 합니다
2. 데이터 조회 시 삭제 상태인 데이터, 즉 `deleted = 1`인 데이터는 조회되지 않도록 필터링이 필요합니다.

데이터 삭제 및 조회 시 주의 사항을 명심하고 있어야, 정상적인 동작을 할 수 있게 됩니다. 그렇지 않다면, 의도하지 않게 Hard Delete가 수행되거나 삭제된 데이터가 조회 결과에 포함되는 결과를 초래할 수 있습니다.

<br>

## Soft Delete를 유용하게 다룰 수 있는 Annotation

앞서 살펴본 Soft Delete의 주의사항은, 개발자가 직접 쿼리나 로직을 신경 써야 한다는 점에서 **실수의 여지가 존재**합니다. 

하지만 다행히도, JPA의 구현체인 **Hibernate**는 이를 보완해 줄 수 있는 유용한 어노테이션들을 제공합니다.

### @SQLDelete

Soft Delete에서는 실제로 데이터를 삭제하지 않고, **삭제 상태를 나타내는 UPDATE 쿼리**를 실행해야 합니다.

Hibernate에서는 이를 위해 `@SQLDelete` 어노테이션을 제공합니다. @SQLDelete는 엔티티가 삭제될 때 Hibernate가 실행할 SQL 쿼리를 직접 지정할 수 있도록 해줍니다.

<br>

**@SQLDelete가 적용된 엔티티 예시**

```kotlin
@Entity
@Table(name = "point")
@EntityListeners(AuditingEntityListener::class)
@SQLDelete(sql = "UPDATE point SET deleted = true WHERE id = ?")
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

- `@SQLDelete`를 통해 Hibernate는 `delete()` 호출 시 `DELETE`가 아닌 `UPDATE point SET deleted = true WHERE id = ?` 쿼리를 실행합니다

<br>    

**@SQLDelete을 이용한 UPDATE 쿼리 실행 테스트 코드**

```kotlin
@Test
fun `SQLDelete 어노테이션 UPDATE 쿼리 실행 확인`() {
    // given
    val savedEntity = pointEntityRepository.save(PointEntity(userId = 10L, amount = 100L))

    // when
    pointEntityRepository.delete(savedEntity) // 데이터 삭제 수행
    
    // then
    val deletedEntity = pointEntityRepository.findById(savedEntity.id!!).orElseThrow { PointNotFoundException("PointEntity(id=${savedEntity.id}) not found") }
    assertThat(deletedEntity.deleted).isTrue // 삭제된 데이터를 조회하여, 해당 데이터의 deleted 상태가 true 인것 확인
}
```

- 포인트를 저장하고, 저장한 포인트를 삭제한 뒤, 다시 해당 포인트를 조회하여 삭제 플래그가 `true` 인지 확인합니다.
    - Soft Delete 일지라도 삭제된 데이터는 `findById` 와 같은 조회를 통해 조회되지 않아야 되지만, 여기서는 아직 Soft Delete 조회 처리를 위한 설정을 하지 않았으므로, 모두 조회됩니다.
    - Soft Delete 조회 처리를 어떻게 해야 될지는 아래 `@Where`, `@SQLRestriction`에서 알아보겠습니다.
- 실행 결과
    
    ![image.png](/assets/img/posts/SQLDelete-update-test-result.png)
    
- UPDATE 실행 로그
    
    ```bash
    Hibernate: 
        UPDATE
            point 
        SET
            deleted = true 
        WHERE
            id = ?
    ```
    
- 이와 같이, 물리적 삭제를 수행하지 않고, 삭제 플래그인 deleted를 true로 설정하는 것으로 삭제 처리한다는 것을 확인할 수 있습니다.

<br>

### @Where, @SQLRestriction

Soft Delete를 적용한 경우, **데이터는 물리적으로 남아 있지만 논리적으로는 삭제된 것처럼 동작해야 합니다.** 즉, `deleted = true`인 데이터는 조회되지 않도록 필터링되어야 합니다.

이를 위해 Hibernate는 SELECT 쿼리 수행 시 특정 조건을 자동으로 추가해주는 어노테이션을 제공하는데, 이는 바로 `@Where`와 `@SQLRestriction`입니다.

Hibernate 6.3 기준으로 `@Where`는 **Deprecated** 되었으며, **공식 문서에서는 `@SQLRestriction` 사용을 권장**하고 있습니다.

우선 `@Where`을 사용하는 방식과 그 한계에 대해서 알아보겠습니다.

<br>

**@Where가 적용된 엔티티 예시**

```kotlin
@Entity
@Table(name = "point")
@EntityListeners(AuditingEntityListener::class)
@SQLDelete(sql = "UPDATE point SET deleted = true WHERE id = ?")
@Where(clause = "deleted = false")
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

- `@Where`을 통해 조회 퀴리 동작 시, `deleted = false` 조건이 자동으로 추가되도록 했습니다.

<br>

**@Where을 이용한 SELECT 쿼리 필터링 테스트 코드**

```kotlin
@Test
fun `Where 어노테이션 SELECT 쿼리 실행 확인`() {
    // given
    val savedEntity = pointEntityRepository.save(PointEntity(userId = 10L, amount = 100L))

    // when
    pointEntityRepository.delete(savedEntity) // 데이터 삭제 수행
        
    // then
    assertThrows<PointNotFoundException> { // 데이터가 조회되지 않아서 예외가 발생하는지 확인
        pointEntityRepository.findById(savedEntity.id!!).orElseThrow { PointNotFoundException("PointEntity(id=${savedEntity.id}) not found") }
    }
}
```

- 포인트를 저장하고, 저장한 포인트를 삭제한 뒤, 삭제한 포인트가 조회가 되지 않는지 확인합니다.
    - 포인트 데이터가 존재하지 않아서 `PointNotFoundException`이 발생하는지 확인합니다.
- 실행 결과
    
    ![image.png](/assets/img/posts/Where-select-test-result.png)
    
- SELECT 실행 로그
    
    ```bash
    Hibernate: 
        select
            pe1_0.id,
            pe1_0.amount,
            pe1_0.created_at,
            pe1_0.deleted,
            pe1_0.updated_at,
            pe1_0.user_id 
        from
            point pe1_0 
        where
            pe1_0.id=? 
            and (
                pe1_0.deleted = 0
            )
    ```
    
    - 실행된 SELECT 쿼리에는 `where deleted = false` 조건이 자동으로 추가된것을 확인할 수 있습니다.

<br>

**@Where의 한계점**

Hibernate 5.x까지는 `@Where`가 **조회(SELECT) 쿼리에만 적용**되어 삭제(DELETE), 수정(UPDATE), 조인(JOIN) 등 에서는 조건이 **무시**되었습니다. 이로 인해 **삭제된 데이터에 대한 예상치 못한 업데이트, 조인 결과 노출** 등이 발생할 수 있었습니다.

Hibernate 6.0부터는 `@Where` 조건이 **모든 쿼리에 일관되게 적용되도록** 개선되었지만, 이전 버전과의 혼란과 불명확한 동작 때문에 결국 **6.3부터 Deprecated 처리**되었습니다. 따라서 더 명확한 방식인 `@SQLRestriction` 사용을 권장합니다

그렇다면 이제, `@Where`의 대체재로 권장되는 `@SQLRestriction`을 사용하는 방식에 대해 정리해보겠습니다.

<br>

**@SQLRestriction가 적용된 엔티티 예시**

```kotlin
@Entity
@Table(name = "point")
@EntityListeners(AuditingEntityListener::class)
@SQLDelete(sql = "UPDATE point SET deleted = true WHERE id = ?")
@SQLRestriction("deleted = false")
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

- `@SQLRestriction`을 통해 **엔티티를 조회할 때 항상** `deleted = false` 조건이 자동으로 WHERE 절에 붙도록 합니다.

<br>

**@SQLRestriction을 이용한 SELECT 쿼리 필터링 테스트 코드**

```kotlin
    @Test
    fun `SQLRestriction 어노테이션 SELECT 쿼리 실행 확인`() {
        // given
        val savedEntity = pointEntityRepository.save(PointEntity(userId = 10L, amount = 100L))

        // when
        pointEntityRepository.delete(savedEntity) // 데이터 삭제 수행
        
        // then
        assertThrows<PointNotFoundException> { // 데이터가 조회되지 않아서 예외가 발생하는지 확인
            pointEntityRepository.findById(savedEntity.id!!).orElseThrow { PointNotFoundException("PointEntity(id=${savedEntity.id}) not found") }
        }
    }
```

- 테스트 로직은 이전 ‘**@Where을 이용한 SELECT 쿼리 필터링 테스트 코드**’와 동일합니다.
- 실행 결과
    
    ![image.png](/assets/img/posts/SQLRestriction-select-test-result.png)
    
- SELECT 실행 로그
    
    ```bash
    Hibernate: 
        select
            pe1_0.id,
            pe1_0.amount,
            pe1_0.created_at,
            pe1_0.deleted,
            pe1_0.updated_at,
            pe1_0.user_id 
        from
            point pe1_0 
        where
            pe1_0.id=? 
            and (
                pe1_0.deleted = 0
            )
    ```
    
    - 역시나 데이터를 조회할 때, where 절을 통해 deleted 필드를 필터링하는 것을 확인할 수 있습니다.

<br>

### @SoftDelete

지금까지 Soft Delete를 위해 여러 어노테이션(`@SQLDelete`, `@Where`, `@SQLRestriction`)을 조합해 설정해왔는데요, Hibernate 6.4 이상에서는 이 과정을 훨씬 간단하게 만들어주는 **`@SoftDelete` 어노테이션**이 새롭게 추가되었습니다.

<br>

**@SoftDelete가 제공하는 기능**

- `deleted`라는 이름의 **Boolean 타입 Soft Delete 플래그**가 내부적으로 자동 생성됩니다.
    - 이 `deleted` 필드는 **엔티티 속성으로 명시적으로 중복해서 선언할 수 없어, 접근도 불가능**합니다.
- `delete()` 호출 시 **자동으로 `deleted = true`로 업데이트 합니다.**
- `findById()`, `findAll()` 등 조회 시 **자동으로 `deleted = false` 조건을 적용합니다.**
- 이로 인해 별도의 `@SQLDelete`, `@Where`, `@SQLRestriction`이 **불필요합니다.**

<br>

**@SoftDelete가 적용된 엔티티 예시**

```kotlin
@Entity
@Table(name = "point")
@EntityListeners(AuditingEntityListener::class)
@SoftDelete
data class PointEntity(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,

    @Column(name = "user_id", nullable = false)
    val userId: Long,

    @Column(name = "amount", nullable = false)
    val amount: Long,
) {
    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    lateinit var createdAt: LocalDateTime

    @LastModifiedDate
    @Column(name = "updated_at")
    var updatedAt: LocalDateTime? = null
}
```

- 이전 예시의 엔티티들과 다르게, `deleted` 필드는 자동으로 관리되며 엔티티에서 선언하지 않아야 합니다.
- 삭제시 수행될 쿼리(`@SQLDelete`)나, 조회시 사용될 조건(`@Where`, `@SQLRestriction`)에 필요한 어노테이션이 사용되지 않습니다.
- 기본 설정 외에도 삭제 여부를 판단하는 전략 설정(`strategy`), 삭제 플래그 컬럼명 설정(`columnName`), 데이터베이스에 저장할 값을 적용할 컨버터 지정(`converter`) 과 같은 설정도 가능합니다
    
    ```kotlin
    @SoftDelete(
        columnName = "is_deleted", // 삭제 플래그 컬럼명을 기본값 deleted → is_deleted 로 변경
        strategy = SoftDeleteType.DELETED, // 기본 삭제 전략 사용 (Boolean true/false로 판단)
        converter = YesNoConverter::class //활성 여부 Y, N로 표기
    )
    ```    
<br>

**@SoftDelete를 이용한 UPDATE, SELECT 쿼리 실행 테스트 코드**

```kotlin
@Test
fun `SoftDelete 어노테이션 UPDATE, SELECT 쿼리 실행 확인`() {
    // given
    val savedEntity = pointEntityRepository.save(PointEntity(userId = 10L, amount = 100L))

    // when
    pointEntityRepository.delete(savedEntity) // 데이터 삭제 수행

    // then
    assertThrows<PointNotFoundException> { // 데이터가 조회되지 않아서 예외가 발생하는지 확인
        pointEntityRepository.findById(savedEntity.id!!).orElseThrow { PointNotFoundException("PointEntity(id=${savedEntity.id}) not found") }
    }
}
```

- 포인트를 저장하고, 저장한 포인트를 삭제한 뒤, 삭제한 포인트가 조회가 되지 않는지 확인하며, 이때 수행되는 쿼리를 확인합니다.
- 실행 결과
    
    ![image.png](/assets/img/posts/SoftDelete-update-select-test-result.png)
    
- 실행 로그
    - UPDATE 실행 로그
        
        ```bash
        Hibernate: 
            update
                point 
            set
                deleted=1 
            where
                id=? 
                and deleted=0
        ```
        
    - SELECT 실행 로그
        
        ```bash
        Hibernate: 
            select
                pe1_0.id,
                pe1_0.amount,
                pe1_0.created_at,
                pe1_0.updated_at,
                pe1_0.user_id 
            from
                point pe1_0 
            where
                pe1_0.deleted=0 
                and pe1_0.id=?
        ```
        
<br>

### @FilterDef와 @Filter

`@FilterDef`와 `@Filter`는 Hibernate에서 **동적으로 SQL 조건을 제어할 수 있는 기능**입니다.

앞서 살펴본 `@Where`나 `@SQLRestriction`은 조회 시 항상 **고정된 조건**을 적용하므로, 상황에 따라 **조건을 유연하게 변경하기 어렵습니다**. 예를 들어, 일반 사용자는 삭제되지 않은 데이터만 조회하되, 관리자는 삭제된 데이터까지 조회해야 하는 경우가 그렇습니다.

이처럼 **Soft Delete 여부에 따라 조건을 동적으로 제어해야 하는 상황**에서는 `@FilterDef`와 `@Filter`를 사용하는 것이 효과적입니다.

<br>

**@FilterDef**

`@FilterDef`는 **필터의 이름과 파라미터를 정의하는 어노테이션**입니다.

Hibernate가 필터를 사용할 수 있도록 **사전 정의**해주는 역할을 하며, 필터 조건에 필요한 파라미터 타입도 함께 명시합니다.

<br>

**@FilterDef를 적용한 예시 코드**

```kotlin
@FilterDef(
    name = "deletedFilter", 
    parameters = [ParamDef(name = "isDeleted", type = Boolean::class)]
)
```

- `name`을 통해 `"deletedFilter"`라는 이름의 필터를 정의합니다.
- `parameters`를 통해 이 필터가 `isDeleted`라는 Boolean 타입 파라미터를 사용할 것임을 명시합니다.

<br>

**@Filter**

`@Filter`는 엔티티에 **실제로 필터 조건을 적용하는 어노테이션**입니다.

`name` 속성으로 사용할 필터를 지정하고, `condition` 속성으로 필터가 활성화될 때 적용할 **SQL 조건절**을 정의합니다.

<br>

**@Filter를 적용한 예시 코드**

```kotlin
@Filter(
    name = "deletedFilter", 
    condition = "deleted = :isDeleted"
)
```

- `name` 속성으로 앞서 `@FilterDef`에서 정의한 `deletedFilter` 필터를 지정합니다.
- `condition`을 통해 필터가 활성화되었을 때 적용할 SQL 조건(`deleted = :isDeleted`)을 설정합니다.
    - `:isDeleted`는 필터를 활성화할 때 넘겨줄 파라미터 이름입니다.

<br>

**@FilterDef와 @Filter가 적용된 엔티티 예시**

```kotlin
@Entity
@Table(name = "point")
@EntityListeners(AuditingEntityListener::class)
@SQLDelete(sql = "UPDATE point SET deleted = true WHERE id = ?")
@FilterDef(
    name = "deletedPointFilter",
    parameters = [ParamDef(name = "isDeleted", type = Boolean::class)]
)
@Filter(
    name = "deletedPointFilter",
    condition = "deleted = :isDeleted"
)
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

- `@FilterDef`를 통해 `"deletedPointFilter"`라는 이름의 필터를 정의합니다.
    - 이 필터는 `Boolean` 타입의 `isDeleted` 파라미터를 받습니다.
- `@Filter`를 통해 위에서 정의한 필터를 이 엔티티에 적용하며, 필터가 활성화되면
    
    `deleted = :isDeleted` 조건이 SQL에 자동으로 추가됩니다.
    
<br>

**@SQLRestriction을 이용한 SQL 조건 제어 테스트 코드**

```kotlin
    @Test
    @Transactional
    fun `SQLRestriction 어노테이션 조건 제어 테스트 코드`() {
        // given
        val savedEntity1 = pointEntityRepository.save(PointEntity(userId = 10L, amount = 100L))
        val savedEntity2 = pointEntityRepository.save(PointEntity(userId = 10L, amount = 100L))
        val savedEntity3 = pointEntityRepository.save(PointEntity(userId = 10L, amount = 100L))
        val filterName = "deletedPointFilter"

        // when
        pointEntityRepository.delete(savedEntity1) // savedEntity1 데이터 삭제 수행 (soft delete)

        val session = entityManager.unwrap(Session::class.java)

        // 필터 활성화 (deleted = false)
        // soft delete 처리되지 않은 데이터(savedEntity2, savedEntity3)를 조회합니다
        session.enableFilter(filterName).setParameter("isDeleted", false)
        val notDeletedPoints = pointEntityRepository.findAll()
        assertThat(notDeletedPoints.size).isEqualTo(2)

        // 필터 활성화 (deleted = true)
        // soft delete 처리된 데이터(savedEntity1)를 조회합니다
        session.enableFilter(filterName).setParameter("isDeleted", true)
        val deletedPoints = pointEntityRepository.findAll()
        assertThat(deletedPoints.size).isEqualTo(1)

        // 필터 비활성화
        // 전체 데이터(savedEntity1, savedEntity2, savedEntity3)를 조회합니다
        session.disableFilter(filterName)
        val allPoints = pointEntityRepository.findAll()
        assertThat(allPoints.size).isEqualTo(3)
    }
```

- `PointEntity` 3건을 저장한 후, 하나의 데이터만 Soft Delete 처리합니다.
- `@Transactional`을 사용하여 Soft Delete 결과가 트랜잭션 내에서 즉시 반영되도록 합니다.
- Hibernate의 `Session`을 이용해 필터를 켜고 끔으로써 필터의 활성화를 설정합니다.
    - 필터를 활성화했을 때
        - `deleted = false` 조건일 때 삭제되지 않은 데이터만 조회되는지 확인합니다.
        - `deleted = true` 조건일 때 삭제된 데이터만 조회되는지 확인합니다.
    - 필터를 비활성화했을 때
        - 전체 데이터가 조회되는지 확인합니다.
- 실행 결과
    
    ![image.png](/assets/img/posts/FilterDef-and-Filter-select-test-result.png)
    
- SELECT 실행 로그
    - 필터 활성화 (`deleted = false`)
        
        ```bash
        Hibernate: 
            select
                pe1_0.id,
                pe1_0.amount,
                pe1_0.created_at,
                pe1_0.deleted,
                pe1_0.updated_at,
                pe1_0.user_id 
            from
                point pe1_0 
            where
                pe1_0.deleted = ?
        ```
        
    - 필터 활성화 (`deleted = true`)
        
        ```bash
        Hibernate: 
            select
                pe1_0.id,
                pe1_0.amount,
                pe1_0.created_at,
                pe1_0.deleted,
                pe1_0.updated_at,
                pe1_0.user_id 
            from
                point pe1_0 
            where
                pe1_0.deleted = ?
        ```
        
    - 필터 비활성화
        
        ```bash
        Hibernate: 
            select
                pe1_0.id,
                pe1_0.amount,
                pe1_0.created_at,
                pe1_0.deleted,
                pe1_0.updated_at,
                pe1_0.user_id 
            from
                point pe1_0
        ```
        
<br>

## 정리

- **Soft Delete**는 데이터를 물리적으로 삭제하지 않고, 삭제 여부를 나타내는 플래그 컬럼을 사용해 논리적으로 삭제하는 방식입니다.
- Soft Delete 구현 시 `@SQLDelete`로 삭제 쿼리를, `@Where`나 `@SQLRestriction`로 조회 조건을 설정할 수 있습니다.
- Hibernate 6.4 이상에서는 `@SoftDelete`를 통해 더 간편하게 Soft Delete를 적용할 수 있습니다.
- 동적 제어가 필요할 땐 `@FilterDef`와 `@Filter`를 활용해 삭제된 데이터 포함 여부를 선택적으로 제어할 수 있습니다.

<br>

---
velog, “[Spring boot] JPA Soft Delete 구현하기”, [https://velog.io/@jsb100800/Spring-boot-JPA-Soft-Delete-구현하기](https://velog.io/@jsb100800/Spring-boot-JPA-Soft-Delete-구현하기), (참고 날짜 2025.06.30)

티스토리, “[JPA] Soft Delete: JPA에서 Soft Delete를 구현하는 방법, @SqlDelete, @Where”, [https://engineerinsight.tistory.com/172](https://engineerinsight.tistory.com/172), (참고 날짜 2025.06.30)

티스토리, “[JPA] Soft Delete 적용하기 : @Where vs @SQLRestriction”, [https://inma.tistory.com/194](https://inma.tistory.com/194), (참고 날짜 2025.07.05)

Hibernate Javadoc, “Annotation Type Where”, [https://docs.jboss.org/hibernate/stable/orm/javadocs/org/hibernate/annotations/Where.html](https://docs.jboss.org/hibernate/stable/orm/javadocs/org/hibernate/annotations/Where.html), (참고 날짜 2025.07.05)

Baeldung, “How to Implement a Soft Delete with Spring JPA”, [https://www.baeldung.com/spring-jpa-soft-delete](https://www.baeldung.com/spring-jpa-soft-delete), (참고 날짜 2025.07.05)

티스토리, “JPA에서 soft delete 처리하기 (@Where, @Filter)”, [https://cjsrhd94.tistory.com/138](https://cjsrhd94.tistory.com/138), (참고 날짜 2025.07.06)

velog, “[Spring] SoftDelete 적용 후 @Where 절로 인해 삭제된 포스트 목록 조회 불가능한 이슈”, [https://velog.io/@zvyg1023/Spring-Boot-SoftDelete-적용-후-Where-절로-인해-삭제된-포스트-목록-조회-불가능한-이슈](https://velog.io/@zvyg1023/Spring-Boot-SoftDelete-적용-후-Where-절로-인해-삭제된-포스트-목록-조회-불가능한-이슈), (참고 날짜 2025.07.06)

velog, “[Spring Boot] Soft Delete와 Hibernate 동적 필터링으로 삭제된 데이터 조회하기 (@Where 이슈 해결)”, [https://velog.io/@eunsilson/Spring-Boot-Soft-Delete와-Hibernate-동적-필터링-Where-이슈-해결](https://velog.io/@eunsilson/Spring-Boot-Soft-Delete와-Hibernate-동적-필터링-Where-이슈-해결), (참고 날짜 2025.07.06)

티스토리, “Hibernate 6.4의 @SoftDelete 사용법 탐구”, [https://ohksj77.tistory.com/249](https://ohksj77.tistory.com/249), (참고 날짜 2025.07.06)

티스토리, “Hibernate 신기능 @SoftDelete 기능”, [https://sundries-in-myidea.tistory.com/165](https://sundries-in-myidea.tistory.com/165), (참고 날짜 2025.07.06)

Github, “hibernate-orm/migration-guide.adoc”, [https://github.com/hibernate/hibernate-orm/blob/6.0/migration-guide.adoc](https://github.com/hibernate/hibernate-orm/blob/6.0/migration-guide.adoc), (참고 날짜 2025.07.06)

티스토리, “[Hibernate] Soft Delete in Hibernate”, [https://eoneunal.tistory.com/31](https://eoneunal.tistory.com/31), (참고 날짜 2025.07.06)



