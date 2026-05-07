notion: https://jyun627.notion.site/modern-java-chapter-6?source=copy_link

## 한 줄 요약

> `collect()`에 **컬렉터(Collector)** 를 넘기면 스트림을 원하는 자료구조(리스트/맵/요약값)로 변환할 수 있다.

## 6.1 컬렉터란?

### 명령형 vs 함수형 비교

```java
// 명령형: 통화별 트랜잭션 그룹화
Map<Currency, List<Transaction>> map = new HashMap<>();
for (Transaction t : transactions) {
    Currency c = t.getCurrency();
    map.computeIfAbsent(c, k -> new ArrayList<>()).add(t);
}

// 함수형: 한 줄로 끝
Map<Currency, List<Transaction>> map =
    transactions.stream().collect(groupingBy(Transaction::getCurrency));
```

### 컬렉터가 하는 일 3가지

1. **요약** — 합계, 평균, 최댓값/최솟값
2. **그룹화** — `groupingBy`
3. **분할** — `partitioningBy` (참/거짓 둘로 나누기)

> 모든 컬렉터는 `Collectors` 클래스의 **정적 팩토리 메서드** 로 제공.
> `import static java.util.stream.Collectors.*;` 후 사용.

---

## 6.2 리듀싱과 요약

### 자주 쓰는 컬렉터

| 목적        | 컬렉터               | 예시                                |
| ----------- | -------------------- | ----------------------------------- |
| 개수 세기   | `counting()`         | `menu.stream().count()` 가 더 간결  |
| 최댓값      | `maxBy(comp)`        | 비어있을 수 있어 `Optional<T>` 반환 |
| 최솟값      | `minBy(comp)`        | `Optional<T>` 반환                  |
| 합계        | `summingInt(fn)`     | `summingInt(Dish::getCalories)`     |
| 평균        | `averagingInt(fn)`   | `Double` 반환                       |
| 통계        | `summarizingInt(fn)` | count, sum, min, avg, max 한 번에   |
| 문자열 연결 | `joining(", ")`      | 내부적으로 `StringBuilder` 사용     |

### 예시

```java
// 합계
int totalCalories = menu.stream().collect(summingInt(Dish::getCalories));

// 통계 한 번에
IntSummaryStatistics stats =
    menu.stream().collect(summarizingInt(Dish::getCalories));
// {count=9, sum=4300, min=120, average=477.77, max=800}

// 문자열 연결
String shortMenu = menu.stream()
                       .map(Dish::getName)
                       .collect(joining(", "));
```

### `reducing` — 범용 리듀싱

위 모든 컬렉터를 `reducing`으로 구현 가능. **인수 3개**:

1. 초깃값
2. 변환 함수
3. 결합 함수 (`BinaryOperator`)

```java
int totalCalories = menu.stream().collect(
    reducing(0, Dish::getCalories, Integer::sum));
```

### 핵심 포인트: collect vs reduce

-   `reduce`는 **불변** 연산용. 누적자(예: 리스트)를 직접 수정하면 병렬 처리 시 망가진다.
-   가변 컨테이너에 누적하려면 **반드시 `collect`** 사용.

### 가독성/성능 팁

같은 결과를 얻는 방법은 여러 가지지만, 가장 특화된 메서드가 가독성·성능 모두 우수:

```java
// 베스트
int totalCalories = menu.stream().mapToInt(Dish::getCalories).sum();
```

---

## 6.3 그룹화 (`groupingBy`)

### 기본

```java
Map<Dish.Type, List<Dish>> dishesByType =
    menu.stream().collect(groupingBy(Dish::getType));
// {FISH=[...], MEAT=[...], OTHER=[...]}
```

### 람다로 분류 함수 만들기

```java
Map<CaloricLevel, List<Dish>> byLevel = menu.stream().collect(
    groupingBy(dish -> {
        if (dish.getCalories() <= 400) return DIET;
        else if (dish.getCalories() <= 700) return NORMAL;
        else return FAT;
    }));
```

### 그룹화 + 추가 연산 (downstream collector)

`groupingBy(분류함수, 다른컬렉터)` 형태로 그룹별 후처리:

```java
// 종류별 개수
Map<Dish.Type, Long> count =
    menu.stream().collect(groupingBy(Dish::getType, counting()));

// 종류별 칼로리 합계
Map<Dish.Type, Integer> total =
    menu.stream().collect(groupingBy(Dish::getType,
        summingInt(Dish::getCalories)));

// 종류별 가장 칼로리 높은 요리
Map<Dish.Type, Optional<Dish>> max = menu.stream().collect(
    groupingBy(Dish::getType,
        maxBy(comparingInt(Dish::getCalories))));
```

### 다수준 그룹화 (groupingBy 중첩)

```java
Map<Dish.Type, Map<CaloricLevel, List<Dish>>> result =
    menu.stream().collect(
        groupingBy(Dish::getType,
            groupingBy(dish -> { /* 칼로리 분류 */ })));
```

### `collectingAndThen` — 결과 변환

`Optional`을 벗기고 싶을 때 유용:

```java
Map<Dish.Type, Dish> mostCaloric = menu.stream().collect(
    groupingBy(Dish::getType,
        collectingAndThen(
            maxBy(comparingInt(Dish::getCalories)),
            Optional::get)));   // Optional 벗기기
```

### 그룹 내 요소 변형

| 컬렉터               | 역할                                          |
| -------------------- | --------------------------------------------- |
| `filtering(p, c)`    | 그룹화 후 각 그룹을 필터링 (빈 그룹도 유지됨) |
| `mapping(fn, c)`     | 그룹별로 매핑                                 |
| `flatMapping(fn, c)` | 그룹별로 flatMap                              |

```java
// 필터를 그룹화 안에 넣어야 빈 그룹(FISH=[])도 유지됨
Map<Dish.Type, List<Dish>> caloric = menu.stream().collect(
    groupingBy(Dish::getType,
        filtering(d -> d.getCalories() > 500, toList())));
```

---

## 6.4 분할 (`partitioningBy`)

### 분할 = 프레디케이트로 두 그룹(true/false)으로 나누기

```java
Map<Boolean, List<Dish>> partitioned =
    menu.stream().collect(partitioningBy(Dish::isVegetarian));
// {false=[고기 요리들], true=[채식 요리들]}

List<Dish> veggies = partitioned.get(true);
```

### 왜 filter 대신 분할?

-   `filter`: 한쪽만 얻음.
-   `partitioningBy`: **양쪽 모두 유지**. 두 번째 컬렉터로 후처리도 가능.

```java
// 채식/비채식 × 종류별
Map<Boolean, Map<Dish.Type, List<Dish>>> result =
    menu.stream().collect(
        partitioningBy(Dish::isVegetarian, groupingBy(Dish::getType)));
```

### 활용: 소수/비소수 분할

```java
public boolean isPrime(int n) {
    int root = (int) Math.sqrt(n);
    return IntStream.rangeClosed(2, root).noneMatch(i -> n % i == 0);
}

public Map<Boolean, List<Integer>> partitionPrimes(int n) {
    return IntStream.rangeClosed(2, n).boxed()
        .collect(partitioningBy(this::isPrime));
}
```

---

## 6.5 Collector 인터페이스

### 시그니처: `Collector<T, A, R>`

-   `T`: 스트림 요소 타입
-   `A`: 누적자(중간 저장소) 타입
-   `R`: 최종 결과 타입

### 구현해야 할 5개 메서드

| 메서드              | 역할                           | toList 예시                                 |
| ------------------- | ------------------------------ | ------------------------------------------- |
| `supplier()`        | 빈 누적자 생성                 | `ArrayList::new`                            |
| `accumulator()`     | 누적자에 요소 추가             | `List::add`                                 |
| `combiner()`        | 병렬 처리 시 누적자끼리 합치기 | `(l1, l2) -> { l1.addAll(l2); return l1; }` |
| `finisher()`        | 누적자 → 최종 결과 변환        | `Function.identity()`                       |
| `characteristics()` | 최적화 힌트 제공               | `IDENTITY_FINISH, CONCURRENT`               |

### Characteristics 3가지

| 값                | 의미                                    |
| ----------------- | --------------------------------------- |
| `UNORDERED`       | 순서가 결과에 영향 없음                 |
| `CONCURRENT`      | 여러 스레드에서 동시에 누적 가능        |
| `IDENTITY_FINISH` | finisher 생략 가능 (누적자 = 최종 결과) |

### 동작 흐름 (순차)

```
1. supplier() → 빈 누적자 생성
2. 스트림 요소마다 accumulator() 호출 → 누적자 갱신
3. finisher() → 최종 결과 변환
```

병렬일 때는 2번을 여러 서브스트림에서 동시 실행 후 `combiner()`로 합침.

---

## 6.6 커스텀 컬렉터로 성능 개선

### 아이디어

-   기존 `partitioningBy(this::isPrime)` 는 매번 `2 ~ √n`의 모든 수로 나눠봄.
-   **이미 발견한 소수로만 나눠보면** 더 빠르다.
-   그런데 컬렉터 사용 중에는 부분결과(지금까지의 소수 리스트)에 접근 불가 → **커스텀 컬렉터 필요**.

### 핵심 코드

```java
public class PrimeNumbersCollector implements
    Collector<Integer,
              Map<Boolean, List<Integer>>,
              Map<Boolean, List<Integer>>> {

    @Override
    public Supplier<Map<Boolean, List<Integer>>> supplier() {
        return () -> new HashMap<>() {{
            put(true, new ArrayList<>());
            put(false, new ArrayList<>());
        }};
    }

    @Override
    public BiConsumer<Map<Boolean, List<Integer>>, Integer> accumulator() {
        return (acc, candidate) -> {
            // 지금까지 발견한 소수 리스트(acc.get(true))를 isPrime에 전달
            acc.get(isPrime(acc.get(true), candidate)).add(candidate);
        };
    }

    @Override
    public BinaryOperator<Map<Boolean, List<Integer>>> combiner() {
        return (m1, m2) -> {
            m1.get(true).addAll(m2.get(true));
            m1.get(false).addAll(m2.get(false));
            return m1;
        };
    }

    @Override
    public Function<Map<Boolean, List<Integer>>,
                    Map<Boolean, List<Integer>>> finisher() {
        return Function.identity();
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Collections.unmodifiableSet(EnumSet.of(IDENTITY_FINISH));
    }

    // 부분결과 활용 + takeWhile (자바 9+)
    public static boolean isPrime(List<Integer> primes, int candidate) {
        int root = (int) Math.sqrt(candidate);
        return primes.stream()
                     .takeWhile(p -> p <= root)
                     .noneMatch(p -> candidate % p == 0);
    }
}
```

### 성능 결과

-   100만개 분할 기준 **약 32% 성능 향상**.

### 인터페이스 구현 안 하는 간단한 방법

`collect`는 람다 3개를 직접 받는 오버로드도 제공:

```java
List<Dish> dishes = menuStream.collect(
    ArrayList::new,    // supplier
    List::add,         // accumulator
    List::addAll);     // combiner
```

간결하지만 **재사용성/가독성은 떨어진다**.

---

## 6.7 핵심 정리 (마치며)

1. `collect`는 스트림을 **다양한 컬렉터로 누적하는 최종 연산**
2. 미리 정의된 컬렉터: 합계/평균/최댓값/최솟값/통계/문자열연결
3. **`groupingBy`** 로 그룹화, **`partitioningBy`** 로 분할
4. 컬렉터는 **다수준 그룹화/분할/리듀싱** 에 적합하게 설계되어 있다
5. `Collector` 인터페이스의 5개 메서드를 구현하면 **커스텀 컬렉터** 가능

---
