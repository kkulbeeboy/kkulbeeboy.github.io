---
title: "코틀린에서는 Checked Exception과 Unchecked Exception을 구별하지 않는다"
date: 2024-10-03 00:00:00 +0900
categories: [language, kotlin]
tags: [kotlin, exception]
---

## 자바의 Checked Exception과 Unchecked Exception

자바에서 예외는 **Checked Exception**과 **Unchecked Exception**으로 나눌 수 있으며, 다음과 같은 특징을 가지고 있습니다.

- **Checked Exception**
    - 예외 처리를 강제하여, 명시적인 처리가 필요합니다. (ex `throws`, `try-catch`)
    - 컴파일 시점에 검사하게 됩니다.
    - 스프링 트랜잭션에서 예외 발생 시 롤백이 되지 않습니다.
    - ex) IOException, SQLException
    
    **Checked Exception을 명시적으로 처리하는 코드**
    
    ```java
    public void callCheckedExceptionMethod() {
        try {
            throwCheckedExceptionMethod();
        } catch (IOException e) {
            logger.error(e.getMessage());
        }
    }
    
    private void throwCheckedExceptionMethod() throws IOException {
        throw new IOException("Checked Exception");
    }
    ```
    
- **Unchecked Exception**
    - 예외 처리를 강제하지 않습니다.
    - 런타임 시점에 검사하게 됩니다.
    - 스프링 트랜잭션에서 예외 발생 시 롤백이 됩니다.
    - RuntimeException 하위의 예외가 해당됩니다.
    - ex) NullPointerException, IllegalArgumentException
    
    **Unchecked Exception을 명시적으로 처리하지 않는 코드**
    
    ```java
    public void callUncheckedExceptionMethod() {
        throwUncheckedExceptionMethod();
    }
        
    private void throwUncheckedExceptionMethod() {
        throw new NullPointerException("Unchecked Exception");
    }
    ```
    

즉, 자바는 **Checked Exception**과 **Unchecked Exception**을 구별하고 있다는 것을 알 수 있습니다.

<br>

## 코틀린은 Checked Exception과 Unchecked Exception을 구별하지 않는다.

앞서 말씀드린 것 같이 자바에서는 Checked Exception에 대해 `try-catch`를 통해 예외를 잡거나, `throws`를 통해 예외를 전파하는 방식처럼 명시적인 예외처리가 필요합니다.

하지만, 코틀린은 자바와 달리, Checked Exception에 대해 강제로 예외 처리를 할 필요가 없는데, **이는 코틀린이 Checked Exception과 Unchecked Exception을 구별하지 않기 때문입니다.**

**코틀린은 기본적으로 모든 예외를 Unchecked Exception으로 간주하여 처리**하며, 이를 통해 예외 처리를 간소화할 수 있습니다. (물론 예외를 잡아서 처리하는 것 또한 가능합니다)

<br>

**자바의 Checked Exception인 IOException을 코틀린에서 명시적인 처리를 하지 않아도 되는 예시 코드**

```kotlin
    fun callIOExceptionMethod() {
        throwIOExceptionMethod()
    }

    fun throwIOExceptionMethod() {
        throw IOException("IO Exception");
    }
```

<br>

### 왜 코틀린은 Checked Exception과 Unchecked Exception을 구별하지 않을까?

이는 자바에서 Checked Exception 예외 처리를 강제하여 명시적으로 처리하는 방법이 그다지 효과적인 경우가 많지 않기 때문입니다.

자바에서 개발자들이 Checked Exception에 대해 명시적으로 처리할 때, 의미 없이 예외를 다시 던지거나, 예외를 잡지만 아무런 처리를 하지 않고 무시하는 경우가 많이 있는데, 이로 인해 Checked Exception이 예외 처리를 강제하도록 함에 불편함만 남았습니다.

즉, Checked Exception을 처리하도록 강제해도, **의미 있는 동작을 하는 경우가 드물고, 오히려 가독성이 좋지 않아지며, 개발 생산성이 낮아지게 되는 문제가 발생했습니다.**

이처럼 Checked Exception이 효과적인가에 대해서는 이전부터 많은 논쟁이 있었으며, 이에 대해 코틀린은 Checked Exception을 구분하지 않도록 설계했습니다.

→ 많은 언어들이 코틀린과 같이 Checked Exception을 구분하지 않습니다. (ex C#, C++, Python …)

<br>

## 그렇다면 코틀린에서 예외 발생 시 스프링 트랜잭션 롤백은 어떻게 동작할까?

스프링은 EJB 관례를 따르기 때문에, 트랜잭션에서 예외가 발생하면, Checked Exception과 같은 경우 롤백이 자동으로 수행되지 않고, Unchecked Exception의 경우에만 롤백이 자동으로 수행됩니다.

하지만 앞서, 코틀린과 같은 경우는 Checked Exception과 Unchecked Exception을 구별하지 않고 모든 예외를 Unchecked Exception으로 간주하여 처리한다고 말했습니다. 그렇다면  발생하는 어떠한 코틀린의 예외든 스프링 트랜잭션 내에서 발생한다면, 해당 트랜잭션은 자동으로 롤백이 수행될까요?

결과적으로 코틀린도, 스프링 트랜잭션의 롤백은 **자바에서 Unchecked Exception으로 구별되는 RuntimeException을 상속받은 예외만 자동으로 롤백 처리가 되고**, 그 외 **자바에서 CheckedException으로 구별되는 예외는 자동으로 롤백 처리가 되지 않습니다.**

<br>

트랜잭션 내부에서 DB에 값을 저장하고 예외를 발생시킨 뒤, 트랜잭션이 롤백 되었는지 확인하는 코드를 통해 테스트를 진행해 보겠습니다. 

→ 만약 **롤백이 진행되었다면 DB에 값이 저장되지 않았을 것**이고, **롤백이 진행되지 않았다면 DB에 값이 저장될 것**입니다.

### RuntimeException을 상속받은 예외의 롤백(O) 테스트 코드

스프링 트랜잭션 내부에서, 자바에서 Unchecked Exception으로 구분하는 예외인 **RuntimeException을 상속받은 예외가 발생**하면 해당 트랜잭션이 **롤백 되는지** 확인해 보겠습니다.

<br>

**트랜잭션 내부에서 RuntimeException을 상속받은 예외(NullPointerException)가 발생하는 코드**

```kotlin
@Transactional
fun saveUser(name: String) {
    userRepository.save(User(name = name))
    throw NullPointerException("RuntimeException")
}
```

<br>

**NullPointerException이 발생했을 때 트랜잭션 롤백이 진행되는 테스트 코드**

```kotlin
@Test
fun runtimeExceptionRollbackTest() {
    // given
    val name = "jude"

    // when
    assertThrows<NullPointerException> {
        userService.saveUser(name = name)
    }

    // then
    assertThat(userRepository.findAllByName(name = name)).isEmpty()
}
```

- **롤백이 진행되어, 저장이 진행되지 않았음을 확인할 수 있습니다.**
- 테스트 실행 결과 - passed
    
    ![image.png](/assets/img/posts/kotlin-runtime-exception-rollback-test-result.png)
    

<br>

### RumtimeException을 상속받지 않은 예외의 트랜잭션 롤백(X) 테스트

스프링 트랜잭션 내부에서, 자바에서 Checked Exception으로 구분하는 예외인 **RuntimeException을 상속받지 않은 예외**가 발생하면 해당 트랜잭션이 **롤백 되지 않는지** 확인해 보겠습니다.

<br>

**트랜잭션 내부에서 RuntimeException을 상속받지 않은 예외(IOException)가 발생하는 코드**

```kotlin
@Transactional
fun saveUser(name: String) {
    userRepository.save(User(name = name))
    throw IOException("Non-RuntimeException")
}
```

<br>

**IOException이 발생했을 때 트랜잭션 롤백이 진행되지 않는 테스트 코드**

```kotlin
@Test
fun nonRuntimeExceptionRollbackTest() {
    // given
    val name = "jude"

    // when
    assertThrows<IOException> {
        userService.saveUser(name = name)
    }

    // then
    assertThat(userRepository.findAllByName(name = name)).isNotEmpty
}
```

- **롤백이 진행되지 않아, 저장이 되었음을 확인할 수 있습니다.**
- 테스트 실행 결과 - passed
    
    ![image.png](/assets/img/posts/kotlin-non-runtime-exception-rollback-test-result.png)
    

<br>

---

Dmitry Jemerov, Svetlana Isakova, ⌜Kotlin In Action⌟, 오현석 옮김, 에이콘출판사, 2017, p.97-98

Kotlin Programming Language, “Exceptions” https://kotlinlang.org/docs/exceptions.html, (참고 날짜 2024.10.01)

velog, “Java의 Checked Exception은 실수다?”, [https://velog.io/@eastperson/Java의-Checked-Exception은-실수다-83omm70j](https://velog.io/@eastperson/Java%EC%9D%98-Checked-Exception%EC%9D%80-%EC%8B%A4%EC%88%98%EB%8B%A4-83omm70j), (참고 날짜 2024.10.01)

공부하는 개발자, “(코틀린 강의 질문) Kotlin + SpringBoot 에서 트랜잭션 예외처리를 할 때 주의할점 - checked exception, unchecked exception”, [https://youtu.be/sQj9_doE18Y?si=fRUbGfuZG-FxniCW](https://youtu.be/sQj9_doE18Y?si=fRUbGfuZG-FxniCW), (참고 날짜 2024.10.03)