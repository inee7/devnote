#  GC종류
### Serial GC
-XX:+UseSerialGC 옵션을 줘서 사용할 수 있는 Serial GC 는 Java SE 5, 6 에서 사용되는 디폴트 가비지 컬렉터이다.
* MinorGC, MajorGC 모두 순차적으로 시행된다.
* Mark-Compact collection method 를 사용한다.

Mark-Compact collection method 란, 새로운 메모리 할당을 빠르게 하기 위해서 기존의 메모리에 있던 오브젝트들을 힙의 시작위치로 옮겨 놓는 방법이다. 창고에서 필요없는 물건들을 버린 후, 창고에 물건을 차곡차곡 쌓기위해 창고안을 정리하는 것이라 생각할 수 있다.    [http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html) 

### Parallel GC
-XX:+UseParallelGC 옵션으로 사용 가능한 Parallel 가비지 컬렉터는 young generation 에 대한 가비지 컬렉션 수행시 멀티스레드를 사용한다. 멀티스레딩을 할 수 있는 ParallelGC 를 사용하도록 옵션을 주었더라도, 호스트 머신이 싱글 CPU 라면 디폴트 가비지 컬렉터(Serial GC)가 사용된다. 하지만, 호스트의 CPU 가 두개 이상이라면 young generation 의 가비지 컬렉션 시간을 줄일 수 있다.
가비지 컬렉터 스레드 개수는 디폴트로 CPU 개수만큼이 할당되는데 -XX:ParallelGCThread=<N> 옵션으로 조절가능하다. 또한, -XX:+UseParallelOldGC 옵션을 사용한다면, old generation 의 가비지 컬렉션에서도 멀티스레딩을 활용할 수 있다.

### Concurrent Mark Sweep (CMS) Collector
Concurrent Low Pause Collector 라고도 불리는 CMS 컬렉터는 -XX:+UseConcMarkSweepGC 옵션으로 사용할 수 있다. 대부분의 가비지 컬렉션 작업을 애플리케이션 스레드와 동시에 수행함으로써 가비지 컬렉션으로 인한 stop-the-world 시간을 최소화하는 GC이다.
CMS 컬렉터는 young generation 에 대한 가비지 컬렉션시 Parallel GC 와 같은 알고리즘을 사용하는데, -XX:ParallelCMSThreads=<N> 옵션으로 스레드 개수를 설정할 수 있다.
일반적으로 CMS 컬렉터는 살아있는 오브젝트들에 대한 compact 작업을 수행하지 않으므로, 메모리의 파편화(Fragmentation) 가 문제가 된다면 더 큰 힙사이즈를 할당해야 한다.

### G1 Garbage Collector
Garbage First 라는 의미의 G1 가비지 컬렉터는 Java 7 부터 사용가능하며, 장기적으로 CMS 컬렉터를 대체하기 위해 만들어졌다. -XX:+UseG1GC 옵션으로 사용가능하다. G1 가비지 컬렉터는 이전까지 이야기한 것들과는 다른 방식으로 동작한다. G1 GC 에 대한 자세한 내용은 ~[[Oracle] Getting Started with the G1 Garbage Collector](http://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)~ 를 참고하기 바란다.