---
tags: [jpa, entity-manager, persistence-context]
---

# entityManager 모방 코드 

```kotlin
class SimpleEntityManager {
    // 영속성 컨텍스트를 모방하기 위한 Map
    private val persistenceContext: MutableMap<Long, User> = mutableMapOf()
    // 데이터베이스를 모방하기 위한 Map
    private val database: MutableMap<Long, User> = mutableMapOf()

    init {
        // 데이터베이스 초기화 예시
        database[1] = User(id = 1, name = "John Doe", age = 30)
        database[2] = User(id = 2, name = "Jane Smith", age = 25)
    }

    // 엔티티를 병합하는 메소드
    fun merge(user: User): User {
        val existingUser = persistenceContext[user.id] ?: database[user.id]

        return if (existingUser != null) {
            existingUser.name = user.name
            existingUser.age = user.age
            persistenceContext[user.id] = existingUser
            existingUser
        } else {
            persistenceContext[user.id] = user
            database[user.id] = user
            user
        }
    }

    // 엔티티를 ID로 찾는 메소드
    fun find(id: Long): User? {
        return persistenceContext[id] ?: database[id]
    }

    // 현재 영속성 컨텍스트 상태를 출력하는 메소드
    fun printPersistenceContext() {
        println("Persistence Context:")
        persistenceContext.values.forEach { println("User(id=${it.id}, name=${it.name}, age=${it.age})") }
    }
}

data class User(
    val id: Long,
    var name: String,
    var age: Int
)

fun main() {
    val entityManager = SimpleEntityManager()

    // 엔티티 찾기 (데이터베이스에서 가져오기)
    val foundUser = entityManager.find(1)
    println("Found User: ${foundUser?.name}")

    // 새 엔티티 병합 (엔티티가 데이터베이스에 없는 경우)
    val newUser = User(id = 3, name = "Alice Johnson", age = 28)
    entityManager.merge(newUser)

    entityManager.printPersistenceContext()
}
```

## 관련 노트

- [[JPA-영속성-관리]] - EntityManager와 영속성 컨텍스트 상세 설명