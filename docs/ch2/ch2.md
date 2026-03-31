참고 링크: https://jyun627.notion.site/modern-java-ch2?source=copy_link

동작 파라미터화: 아직은 어떻게 실행할 것인지 결정하지 않은 코드 블록

→ 나중에 프로그램에서 호출 → 코드 블록의 실행이 나중으로 미뤄지는 것

## 2.1 변화하는 요구사항에 대응하기

```java
public static List<Apple> filterGreenApples(List<Apple> inventory) {
  List<Apple> result = new ArrayList<>(); // 사과 누적 리스트
  for (Apple apple : inventory) {
    if (apple.getColor() == Color.GREEN) { // 녹색 사과만 선택
      result.add(apple);
    }
  }
  return result;
}
```

그런데 클라이언트가 갑자기 녹색 말고 빨간 사과도 필터링하고 싶어졌다.

→ 메서드를 복사해서 `filterRedApples` 라는 새로운 메서드를 만들고, if문의 조건을 빨간 사과로 바꾸기?
나중에 농부가 다양한 색으로 필터링하는 등의 변화에는 적절하게 대응할 수 없다.

> 거의 비슷한 코드가 반복 존재한다면 그 코드를 추상화한다.
>

`if (apple.getColor().equals(color))`

이렇게 색을 파라미터화하면 될 것이다.

무게 정보에 대한 요구사항이 들어오면?

```java
public static List<Apple> filterApplesByWeight(List<Apple> inventory, int weight) {
  List<Apple> result = new ArrayList<>();
  for (Apple apple : inventory) {
    if (apple.getWeight() > weight) {
      result.add(apple);
    }
  }
  return result;
}
```

구현 코드를 자세히 보면

1. 목록 검색
2. 각 사과에 필터링 조건

등을 적용하는 부분이 대부분 색 필터링 코드와 중복된다. 이는 소프트웨어 공학의 DRY (don’t repeat yourself)원칙을 어기는 것이다.

## 2.2 동작 파라미터화

사과의 어떤 속성에 기초해서 불리언값을 반환하는 방법이 있다.
참 또는 거짓을 반환하는 함수를 프레디케이트라고 한다.

p72. 전략 디자인 패턴

조건에 따라 filter 메서드가 다르게 동작 → 전략 디자인 패턴 stategy design patern 이라고 부른다.

<aside>
💡

**전략 디자인 패턴**

각 알고리즘 (전략이라 불리는)을 캡슐화하는 알고리즘 패밀리를 정의해둔 다음
런타임에 알고리즘을 선택하는 기법이다.

</aside>

`filterApples` 메서드가 `ApplePredicate` 객체를 인수로 받도록 하자.

이렇게 하면 `filterApples` 메서드 내부에서 컬렉션을 반복하는 로직과 컬렉션의 각 요소에 적용할 동작(우리 예제에서는 프레디케이트)을 **분리**할 수 있다는 점에서 소프트웨어 엔지니어링적으로 큰 이득을 얻는다.

**추상적 조건으로 필터링**

```java
public static List<Apple> filter(List<Apple> inventory, ApplePredicate p) {
  List<Apple> result = new ArrayList<>();
  for (Apple apple : inventory) {
    if (p.test(apple)) { // 프레디케이트 객체로 사과 검사 조건을 캡슐화했다. 
      result.add(apple);
    }
  }
  return result;
}
```

이제 농부가 150그램이 넘는 빨간 사과를 검색해달라고 부탁하면 `ApplePredicate`를 적절하게 구현하는 클래스만 만들면 된다.

## 복잡한 과정 간소화

`filterApples` 메서드로 새로운 동작을 전달하려면 `ApplePredicate` 인터페이스를 구현하는 여러 클래스를 정의한 다음에 인스턴스화해야 한다. → 번거롭고 시간 낭비

```java
// 1. 빨간 사과인지 확인하는 클래스를 정의
public class AppleRedColorPredicate implements ApplePredicate {
    public boolean test(Apple apple) {
        return "red".equals(apple.getColor());
    }
}

// 2. 무거운 사과인지 확인하는 클래스를 정의
public class AppleHeavyWeightPredicate implements ApplePredicate {
    public boolean test(Apple apple) {
        return apple.getWeight() > 150;
    }
}

// 사용 시: 매번 인스턴스를 생성해서 전달
List<Apple> redApples = filter(inventory, new AppleRedColorPredicate());
List<Apple> heavyApples = filter(inventory, new AppleHeavyWeightPredicate());
```

> 자바는 클래스의 선언과 인스턴스화를 동시에 수행할 수 있는 **익명 클래스**를 제공한다.
>

**익명 클래스**

```java
// 클래스 이름 없이 메서드 호출부에서 바로 로직을 정의
List<Apple> redApples = filter(inventory, new ApplePredicate() {
    @Override
    public boolean test(Apple apple) {
        return "red".equals(apple.getColor());
    }
});

List<Apple> heavyApples = filter(inventory, new ApplePredicate() {
    @Override
    public boolean test(Apple apple) {
        return apple.getWeight() > 150;
    }
});
```

많이 간소해졌다. 하지만 아직 `new ApplePredicate()`, `@Override`, `public boolean test` 같은 '보일러플레이트(반복되는 코드)'가 많아 코드가 지저분해 보인다.

**람다식 사용**

```java
// 훨씬 간결하고 가독성이 높음
List<Apple> redApples = filter(inventory, (Apple apple) -> "red".equals(apple.getColor()));
List<Apple> heavyApples = filter(inventory, (Apple apple) -> apple.getWeight() > 150);
```
<aside>
💡

동작 파라미터화 (Behavior Parameterization)

메서드가 실행할 **코드 블록** 혹은 **동작을 인수로 전달**하는 개발 패턴
”무엇을 할 것인가”를 메서드 내부가 아니라, **메서드를 호출하는 쪽에서 결정**하도록 만드는 것

</aside>

**리스트 형식**

```java
public static <T> List<T> filter(List<T> list, Predicate<T> p) {
    List<T> result = new ArrayList<>();
    for (T e : list) {
        if (p.test(e)) {
            result.add(e);
        }
    }
    return result;
}
```

메서드 시그니처에 `<T>`를 추가하면 `Apple` 뿐만 아니라 `Banana`, `Orange`, 혹은 `String`이나 `Integer` 리스트까지 처리할 수 있게 만든다.

## 2.4 실전 예제

**Comparator로 정렬**

자바 8의 List에선 sort 메서드가 포함되어 있다(물론 Collections.sort도 존재한다.)

```java
// java.util.Comparator
public interface Comparator<T> {
    // 두 객체 t1, t2를 비교함
    int compare(T t1, T t2);
}
```

Comparator를 구현해서 sort메서드의 동작을 다양화할 수 있다.
아래는 익명 클래스를 이용해서 무게가 적은 순서로 목록에서 사과를 정렬한 예시이다.

```java
import java.util.Comparator;

// inventory 리스트의 sort 메서드에 익명 클래스를 전달
inventory.sort(new Comparator<Apple>() {
    @Override
    public int compare(Apple a1, Apple a2) {
        // 첫 번째 사과의 무게가 두 번째보다 작으면 음수를 반환하여 앞으로 보냄 (오름차순)
        return Integer.compare(a1.getWeight(), a2.getWeight());
    }
});
```

a1의 무게 < a2의 무게라면 a1,a2 순으로 (오름차순) 정렬된다.

농부의 요구사항이 바뀌면 새로운  Comparator 를 만들어 sort  메서드에 전달할 수 있다.

**Runnable 코드 블록 실행하기**

자바 스레드를 이용하면 병렬로 코드 블록을 실행할 수 있다.

- 여러 스레드가 각자 다른 코드 실행 가능
- 나중에 실행할 수 있는 코드를 구현할 방법 필요

자바 8까지는 Thread 생성자에 객체만을 전달할 수 있었음 → 결과를 반환하지 않는 void run 메서드를 포함하는 익명 클래스가 Runnable 인터페이스를 구현하도록 하는 것이 일반적인 방법

```java
Thread t = new Thread(new Runnable() {
    @Override
    public void run() {
        // 실행할 동작을 여기에 정의
        System.out.println("익명 클래스로 스레드 실행 중!");
    }
});

t.start(); // 스레드 시작
```

`Runnable` 인터페이스를 사용하여 스레드를 실행하는 방법도 앞서 배운 **동작 파라미터화**의 전형적인 사례이다.  `Thread` 클래스에 "실행할 작업(동작)"을 인수로 전달하는 방식

<aside>
💡

**Runnable**

‘실행할 수 있는 작업’을 추상화한 인터페이스
Runnable은 병렬적으로 실행할 코드 블록을 정의한다.
```

@FunctionalInterface
public interface Runnable {
public abstract void run();
}
```

</aside>