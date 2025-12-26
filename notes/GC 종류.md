# **Java Garbage Collector 종류 정리**

---

## **🔹 Serial GC**

- **옵션**: -XX:+UseSerialGC
    
- **기본 적용**: Java SE 5, 6에서 기본 가비지 컬렉터
    
- **방식**: 단일 스레드 기반의 **순차 처리 GC**
    
    - Minor GC, Major GC 모두 **Stop-The-World(STW)** 후 순차적으로 실행됨
        
    - **Mark-Compact 방식** 사용
        
    

  

### **📌 Mark-Compact란?**

- 살아 있는 객체들을 **힙의 시작 위치로 이동**시켜 새로운 메모리 할당을 빠르게 만드는 방식
    
- 창고에 남은 물건들을 앞으로 밀어 넣고 빈 공간을 확보하는 정리 방식과 유사
    
- [참고 링크 🔗](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)
    

---

## **🔹 Parallel GC**

- **옵션**: -XX:+UseParallelGC
    
- **특징**: Minor GC 시 **멀티 스레드** 활용
    
- **CPU 2개 이상** 환경에서 효율적 (단일 CPU면 Serial GC 사용됨)
    
- **스레드 수 조절**:
    
    - -XX:ParallelGCThreads=<N> 옵션 사용 가능
        
    
- **Old Generation에도 병렬 GC 사용**:
    
    - -XX:+UseParallelOldGC 옵션 설정 시
        
    

---

## **🔹 CMS (Concurrent Mark Sweep) GC**

- **옵션**: -XX:+UseConcMarkSweepGC
    
- **별칭**: **Concurrent Low Pause Collector**
    
- **목표**: Stop-the-World 시간을 **최소화**
    
- **특징**:
    
    - 대부분의 GC 작업을 **애플리케이션 스레드와 동시에 수행**
        
    - Young Generation은 Parallel GC 방식 사용
        
    - -XX:ParallelCMSThreads=<N> 로 스레드 수 조정 가능
        
    
- **주의점**:
    
    - Compact 작업(조각 정리)을 수행하지 않기 때문에 **메모리 단편화(Fragmentation)** 발생 가능
        
    - 단편화를 방지하려면 **더 큰 힙 사이즈**가 필요할 수 있음
        
    

---

## **🔹 G1 (Garbage First) GC**

- **옵션**: -XX:+UseG1GC
    
- **도입 시점**: Java 7부터 사용 가능
    
- **설계 목적**: **CMS GC를 대체**하기 위한 새로운 GC
    
- **특징**:
    
    - 힙을 Region 단위로 나누어 **병렬 처리 및 예측 가능한 GC 시간 확보**
        
    - 기존 GC 방식(Mark-Compact, Sweep 등)과는 다른 내부 구조 사용
        
    
- **자세한 설명**:
    
    - [Getting Started with the G1 Garbage Collector (Oracle)](http://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)



#java #gc 