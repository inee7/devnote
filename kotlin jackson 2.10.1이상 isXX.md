# kotlin jackson 2.10.1이상 isXX 
[kotlin jackson 조합시 isXXX 프로퍼티 스펙변경](https://multifrontgarden.tistory.com/269)
 
jackson-module-kotlin 2.10.0 까지는 data class 를 json 으로 serialize 시 is 로 시작하는 프로퍼티에 대해 is 를 제거한 key 로 serialize 했다. 쉽게 코드를 보고 얘기하자면
```kotlin
data class Person(           
    val isDeveloper: Boolean 
)                            
```
이 클래스를 serialize 시 json 형태는 이런 형태였다.
`{"developer":true}`
jackson-kotlin-module 2.10.1 부터는 이 스펙이 변경되어 isXXX 프로퍼티에대해 is 를 포함하게된다.
`{"isDeveloper":true}`
서버개발에서 kotlin 을 사용시 spring boot 를 조합해서 사용하는 경우가 많을거고, spring boot 는 json 라이브러리로 jackson 을 이용하기때문에 이번 변경이 생각보다 치명적일 수 있다. api 응답 DTO 에 isXXX 프로퍼티가 존재한다면 spring boot 버전을 올리는것 만으로 api 스펙이 변경될 여지가 있기때문이다.
 
jackson 에서 관련이슈는  [https://github.com/FasterXML/jackson-module-kotlin/issues/80](https://github.com/FasterXML/jackson-module-kotlin/issues/80)  를 통해 살펴볼 수 있다. 사실 serialize/deserialize 간에 이슈가 있었고, 자바에서는 is 가 포함돼서 serialize 되고있었기때문에 스펙변경이라기보다는 버그픽스라고 보는게 맞을것 같다. 하지만 기존 스펙(버그)에 기대어 작성된 api 가 있다면 어쩔수없이 해당 부분까지 안고가야한다. 버전을 올리면서 기존 api 스펙을 유지하기위해서는(isXXX 에서 serialize 시 기존대로 is 를 제거하려면) 현재 상태로는 @JsonProperty 를 사용해 명시적으로 key 를 작성해주는 방법밖에 없어보인다.
```kotlin
data class Person(
    @JsonProperty("developer")
    val isDeveloper: Boolean
)
```

#kotlin