# Chapter 7. 병렬 데이터 처리와 성능

> **모던 자바 인 액션** | 2부 - 함수형 데이터 처리

---

## 이 장의 내용

- 병렬 스트림으로 데이터를 병렬 처리하기
- 병렬 스트림의 성능 분석
- 포크/조인 프레임워크
- Spliterator로 스트림 데이터 쪼개기

---

## 개요

4~6장에서는 새로운 스트림 인터페이스를 이용해서 데이터 컬렉션을 선언형으로 제어하는 방법을 살펴봤다. 또한 외부 반복을 내부 반복으로 바꾸면 네이티브 자바 라이브러리가 스트림 요소의 처리를 제어할 수 있음을 확인했다.

자바 7이 등장하기 전에는 데이터 컬렉션을 병렬로 처리하기가 어려웠다. 우선 데이터를 서브파트로 분할해야 한다. 그리고 분할된 서브파트를 각각의 스레드로 할당한다. 스레드로 할당한 다음에는 의도치 않은 **레이스 컨디션(race condition)** 이 발생하지 않도록 적절한 동기화를 추가해야 하며, 마지막으로 부분 결과를 합쳐야 한다. 자바 7은 더 쉽게 병렬화를 수행하면서 에러를 최소화할 수 있도록 **포크/조인 프레임워크(fork/join framework)** 기능을 제공한다.

이 장에서는 스트림으로 데이터 컬렉션 관련 동작을 얼마나 쉽게 병렬로 실행할 수 있는지 살펴본다.

---

## 7.1 병렬 스트림

> 컬렉션에 `parallelStream`을 호출하면 **병렬 스트림**이 생성된다.

### 병렬 스트림이란?

병렬 스트림이란 각각의 스레드에서 처리할 수 있도록 스트림 요소를 **여러 청크로 분할한 스트림**이다. 이를 이용하면 모든 멀티코어 프로세서가 각각의 청크를 처리하도록 할당할 수 있다.

### 예시: 숫자 n이 주어졌을 때 1부터 n까지의 모든 숫자의 합계를 반환하는 메서드

```java
public long sequentialSum(long n) {
    return Stream.iterate(1L, i -> i + 1)  // 무한 자연수 스트림 생성
                 .limit(n)                  // n개 이하로 제한
                 .reduce(0L, Long::sum);    // 모든 숫자를 더하는 스트림 리듀싱 연산
}
```

전통적인 자바에서는 다음과 같이 반복문으로 이를 구현할 수 있다:

```java
public long iterativeSum(long n) {
    long result = 0;
    for (long i = 1L; i <= n; i++) {
        result += i;
    }
    return result;
}
```

---

### 7.1.1 순차 스트림을 병렬 스트림으로 변환하기

순차 스트림에 `parallel` 메서드를 호출하면 기존의 함수형 리듀싱 연산(숫자 합계 계산)이 병렬로 처리된다.

```java
public long parallelSum(long n) {
    return Stream.iterate(1L, i -> i + 1)
                 .limit(n)
                 .parallel()    // <-- 스트림을 병렬 스트림으로 변환
                 .reduce(0L, Long::sum);
}
```

#### 병렬 스트림의 동작 원리

```
스트림:  [1] [2] [3] [4] [5] [6] [7] [8] [9] [10]

            첫 번째 청크          두 번째 청크
           ┌─────────┐         ┌─────────┐
           │ 1 2 3 4 5│         │ 6 7 8 9 10│
           └────┬────┘         └────┬─────┘
                │                    │
             합계: 15              합계: 40
                │                    │
                └───────┬───────────┘
                        │
                    최종 합계: 55
```

- 스트림이 여러 청크로 분할되어 각각의 청크가 **서로 다른 스레드에서** 리듀싱 연산을 수행한다
- 마지막으로 리듀싱 연산으로 생성된 부분 결과를 다시 합쳐서 최종 결과를 도출한다

#### parallel()과 sequential() 전환

```java
stream.parallel()
      .filter(...)
      .sequential()
      .map(...)
      .parallel()    // <-- 최종적으로 parallel이 적용됨
      .reduce();
```

> **핵심:** `parallel`과 `sequential` 두 메서드 중 **최종적으로 호출된 메서드**가 전체 파이프라인에 영향을 미친다. 위 예제에서 마지막 호출은 `parallel`이므로 파이프라인은 전체적으로 **병렬로** 실행된다.

#### 병렬 스트림에서 사용하는 스레드 풀 설정

병렬 스트림은 내부적으로 `ForkJoinPool`을 사용한다. 기본적으로 `ForkJoinPool`은 프로세서 수, 즉 `Runtime.getRuntime().availableProcessors()`가 반환하는 값에 상응하는 스레드를 갖는다.

```java
System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "12");
```

> 위 예제는 전역 설정 코드이므로 이후 모든 병렬 스트림 연산에 영향을 준다. 현재는 하나의 병렬 스트림에 사용할 수 있는 **특정 값을 지정할 수 없다.** 일반적으로 기기의 프로세서 수와 같으므로 특별한 이유가 없다면 **기본값을 그대로 사용할 것을 권장**한다.

---

### 7.1.2 스트림 성능 측정

> "병렬화를 이용하면 순차나 반복에 비해 성능이 더 좋아질 것"이라 추측했다. 하지만 소프트웨어 공학에서 추측은 위험한 방법이다. **측정을 해야 한다!**

#### JMH (Java Microbenchmark Harness) 사용

성능을 정확하게 측정하려면 **JMH**라는 라이브러리를 이용해야 한다. JMH를 이용하면 간단하고, 어노테이션 기반의 방식을 지원하며, 안정적으로 자바 프로그램이나 JVM을 대상으로 하는 벤치마크를 구현할 수 있다.

```java
@BenchmarkMode(Mode.AverageTime)           // 벤치마크 대상 메서드를 실행하는 데 걸린 평균 시간 측정
@OutputTimeUnit(TimeUnit.MILLISECONDS)      // 벤치마크 결과를 밀리초 단위로 출력
@Fork(2, jvmArgs={"-Xms4G", "-Xmx4G"})     // 4Gb의 힙 공간을 제공한 환경에서 두 번 벤치마크를 수행
public class ParallelStreamBenchmark {
    private static final long N = 10_000_000L;

    @Benchmark
    public long sequentialSum() {
        return Stream.iterate(1L, i -> i + 1).limit(N)
                     .reduce(0L, Long::sum);
    }

    @TearDown(Level.Invocation)             // 매 번 벤치마크를 실행한 다음에는 가비지 컬렉터 동작 시도
    public void tearDown() {
        System.gc();
    }
}
```

#### 벤치마크 결과 비교

| 방식 | 성능 (ms/op) | 설명 |
|-----|-----------|-----|
| **순차 스트림** (`sequentialSum`) | ~121 ms | `Stream.iterate` 사용 |
| **반복문** (`iterativeSum`) | ~3.3 ms | 전통적인 for 루프 |
| **병렬 스트림** (`parallelSum`) | ~604 ms | `Stream.iterate` + `parallel()` |

#### 왜 병렬 스트림이 더 느린가?

> 병렬 버전이 순차 버전에 비해 **다섯 배나 느리다!** 두 가지 문제가 있다:

1. **반복 결과로 박싱된 객체가 만들어지므로** 숫자를 더하려면 **언박싱**을 해야 한다
2. **`iterate`는 본질적으로 순차적이다.** 이전 연산의 결과에 따라 다음 함수의 입력이 달라지기 때문에 `iterate` 연산을 **청크로 분할하기가 어렵다**

> **교훈:** 병렬 프로그래밍을 오용(예를 들어 병렬화할 수 없는 `iterate`를 병렬화하려 할 때)하면 오히려 전체 프로그램의 성능이 더 나빠질 수 있다.

#### 더 특화된 메서드 사용: `LongStream.rangeClosed`

`LongStream.rangeClosed`는 `iterate`에 비해 다음과 같은 장점이 있다:
- 기본형 `long`을 직접 사용하므로 **박싱과 언박싱 오버헤드가 없다**
- 쉽게 청크로 분할할 수 있는 **숫자 범위를 생산**한다 (예: 1~20 범위를 1~5, 6~10, 11~15, 16~20으로 분할)

```java
@Benchmark
public long rangedSum() {
    return LongStream.rangeClosed(1, N)
                     .reduce(0L, Long::sum);
}
```

| 방식 | 성능 (ms/op) |
|-----|-----------|
| `rangedSum` (순차) | ~5.3 ms |
| `parallelRangedSum` (병렬) | ~2.7 ms |

```java
@Benchmark
public long parallelRangedSum() {
    return LongStream.rangeClosed(1, N)
                     .parallel()
                     .reduce(0L, Long::sum);
}
```

> **드디어** 순차 실행보다 빠른 성능을 갖는 병렬 리듀싱을 만들었다! 올바른 자료구조를 선택해야 병렬 실행도 최적의 성능을 발휘할 수 있다.

#### 핵심 정리

하지만 병렬화가 완전 공짜는 아니라는 사실을 기억하자:
- **병렬화를 이용하려면** 스트림을 재귀적으로 분할해야 하고
- 각 서브스트림을 서로 **다른 스레드의 리듀싱 연산으로 할당**해야 하고
- 이들 결과를 하나의 값으로 **합쳐야** 한다
- 멀티코어 간의 데이터 이동은 우리 생각보다 **비싸다**

---

### 7.1.3 병렬 스트림의 올바른 사용법

> 병렬 스트림을 잘못 사용하면서 발생하는 많은 문제는 **공유된 상태를 바꾸는 알고리즘**을 사용하기 때문에 일어난다.

#### 잘못된 예시: 공유된 누적자(accumulator) 사용

```java
public long sideEffectSum(long n) {
    Accumulator accumulator = new Accumulator();
    LongStream.rangeClosed(1, n).forEach(accumulator::add);
    return accumulator.total;
}

public class Accumulator {
    public long total = 0;
    public void add(long value) { total += value; }
}
```

위 코드의 문제점:
- `total += value`는 **원자적(atomic) 연산이 아니다**
- 여러 스레드에서 동시에 `total`에 접근하면 **레이스 컨디션** 발생

#### 병렬로 실행하면?

```java
public long sideEffectParallelSum(long n) {
    Accumulator accumulator = new Accumulator();
    LongStream.rangeClosed(1, n).parallel().forEach(accumulator::add);
    return accumulator.total;
}
```

**실행 결과 (10번 실행):**
```
Result: 5959989000692
Result: 7425264100768
Result: 6432292556402
Result: 7192970417739
Result: 6741475975331
Result: 7493903958907
Result: 6453348646885
Result: 6993884894209
Result: 7459192292481
Result: 7357352292681
```

> **올바른 결과(50000005000000)가 나오지 않는다!** 여러 스레드에서 동시에 공유된 `total` 변수에 접근하면서 데이터 레이스 문제가 발생했다. 병렬 스트림과 공유된 가변 상태는 **절대 함께 사용하면 안 된다.**

---

### 7.1.4 병렬 스트림 효과적으로 사용하기

> "천 개 이상의 요소가 있을 때만 병렬 스트림을 사용하라"와 같이 양을 기준으로 병렬 스트림 사용을 결정하는 것은 적절하지 않다.

#### 병렬 스트림 사용 가이드라인

| 가이드라인 | 설명 |
|---------|-----|
| **확신이 서지 않으면 직접 측정하라** | 순차 스트림을 병렬 스트림으로 쉽게 바꿀 수 있지만, 무조건 바꾸는 것이 능사는 아니다. 항상 **벤치마크로 성능을 확인**하라 |
| **박싱을 주의하라** | 자동 박싱과 언박싱은 성능을 크게 저하시킨다. `IntStream`, `LongStream`, `DoubleStream` 등 **기본형 특화 스트림**을 사용하라 |
| **순차적인 연산에 주의하라** | `limit`이나 `findFirst`처럼 요소의 순서에 의존하는 연산은 병렬 스트림에서 비싸다 |
| **스트림의 소스를 확인하라** | 분할하기 쉬운 자료구조인지 확인하라 (아래 표 참조) |
| **중간 연산의 특성을 확인하라** | `filter` 같은 연산은 스트림 크기를 예측할 수 없으므로 효과적인 병렬 처리가 어렵다 |
| **최종 연산의 비용을 확인하라** | 합치는 과정(예: `combiner`)의 비용이 크면 병렬 스트림의 이점이 상쇄된다 |

#### 스트림 소스와 분해성

| 소스 | 분해성 |
|------|------|
| `ArrayList` | **훌륭함** |
| `LinkedList` | **나쁨** |
| `IntStream.range` | **훌륭함** |
| `Stream.iterate` | **나쁨** |
| `HashSet` | **좋음** |
| `TreeSet` | **좋음** |

> **핵심:** `ArrayList`는 요소를 분할하지 않고도 리스트를 분할할 수 있지만, `LinkedList`는 모든 요소를 탐색해야 분할할 수 있으므로 병렬 처리에 부적합하다.

---

## 7.2 포크/조인 프레임워크

> 포크/조인 프레임워크는 병렬화할 수 있는 작업을 **재귀적으로 작은 작업으로 분할**한 다음에 서브태스크 각각의 결과를 합쳐서 전체 결과를 만들도록 설계되었다.

포크/조인 프레임워크에서는 서브태스크를 스레드 풀(`ForkJoinPool`)의 작업자 스레드에 분산 할당하는 `ExecutorService` 인터페이스를 구현한다.

---

### 7.2.1 RecursiveTask 활용

스레드 풀을 이용하려면 `RecursiveTask<R>`의 서브클래스를 만들어야 한다.

> `RecursiveTask`를 정의하려면 추상 메서드 `compute`를 구현해야 한다.

#### compute 메서드의 알고리즘 (의사코드)

```
if (태스크가 충분히 작거나 더 이상 분할 수 없으면) {
    순차적으로 태스크 계산
} else {
    태스크를 두 서브태스크로 분할
    태스크가 다시 서브태스크로 분할되도록 이 메서드를 재귀적으로 호출
    모든 서브태스크의 연산이 완료될 때까지 기다림
    각 서브태스크의 결과를 합침
}
```

> 이 알고리즘은 **분할 후 정복(divide and conquer)** 알고리즘의 병렬화 버전이다.

#### 그림: 포크/조인 과정

```
         ┌─────────────────────────────┐
         │  ForkJoinSumCalculator       │
         │  (전체 배열)                  │
         └──────────┬──────────────────┘
                    │ 분할
         ┌──────────┴──────────┐
    ┌────┴────┐          ┌────┴────┐
    │ 왼쪽 절반 │          │ 오른쪽 절반│
    └────┬────┘          └────┬────┘
         │ 분할                │ 분할
    ┌────┴────┐          ┌────┴────┐
    │  ...    │          │  ...    │
    └────┬────┘          └────┬────┘
         │                    │
      순차 계산              순차 계산
         │                    │
         └────────┬───────────┘
              부분 결과 조합
                  │
              최종 결과
```

#### ForkJoinSumCalculator 구현

```java
public class ForkJoinSumCalculator
        extends java.util.concurrent.RecursiveTask<Long> {

    private final long[] numbers;     // 더할 숫자 배열
    private final int start;          // 이 서브태스크에서 처리할 배열의 초기 위치
    private final int end;            // 이 서브태스크에서 처리할 배열의 최종 위치
    public static final long THRESHOLD = 10_000;  // 이 값 이하의 서브태스크는 더 이상 분할 불가

    // 메인 태스크를 생성할 때 사용할 공개 생성자
    public ForkJoinSumCalculator(long[] numbers) {
        this(numbers, 0, numbers.length);
    }

    // 메인 태스크의 서브태스크를 재귀적으로 만들 때 사용할 비공개 생성자
    private ForkJoinSumCalculator(long[] numbers, int start, int end) {
        this.numbers = numbers;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;
        if (length <= THRESHOLD) {
            // 기준값과 같거나 작으면 순차적으로 결과를 계산
            return computeSequentially();
        }

        // 배열의 첫 번째 절반을 더하도록 서브태스크를 생성
        ForkJoinSumCalculator leftTask =
            new ForkJoinSumCalculator(numbers, start, start + length/2);

        // ForkJoinPool의 다른 스레드로 새로 생성한 태스크를 비동기적으로 실행
        leftTask.fork();

        // 배열의 나머지 절반을 더하도록 서브태스크를 생성
        ForkJoinSumCalculator rightTask =
            new ForkJoinSumCalculator(numbers, start + length/2, end);

        // 두 번째 서브태스크를 동기 실행 (이 스레드 재활용)
        Long rightResult = rightTask.compute();

        // 첫 번째 서브태스크의 결과를 읽거나 아직 결과가 없으면 기다림
        Long leftResult = leftTask.join();

        // 두 서브태스크의 결과를 조합한 값이 이 태스크의 결과
        return leftResult + rightResult;
    }

    // 더 분할할 수 없을 때 서브태스크의 결과를 계산하는 단순한 알고리즘
    private long computeSequentially() {
        long sum = 0;
        for (int i = start; i < end; i++) {
            sum += numbers[i];
        }
        return sum;
    }
}
```

#### 사용 방법

```java
public static long forkJoinSum(long n) {
    long[] numbers = LongStream.rangeClosed(1, n).toArray();
    ForkJoinTask<Long> task = new ForkJoinSumCalculator(numbers);
    return new ForkJoinPool().invoke(task);
}
```

> `LongStream`으로 n까지의 자연수를 포함하는 배열을 생성하고, `ForkJoinSumCalculator`의 생성자로 전달해서 `ForkJoinPool`에서 `ForkJoinTask`를 실행한다.

#### 실행 결과

```
ForkJoin sum done in: 41 msecs
```

병렬 스트림을 이용할 때보다 성능이 나빠졌다. 하지만 이는 `ForkJoinSumCalculator` 태스크에서 사용할 수 있도록 전체 스트림을 `long[]`으로 변환했기 때문이다.

---

### 7.2.2 포크/조인 프레임워크를 제대로 사용하는 방법

| 규칙 | 설명 |
|------|-----|
| **`join` 메서드 호출은 신중하게** | `join` 메서드를 태스크에 호출하면 태스크가 생산하는 결과가 준비될 때까지 호출자를 **블록**시킨다. 따라서 두 서브태스크가 **모두 시작된 다음에** `join`을 호출해야 한다 |
| **`invoke` 대신 `fork`/`compute` 사용** | `RecursiveTask` 내에서는 `invoke`를 직접 사용하지 말아야 한다. 대신 `compute`나 `fork` 메서드를 직접 호출한다. 순차 코드에서 병렬 계산을 시작할 때만 `invoke`를 사용한다 |
| **한쪽은 `fork`, 한쪽은 `compute`** | 왼쪽과 오른쪽 서브태스크에 **모두 `fork`를 호출하는 것보다**, 한쪽에는 `compute`를 호출하는 것이 효율적이다. 같은 스레드를 **재활용**할 수 있기 때문이다 |
| **디버깅이 어렵다** | 포크/조인 프레임워크를 이용하는 병렬 계산은 디버깅하기 어렵다. `fork`라 불리는 다른 스레드에서 `compute`를 호출하므로 스택 트레이스가 도움이 되지 않는다 |

> **주의:** 포크/조인 프레임워크를 사용하면 무조건 빠를 거라는 생각은 버려야 한다. 태스크를 여러 독립적인 서브태스크로 분할할 수 있어야 하며, 각 서브태스크의 실행시간은 새로운 태스크를 포킹하는 데 드는 시간보다 길어야 한다.

---

### 7.2.3 작업 훔치기 (Work Stealing)

> `ForkJoinSumCalculator` 예제에서는 덧셈을 수행할 숫자가 만 개 이하면 서브태스크 분할을 중단했다. 기준값을 바꿔가며 실험해봐도 딱 맞는 기준을 찾기 어렵다. 그래서 **작업 훔치기(work stealing)** 기법이 존재한다.

#### 작업 훔치기의 원리

- 각 스레드는 자신에게 할당된 태스크를 포함하는 **이중 연결 리스트(doubly linked list)** 를 참조한다
- 작업이 끝날 때마다 큐의 **헤드**에서 다른 태스크를 가져와 작업을 처리한다
- 한 스레드가 다른 스레드보다 작업을 빨리 끝내면, **다른 스레드 큐의 꼬리(tail)** 에서 작업을 훔쳐온다
- 모든 태스크가 작업을 끝낼 때까지, 즉 **모든 큐가 빌 때까지** 이 과정을 반복한다

```
작업자1:  [태스크A] [태스크B] [태스크C] [태스크D]
           ↑ (헤드에서 가져감)

작업자2:  [태스크E] [태스크F]                        ← 먼저 끝남!
                                     ↓
작업자3:  [태스크G]                   훔치기!
                              작업자1의 꼬리에서 [태스크D]를 가져감

작업자4:  [태스크H] [태스크I]
```

> 풀에 있는 작업자 스레드의 태스크를 **재분배하고 균형을 맞출 때** 작업 훔치기 알고리즘을 사용한다. 작업자의 큐에 있는 태스크를 두 개의 서브 태스크로 분할했을 때 둘 중 하나의 태스크를 다른 유휴 작업자가 훔쳐갈 수 있다.

---

## 7.3 Spliterator 인터페이스

> `Spliterator`는 '분할할 수 있는 반복자(splitable iterator)'라는 의미다.

자바 8은 `Spliterator`라는 새로운 인터페이스를 제공한다. `Iterator`처럼 소스의 요소 탐색 기능을 제공하지만, **병렬 작업에 특화**되어 있다.

#### Spliterator 인터페이스

```java
public interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);  // 요소를 하나씩 순차적으로 소비하며 탐색할 요소가 남아있으면 true 반환
    Spliterator<T> trySplit();                        // 일부 요소를 분할해서 두 번째 Spliterator를 생성
    long estimateSize();                              // 탐색해야 할 요소 수 정보 제공
    int characteristics();                            // Spliterator의 특성을 정의
}
```

| 메서드 | 설명 |
|--------|------|
| `tryAdvance` | Spliterator의 요소를 하나씩 순차적으로 소비하면서 탐색할 요소가 남아있으면 `true` 반환 |
| `trySplit` | Spliterator의 일부 요소를 분할해서 **두 번째 Spliterator를 생성** |
| `estimateSize` | 탐색해야 할 요소 수 정보를 제공 |
| `characteristics` | Spliterator의 특성을 정의하는 int를 반환 |

---

### 7.3.1 분할 과정

스트림을 여러 스트림으로 분할하는 과정은 **재귀적**으로 일어난다.

#### 분할 과정 (4단계)

```
1단계:  [Spliterator1]
            │
         trySplit()
            │
        ┌───┴───┐
  [Spliterator1] [Spliterator2]

2단계:     trySplit()          trySplit()
        ┌───┴───┐          ┌───┴───┐
  [Spl1] [Spl2]       [Spl3] [Spl4]

3단계: 각각에 trySplit() 호출
       → 더 분할 가능하면 계속 분할
       → trySplit()이 null을 반환하면 분할 종료

4단계: 모든 Spliterator에 trySplit() 결과가 null이면
       → 재귀 분할 과정 종료!
```

1. 첫 번째 `Spliterator`에 `trySplit`을 호출하면 두 번째 `Spliterator`가 생성된다
2. 두 개의 `Spliterator`에 다시 `trySplit`을 호출하면 네 개의 `Spliterator`가 생성된다
3. `trySplit`이 `null`을 반환할 때까지(더 이상 분할할 수 없음을 의미) 이 과정을 반복한다
4. 모든 `trySplit`의 결과가 `null`이면 재귀 분할 과정이 종료된다

> 이 분할 과정은 `characteristics` 메서드로 정의하는 Spliterator의 특성에 영향을 받는다.

#### Spliterator 특성

| 특성 | 의미 |
|------|------|
| `ORDERED` | 리스트처럼 요소에 정해진 순서가 있으므로 Spliterator는 요소를 탐색하고 분할할 때 이 순서에 유의해야 한다 |
| `DISTINCT` | x, y 두 요소를 방문했을 때 `x.equals(y)`는 항상 `false`를 반환한다 |
| `SORTED` | 탐색된 요소는 미리 정의된 정렬 순서를 따른다 |
| `SIZED` | 크기가 알려진 소스로 생성된 `Set`이면 `estimateSize`는 정확한 값을 반환한다 |
| `NON-NULL` | 탐색하는 모든 요소는 `null`이 아니다 |
| `IMMUTABLE` | 이 Spliterator의 소스는 불변이다. 즉, 요소를 탐색하는 동안 요소를 추가하거나, 삭제하거나, 고칠 수 없다 |
| `CONCURRENT` | 동기화 없이 Spliterator의 소스를 여러 스레드에서 동시에 고칠 수 있다 |
| `SUBSIZED` | 이 Spliterator 그리고 분할되는 모든 Spliterator는 `SIZED` 특성을 갖는다 |

---

### 7.3.2 커스텀 Spliterator 구현하기

#### 예제: 문자열의 단어 수를 세는 메서드

먼저 반복형으로 단어 수를 세는 메서드를 살펴보자:

```java
public int countWordsIteratively(String s) {
    int counter = 0;
    boolean lastSpace = true;
    for (char c : s.toCharArray()) {
        if (Character.isWhitespace(c)) {
            lastSpace = true;
        } else {
            if (lastSpace) counter++;  // 공백 뒤에 나오는 문자의 횟수를 세는 것(즉 단어의 수)
            lastSpace = false;
        }
    }
    return counter;
}
```

#### 함수형으로 단어 수를 세는 메서드 재구현하기

`String`을 스트림으로 변환해야 한다. `int`, `long`, `double` 기본형 스트림만 제공하므로 `Stream<Character>`를 사용해야 한다:

```java
Stream<Character> stream = IntStream.range(0, SENTENCE.length())
                                    .mapToObj(SENTENCE::charAt);
```

스트림에 리듀싱 연산을 실행하면서 단어 수를 계산할 수 있다. 이때 자바에는 튜플(tuple)이 없으므로, 문자를 하나씩 탐색하면서 **현재까지 발견한 단어 수**를 계산하는 변수와 **마지막 문자가 공백이었는지 여부**를 기억하는 `Boolean` 변수 두 가지 변수가 필요하다.

이를 위해 새로운 클래스 `WordCounter`를 만든다:

```java
class WordCounter {
    private final int counter;
    private final boolean lastSpace;

    public WordCounter(int counter, boolean lastSpace) {
        this.counter = counter;
        this.lastSpace = lastSpace;
    }

    // 문자를 하나씩 탐색하다가 새로운 단어를 찾으면 카운터를 증가시킨다
    public WordCounter accumulate(Character c) {
        if (Character.isWhitespace(c)) {
            return lastSpace ? this :
                   new WordCounter(counter, true);
        } else {
            return lastSpace ?
                   new WordCounter(counter + 1, false) :  // 공백 → 문자 전환 시 단어 카운트 증가
                   this;
        }
    }

    // 두 WordCounter의 counter를 더한다
    public WordCounter combine(WordCounter wordCounter) {
        return new WordCounter(counter + wordCounter.counter,
                               wordCounter.lastSpace);
    }

    public int getCounter() {
        return counter;
    }
}
```

#### 문자 c를 탐색했을 때 WordCounter의 상태 변환

```
                 ┌──────────────────────┐
                 │    새로운 문자 c       │
                 └──────────┬───────────┘
                            │
              ┌─────────────┴──────────────┐
              │                            │
       c는 공백인가?                   c는 문자인가?
              │                            │
     lastSpace == false?          lastSpace == true?
         │        │                  │         │
        Yes      No                Yes        No
         │        │                  │         │
   WordCounter  this         WordCounter    this
   (counter,              (counter+1,
    true)                  false)
```

#### 스트림으로 단어 수 세기

```java
private int countWords(Stream<Character> stream) {
    WordCounter wordCounter = stream.reduce(new WordCounter(0, true),
                                            WordCounter::accumulate,
                                            WordCounter::combine);
    return wordCounter.getCounter();
}
```

```java
Stream<Character> stream = IntStream.range(0, SENTENCE.length())
                                    .mapToObj(SENTENCE::charAt);
System.out.println("Found " + countWords(stream) + " words");
// 출력: Found 19
words
```

#### WordCounter 병렬로 수행하기

```java
System.out.println("Found " + countWords(stream.parallel()) + " words");
// 출력: Found 25   ← 원하는 결과가 나오지 않는다!
```

> **왜 잘못된 결과가 나올까?** 원래 문자열을 **임의의 위치에서 분할**하기 때문이다. 한 단어가 둘로 나뉘면서 두 번 카운트될 수 있다.

이 문제를 해결하려면 **단어가 끝나는 위치에서만 분할하는** 커스텀 `Spliterator`가 필요하다.

#### WordCounterSpliterator 구현

```java
class WordCounterSpliterator implements Spliterator<Character> {
    private final String string;
    private int currentChar = 0;

    public WordCounterSpliterator(String string) {
        this.string = string;
    }

    @Override
    public boolean tryAdvance(Consumer<? super Character> action) {
        // 현재 문자를 소비한다
        action.accept(string.charAt(currentChar++));
        // 소비할 문자가 남아있으면 true를 반환
        return currentChar < string.length();
    }

    @Override
    public Spliterator<Character> trySplit() {
        int currentSize = string.length() - currentChar;

        if (currentSize < 10) {
            // 파싱할 문자열을 순차 처리할 수 있을 만큼 충분히 작아졌음을 알리는 null을 반환
            return null;
        }

        // 파싱할 문자열의 중간을 분할 위치로 설정
        for (int splitPos = currentSize / 2 + currentChar;
             splitPos < string.length(); splitPos++) {

            // 다음 공백이 나올 때까지 분할 위치를 뒤로 이동
            if (Character.isWhitespace(string.charAt(splitPos))) {
                // 처음부터 분할 위치까지 문자열을 파싱할
                // 새로운 WordCounterSpliterator를 생성한다
                Spliterator<Character> spliterator =
                    new WordCounterSpliterator(
                        string.substring(currentChar, splitPos));

                // 이 WordCounterSpliterator의 시작 위치를
                // 분할 위치로 설정한다
                currentChar = splitPos;
                return spliterator;
            }
        }
        return null;
    }

    @Override
    public long estimateSize() {
        return string.length() - currentChar;
    }

    @Override
    public int characteristics() {
        return ORDERED + SIZED + SUBSIZED + NON_NULL + IMMUTABLE;
    }
}
```

#### WordCounterSpliterator의 핵심 동작

| 메서드 | 동작 |
|--------|------|
| `tryAdvance` | 문자열에서 현재 인덱스에 해당하는 문자를 `Consumer`에 제공한 다음 인덱스를 증가시킨다 |
| `trySplit` | 반으로 분할하되 **단어 중간이 아닌 공백에서만** 분할한다. 문자열을 임의의 위치에서 분할하지 않고 단어가 끝나는 위치에서만 분할하므로 올바른 결과를 보장한다 |
| `estimateSize` | 파싱해야 할 문자열 전체 길이(`string.length() - currentChar`)와 현재 반복 중인 위치의 차이 |
| `characteristics` | `ORDERED`(문자열의 문자 등장 순서가 유의미), `SIZED`(`estimateSize` 메서드의 반환값이 정확함), `SUBSIZED`(`trySplit`으로 생성된 `Spliterator`도 정확한 크기를 가짐), `NON_NULL`(문자열에는 `null` 문자가 존재하지 않음), `IMMUTABLE`(문자열 자체가 불변 클래스이므로 파싱하면서 속성이 추가되지 않음) |

#### 사용

```java
Spliterator<Character> spliterator = new WordCounterSpliterator(SENTENCE);
Stream<Character> stream = StreamSupport.stream(spliterator, true);  // true = 병렬 스트림

System.out.println("Found " + countWords(stream) + " words");
// 출력: Found 19   ← 올바른 결과!
```

> `Spliterator`에서 어떻게 자료구조를 분할하는지가 중요하다. 분할 기준을 잘 정하면 병렬 스트림으로도 올바른 결과를 얻을 수 있다.

---

## 7.4 마치며

### 핵심 요약

- **내부 반복**을 이용하면 명시적으로 다른 스레드를 사용하지 않고도 스트림을 병렬로 처리할 수 있다

- **간단하게** 스트림을 병렬로 처리할 수 있지만 항상 병렬 처리가 빠른 것은 아니다. 병렬 소프트웨어 동작 방법과 성능은 직관적이지 않을 때가 많으므로 **병렬 처리를 사용했을 때 성능을 직접 측정**해봐야 한다

- 병렬 스트림으로 데이터 집합을 병렬 실행할 때 특히 처리해야 할 데이터가 아주 많거나 각 요소를 처리하는 데 오랜 시간이 걸릴 때 성능을 높일 수 있다

- 가능하면 **기본형 특화 스트림**을 사용하는 등 올바른 자료구조 선택이 어떤 연산을 병렬로 처리하는 것보다 성능적으로 더 큰 영향을 미칠 수 있다

- **포크/조인 프레임워크**에서는 병렬화할 수 있는 태스크를 작은 태스크로 분할한 다음에 분할된 태스크를 각각의 스레드로 실행하며 서브태스크 각각의 결과를 합쳐서 최종 결과를 생산한다

- `Spliterator`는 탐색하려는 데이터를 포함하는 스트림을 **어떻게 병렬화할 것인지** 정의한다

---

## 한눈에 보는 챕터 요약

```
                    ┌───────────────────────────────────────┐
                    │     Chapter 7: 병렬 데이터 처리와 성능    │
                    └───────────────────┬───────────────────┘
                                        │
              ┌─────────────────────────┼──────────────────────────┐
              │                         │                          │
    ┌─────────┴─────────┐    ┌─────────┴─────────┐    ┌──────────┴──────────┐
    │   7.1 병렬 스트림    │    │ 7.2 포크/조인       │    │ 7.3 Spliterator     │
    │                    │    │    프레임워크         │    │    인터페이스         │
    └────────┬───────────┘    └────────┬───────────┘    └─────────┬───────────┘
             │                         │                          │
    ・parallel() 호출        ・RecursiveTask 활용         ・trySplit()으로 분할
    ・성능 측정 (JMH)         ・분할 후 정복 알고리즘        ・tryAdvance()로 탐색
    ・올바른 사용법            ・작업 훔치기 기법           ・커스텀 구현 가능
    ・공유 상태 주의!          ・ForkJoinPool 활용         ・characteristics 정의
```
