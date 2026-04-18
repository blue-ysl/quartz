---
aliases:
title: Function With Receiver + 사용 예시 (Gradle)
---
`마지막 수정 일시: 2026년 4월 18일 토요일, 16시 (KST)`

## Function with receiver

한국어로 번역하기 참 애매한 개념입니다..
따라서 이름의 뜻 보다는 동작을 보는 편이 낫다고 생각합니다.

간단히 말하면, 어떤 개체(receiver)의 함수처럼 실행할 수 있는 람다입니다.

예를 보겠습니다. 다음은 `MyInt` 클래스의 정의입니다.

```kotlin
class MyInt(i: Int) {  
    var num: Int = i  
  
    fun add(other: Int): Int {
	    num += other
        return num  
    }  
}
```

`num`의 초기값이 5이고, 여기에 5를 더하고 싶다면 다음과 같이 `add` 함수를 호출하면 될 것입니다.

```kotlin
val myObj = MyInt(5)  
myObj.add(5)  
println(myObj.num) // 10
```

똑같은 동작을 *function with receiver* 람다를 활용해서 다음과 같이 구성할 수도 있습니다. 

```kotlin
val myObj2 = MyInt(5)  
val sumFunc : MyInt.(Int) -> Unit = { other -> add(other) }  
//                                  { other -> this.add(other) } <-- 똑같이 동작
myObj2.sumFunc(5)  
println(myObj2.num) // 10
```

위 예에서 `sumFunc`는
- `Int` 하나를 `other`라는 인자로 받아서
- `MyInt` **개체 컨텍스트 안에서** 정의된 동작(`add` 메서드 호출)을 수행하는
람다 타입으로 정의되었습니다.

`MyInt`라는 개체의 컨텍스트를 람다 안에서 참조하고 있으므로, 마치 `MyInt` 개체의 함수처럼 실행되는 것처럼 볼 수도 있겠지요.

다음과 같은 경우에는 컴파일 에러가 납니다.
- `sumFunc` 람다에서 `MyInt`에 정의되지 않은 메서드를 호출하는 경우
- `MyInt` 타입이 아닌 변수에 대해서 `sumFunc`를 사용하려는 경우

```kotlin
// sumFunc 람다에서 MyInt에 정의되지 않은 메서드를 호출하는 경우
val sumFunc : MyInt.(Int) -> Unit = { other -> unknownFunc(other) } // error

// MyInt 타입이 아닌 변수에 대해서 sumFunc를 사용하려는 경우
val myNotObj = NotMyInt(5)  
myNotObj.sumFunc(3) // error
```

## 사용 예시 (Gradle)

의존성 관리 도구인 Gradle에서도 function with receiver가 활용되는 사례를 확인할 수 있습니다.

Gradle 빌드 스크립트를 작성할 때 이런 블록을 한 번은 보셨을 겁니다..

```kotlin
// 파일: build.gradle.kts
dependencies {  
    implementation("org.postgresql:postgresql:42.2.18")  
}
```

사실 이 `dependencies`는 Gradle API `Project` 클래스의 확장 함수입니다.
[Gradle GitHub: ProjectExtensions.kt 코드](https://github.com/gradle/gradle/blob/master/platforms/core-configuration/kotlin-dsl/src/main/kotlin/org/gradle/kotlin/dsl/ProjectExtensions.kt)

시그니처는 이렇게 생겼습니다.

```kotlin
fun Project.dependencies(configuration: DependencyHandlerScope.() -> Unit) =
    DependencyHandlerScope.of(dependencies).configuration()
```

`DependencyHandlerScope` 타입의 개체를 receiver로 하는 람다를 `configuration`이라는 이름의 인자로 넘겨주고 있지요.

함수 구현체를 보면, 결국 `DependencyHandlerScope.of(dependencies)`에서 반환된 개체에 대하여, Gradle 빌드스크립트에 썼던 람다 `{ implementation(...) }`가 실행되는 것입니다.

 > 참고 사항: 여기서 중요한 것은 아니지만, `implementation`은 `DependencyHandler` 인터페이스의 `add` 메서드를 구현으로 정의한 확장 함수입니다.  
 
## 나가며..

C나 Java 같은 전통적 언어를 주로 썼던 분이라면 Kotlin의 function with receiver 문법이 다소 생소하실 수도 있을 듯 합니다. 하지만 특정 타입의 개체에 대해서 실행하는 함수를 정의한다고 생각하면 그렇게 어렵지는 않으리라 생각합니다. 

function with receiver를 실질적으로 유용하게 활용할 수 있는 사용례에 대해서는 추후 여건이 허락하면 업데이트 하겠습니다.