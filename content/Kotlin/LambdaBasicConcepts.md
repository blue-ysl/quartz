---
aliases:
title: Lambda 기본 개념과 문법
---
`마지막 수정 일시: 2026년 4월 18일 토요일, 15시 (KST)`

이 글에서는 Kotlin의 Lambda와 관련된 문법에 대해 알아봅니다.
어려운 내용은 아니지만, C, Java 같은 언어를 쓰다 Kotlin을 처음 본 분을 독자로 상정하고 작성했습니다.

## Lambda란?

람다(lambda)는 값처럼 쓸 수 있는 함수입니다.

```kotlin
val sumFn: (Int, Int) -> Int = { x, y -> x + y }
println(sumFn(3,5)) // 8
println(sumFn(2,5))
```

변수 `sumFn`은 `Int 인자 두 개를 받아 Int 하나를 반환하는 함수` 타입의 변수입니다.

함수 타입에 `typealias` 키워드로 별칭(alias)을 붙여 더 간단하게 표현할 수도 있습니다.

```kotlin
typealias SumFn = (Int, Int) -> Int

val mySumFn: SumFn = { x, y -> x + y }
println(mySumFn(3,5))
```

## Lambda를 함수 파라미터로 전달하기

위 예에서 `println` 함수에 전달된 것은 더하기 람다의 반환 값입니다.
하지만 **람다 자체**를 함수 인자로 넘겨줄 수도 있지 않을까요?
이는 다음과 같이 작성하면 됩니다.

```kotlin
fun multiplyNumber(number: Int, multiple: Int, execFun: (Int, Int) -> Int) {  
    val res = execFun(number, multiple)  
    println("--> Result: $res")  
}

multiplyNumber(3, 5, { a, b -> a * b}) // --> Result: 15 
```

위 예에서 함수는 `execFun`이라는 이름으로  `Int 인자 두 개를 받아 Int 하나를 반환하는 함수` 타입의 값을 받습니다. `multiplyNumber`에 `Int` 타입 `a`, `b`를 받아 곱한 값을 반환하는 함수를 전달하고 있지요.

참고로 함수에 전달하는 마지막 인자가 람다일 경우, 이를 괄호 밖으로 빼는 것도 허용됩니다.
중요한 건 아니지만 이러한 형식의 구문을  '*trailing lambda*'라고도 합니다.

```Kotlin
multiplyNumber(3, 5, { a, b -> a * b})
multiplyNumber(3, 5) { a, b -> a * b } // 같은 동작을 하는 코드
```

이러한 문법 때문에, 아래와 같이 람다를 코드블록처럼 작성하는 사례도 종종 보실 수 있을 것입니다.

```kotlin
multiplyNumber(3, 5) { a, b ->
	a * b
}

// Another example
fun doReceivedAction(actFunc: () -> Unit) {  
    println("--- doReceivedAction start ---")  
    actFunc()  
    println("--- doReceivedAction end   ---")  
}

doReceivedAction {  
    println("Do something here and there")  
}

doReceivedAction({println("Do Something here and there")}) // 똑같이 동작하는 코드

/* 출력
--- doReceivedAction start ---
Do something here and there
--- doReceivedAction end   ---
*/
```

## Lambda의 인자가 하나일 때: `it`

람다가 받는 인자가 단 한 개라면, 이를 `it`로 표기할 수 있습니다.

다음 예에서는 List 개체에 대해 `filter`에 넘겨주는 판별식 람다에 `it` 키워드가 쓰였습니다.

```kotlin
val nums = listOf(1, 2, 3, 4, 5, 6)  
val evenNums = nums.filter { it % 2 == 0 }  
```

`filter` 함수는 `predicate`라는 이름으로 람다 인자를 받는데, 이 람다의 인자는 `T` 하나 뿐입니다.

```kotlin
public inline fun <T> Iterable<T>.filter(predicate: (T) -> Boolean): List<T> {  
    return filterTo(ArrayList<T>(), predicate)  
}
```

람다의 인자가 하나일 때, `it`라는 키워드는 바로 그 인자를 가리키는 것임을 약속한 것이지요.
아래 예에서 세 줄 모두 똑같은 역할을 합니다. 

```kotlin
val evenNums = nums.filter { it % 2 == 0 }  
val evenNumsSimpler = nums.filter { num -> num % 2 == 0 }  
val evenNumsVerbose = nums.filter { num: Int -> num % 2 == 0 }
```




---
