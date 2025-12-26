# Full GC 잦을때

아래 목표에 따라 달라진다. 

- 메모리 최적화
- GC 횟수 감소 
- GC 소요 시간 단축 
- 어플리케이션 성능 향상 

### GC로그 수집 방법

- JVM Option 에 GC 로그를 수집하기 위해 `-verbosegc` 옵션을 사용

### Concurrent GC 적용

- "Concurrent GC" 사용 시 Full GC 시간과 횟수를 줄일 수 있다고 하지만 Parallel GC는 New Size가 고정되지 않는 문제가 있으며, Concurrent GC에서도 큰 성능 향상을 기대하기 어렵다고한다.

### Default GC 사용 및 튜닝 시도

**Full GC 분석**:

- 규칙적인 간격 또는 시간에 Full GC 발생 시 -> 1로 진행

- Full GC 빈도가 감소하거나 시간이 증가할 경우 -> 2로 진행
1. **JVM 튜닝**:
   
   1. **NewSize 조정**:
      
      - `old:new` 비율을 2:1 또는 4:1로 설정하여 New 영역을 늘린다.
      - Minor GC 횟수 감소 및 Full GC 횟수 감소에 도움이 된다.
   
   2. **전체 Heap 크기 조정**:
      
      - 전체 Heap 크기를 줄여서 Full GC를 더 자주 수행하도록 유도한다.
      - 사용자 체감 속도가 저하될 수 있으므로 주의가 필요하다.

2. **애플리케이션 문제 추적**:
   
   - 메모리 누수 등의 애플리케이션 문제를 추적하고 해결한다.
   - 캐시 클래스 등이 메모리를 계속 쌓아두어 Full GC 시간이 늘어나는 현상 등을 예로 들며, 툴을 활용하여 문제를 찾는다.
   - jeniffer 같은 툴을 사용하여 보다 쉽게 찾아낼 수 있습니다.

### Full GC 속도 개선 방법

- 트래픽 증가 시 Full GC가 느려지는 상황을 완전히 막을 방법은 없으나, 멈추는 시간을 최소화한다.
- Old 영역을 줄이거나 ParallelGC를 활용하여 성능을 개선한다.
- Old 영역을 줄이면 Full GC 시간이 감소하고, 병렬적으로 GC를 수행하여 성능을 향상시킨다.
- Old영역 줄이는 명령어는 New영역을 지정함으로써 조정가능 Xms2G -Xmx2G -XX:NewSize=1500m -XX:MaxNewSize=1500m의 경우에는 전체 메모리를 2기가로 잡고 New 영역을 1500메가로 잡음으로써 Old영역을 500메가로 잡습니다. 두번째 ParallelCG는 -XX:+UseParallelOldGC 옵션을 줌으로써 조절 가능
#java #gc
