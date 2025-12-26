# 적절하게 null을 처리 하라 
**null이 최대한 의미를 갖어야 한다.** 

```kotlin
printer?.print() // safe call
if(printer != null) printer.print() // smart casting 

//elvis
printer?.name ?: "unnamed"
printer?.name ?: return 
printer?.name ?: throw Error()
```

스마트 캐스팅은 코틀린의 contracts feature (규약기능) 을 지원 
```kotlin
val name = readLine()
if (!name.isNullOrBlank()) {
    print("${name.length}")
}

//contract가 적용된 코드 
@kotlin.internal.InlineOnly
public inline fun CharSequence?.isNullOrBlank(): Boolean {
    contract {
        returns(false) implies (this@isNullOrBlank != null)
    }

    return this == null || this.isBlank()
}

```


**당연히 null이 되리라 예상하지 못한다면 오류를 발생 시켜야 한다** 
* throw, !!, requireNotNull, checkNotNull... 

**!! 를 사용하는건 자바와 같은 문제가 발생** 
지금은 확실하더라도 미래에 확실하지 않게 될수 있기에 조심해야한다 
예를 들어` listOf(a,b,c,d).max() `가 확실히 notnull로 생각하고 !!를 썼는데 리팩토링에 의해 nullable이 나오게되면 추적하기가 힘들어진다  

**최대한 nullable을 지우면 된다**


 
#kotlin 