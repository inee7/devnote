---
tags: [kotlin, resource-management, try-with-resources]
---

# finally close -> use 로 간단히 
Closeable/AutoCloseable 구현한 객체를 쉽고 안전하게  
use를 써서 해결 할수 있다 
```kotlin
BufferedReader(FileReader(path)).use { reader -> 
	return reader.lineSequence().sumBy {it.length}
}
```