https://clover-shift-99f.notion.site/Chapter-5-33b0b82d4e9c80afb497f8159db1f073?source=copy_link


이 장에서는 스트림 API가 지원하는 다양한 연산을 살펴본다. 스트림 API가 지원하는 연산을 이용해서 필터링, 슬라이싱, 매핑, 검색, 매칭, 리듀싱 등 다양한 데이터 처리 질의를 표현할 수 있다.

## 5.1 필터링

### 5.1.1 프레디케이트로 필터링

스트림 인터페이스는 filter 메서드를 지원한다. filter는 Predicate를 인수로 받아서 일치하는 모든 요소를 포함하는 스트림을 반환한다.

```java
List<Dish> vegetarianMenu = menu.stream()
                    .filter(Dish::isVegetarian)
                    .collect(toList());
```

다음의 코드는 모든 채식요리를 필터링해서 채식 메뉴를 만든다.

### 5.1.2 고유 요소 필터링

distinct 메서드는 고유 요소로 이루어진 스트림을 반환한다. 

다음 코드는 리스트의 모든 짝수를 선택하고 중복을 필터링한다.

```java
List<Integer> numbers = Arrays.asList(1, 2, 1, 3, 3, 2, 4);
    numbers.stream()
            .filter(i -> i % 2 == 0)
            .distinct()
            .forEach(System.out::println);
```

## 5.2 스트림 슬라이싱

### 5.2.1 프레디케이트를 이용한 슬라이싱

자바 9는 스트림의 요소를 효과적으로 선택할 수 있도록 takeWhile, dropWhile 두 가지 새로운 메서드를 지원한다.

**TAKEWHILE 활용**

다음과 같은 요리 목록을 갖고 있다고 가정하자.

```java
List<Dish> specialMenu = Arrays.asList(
        new Dish("season fruit", true, 120, Dish.Type.OTHER),
        new Dish("prawns", false, 300, Dish.Type.FISH),
        new Dish("rice", true, 350, Dish.Type.OTHER),
        new Dish("chicken", false, 400, Dish.Type.MEAT),
        new Dish("french fries", true, 530, Dish.Type.OTHER));
```

320 칼로리 이하의 요리를 선택하려면 filter를 이용할 수 있을 것이다.

```java
List<Dish> filteredMenu = specialMenu.stream()
        .filter(dish -> dish.getCalories() < 320)
        .collect(toList());
```

filter 연산을 이용하면 전체 스트림을 반복하면서 각 요소에 프레디케이트를 적용하게 된다. 리스트가 정렬되어 있는 상태에서, 320 칼로리보다 큰 음식이 등장하면 검사를 멈추는 것이 효율적일 것이다. 그러나, filter는 계속해서 검사를 하게 된다. 하지만 takeWhile 연산을 이용하면 이미 탈락 확정인 요소를 쓸데없이 검사하는 낭비를 막을 수 있다.

```java
List<Dish> slicedMenu1 = specialMenu.stream()
        .takeWhile(dish -> dish.getCalories() < 320)
        .collect(toList());
```

**DROPWHILE 활용**

320 칼로리보다 큰 요소는 어떻게 탐색할까? dropWhile을 이용해 이 작업을 완료할 수 있다.

```java
List<Dish> slicedMenu2 = specialMenu.stream()
        .dropWhile(dish -> dish.getCalories() < 320)
        .collect(toList());
```

dropWhile은 takeWhile과 정반대의 작업을 수행한다. dropWhile은 프레디케이트가 처음으로 거짓이 되는 지점까지 발견된 요소를 버린다. 프레디케이트가 거짓이 되면 그 지점에서 작업을 중단하고 남은 모든 요소를 반환한다.

### 5.2.2 스트림 축소

스트림은 주어진 값 이하의 크기를 갖는 새로운 스트림을 반환하는 limit 메서드를 지원한다.

다음처럼 300칼로리 이상의 세 요리를 선택해서 리스트를 만들 수 있다.

```java
List<Dish> dishesLimit3 = menu.stream()
        .filter(d -> d.getCalories() > 300)
        .limit(3)
        .collect(toList());
```

### 5.2.3 요소 건너뛰기

스트림은 처음 n개 요소를 제외한 스트림을 반환하는 skip 메서드를 지원한다. n개 이하의 요소를 포함하는 스트림에 skip을 호출하면 빈 스트림이 반환된다.

다음 코드는 300칼로리 이상의 처음 두 요리를 건너뛴 다음에 300칼로리가 넘는 나머지 요리를 반환한다.

```java
List<Dish> dishesSkip2 = menu.stream()
        .filter(d -> d.getCalories() > 300)
        .skip(2)
        .collect(toList());
```

## 5.3 매핑

스트림 API의 map과 flatMap 메서드는 특정 데이터를 선택하는 기능을 제공한다.

### 5.3.1 스트림의 각 요소에 함수 적용하기

스트림은 함수를 인수로 받는 map 메서드를 지원한다.  함수는 각 요소에 적용되며 함수를 적용한 결과가 새로운 요소로 매핑된다. 

다음은 Dish의 getName을 map의 메서드로 전달해서 요리명을 추출한다.

```java
List<String> dishNames = menu.stream()
        .map(Dish::getName)
        .collect(toList());
```

각 요리명의 길이를 알고 싶다면 다음처럼 다른 map 메서드를 연결할 수 있다.

```java
List<Integer> dishNameLengths = menu.stream()
        .map(Dish::getName)
        .map(String::length)
        .collect(toList());
```

### 5.3.2 스트림 평면화

리스트에서 고유 문자로 이루어진 리스트를 반환해보자.

[”Hello”, “World”] 리스트가 있다면 결과로 [”H”, “e”, “l”, “o”, “W”, “r”, “d”]를 포함하는 리스트를 반환받고 싶다.

map과 distinct를 이용해서 문제를 해결할 것이라고 생각할 수도 있다.

```java
words.stream()
            .map(word -> word.split(" "))
            .distinct()
            .collect(toList());
```

map으로 전달한 람다는 String[]을 반환한다. 우리가 기대하는 스트림 반환 형식은 **`List<String>`**인데, **`List<String[]>`**을 반환하게 된다.

이 문제는 flatMap이라는 메서드를 이용해서 해결할 수 있다.

**map과 Arrays.stream 활용**

문자열 스트림이 필요하다. 문자열을 받아 스트림을 만드는 **`Arrays.stream()`** 메서드가 있다.

```java
String[] arrayOfWords = {"GoodBye", "Word"};
Stream<String> stream = Arrays.stream(arrayOfWords);
```

위 예제의 파이프라인에 이 메서드를 적용해보자.

```java
words.stream()
            .map(word -> word.split(" "))
            .map(Arrays::stream)
            .distinct()
            .collect(toList());
```

이 코드를 통해 반환받는 것은 **`List<Stream<String>>`**이다. 문제를 해결하려면 먼저 각 단어를 개별 문자열로 이루어진 배열로 만든 다음에 각 배열을 별도의 스트림으로 만들어야 한다.

**flatMap 사용**

flatMap을 사용하면 다음처럼 문제를 해결할 수 있다.

```java
words.stream()
            .map(word -> word.split(" "))
            .flatMap(Arrays::stream) // 생성된 스트림을 하나의 스트림으로 평면화
            .distinct()
            .collect(toList());
```

flatMap은 각 배열을 스트림이 아니라 스트림의 콘텐츠로 매핑한다. flatMap 메서드는 스트림의 각 값을 다른 스트림으로 만든 다음에 모든 스트림을 하나의 스트림으로 연결하는 기능을 수행한다.

## 5.4 검색과 매칭

스트림 API는 allMatch, anyMatch, noneMatch, findFirst, findAny 등 다양한 메서드를 제공한다.

### 5.4.1 프레디케이트가 적어도 한 요소와 일치하는지 확인

anyMatch 메서드를 사용하면 주어진 스트림에서 적어도 한 요소와 일치하는지 확인할 수 있다. 

다음 코드는 menu에 채식 요리가 있는지 확인한다.

```java
if(menu.stream().anyMatch(Dish::isVegetarian))
```

이는 불리언을 반환하므로 최종 연산이다.

### 5.4.2 프레디케이트가 모든 요소와 일치하는지 검사

allMatch 메서드는 모든 요소가 주어진 프레디케이트와 일치하는지 검사한다.

예를 들어 모든 요리가 1000 칼로리 이하인지 확인할 수 있다.

```java
boolean isHealthy = menu.stream()
            .allMatch(dish -> dish.getCalories() < 1000);
```

noneMatch 메서드는 allMatch와 반대 연산을 수행한다. 주어진 프레디케이트와 일치하는 요소가 없는지 확인한다.

```java
boolean isHealthy = menu.stream()
            .noneMatch(dish -> dish.getCalories() < 1000);
```

<aside>

💡 쇼트서킷 평가란?

때로는 전체 스트림을 처리하지 않았더라도 결과를 반환할 수 있다. allMatch, noneMatch, findFirst, findAny 등의 연산은 모든 스트림의 요소를 처리하지 않고도 결과를 반환할 수 있다.원하는 요소를 찾았으면 즉시 결과를 반환할 수 있다.

</aside>

### 5.4.3 요소 검색

findAny 메서드는 현재 스트림에서 임의의 요소를 반환한다.

예를 들어 다음 코드처럼 filter와 findAny를 이용해서 채식 요리를 선택할 수 있다.

```java
Optional<Dish> dish = 
		menu.stream()
		.filter(Dish::isVegetarian)
		.findAny();
```

쇼트서킷을 이용해서 결과를 찾는 즉시 실행을 종료한다.

### 5.4.4 첫 번째 요소 찾기

일부 스트림에는 논리적인 순서가 정해져 있을 수 있다. 이런 스트림에서 첫 번째 요소를 찾으려면 findAny를 사용한다.

다음은 숫자 리스트에서 3으로 나누어 떨어지는 첫 번째 제곱값을 반환하는 코드이다.

```java
Optional<Integer> firstSquareDivisibleByThree =
            someNumbers.stream()
                    .map(n -> n * n)
                    .filter(n % 3 == 0)
                    .findFirst();
```

**findFirst vs findAny**

병렬성때문에 findFirst와 findAny를 모두 사용한다.

병렬 실행에서는 첫 번째 요소를 찾기 어렵다. 요소의 반환 순서가 중요하지 않다면 병렬 스트림에서는 제약이 적은 findAny를 사용한다.

## 5.5 리듀싱

리듀스 연산을 이용해서 스트림 요소를 조합해서 더 복잡한 질의를 표현하는 방법을 설명한다. 이러한 질의를 수행하려면 Integer 같은 결과가 나올 때까지 스트림의 모든 요소를 반복적으로 처리해야 한다. 이런 질의를 **리듀싱 연산**이라고 한다.

### 5.5.1 요소의 합

for-each 루프를 이용해서 리스트의 숫자 요소를 더하는 코드를 보자.

```java
int sum = 0;
for(int x : numbers) {
		sum += x;
}
```

numbers의 각 요소는 결과에 반복적으로 더해진다. 

reduce를 이용하면 애플리케이션의 반복된 패턴을 추상화할 수 있다.

```java
int sum = numbers.stream().reduce(0, (a, b) -> a + b);
```

reduce는 두 개의 인수를 갖는다.

- 초깃값 0
- BinaryOperator<T>

메서드 참조를 이용해서 이 코드를 좀 더 간결하게 만들 수 있다. 자바 8에서 Integer 클래스에 두 숫자를 더하는 정적 메서드인 sum을 제공하므로 직접 람다를 구현할 필요가 없다.

```java
int sum = numbers.stream().reduce(0, Integer::sum);
```

초깃값을 받지 않도록 오버로드된 reduce도 있다. 이는 Optional 객체를 반환한다.

```java
Optional<Integer> sum = numbers.stream().reduce((a, b) -> (a + b));
```

스트림에 아무 요소도 없다면 합계가 없음을 가리킬 수 있도록 Optional 객체로 감싼 결과를 반환한다.

### 5.5.2 최댓값과 최솟값

두 요소에서 최댓값을 반환하는 람다만 있으면 최댓값을 구할 수 있다. 즉, reduce 연산은 새로운 값을 이용해서 모든 요소를 소비할 때까지 람다를 반복 수행하면서 최댓값을 생산한다.

```java
 Optional<Integer> max = numbers.stream().reduce(Integer::max);
```

**`Integer.max`** 대신 **`Integer.min`**을 reduce로 넘겨주면 최솟값을 찾을 수 있다.

## 5.7 숫자형 스트림

reduce 메서드로 스트림 요소의 합을 구하는 예제를 살펴봤다. 다음처럼 메뉴의 칼로리 합계를 계산할 수 있다.

```java
int calories = menu.stream()
                    .map(Dish::getCalories)
                    .reduce(0, Integer::sum);
```

그러나 위 코드에서는 내부적으로 합계를 계산하기 위해 Integer를 기본형으로 언박싱해야 한다.

```java
int calories = menu.stream()
                    .map(Dish::getCalories)
                    .sum();
```

하지만 그렇다고 해서 sum을 직접 호출할 수 없다. map 메서드는 Stream<T>를 생성한다. 다행히도 스트림 API 숫자 스트림을 효율적으로 처리할 수 있도록 **기본형 특화 스트림**을 제공한다.

### 5.7.1 기본형 특화 스트림

자바 8에서는 IntStream, DoubleStream, LongStream의 세 가지 기본형 특화 스트림을 제공한다. 각각의 인터페이스는 sum, max와 같이 자주 사용하는 숫자 관련 리듀싱 연산 수행 메서드를 제공한다.

스트림을 특화 스트림으로 변환할 때는 mapToInt, mapToDouble, mapToLong 세 가지 메서드를 가장 많이 사용한다. map과 같은 기능을 수행하지만, **Stream<T> 대신 특화된 스트림을 반환한다.**

```java
int calories = menu.stream()
                    .mapToInt(Dish::getCalories)
                    .sum();
```

Dish에서 Integer 형식의 칼로리를 추출하여 IntStream을 반환한다. IntStream은 max, min, average 등 다양한 유틸리티 메서드도 지원한다.

boxed 메서드를 이용하면 원상태인 특화되지 않은 스트림으로 복원할 수 있다.

```java
IntStream intStream = menu.stream().mapToInt(Dish::getCalories);
Stream<Integer> stream = intStream.boxed();
```

### 5.7.2 숫자 범위

특정 범위의 숫자를 이용해야 하는 상황이 자주 발생할 것이다. 1에서 100 사이의 숫자를 생성하려 한다고 가정하자. IntStream과 LongStream에서는 **range**와 **rangeClosed**라는 두 가지 정적 메서드를 제공한다.

range는 시작값과 종료값이 결과에 포함되지 않고, rangeClosed는 포함된다.
