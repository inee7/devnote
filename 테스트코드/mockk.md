
mockk 에서 mock 을 relexed = true하면 모킹 안해도 되고 verify 로 호출됐는지 확인으로 끝낼수 있다

verify {객체변수 WasNot Called } 하면 호출 되지 않음을 알수 있다

## 람다를 mocking 할때
~~~kotlin
class TestClass(val mockTarget:MockTarget) {
  fun f():String {
    return mockTarget.start { "$it 굳" }
  }
}

class MockTarget {
  fun start(onSuccess: String -> String): String {
	return onSucess(generaterFromExternal())
  }
}
~~~
Engine.start를 모킹해야하는데 인자가 람다라면 
~~~kotlin
val slotOnSuccess = slot<String->String>()

every { engine.start( capture(slotOnSuccess)) } answers {
  slotOnSuccess.captured.invoke("success")
}

val result = testClass.f()
assertEquals(result, "success 굳")
~~~
그럼 start 실행될때 onSucess(generaterFromExternal())가 모킹되어 onSuccess에 파라미터 값으로 “success” 가 가게 되고 그 밖 코드 기준 그대로 따라 가게 된다 
