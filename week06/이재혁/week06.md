# 자바 최적화 2판 13장: 동시성 성능 기법 정리

2026-10-26

## 개요

13장의 결론은 하나다. 동시성 성능은 **직렬 구간과 공유 가변 상태를 얼마나 줄이느냐**로 정해진다. 장은 저수준 메모리 보장에서 시작해, 그 위의 도구, 작업 분배, 그리고 락을 피하거나 스레드 모델을 바꾸는 최신 기법 순으로 쌓아 올린다.

1. 병렬 처리 기초와 JMM(Java Memory Model, 자바 메모리 모델): 왜 어렵고, 무엇이 보장되는가
2. VarHandle, Atomic, CAS(Compare-And-Swap): 모든 도구의 토대가 되는 원자 연산
3. `java.util.concurrent`: 락, 동시성 컬렉션, 래치와 배리어
4. Executor, Fork/Join, 병렬 스트림: 작업을 나눠 맡기는 방법
5. 락-프리, 액터, 가상 스레드: 락을 피하거나 스레드 모델 자체를 바꾸기

책 원문이 아닌 이 장이 다루는 주제를 기준으로 설명과 퀴즈로 정리한 노트다. 절 구성은 책과 다를 수 있다.

## 파트 1. 병렬 처리 기초와 JMM

### 왜 동시성인가

CPU 클럭은 발열과 전력 문제로 정체됐고, 제조사는 코어 수를 늘리는 쪽으로 갔다. 이제 코드는 여러 코어를 쓰도록 짜야 빨라진다.

### 암달의 법칙 (Amdahl's Law)

P = 병렬화 가능 비율, N = 코어 수일 때 속도 향상은 다음과 같다.

```
S(N) = 1 / ((1 − P) + P/N)
```

- 직렬 구간 (1 − P)가 상한을 정한다. P = 0.95면 코어가 무한해도 최대 20배, P = 0.9면 10배다.
- 튜닝 교훈: 코어를 더 붙이기보다 **락 경합 같은 직렬 구간을 줄여라**.
- 공유 상태가 거의 없어 나누기만 하면 되는 작업은 처치 곤란 병렬(embarrassingly parallel)이라 부르고, 거의 선형으로 확장된다.

### 공유 가변 상태

`count++`는 읽기 → 더하기 → 쓰기 세 단계다. 두 스레드가 같은 값을 읽으면 한쪽 증가분이 사라진다(갱신 유실, lost update). 문제의 근원은 **공유 + 가변**이고, 불변 객체나 스레드 한정으로 둘 중 하나만 없애도 사라진다.

### JMM (Java Memory Model)

CPU마다 메모리 일관성 수준이 다르고(x86은 강함, ARM은 약함), CPU와 JIT(Just-In-Time) 컴파일러는 명령을 재정렬하고 값을 캐시에 둔다. JMM은 이를 추상화한 자바 차원의 계약이다.

- **happens-before**: A가 B보다 happens-before면 B는 A의 결과를 반드시 본다. 이 관계가 없으면 보장도 없다.
- **synchronizes-with**: 같은 락의 해제 → 획득, volatile 쓰기 → 이후 읽기에서 스레드 간 happens-before가 생긴다.
- **as-if-serial**: 단일 스레드 안에서 결과만 같으면 재정렬을 허용한다. 다른 스레드는 그 재정렬을 관찰할 수 있다.

| 도구 | 가시성 | 원자성/상호배제 |
| --- | --- | --- |
| `synchronized` | O (해제 → 획득 시 보장) | O |
| `volatile` | O | X — `count++`는 여전히 깨짐 |
| `final` 필드 | 생성자 완료 후 안전하게 공개 | 불변이라 불필요 |

**함정**: volatile은 "보이는 것"만 보장하고 "쪼개지지 않는 것"은 보장하지 않는다. 상태 플래그에는 맞고 카운터에는 틀리다.

## 파트 2. VarHandle, Atomic, CAS

### Unsafe에서 VarHandle로

예전 JDK 라이브러리는 비공식 내부 API `sun.misc.Unsafe`로 CAS와 메모리 접근을 했다. Java 9의 **VarHandle**은 이를 대체하는 타입 안전한 공식 API다.

```java
private static final VarHandle COUNT = MethodHandles.lookup()
    .findVarHandle(Counter.class, "count", int.class);
COUNT.compareAndSet(this, expected, newValue);
```

메모리 순서 모드를 고를 수 있다. 약한 순서부터 plain → opaque → acquire/release → volatile이며, 필요한 만큼만 보장을 걸어 비용을 줄인다.

### CAS (Compare-And-Swap)

"현재 값이 기대값과 같으면 새 값으로 바꾸고 true, 아니면 false"를 한 번에 수행하는 하드웨어 원자 명령이다(x86의 `LOCK CMPXCHG`). 낙관적 방식으로, 실패하면 다시 읽고 재시도한다.

```java
int prev, next;
do {
    prev = value;
    next = prev + 1;
} while (!CAS(value, prev, next));
```

`AtomicInteger.incrementAndGet()`이 개념적으로 이 구조이며, JIT는 x86의 `LOCK XADD` 같은 단일 명령으로 인트린직 처리한다.

| | 락(블로킹) | CAS(락-프리) |
| --- | --- | --- |
| 실패 시 | OS 수준 대기, 컨텍스트 스위칭 | 즉시 재시도(CPU 사용) |
| 경합 낮음 | 오버헤드 있음 | 매우 빠름 |
| 경합 높음 | 줄 서서 처리 | 재시도 폭주, CPU 낭비 |

- **ABA 문제**: 값이 A → B → A로 바뀌면 CAS는 변경이 없었다고 판단한다. 참조 기반 락-프리 자료구조에서 버그가 된다. 버전을 함께 비교하는 `AtomicStampedReference`로 해결한다.
- **LongAdder (Java 8)**: 값을 여러 셀로 쪼개 스레드마다 다른 셀을 증가시키고 `sum()`에서 합친다. 쓰기 경합이 크게 줄지만 `sum()`은 정확한 순간 스냅샷이 아니고 메모리를 더 쓴다. 메트릭 카운터에 적합하다.

### 스핀 락과 synchronized 내부

CAS로 가장 단순한 락을 만들 수 있다. 잠들지 않고 돌며 기다리는 **스핀 락**이다.

```java
while (!locked.compareAndSet(false, true)) {
    Thread.onSpinWait();  // Java 9+, x86 PAUSE 힌트
}
```

임계 구역이 아주 짧으면 OS 수준 대기보다 싸고, 길면 CPU만 태운다. JVM의 `synchronized`도 단계적으로 동작한다.

1. 경량 락: 경합이 없으면 객체 헤더에 CAS 한 번으로 획득
2. 적응형 스피닝: 경합 시 잠깐 스핀, 최근 성공 이력에 따라 시도 시간 조절
3. 팽창: 그래도 안 되면 무거운 모니터로 바뀌고 OS 수준 대기

편향 락(biased locking)은 JDK 15에서 비활성화·사용 중단됐고 이후 제거됐다.

## 파트 3. java.util.concurrent: 락, 컬렉션, 래치와 배리어

### ReentrantLock

| 기능 | synchronized | ReentrantLock |
| --- | --- | --- |
| 재진입 | O | O |
| `tryLock` (시도 후 포기) | X | O |
| 타임아웃 | X | O |
| 대기 중 인터럽트 (`lockInterruptibly`) | X | O |
| 공정성 옵션 | X | O |
| 조건 변수 | 1개 (`wait/notify`) | 여러 개 (`Condition`) |
| 해제 | 블록 끝에서 자동 | `finally`에서 직접 `unlock()` |

- 기본값은 **비공정**이다. 공정 락은 대기 순서를 지키느라 잠든 스레드를 깨워야 해서 컨텍스트 스위칭이 늘고 처리량이 떨어진다.
- **AQS(AbstractQueuedSynchronizer)**: ReentrantLock, Semaphore, CountDownLatch의 공통 토대다. `int state` 하나와 대기 큐로 구성되고 state를 CAS로 바꾼다. state는 락이면 보유 횟수, 세마포어면 남은 허가 수, 래치면 남은 카운트다.

### 읽기/쓰기 락

- **ReentrantReadWriteLock**: 읽기는 여럿이 동시에, 쓰기는 하나만 가능하다. 읽기 락을 쥔 채 쓰기 락으로 업그레이드하면 **데드락**이 난다(다운그레이드는 가능). 읽기가 끊이지 않으면 쓰기가 굶을 수 있다.
- **StampedLock (Java 8)**: 락 없이 스탬프만 받아 읽고 `validate(stamp)`로 확인하는 낙관적 읽기를 지원한다. 실패하면 진짜 읽기 락을 잡는다. 빠르지만 재진입이 안 된다.

### Semaphore

허가 N개짜리 카운터로, `acquire()`와 `release()`로 **동시 접근 수를 제한**한다(외부 API 동시 호출 제한, 커넥션 풀 등). 소유자 개념이 없어 다른 스레드가 release할 수도 있다.

### 동시성 컬렉션

`Collections.synchronizedMap()`은 전체에 락 하나라 모든 접근이 직렬화된다. 동시성 컬렉션은 이를 피한다.

**ConcurrentHashMap**

- Java 7은 세그먼트(기본 16개) 단위 락, Java 8 이후는 버킷 단위다. 빈 버킷은 CAS로 넣고, 차 있으면 첫 노드에 `synchronized`를 건다.
- 한 버킷 충돌이 8개를 넘고 테이블이 64 이상이면 리스트를 트리로 바꾼다.
- 읽기는 대부분 락이 없고, `size()`는 LongAdder 같은 분산 카운터로 계산한다.
- null 키/값 금지: `get()`의 null이 "없음"인지 "값이 null"인지 구분할 수 없어서다.
- 반복자는 약한 일관성이라 `ConcurrentModificationException`이 없다.
- 복합 연산은 `get` → `put`으로 나누지 말고 `putIfAbsent`, `computeIfAbsent`, `merge`를 쓴다.

**CopyOnWriteArrayList**: 쓰기마다 배열 전체를 복사한다. 읽기는 락이 없고 반복자는 스냅샷을 순회한다. 읽기가 압도적이고 쓰기가 드문 경우(이벤트 리스너 목록)에 쓴다.

**BlockingQueue**: 비면 `take()`, 차면 `put()`이 블로킹된다. 생산자-소비자 패턴의 기본이다. `ArrayBlockingQueue`는 크기 고정, `LinkedBlockingQueue`는 크기를 안 정하면 사실상 무제한이라 메모리 폭주 위험이 있다.

### 래치와 배리어

| | CountDownLatch | CyclicBarrier |
| --- | --- | --- |
| 개념 | 카운트가 0이 될 때까지 대기 | N개 스레드가 다 모일 때까지 서로 대기 |
| 대기 주체 | 보통 메인 스레드가 작업 스레드를 기다림 | 참여 스레드끼리 |
| 재사용 | 불가 (일회용) | 가능 |
| 부가 기능 | — | 모두 도착 시 배리어 액션 실행 |
| 예시 | 초기화 작업 5개 완료 후 서버 시작 | 시뮬레이션 매 단계 동기화 |

## 파트 4. Executor, Fork/Join, 병렬 스트림

### 왜 Executor인가

요청마다 `new Thread()`를 하면 생성 비용이 크고, 스레드 수에 상한이 없으며, 작업과 실행 정책이 섞인다. Executor 프레임워크는 작업(`Runnable`/`Callable`)과 실행 정책(`ExecutorService`)을 분리한다.

### ThreadPoolExecutor

| 파라미터 | 의미 |
| --- | --- |
| corePoolSize | 기본 유지 스레드 수 |
| maximumPoolSize | 최대 스레드 수 |
| workQueue | 스레드가 바쁠 때 작업이 기다리는 큐 |
| keepAliveTime | core 초과 스레드가 놀면 정리되는 시간 |
| RejectedExecutionHandler | 큐도 차고 max도 찼을 때의 정책 |

작업이 들어올 때의 순서:

1. 실행 중 스레드 < core → 새 스레드 생성
2. core가 다 찼으면 → 큐에 적재
3. 큐가 꽉 찼으면 → max까지 스레드 추가
4. 그래도 안 되면 → 거부 정책 (`AbortPolicy`는 예외, `CallerRunsPolicy`는 호출 스레드가 실행해 배압 효과)

**큐가 꽉 차야만 max까지 늘어난다.** 큐가 무제한이면 maximumPoolSize는 의미가 없다.

**`Executors` 팩토리 함정**

- `newFixedThreadPool(n)`: 무제한 `LinkedBlockingQueue` → 작업이 쌓여 OOM(OutOfMemoryError) 위험
- `newCachedThreadPool()`: max가 사실상 무제한 → 스레드 무한 생성 위험
- 운영 환경에서는 `ThreadPoolExecutor`를 직접 만들어 크기 제한 큐와 거부 정책을 명시한다.

### 스레드 풀 크기

- CPU 집약 작업: 코어 수(또는 + 1)
- I/O 집약 작업: 아래 경험식이 출발점이다. 예: 8코어, 계산 20ms, 대기 80ms → 8 × (1 + 4) = 40

```
스레드 수 ≈ 코어 수 × (1 + 대기 시간 / 계산 시간)
```

공식은 출발점이고 측정으로 조정한다. DB 커넥션 풀처럼 하위 자원의 한계도 함께 본다.

실무 연결: Tomcat 요청 스레드는 기본 최대 200개다. Spring Boot가 `@Async`용으로 자동 설정하는 `ThreadPoolTaskExecutor`는 기본 큐가 무제한이라 `newFixedThreadPool`과 같은 함정이 있다.

### Fork/Join과 작업 훔치기

`RecursiveTask`로 큰 작업을 쪼개 `fork()`로 던지고 `join()`으로 합치는 분할 정복용 풀이다(Java 7).

- 워커마다 자기 **덱(deque)**을 갖고 한쪽 끝에서 꺼내 처리한다.
- 할 일이 없는 워커는 다른 워커 덱의 **반대쪽 끝**에서 작업을 훔친다. 끝을 나눠 쓰니 경합이 적고, 반대쪽 끝에는 보통 큰 덩어리가 있다.
- **공용 풀** `ForkJoinPool.commonPool()`은 JVM 전체가 공유하며 기본 병렬도는 코어 수 − 1이다. 병렬 스트림과 `CompletableFuture`의 기본 비동기 메서드가 쓴다.

### 병렬 스트림

`parallelStream()`은 공용 풀에서 실행된다. 항상 빨라지지 않는다.

| 조건 | 유리 | 불리 |
| --- | --- | --- |
| 데이터 크기 | 원소 수 × 원소당 비용이 큼 (대략 1만 이상) | 작음 |
| 소스 분할성 | 배열, `ArrayList`, `IntStream.range` | `LinkedList`, `Stream.iterate`, I/O 스트림 |
| 연산 | 순서 무관 (`sum`, `findAny`) | 순서 의존 (`sorted`, `limit`, `findFirst`) |
| 타입 | 기본형 스트림 (`IntStream`) | 박싱 (`Stream<Integer>`) |

**위험**

- 람다에서 외부 컬렉션에 `add` 같은 공유 가변 상태 수정 → 경쟁 조건. `collect()`/`reduce()`로 모은다.
- 병렬 스트림 안의 블로킹 I/O → 작은 공용 풀이 묶여 JVM 전체의 다른 작업이 굶는다. I/O는 전용 Executor로 분리한다.

## 파트 5. 락-프리, 액터, 가상 스레드

### LMAX Disruptor와 기계적 공감

금융 거래소용 스레드 간 메시지 전달 라이브러리로, 큐 기반 설계보다 지연 시간이 훨씬 짧다.

- **링 버퍼**: 미리 할당한 원형 배열을 재사용한다. 메시지마다 객체를 만들지 않아 GC(Garbage Collection) 부담이 거의 없고 캐시에도 유리하다.
- **단일 작성자 원칙**: 값 하나는 한 스레드만 쓴다. 쓰기 경합이 없으니 락도 CAS도 거의 필요 없다.
- **기계적 공감(mechanical sympathy)**: 하드웨어가 실제로 동작하는 방식에 맞춰 코드를 짠다는 철학이다.

### 거짓 공유 (false sharing)

CPU 캐시는 캐시 라인(보통 64바이트) 단위로 동작한다. 서로 다른 스레드가 서로 다른 변수를 써도, 같은 캐시 라인에 있으면 쓸 때마다 상대 코어의 캐시 라인이 무효화된다.

- 해결: 패딩으로 변수를 다른 캐시 라인에 떨어뜨린다. JDK 내부는 `@Contended`를 쓴다.
- LongAdder의 셀이 바로 이걸로 거짓 공유를 막는다.
- 락-프리는 강력하지만 정확한 구현이 매우 어렵다. 대부분은 검증된 라이브러리를 쓰는 게 맞다.

### 액터 모델

공유 가변 상태 문제를 **공유를 없애서** 푼다. 대표 구현은 Akka다.

- 액터는 자기 상태를 가지며 외부에서 직접 접근할 수 없다.
- 액터끼리는 비동기 메시지만 주고받는다.
- 메일박스에서 메시지를 한 번에 하나씩 처리하므로 액터 내부는 락이 필요 없다.
- 위치 투명성 덕분에 분산 시스템으로 확장하기 좋다.
- 트레이드오프: 메시지 전달 비용, 비동기 흐름의 디버깅·추적 어려움, 메일박스 적체 시 배압 설계 필요.

### 가상 스레드 (Java 21 정식)

플랫폼 스레드는 OS 스레드와 1:1이라 수천 개가 한계였고, 그래서 리액티브 같은 복잡한 비동기 코드로 우회했다. 가상 스레드는 JVM이 관리하는 가벼운 스레드다.

- 소수의 **캐리어 스레드**(내부적으로 Fork/Join 풀) 위에 M:N으로 실행된다.
- 블로킹되면 스택을 힙에 저장하고 캐리어에서 내려온다(unmount). 캐리어는 다른 가상 스레드를 실행하고, I/O가 끝나면 아무 캐리어에 다시 올라탄다(mount).
- 평범한 블로킹 코드로 높은 동시성을 얻어, 요청당 스레드 모델이 다시 실용적이 됐다.
- **더 빠른 스레드가 아니다.** 늘어나는 건 처리량이고, I/O 대기가 많은 작업에서만 이득이다. CPU 집약 작업은 코어 수가 상한이다.

**주의점**

1. 풀링하지 않는다. 작업마다 새로 만든다(`Executors.newVirtualThreadPerTaskExecutor()`). 동시성 제한은 Semaphore로 한다.
2. **피닝(pinning)**: 가상 스레드가 캐리어를 붙잡는 현상. 네이티브 호출 중, 그리고 Java 21~23에서는 `synchronized` 안에서 블로킹할 때 발생했다. 당시 권장은 `ReentrantLock`으로 교체였고, JDK 24(JEP 491)에서 `synchronized` 피닝은 대부분 해소됐다.
3. ThreadLocal 남용은 스레드 수만큼 메모리를 쓴다. 대안은 ScopedValue다.
4. 구조적 동시성(Structured Concurrency)은 하위 작업을 한 단위로 묶어 실패·취소를 함께 처리하는 API로, 아직 프리뷰다.

실무 연결: Spring Boot 3.2부터 `spring.threads.virtual.enabled=true`로 Tomcat 요청 처리와 `@Async`를 가상 스레드로 돌릴 수 있다. DB 커넥션 풀 크기와의 관계는 여전히 신경 써야 한다.

## 상황별 도구 선택표

| 상황 | 쓸 도구 | 피할 것 |
| --- | --- | --- |
| 단순 상태 플래그 (`running = false`) | `volatile` | 무거운 락 |
| 경합 낮은 카운터 | `AtomicInteger` / `AtomicLong` | `volatile` + `++` |
| 경합 높은 메트릭 카운터 | `LongAdder` | 단일 `AtomicLong` |
| 타임아웃·인터럽트·여러 조건이 필요한 락 | `ReentrantLock` | `synchronized` |
| 읽기 위주 공유 데이터 | `ReentrantReadWriteLock`, `StampedLock` | 읽기 → 쓰기 락 업그레이드 |
| 동시 접근 수 제한 | `Semaphore` | 스레드 풀 크기로 제한 (가상 스레드에서) |
| 동시 맵 + 복합 연산 | `ConcurrentHashMap.computeIfAbsent` / `merge` | `get` 후 `put` |
| 읽기 압도적인 리스너 목록 | `CopyOnWriteArrayList` | 쓰기 잦은 곳에 사용 |
| 생산자-소비자 | 크기 제한 `ArrayBlockingQueue` | 무제한 `LinkedBlockingQueue` |
| 작업 N개 완료 대기 (1회) | `CountDownLatch` | 반복 단계에 사용 |
| 단계마다 스레드 동기화 | `CyclicBarrier` | `CountDownLatch` 재생성 |
| 운영용 스레드 풀 | `ThreadPoolExecutor` 직접 생성 (제한 큐 + 거부 정책) | `newFixedThreadPool`, `newCachedThreadPool` |
| 대용량 CPU 계산 | 배열 + `IntStream` 병렬 스트림, Fork/Join | 작은 데이터, `LinkedList` |
| 블로킹 I/O 병렬 실행 | 전용 Executor 또는 가상 스레드 | 병렬 스트림 (공용 풀) |
| 대량 동시 요청 I/O 서버 | 가상 스레드 | CPU 집약 작업에 기대 |

한 줄 요약: **먼저 직렬 구간을 줄이고, 공유를 줄이고, 그다음 도구를 고르고, 끝으로 측정한다.**
