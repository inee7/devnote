# Int.() -> Unit 과 Int -> Unit 차이 

`int.() -> Unit`와 `Int -> Unit`는 Kotlin에서 서로 다른 함수 시그니처(signature)를 나타냅니다.

1. `int.() -> Unit`: 이것은 확장 함수(extension function)의 시그니처입니다. `int`는 수신 객체의 타입을 나타내고, `() -> Unit`은 해당 수신 객체에 대한 람다 함수를 나타냅니다. 이 람다 함수 내에서는 수신 객체에 직접 접근할 수 있습니다. 아래는 예시 코드입니다:

   ```kotlin
   fun Int.myExtensionFunction(block: Int.() -> Unit) {
       // 수신 객체(this)에 대한 작업을 수행
       block()
   }
   
   val number = 42
   number.myExtensionFunction {
       println("This is an extension function on Int: $this")
   }
   ```

2. `Int -> Unit`: 이것은 일반적인 함수의 시그니처입니다. 여기서 `Int`는 매개변수의 타입을 나타내고, `-> Unit`은 반환 타입이 없음을 나타냅니다. 아래는 예시 코드입니다:

   ```kotlin
   fun myFunction(value: Int) {
       // 매개변수로 전달된 Int 값에 대한 작업 수행
       println("Received value: $value")
   }
   
   val number = 42
   myFunction(number)
   ```

따라서 `int.() -> Unit`은 확장 함수이고, `Int -> Unit`은 일반적인 함수를 나타냅니다. 전자는 수신 객체에 직접 접근할 수 있지만, 후자는 매개변수를 통해 값을 받습니다.

#kotlin