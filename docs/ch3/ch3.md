참고 링크: https://www.notion.so/33477ca3c78280efa1dffd84f1a4156e

# 3장 람다 표현식

## 3.1 람다란 무엇인가?

- 람다 표현식은 **이름 없는 함수(익명 함수)** 다.
- 메서드처럼 매개변수, 본문, 반환 개념을 가지지만 이름은 없다.
- 핵심은 동작을 값처럼 전달할 수 있게 해준다는 점이다.

```java
// 익명 클래스
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello");
    }
};

// 람다
Runnable r2 = () -> System.out.println("Hello");
```

```java
Comparator<Integer> numeric = (a, b) -> Integer.compare(a, b);
```

---

## 3.2 어디에, 어떻게 람다를 사용할까?

람다는 아무 위치에서나 쓸 수 있는 문법이 아니다.  
**함수형 인터페이스를 기대하는 자리(대상 형식, target type)** 에서만 사용할 수 있다.

### 3.2.1 함수형 인터페이스

- 추상 메서드가 정확히 하나인 인터페이스
- 디폴트/정적 메서드는 여러 개 있어도 된다.
- `@FunctionalInterface`를 붙이면 의도를 명확히 하고 컴파일 검증을 받을 수 있다.

```java
@FunctionalInterface
interface ApplePredicate {
    boolean test(Apple apple);
}
```

```java
@FunctionalInterface
interface Runnable {
    void run();
}
```

### 3.2.2 함수 디스크립터

함수형 인터페이스의 추상 메서드 시그니처를 말한다.

- `Runnable` : `() -> void`
- `Callable<String>` : `() -> String`
- `Predicate<Apple>` : `Apple -> boolean`
- `Comparator<Apple>` : `(Apple, Apple) -> int`

```java
Callable<String> c = () -> "result";
Predicate<Apple> p = a -> a.getWeight() > 150;
Comparator<Apple> cmp = (a1, a2) -> a1.getWeight().compareTo(a2.getWeight());
```

포인트: 람다 자체에 타입이 있는 게 아니라, **어디에 대입/전달되는지**가 타입을 결정한다.

---

## 3.3 람다 활용: 실행 어라운드 패턴

자원 처리 코드는 보통 아래 패턴을 반복한다.

1. 자원 열기
2. 실제 처리
3. 자원 닫기

설정/정리 코드는 고정, 중간 처리 로직만 자주 바뀐다.  
이때 중간 로직만 람다로 받는 방식이 실행 어라운드 패턴이다.

### 3.3.1 1단계: 동작 파라미터화 기억

```java
public static String processFile() throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader("data.txt"))) {
        return br.readLine();
    }
}
```

### 3.3.2 2단계: 함수형 인터페이스로 동작 전달

```java
@FunctionalInterface
public interface BufferedReaderProcessor {
    String process(BufferedReader b) throws IOException;
}
```

```java
public static String processFile(BufferedReaderProcessor p) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader("data.txt"))) {
        return p.process(br);
    }
}
```

### 3.3.3 3단계: 동작 실행

```java
String oneLine = processFile((BufferedReader br) -> br.readLine());
```

### 3.3.4 4단계: 람다 전달

```java
String twoLines = processFile(br -> br.readLine() + br.readLine());
```

---

## 3.4 함수형 인터페이스 사용

`java.util.function` 패키지는 자주 쓰는 함수 모양을 표준 인터페이스로 제공한다.

### 3.4.1 Predicate

`Predicate<T>` 는 `T -> boolean`

```java
public static <T> List<T> filter(List<T> list, Predicate<T> p) {
    List<T> results = new ArrayList<>();
    for (T t : list) {
        if (p.test(t)) {
            results.add(t);
        }
    }
    return results;
}

List<Apple> heavy = filter(inventory, a -> a.getWeight() > 150);
List<Apple> green = filter(inventory, a -> GREEN.equals(a.getColor()));
```

### 3.4.2 Consumer

`Consumer<T>` 는 `T -> void`  
값을 받아서 출력/저장/로그 같은 부수효과를 수행할 때 사용한다.

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}

public static <T> void forEach(List<T> list, Consumer<T> c) {
    for (T t : list) {
        c.accept(t);
    }
}

forEach(Arrays.asList(1, 2, 3, 4, 5), (Integer i) -> System.out.println(i));
```

### 3.4.3 Function

`Function<T, R>` 는 `T -> R`  
입력을 출력으로 변환할 때 사용한다.

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}

public static <T, R> List<R> map(List<T> list, Function<T, R> f) {
    List<R> result = new ArrayList<>();
    for (T s : list) {
        result.add(f.apply(s));
    }
    return result;
}

List<Integer> lengths = map(Arrays.asList("lambdas", "in", "action"), String::length);
```

### 3.4.4 그 밖의 기본 인터페이스

- `Supplier<T>` : `() -> T`
- `UnaryOperator<T>` : `T -> T`
- `BinaryOperator<T>` : `(T, T) -> T`

```java
Supplier<Apple> supplier = Apple::new;
Apple apple = supplier.get();
```

#### 기본형 특화 인터페이스

오토박싱/언박싱 비용을 줄이기 위해 기본형 전용 인터페이스를 제공한다.

- `IntPredicate`, `LongPredicate`, `DoublePredicate`
- `ToIntFunction<T>`, `IntFunction<R>`
- `IntUnaryOperator`, `IntBinaryOperator`

```java
IntPredicate evenNumbers = i -> i % 2 == 0;          // int -> boolean
Predicate<Integer> oddNumbers = i -> i % 2 != 0;     // 박싱 가능성

ToIntFunction<Apple> weightExtractor = Apple::getWeight;  // Apple -> int
IntUnaryOperator square = x -> x * x;                      // int -> int
```

### 3.4.5 예외, 람다, 함수형 인터페이스 관계

람다가 던질 수 있는 예외는 대상 함수형 인터페이스 시그니처가 허용하는 범위 안에서만 가능하다.

```java
// Consumer.accept는 checked exception을 선언하지 않음
files.forEach(f -> {
   String line = new BufferedReader(new FileReader(f)).readLine();
});
// IOException 처리 때문에 컴파일 에러 가능
```

```java
@FunctionalInterface
interface BufferedReaderProcessor {
    String process(BufferedReader b) throws IOException;
}

String line = processFile(br -> br.readLine()); // IOException 허용
```

실무 대응:

1. 체크 예외를 선언한 커스텀 함수형 인터페이스 사용
2. 람다 내부에서 try-catch 후 런타임 예외로 래핑

---

## 3.5 형식 검사, 형식 추론, 제약

### 3.5.1 형식 검사

컴파일러는 대상 형식을 먼저 찾고, 그 함수 디스크립터와 람다가 맞는지 검사한다.

```java
Predicate<Apple> p = a -> a.getWeight() > 150; // OK
// Callable<String> c = () -> 42; // 에러
```

### 3.5.2 같은 람다, 다른 함수형 인터페이스

같은 람다가 여러 함수형 인터페이스와 호환될 수 있다.

```java
Callable<Integer> c1 = () -> 42;
PrivilegedAction<Integer> c2 = () -> 42;
```

```java
void doWork(Callable<Integer> c) {}
void doWork(PrivilegedAction<Integer> a) {}

// doWork(() -> 42); // 모호성 에러
doWork((Callable<Integer>) () -> 42);
```

#### 다이아몬드 연산자와 람다

`<>`와 람다, 오버로드가 겹치면 추론이 더 어려워질 수 있다.  
필요하면 타입을 명시해 모호성을 제거한다.

```java
ExecutorService executor = Executors.newCachedThreadPool();
Callable<Integer> task = () -> 42;
executor.submit(task);
// executor.submit((Callable<Integer>) () -> 42);
```

#### 특별한 void 호환 규칙

- 대상 형식이 `void`면, 람다 본문의 계산값은 버릴 수 있다.
- 값을 반환해야 하는 대상 형식에서는 모든 경로가 반환해야 한다.

```java
Runnable r1 = () -> System.out.println("hello");
Runnable r2 = () -> { if (System.currentTimeMillis() > 0) return; };

Function<String, Integer> f;
// f = s -> { System.out.println(s); }; // 에러
f = s -> { System.out.println(s); return s.length(); };
```

### 3.5.3 형식 추론

컴파일러가 대상 형식으로 람다 파라미터 타입을 추론한다.

```java
List<Apple> greenApples =
    filter(inventory, apple -> GREEN.equals(apple.getColor()));
```

```java
Comparator<Apple> c =
    (Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight());

Comparator<Apple> c2 =
    (a1, a2) -> a1.getWeight().compareTo(a2.getWeight());
```

타입 명시/생략 중 무엇이 더 좋은지는 문맥과 가독성에 따라 판단한다.

### 3.5.4 지역 변수 사용

캡처한 지역 변수는 `final` 또는 effectively final이어야 한다.

```java
int portNumber = 1337;
Runnable r = () -> System.out.println(portNumber);
// portNumber = 31337; // 컴파일 에러
```

```java
public void process() {
    int count = 0;
    Runnable r = () -> {
        // count++; // 컴파일 에러
        System.out.println(count);
    };
}
```

### 3.5.5 클로저(closure)

- 클로저: 바깥 스코프 변수를 캡처하는 함수 인스턴스
- 자바 람다도 외부 변수 캡처가 가능해 클로저와 유사
- 다만 지역 변수 변경은 허용되지 않음(effectively final 제약)

```java
class Counter {
    private int value = 0;
    Runnable inc() {
        return () -> value++; // 인스턴스 변수는 변경 가능
    }
}
```

```java
public Runnable bad() {
    int local = 0;
    // return () -> local++; // 에러
    return () -> System.out.println(local);
}
```

---

## 3.6 메서드 참조

람다가 기존 메서드 호출만 한다면 메서드 참조가 더 읽기 좋다.

### 3.6.1 요약

1. 정적 메서드 참조: `ClassName::staticMethod`
2. 특정 객체 인스턴스 메서드 참조: `instanceRef::instanceMethod`
3. 임의 객체 인스턴스 메서드 참조: `ClassName::instanceMethod`

```java
Function<String, Integer> strToInt = Integer::parseInt;
Transaction expensiveTransaction = new Transaction();
Supplier<Integer> value = expensiveTransaction::getValue;
Comparator<Apple> byWeight = Comparator.comparing(Apple::getWeight);
```

```java
List<String> str = Arrays.asList("a", "b", "A", "B");
str.sort((s1, s2) -> s1.compareToIgnoreCase(s2));
str.sort(String::compareToIgnoreCase);
```

### 3.6.2 생성자 참조

생성자도 `ClassName::new`로 참조 가능하며, 함수형 인터페이스 시그니처와 생성자 시그니처가 맞아야 한다.

```java
Supplier<Apple> c1 = Apple::new;
Apple a1 = c1.get();

Function<Integer, Apple> c2 = Apple::new;
Apple a2 = c2.apply(150);

BiFunction<String, Integer, Apple> c3 = Apple::new;
Apple a3 = c3.apply("green", 160);
```

```java
@FunctionalInterface
interface TriFunction<T, U, V, R> {
    R apply(T t, U u, V v);
}

TriFunction<Integer, Integer, Integer, Color> rgb = Color::new;
Color c = rgb.apply(255, 0, 0);
```

#### 생성자 참조로 팩터리 만들기

```java
Map<String, Function<Integer, Fruit>> map = new HashMap<>();
map.put("apple", Apple::new);
map.put("orange", Orange::new);

public static Fruit giveMeFruit(String fruit, Integer weight,
                                Map<String, Function<Integer, Fruit>> map) {
    return map.get(fruit.toLowerCase()).apply(weight);
}

Fruit f1 = giveMeFruit("apple", 150, map);
Fruit f2 = giveMeFruit("orange", 120, map);
```

장점: `if-else/switch` 분기 감소, 확장 시 맵 엔트리만 추가.

---

## 3.7 람다, 메서드 참조 활용하기

### 3.7.1 1단계: 코드 전달

```java
public class AppleComparator implements Comparator<Apple> {
    @Override
    public int compare(Apple a1, Apple a2) {
        return a1.getWeight().compareTo(a2.getWeight());
    }
}
inventory.sort(new AppleComparator());
```

### 3.7.2 2단계: 익명 클래스 사용

```java
inventory.sort(new Comparator<Apple>() {
    @Override
    public int compare(Apple a1, Apple a2) {
        return a1.getWeight().compareTo(a2.getWeight());
    }
});
```

### 3.7.3 3단계: 람다 표현식 사용

```java
inventory.sort((a1, a2) -> a1.getWeight().compareTo(a2.getWeight()));
```

### 3.7.4 4단계: 메서드 참조 사용

```java
inventory.sort(Comparator.comparing(Apple::getWeight));
```

`Comparator.comparing`은 \"비교 키 추출 함수\"를 받아 Comparator를 만들어 준다.  
의도가 잘 드러나고 `reversed`, `thenComparing`으로 확장하기 쉽다.

```java
List<Apple> greenApples = filter(inventory, a -> GREEN.equals(a.getColor()));
List<Apple> heavyApples = filter(inventory, a -> a.getWeight() > 150);
```

---

## 3.8 람다 표현식을 조합할 수 있는 유용한 메서드

### 3.8.1 Comparator 조합

```java
Comparator<Apple> c1 = Comparator.comparing(Apple::getWeight);
Comparator<Apple> c2 = Comparator.comparing(Apple::getWeight).reversed();
Comparator<Apple> c3 = Comparator
    .comparing(Apple::getWeight)
    .reversed()
    .thenComparing(Apple::getCountry);
```

### 3.8.2 Predicate 조합

```java
Predicate<Apple> redApple = apple -> RED.equals(apple.getColor());
Predicate<Apple> notRedApple = redApple.negate();
Predicate<Apple> redAndHeavyApple = redApple.and(apple -> apple.getWeight() > 150);
Predicate<Apple> redAndHeavyOrGreenApple = redAndHeavyApple
    .or(apple -> GREEN.equals(apple.getColor()));
```

### 3.8.3 Function 조합

```java
Function<Integer, Integer> f = x -> x + 1;
Function<Integer, Integer> g = x -> x * 2;

Function<Integer, Integer> h1 = f.andThen(g); // g(f(x))
Function<Integer, Integer> h2 = f.compose(g); // f(g(x))

System.out.println(h1.apply(1)); // 4
System.out.println(h2.apply(1)); // 3
```

```java
Function<String, String> addHeader = Letter::addHeader;
Function<String, String> transformationPipeline = addHeader
    .andThen(Letter::checkSpelling)
    .andThen(Letter::addFooter);
```

---

## 3.9 비슷한 수학적 개념

### 3.9.1 적분

적분은 함수 `f(x)`와 구간 `[a, b]`를 받아 계산하는 연산이다.  
핵심은 **함수를 인수로 받는다**는 점.

```java
public static double integrate(DoubleFunction<Double> f, double a, double b) {
    return (f.apply(a) + f.apply(b)) * (b - a) / 2.0;
}
```

### 3.9.2 자바 8 람다로 연결

람다를 이용하면 \"계산 규칙(함수)\" 자체를 간단히 전달할 수 있다.

```java
double result = integrate(x -> x + 10, 3, 7);

double area1 = integrate(x -> x, 0, 10);
double area2 = integrate(x -> x * x, 0, 10);
double area3 = integrate(Math::sin, 0, Math.PI);
```

즉 3.9는 람다를 단순 문법이 아니라, \"함수를 값으로 다루는 사고\"로 연결한다.

---

## 3.10 마치며

- 람다는 동작 파라미터화를 간결하게 만든다.
- 함수형 인터페이스와 대상 형식을 이해하면 람다 타입 문제를 제대로 다룰 수 있다.
- 메서드 참조/생성자 참조로 가독성과 재사용성을 높일 수 있다.
- `Comparator`, `Predicate`, `Function` 조합 메서드는 선언형 스타일의 핵심 도구다.
