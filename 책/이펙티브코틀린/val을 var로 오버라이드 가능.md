# val을 var로 오버라이드 가능 

``` kotlin
interface Element {
	val active: Boolean
}

class AElement: Element {
	override var active: Boolean = false 
}
```

#kotlin
