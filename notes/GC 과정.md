# GC 과정 
[자바 메모리 관리 - 가비지 컬렉션](https://yaboong.github.io/java/2018/06/09/java-garbage-collection/)

스택이 가르치지 않는 힙영역의 인스턴스를 free한다.
Garbage Collection 과정은 Mark and Sweep 이라고도 한다.
JVM의 Garbage Collector 가 스택의 모든 변수를 스캔하면서 각각 어떤 오브젝트를 레퍼런스 하고 있는지 찾는과정이 Mark 다.
Reachable 오브젝트가 레퍼런스하고 있는 오브젝트 또한 marking 한다.
첫번째 단계인 marking 작업을 위해 모든 스레드는 중단되는데 이를 stop the world 라고 부르기도 한다.
그리고 나서 mark 되어있지 않은 모든 오브젝트들을 힙에서 제거하는 과정이 Sweep 이다.
Garbage Collection 이라고 하면 garbage 들을 수집할 것 같지만 실제로는 garbage 를 수집하여 제거하는 것이 아니라, garbage 가 아닌 것을 따로 mark 하고 그 외의 것은 모두 지우는 것이다. 만약 힙에 garbage 만 가득하다면 제거 과정은 즉각적으로 이루어진다.
JAVA8부터는 perm영역이 metaspace영역으로 바꼈음


![[resources/images/gc-area.png]]

### GC 프로세스
1. 새로운 오브젝트는 Eden 영역에 할당된다. 두개의 Survivor Space 는 비워진 상태로 시작한다.

2. Eden 영역이 가득차면, MinorGC 가 발생한다.

3. MinorGC 가 발생하면, Reachable 오브젝트들은 S0 으로 옮겨진다. Unreachable 오브젝트들은 Eden 영역이 클리어 될때 함께 메모리에서 사라진다.

4. 다음 MinorGC 가 발생할때, Eden 영역에는 3번과 같은 과정이 발생한다. Unreachable 오브젝트들은 지워지고, Reachable 오브젝트들은 Survivor Space 로 이동한다. 기존에 S0 에 있었던 Reachable 오브젝트들은 S1 으로 옮겨지는데, 이때, age 값이 증가되어 옮겨진다. 살아남은 모든 오브젝트들이 S1 으로 모두 옮겨지면, S0 와 Eden 은 클리어 된다.

Survivor Space 에서 Survivor Space 로의 이동은 이동할때마다 age 값이 증가한다.

5. 다음 MinorGC 가 발생하면, 4번 과정이 반복되는데, S1 이 가득차 있었으므로 S1 에서 살아남은 오브젝트들은 S0 로 옮겨지면서 Eden 과 S1 은 클리어 된다. 이때에도, age 값이 증가되어 옮겨진다.

Survivor Space 에서 Survivor Space 로의 이동은 이동할때마다 age 값이 증가한다.

6. Young Generation 에서 계속해서 살아남으며 age 값이 증가하는 오브젝트들은 age 값이 특정값 이상이 되면 Old Generation 으로 옮겨지는데 이 단계를 Promotion 이라고 한다.

7. MinorGC 가 계속해서 반복되면, Promotion 작업도 꾸준히 발생하게 된다.

8. Promotion 작업이 계속해서 반복되면서 Old Generation 이 가득차게 되면 MajorGC 가 발생하게 된다.

* MinorGC : Young Generation 에서 발생하는 GC
* MajorGC : Old Generation (Tenured Space) 에서 발생하는 GC
* FullGC : Heap 전체를 clear 하는 작업 (Young/Old 공간 모두)

#java #gc 