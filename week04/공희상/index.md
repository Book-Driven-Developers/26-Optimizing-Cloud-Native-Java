
## 트레이드 오프와 플러그형 컬렉터

가비지 컬렉서 서브 시스템은 플러그인이 가능한 형태이다.
-> 다른 가비지 컬렉터로 바꾸어 실행 가능하다.
-> 프로그램은 동일한 의미로 동작하지만, 컬렉터의 성능에 따라 프로그래밍의 성능이 달라진다.

> [!NOTE]
> 모든 문제를 동시에 최적화할 수 있는 단일 가비지 컬렉션은 없다! -> 은총알은 없다.

가비지 컬렉터 선택 시 고려 사항
- 애플리케이션 STW 시간
- 애플리케이션 실행 시간 대비 GC 실행 시간
- GC가 애플리케이션을 중지하는 빈도
- 단일 가비지 컬렉터가 수집할 수 있는 가비지의 양
- 일시 중지 시간의 일관성

## 동시 GC 이론

1. GC의 비결정성(예축 뷸거눙성)은 직접적인 메모리 할당 동작으로 발생
2. 자바 시스템은 가변적인 메모리 할당
3. 범용적으로 설계된 GC는 애플리케이션의 메모리 할당 패턴을 예측불가

현대 GC의 출발점은 STW의 지속 시간과 빈도가 예측 불가능적이라는 점에서 시작했다.

### JVM Safe-point
STW GC를 수행하려면, 모든 스레드가 중지해야 한다.
-> 

What is JVM Safepoint?  JVM 세이프포인트란 무엇인가요?

자바 가상 머신(JVM)에서 세이프포인트(safepoint)란, 프로그램 실행 중 JVM 내의 모든 스레드가 명확하게 정의된 상태에서 일시 정지된 지점을 말합니다. 이를 통해 JVM은 힙 및 스레드 상태에 대한 일관된 관점이 필요한 특정 작업을 수행할 수 있습니다. 세이프포인트는 다음과 같은 다양한 JVM 기능을 구현하는 데 필수적입니다:

Garbage Collection: The garbage collector needs to ensure that the heap is in a consistent state and that no objects are being modified during the collection process. Safepoints ensure that all threads pause, making it safe to perform garbage collection.
가비지 컬렉션: 가비지 컬렉터는 힙이 일관된 상태를 유지하고, 컬렉션 과정 중에 어떤 객체도 수정되지 않도록 보장해야 합니다. 세이프포인트는 모든 스레드가 일시 정지되도록 하여 가비지 컬렉션을 안전하게 수행할 수 있게 합니다.
Thread Stack Dump: When generating a thread dump (e.g., for debugging or profiling), the JVM needs all threads to be at a safe point to get a consistent snapshot of the stack traces.
스레드 스택 덤프: 스레드 덤프를 생성할 때(예: 디버깅이나 프로파일링을 위해), JVM은 스택 추적의 일관된 스냅샷을 얻기 위해 모든 스레드가 안전한 지점에 있어야 합니다.
Deoptimization: During the execution of optimized code, the JVM may need to revert to interpreted mode or less optimized code. Safepoints ensure that this transition happens safely.
비최적화: 최적화된 코드가 실행되는 동안 JVM은 해석 모드로 되돌아가거나 최적화 수준이 낮은 코드로 전환해야 할 수도 있습니다. 세이프포인트는 이러한 전환이 안전하게 이루어지도록 보장합니다.
Class Redefinition: When classes are redefined or modified at runtime, such as during debugging or hot swapping in development, safepoints are used to ensure a consistent state for these changes.
클래스 재정의: 개발 과정에서 디버깅이나 핫 스와핑과 같이 런타임에 클래스가 재정의되거나 수정될 때, 이러한 변경 사항에 대해 일관된 상태를 보장하기 위해 세이프포인트가 사용됩니다.

How Safepoints Work  세이프포인트의 작동 원리

Safepoint Polling: Threads periodically check if a safepoint is required. This can be done through polling mechanisms like explicit checks in the bytecode or through periodic interrupts.
세이프포인트 폴링: 스레드는 주기적으로 세이프포인트가 필요한지 확인합니다. 이는 바이트코드 내의 명시적 확인과 같은 폴링 메커니즘을 통해, 또는 주기적인 인터럽트를 통해 수행될 수 있습니다.
Safepoint Mechanism: When a safepoint is needed, the JVM will request all threads to reach a safepoint. Threads that are already at a safepoint will stop immediately, while others will continue to execute until they reach the next safepoint poll.
세이프포인트 메커니즘: 세이프포인트가 필요한 경우, JVM은 모든 스레드에 세이프포인트에 도달할 것을 요청합니다. 이미 세이프포인트에 있는 스레드는 즉시 중지되는 반면, 그 외의 스레드는 다음 세이프포인트 폴링에 도달할 때까지 실행을 계속합니다.
Global Synchronization: Once all threads have reached a safepoint, the JVM can safely perform the necessary operations.
전역 동기화: 모든 스레드가 세이프포인트에 도달하면, JVM은 필요한 작업을 안전하게 수행할 수 있습니다.

Safepoint Bias  세이프포인트 편향

Safepoints are designed to minimize the performance impact on running applications. However, excessive safepoint operations can still cause performance issues, such as:
세이프포인트는 실행 중인 애플리케이션에 미치는 성능 영향을 최소화하도록 설계되었습니다. 하지만 세이프포인트 작업이 지나치게 빈번하게 수행될 경우 다음과 같은 성능 문제가 발생할 수 있습니다.

Pause Times: While safepoints are designed to be quick, frequent safepoints can introduce noticeable pauses in application execution.
지연 시간: 세이프포인트는 신속하게 처리되도록 설계되었지만, 세이프포인트가 너무 빈번하게 발생하면 애플리케이션 실행에 눈에 띄는 지연이 발생할 수 있습니다.
Throughput Impact: Frequent safepoints can reduce overall throughput, especially in applications with high concurrency and tight performance requirements.
처리량에 미치는 영향: 세이프포인트가 빈번하게 발생하면, 특히 동시성이 높고 성능 요구 사항이 까다로운 애플리케이션에서 전체 처리량이 저하될 수 있습니다.

Understanding and optimizing safepoint behavior is crucial for maintaining the performance and stability of Java applications, particularly those running in high-throughput or low-latency environments.
세이프포인트의 동작을 이해하고 최적화하는 것은 자바 애플리케이션, 특히 높은 처리량이나 낮은 지연 시간 환경에서 실행되는 애플리케이션의 성능과 안정성을 유지하는 데 매우 중요합니다. 