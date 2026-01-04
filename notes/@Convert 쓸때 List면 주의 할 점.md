---
tags: [jpa, hibernate, attributeconverter, dirty-checking]
---

# @Convert 쓸때 List면 주의 할 점

## 한 줄 요약

JPA `@Convert`로 List 타입 변환 시 기존 리스트에 add하지 말고 새로운 리스트 객체를 할당해야 변경 감지가 정상 작동한다

## 핵심 정리

- `@Convert`를 사용한 컬렉션 타입은 영속성 컨텍스트가 변경을 추적하지 못함
- 기존 리스트에 `add()` 하면 더티 체킹(Dirty Checking)이 작동하지 않음
- **새로운 리스트 객체를 생성해서 할당**해야 JPA가 변경을 감지함

## 상세 내용

### 문제 상황

```java
@Entity
public class Article {
    @Convert(converter = TagListConverter.class)
    private List<String> tags = new ArrayList<>();

    public void addTag(String tag) {
        this.tags.add(tag);  // ❌ 변경 감지 안 됨
    }
}
```

위 코드에서 `addTag()`를 호출해도 JPA는 변경을 감지하지 못하고 DB에 저장되지 않는다.

### 왜 이런 문제가 발생하는가?

JPA의 더티 체킹은 **객체의 참조 변경**을 감지한다. `@Convert`로 변환되는 컬렉션은 JPA가 관리하는 특수한 컬렉션(PersistentBag 등)이 아니라 일반 컬렉션이므로:

1. 내부 요소가 변경되어도 참조는 동일함
2. JPA는 컬렉션 내부까지 깊은 비교를 하지 않음
3. 결과적으로 변경을 감지하지 못함

### 해결 방법

```java
@Entity
public class Article {
    @Convert(converter = TagListConverter.class)
    private List<String> tags = new ArrayList<>();

    public void addTag(String tag) {
        List<String> newTags = new ArrayList<>(this.tags);
        newTags.add(tag);
        this.tags = newTags;  // ✅ 새 객체 할당으로 변경 감지됨
    }

    public void setTags(List<String> tags) {
        this.tags = new ArrayList<>(tags);  // ✅ 방어적 복사
    }
}
```

## 실무 적용

### 대안 1: @ElementCollection 사용

간단한 컬렉션이라면 `@Convert` 대신 `@ElementCollection` 사용 고려:

```java
@Entity
public class Article {
    @ElementCollection
    @CollectionTable(name = "article_tags")
    private List<String> tags = new ArrayList<>();

    public void addTag(String tag) {
        this.tags.add(tag);  // ✅ @ElementCollection은 변경 감지됨
    }
}
```

**단점**: 별도 테이블 생성, 성능 이슈 가능성

### 대안 2: JSON 직렬화 + 불변 리스트

```java
@Entity
public class Article {
    @Convert(converter = JsonListConverter.class)
    @Column(columnDefinition = "TEXT")
    private List<String> tags = new ArrayList<>();

    public void updateTags(List<String> newTags) {
        this.tags = List.copyOf(newTags);  // 불변 리스트로 재할당
    }
}
```

### 주의사항

- 성능: 리스트를 매번 새로 생성하므로 대용량 컬렉션에서는 부담
- 동시성: 멀티스레드 환경에서는 불변 컬렉션 사용 권장
- Converter 구현 시 null 처리 필수

## 관련 노트

- [[JPA-영속성-관리]] - 변경 감지(Dirty Checking) 원리
- [[JPA-엔티티-매핑]] - @Embeddable과 값 타입 불변성
