---
tags: [java, gc, garbage-collection, jvm, memory, performance]
---

## 한 줄 요약

GC(Garbage Collection)는 JVM이 힙 영역에서 사용하지 않는 객체를 자동으로 제거하는 메모리 관리 기법으로, Mark and Sweep 방식을 사용하며 애플리케이션 특성에 따라 Serial, Parallel, CMS, G1 등의 GC 알고리즘을 선택할 수 있다.

## 핵심 정리

- GC는 스택이 가리키지 않는 힙 영역의 객체를 자동으로 제거
- Mark and Sweep 방식: Reachable 객체 마킹 후 나머지 제거
- Stop-The-World: GC 실행 시 모든 애플리케이션 스레드 일시 중단
- Young Generation(Eden, S0, S1)과 Old Generation으로 구분
- Minor GC(Young), Major GC(Old), Full GC(전체 Heap)
- GC 종류: Serial(단일 스레드), Parallel(멀티 스레드), CMS(동시 처리), G1(Region 기반)
- Java 8부터 Perm 영역이 Metaspace로 변경
- GC 튜닝: NewSize 조정, Heap 크기 조정, GC 알고리즘 선택

## 상세 내용

### GC(Garbage Collection)란

#### 정의

GC는 JVM의 Garbage Collector가 **스택이 가리키지 않는 힙 영역의 인스턴스를 자동으로 free하는 과정**입니다.

목적:
- 메모리 누수 방지
- 개발자의 수동 메모리 관리 부담 제거
- 프로그램 안정성 향상

### GC 과정: Mark and Sweep

#### Mark 단계

JVM의 Garbage Collector가 **스택의 모든 변수를 스캔**하면서 각각 어떤 객체를 레퍼런스하고 있는지 찾습니다.

동작:
1. 스택의 모든 변수 스캔
2. 각 변수가 참조하는 객체 마킹
3. Reachable 객체가 참조하는 객체도 재귀적으로 마킹

#### Stop-The-World

**Mark 작업을 위해 모든 애플리케이션 스레드가 중단**됩니다.

- GC 실행 시 필수적으로 발생
- STW 시간 최소화가 GC 튜닝의 핵심 목표
- 사용자 체감 성능에 직접적 영향

#### Sweep 단계

**Mark되지 않은 모든 객체를 힙에서 제거**합니다.

특징:
- Garbage를 수집하는 것이 아니라, Garbage가 **아닌 것**을 마크하고 나머지를 제거
- 힙에 Garbage만 가득하다면 제거 과정은 즉각적으로 이루어짐

### 힙 메모리 구조

#### Java 8 이전

```
Heap
├─ Young Generation
│  ├─ Eden
│  ├─ Survivor 0 (S0)
│  └─ Survivor 1 (S1)
├─ Old Generation (Tenured)
└─ Permanent Generation (Perm)
```

#### Java 8 이후

```
Heap
├─ Young Generation
│  ├─ Eden
│  ├─ Survivor 0 (S0)
│  └─ Survivor 1 (S1)
└─ Old Generation (Tenured)

Metaspace (Native Memory)
```

변경 사항:
- **Perm 영역이 Metaspace로 변경**
- Metaspace는 Native Memory 영역에 위치
- 크기 제한 없음 (OS가 허용하는 만큼)

### GC 프로세스

#### 1. 객체 생성

새로운 객체는 **Eden 영역에 할당**됩니다.
- 두 개의 Survivor Space(S0, S1)는 비워진 상태로 시작

#### 2. Minor GC 발생

Eden 영역이 가득 차면 **Minor GC**가 발생합니다.

동작:
- Reachable 객체들은 S0으로 이동
- Unreachable 객체들은 Eden 클리어 시 함께 제거

#### 3. 다음 Minor GC

Eden이 다시 가득 차면:
- Eden의 Unreachable 객체 제거
- Eden의 Reachable 객체 → Survivor Space로 이동
- 기존 S0의 Reachable 객체 → S1으로 이동 (age 증가)
- S0와 Eden 클리어

**중요**: Survivor Space 간 이동 시마다 **age 값 증가**

#### 4. Survivor Space 교대

다음 Minor GC 발생 시:
- S1 → S0으로 이동 (age 증가)
- Eden → S0으로 이동
- Eden과 S1 클리어

**항상 하나의 Survivor Space는 비어있음**

#### 5. Promotion

age 값이 특정 임계값 이상이 되면 **Old Generation으로 승격**됩니다.

- 임계값: 기본 15 (설정 가능)
- 오래 살아남은 객체는 Old로 이동
- Young Generation의 공간 확보

#### 6. Major GC 발생

Promotion이 반복되어 Old Generation이 가득 차면 **Major GC** 발생합니다.

- Old Generation 영역 정리
- STW 시간이 Minor GC보다 길음
- 애플리케이션 성능에 큰 영향

#### 7. Full GC

**Heap 전체(Young + Old)를 정리**합니다.

- 가장 긴 STW 시간
- 최대한 발생 빈도를 줄여야 함
- 성능 문제의 주요 원인

### GC 종류

#### Serial GC

```bash
-XX:+UseSerialGC
```

특징:
- **단일 스레드** 기반 순차 처리
- Java SE 5, 6에서 기본 GC
- Mark-Compact 방식 사용
- Minor GC, Major GC 모두 STW 후 순차 실행

**Mark-Compact**:
- 살아있는 객체들을 힙의 시작 위치로 이동
- 메모리 단편화 방지
- 새로운 메모리 할당을 빠르게 처리

적합한 환경:
- CPU 코어가 1개인 환경
- 힙 크기가 작은 환경 (수백 MB 이하)

#### Parallel GC

```bash
-XX:+UseParallelGC
```

특징:
- Minor GC 시 **멀티 스레드** 활용
- **CPU 2개 이상** 환경에서 효율적
- 처리량(Throughput) 중심

옵션:
```bash
# GC 스레드 수 조절
-XX:ParallelGCThreads=<N>

# Old Generation도 병렬 GC 사용
-XX:+UseParallelOldGC
```

적합한 환경:
- 멀티 코어 환경
- 배치 작업처럼 처리량이 중요한 경우
- STW 시간보다 전체 처리량이 중요한 경우

#### CMS (Concurrent Mark Sweep) GC

```bash
-XX:+UseConcMarkSweepGC
```

특징:
- **Concurrent Low Pause Collector**
- Stop-the-World 시간 **최소화**가 목표
- 대부분의 GC 작업을 애플리케이션 스레드와 **동시에 수행**
- Young Generation은 Parallel GC 방식 사용

옵션:
```bash
# CMS 스레드 수 조절
-XX:ParallelCMSThreads=<N>
```

장점:
- 짧은 응답 시간
- Low Latency

단점:
- **Compact 작업을 수행하지 않음**
- **메모리 단편화(Fragmentation)** 발생 가능
- 단편화 방지를 위해 더 큰 힙 사이즈 필요
- CPU 리소스를 더 많이 사용

적합한 환경:
- 응답 시간이 중요한 웹 애플리케이션
- STW 시간을 최소화해야 하는 경우

#### G1 (Garbage First) GC

```bash
-XX:+UseG1GC
```

특징:
- Java 7부터 사용 가능
- **CMS GC를 대체**하기 위해 설계
- 힙을 **Region 단위로 분할**하여 관리
- **병렬 처리 및 예측 가능한 GC 시간** 확보

동작 방식:
- 전체 힙을 동일한 크기의 Region으로 나눔
- 각 Region은 Eden, Survivor, Old 역할 중 하나
- Garbage가 많은 Region부터 우선 수집 (Garbage First)

장점:
- 큰 힙에서도 예측 가능한 STW 시간
- 메모리 단편화 해결 (Compaction 수행)
- 튜닝 옵션이 적어 관리 편의성

적합한 환경:
- **대용량 힙 (4GB 이상)**
- Java 9 이후 기본 GC
- 대부분의 현대 애플리케이션

### Full GC 문제 분석 및 튜닝

#### Full GC가 잦을 때 목표

- 메모리 최적화
- GC 횟수 감소
- GC 소요 시간 단축
- 애플리케이션 성능 향상

#### GC 로그 수집

```bash
# GC 로그 수집
-verbose:gc

# 상세 로그 (Java 8)
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/path/to/gc.log

# 상세 로그 (Java 9+)
-Xlog:gc*:file=/path/to/gc.log
```

#### Full GC 분석

**1. 규칙적인 간격 또는 시간에 Full GC 발생 시**

→ JVM 튜닝 진행

**a. NewSize 조정**

```bash
# Old:New 비율을 2:1 또는 4:1로 설정
-XX:NewRatio=2  # Old:New = 2:1
-XX:NewRatio=4  # Old:New = 4:1
```

효과:
- New 영역을 늘림
- Minor GC 횟수 감소
- Full GC 횟수 감소

**b. 전체 Heap 크기 조정**

```bash
# 전체 Heap 크기 설정
-Xms2G -Xmx2G

# New 영역 크기 설정
-XX:NewSize=1500m
-XX:MaxNewSize=1500m
```

예시:
```bash
# Old 영역 500MB, New 영역 1500MB
-Xms2G -Xmx2G -XX:NewSize=1500m -XX:MaxNewSize=1500m
```

주의:
- 전체 Heap을 줄이면 Full GC가 더 자주 발생
- 사용자 체감 속도 저하 가능성

**2. Full GC 빈도가 감소하거나 시간이 증가하는 경우**

→ 애플리케이션 문제 추적

원인:
- 메모리 누수
- 캐시 클래스가 메모리를 계속 쌓아둠
- 비정상적인 객체 생성 패턴

도구:
- jProfiler
- VisualVM
- JMC (Java Mission Control)
- jennifer (APM 도구)

#### Full GC 속도 개선 방법

**1. Old 영역 줄이기**

```bash
# New 영역을 크게 설정하여 Old 영역 축소
-Xms2G -Xmx2G -XX:NewSize=1500m -XX:MaxNewSize=1500m
```

효과:
- Full GC 시간 감소
- Old에 쌓이는 객체 수 감소

**2. Parallel Old GC 사용**

```bash
-XX:+UseParallelOldGC
```

효과:
- Old Generation GC를 병렬로 처리
- STW 시간 단축

#### Concurrent GC 고려사항

**CMS GC 사용 시**:
- Full GC 시간과 횟수 감소 효과
- 하지만 큰 성능 향상 기대하기 어려움

**Parallel GC 문제점**:
- New Size가 고정되지 않는 문제
- 예측 가능성 저하

**권장**:
- 기본 GC 사용 후 튜닝 시도
- 성능 측정 후 Concurrent GC 적용 여부 결정

## 실무 적용

### GC 모니터링 체크리스트

- [ ] GC 로그를 수집하고 분석하고 있는가?
- [ ] Full GC 빈도와 STW 시간을 모니터링하는가?
- [ ] 힙 메모리 사용률을 주기적으로 확인하는가?
- [ ] Old Generation 증가 추세를 파악하는가?
- [ ] GC 알고리즘이 애플리케이션 특성에 적합한가?

### GC 튜닝 전략

**1단계: 현재 상태 파악**
- GC 로그 수집 및 분석
- Full GC 빈도, STW 시간 측정
- 메모리 사용 패턴 파악

**2단계: 목표 설정**
- 허용 가능한 STW 시간 정의
- Full GC 빈도 목표 설정
- 처리량 vs 응답시간 우선순위 결정

**3단계: 튜닝 적용**
- Heap 크기 조정
- New/Old 비율 조정
- GC 알고리즘 선택

**4단계: 모니터링 및 검증**
- 성능 측정
- 문제 발생 시 롤백
- 점진적 튜닝

### 금융권에서의 고려사항

- **거래 처리 안정성**: STW 시간 최소화 필수 (거래 지연 방지)
- **대용량 데이터 처리**: 배치 작업 시 Parallel GC 고려
- **실시간 서비스**: G1 GC 또는 CMS GC 권장 (낮은 Latency)
- **메모리 누수 관리**: 정기적인 힙 덤프 분석, 메모리 프로파일링
- **GC 로그 보관**: 문제 발생 시 원인 분석을 위한 로그 필수
- **모니터링**: APM 도구로 실시간 GC 모니터링
- **장애 대응**: Full GC로 인한 서비스 지연 시 알림 설정

### GC 알고리즘 선택 가이드

| 환경 | 권장 GC | 이유 |
|------|---------|------|
| 소규모 애플리케이션 (< 100MB) | Serial GC | 단순하고 오버헤드 적음 |
| 배치 작업 (처리량 중시) | Parallel GC | 높은 처리량 |
| 웹 서비스 (응답시간 중시) | G1 GC | 예측 가능한 STW |
| 대용량 힙 (> 4GB) | G1 GC | Region 기반 효율적 관리 |
| 레거시 시스템 (Java 7 이하) | CMS GC | Low Latency |

---

**출처**
- [자바 메모리 관리 - 가비지 컬렉션](https://yaboong.github.io/java/2018/06/09/java-garbage-collection/)
- [Oracle GC Tuning Guide](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)
- [Getting Started with the G1 Garbage Collector](http://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)
