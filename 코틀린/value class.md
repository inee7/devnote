# value class

`Kotlin 1.5`
 버전이 릴리즈 되면서 `value class` 추가

data class로 이뤄진 Money의 경우
![](Untitled.png)
![](Untitled%201.png)value class로 변경 
![](Untitled%202.png)
![](Untitled%203.png)

```
❗data class 와 차이점 
1. copy(), componentN() 자동 생성 안함
2. ===연산 지원안함 
3. val 프로퍼티만 허용
4. 아직은 하나의 프로퍼티만 허용
```

#dev/kotlin