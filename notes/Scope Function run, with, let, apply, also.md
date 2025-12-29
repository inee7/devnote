# Scope Function: run, with, let, apply, also

![[resources/images/scope-function-table.png]]
![[resources/images/scope-function-diagram.jpg]]
[코틀린 의 apply, with, let, also, run 은 언제 사용하는가? | by GM.Lim | Medium](https://medium.com/@limgyumin/%EC%BD%94%ED%8B%80%EB%A6%B0-%EC%9D%98-apply-with-let-also-run-%EC%9D%80-%EC%96%B8%EC%A0%9C-%EC%82%AC%EC%9A%A9%ED%95%98%EB%8A%94%EA%B0%80-4a517292df29)

apply는 유창한 설정자에 쓰이기도 한다. 
```kotlin
data class Mail(private var _message:String?=null) {
    fun message(message:String) = apply { _message=message }
}
```

https://hyeon9mak.github.io/using-kotlin-scope-function-correctly/

#kotlin