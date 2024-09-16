---
title: "코틀린 객체 생성 시, 속성값을 검증하고 싶다면?"
date: 2024-09-16 00:00:00 +0900
categories: [language]
tags: [kotlin]
---

## 객체를 생성할 때, 검증이 필요한 상황

데이터 정합성을 위해 객체 생성 시, 속성값에 대해 검증이 필요한 상황이 발생할 수 있습니다. 예를 들어, ‘종료일은 시작일 이후여야 한다’ 또는 ‘사각형의 가로 길이와 세로 길이는 특정 비율에 맞아야 한다’와 같이 객체가 가지는 속성값에 대한 제약사항이 있을 수 있습니다.

<br>


‘**사각형의 가로 길이와 세로 길이의 비율은 특정 비율을 유지해야 된다**’라는 제약조건이 있는 상황에서 변화하는 몇 가지 요구사항에 따라, 객체 생성 시 데이터의 무결성을 유지할 수 있도록 사각형 클래스를 만들어 보겠습니다.

<br>

> 우선 첫 번째로, 만들어지는 모든 사각형 객체는 무조건 ‘가로와 세로 길이가 4:3 비율을 유지해야 된다.’는 요구사항이 있다고 가정하겠습니다.
> 

<br>

우선 간단하게 사각형의 속성값에는 가로 길이와, 세로 길이만을 가지며, 다음과 같이 데이터 클래스를 설계할 수 있습니다.

**가로 길이와 세로 길이를 속성값으로 가지는 사각형 데이터 클래스**

```kotlin
data class Rectangle(val width: Int, val height: Int)
```

하지만, 해당 클래스는 객체 생성 시, 가로와 세로의 비율이 4:3 비율에 맞는지에 대한 아무런 검증을 수행하지 않기 때문에, 요구사항과 다르게 가로와 세로의 비율이 4:3 이 아닌, 데이터 무결성을 해칠 수 있는 객체가 만들어질 수 있습니다.

이러한 상황에서 어떻게 해야 요구사항에 맞는 데이터 무결성을 유지할 수 있을까요?

<br>

## 초기화 블록을 이용한 검증

객체를 생성할 때, 실행되는 초기화 블록을 통해 검증을 수행할 수 있습니다.

```kotlin
data class Rectangle(val width: Int, val height: Int) {
    init {
        validateAspectRatio()
    }

    private fun validateAspectRatio() {
        require(width * 3 == height * 4) {
            "Width and height must maintain a 4:3 ratio."
        }
    }
}
```

초기화 블록에서 비율을 검증하는 `validateAspectRatio` 메서드를 수행하여, 가로와 세로의 비율이 4:3 비율인지 확인합니다.

객체를 생성할 때, 의도한 대로 검증을 수행하는지 확인해 보겠습니다.

<br>

**가로와 세로의 비율이 4:3이 아닌 사각형 객체를 생성하는 코드**

```kotlin
@Test
fun invalidAspectRatioTest(){
    val rectangle = Rectangle(width = 10, height = 10)
    println("rectangle: $rectangle")
}
```

- 출력 결과
    
    ```bash
    java.lang.IllegalArgumentException: Width and height must maintain a 4:3 ratio.
    ```
    
- 객체 생성 시 의도한 대로, 사각형의 비율에 대한 검증을 수행하는 것을 확인할 수 있습니다.

<br>    

> 그렇다면 요구사항이 변경되어 사각형의 비율을 4:3으로 유지하는 것이 아닌, 비율과 [스케일 팩터](https://ko.wikipedia.org/wiki/%EC%8A%A4%EC%BC%80%EC%9D%BC_%ED%8C%A9%ED%84%B0)를 받아, 이에 대한 사각형을 만들어 내야 한다면, 어떻게 해야 될까요?
> 

<br>

## 팩토리 메서드를 통한 객체 생성

새로운 요구사항에 맞춰 비율과 스케일 팩터를 받아 이에 맞는 적절한 사각형 객체를 만들어 내는 팩토리 메서드를 만들 수 있습니다.

<br>

**비율과 스케일 팩터를 받아 사각형 객체를 만들어내는 팩토리 메서드 코드**

```kotlin
data class Rectangle(val width: Int, val height: Int) {
    companion object {
        fun fromRatio(ratioWidth: Int, ratioHeight: Int, scaleFactor: Int): Rectangle {
            return Rectangle(ratioWidth * scaleFactor, ratioHeight * scaleFactor)
        }
    }
}
```

<br>

팩토리 메서드를 통해 가로와 세로의 비율이 16:9인 사각형 객체를 만들어 보겠습니다.

**팩토리 메서드를 통해 사각형 객체를 만드는 코드**

```kotlin
@Test
fun generateByFactoryMethodTest(){
    val rectangle = Rectangle.fromRatio(ratioWidth = 16, ratioHeight = 9, scaleFactor = 2)
    println("rectangle: $rectangle")
}
```

- 출력 결과
    
    ```kotlin
    rectangle: Rectangle(width=32, height=18)
    ```
    

- 팩토리 메서드인 `fromRatio`을 통해 요구 사항인, 의도한 비율의 사각형을 만들어 낼 수 있었습니다.

### 기본 생성자 외부 호출 방지

하지만, 현재 상황에서는 `Rectangle`의 기본 생성자를 호출할 수 있는 상황입니다. 즉, **개발자가 의도하지 않은 비율의 사각형 객체가 만들어질 수 있는 상황입니다.**

이를 방지하기 위해, `private constructor`를 이용해 외부에서 기본 생성자의 호출을 막을 수 있습니다.

<br>

**기본 생성자의 외부 호출을 private로 막은 코드**

```kotlin
data class Rectangle private constructor(val width: Int, val height: Int) {
    companion object {
        fun fromRatio(ratioWidth: Int, ratioHeight: Int, scaleFactor: Int): Rectangle {
            return Rectangle(ratioWidth * scaleFactor, ratioHeight * scaleFactor)
        }
    }
}
```

- 외부에서 기본 생성자 호출 시, 발생하는 에러 메시지
    
    ```bash
    Cannot access '<init>': it is private in 'Rectangle'
    ```
    
- 이를 통해, 비율로 사각형 객체를 생성하는 `fromRatio` 메서드를 통해서만 `Rectangle` 객체를 생성할 수 있도록 제한했습니다.

<br>

## 정리

코드를 작성하다 보면, 의도하지 않게 데이터 무결성이 깨지는 인스턴스를 생성하게 될 수 있습니다. 이러한 문제를 사전에 방지하도록 다음과 같이 객체를 생성하는 몇 가지 방법을 고려할 수 있습니다.

<br>

---

kakaopay tech, “코틀린, 저는 이렇게 쓰고 있습니다”, [https://tech.kakaopay.com/post/katfun-joy-kotlin](https://tech.kakaopay.com/post/katfun-joy-kotlin), (참고 날짜 2024.09.15)

Kotlin Programming Language, “classes - constructors” [https://kotlinlang.org/docs/classes.html#constructors](https://kotlinlang.org/docs/classes.html#constructors), (참고 날짜 2024.09.15)