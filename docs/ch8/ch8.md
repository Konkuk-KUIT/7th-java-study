# 8.1 컬렉션 팩토리

## 작은 컬렉션 객체 생성 방법

```java
List<String> friends = new ArrayList<>();
friends.add("Raphael");
friends.add("Olivia");
friends.add("Thibaut");
```

### Arrays.asList()

```java
List<String> friends = Arrays.asList("Raphael", "Olivia", "Thibaut");
```

- 고정 크기 리스트 생성
    - 요소 갱신 가능: set()
    - 요소 추가/삭제 불가
- .add로 요소 추가 시도 ⇒ UnsupportedOperationException 발생

### 집합

- Arrays.asSet()이 없음
    
    ```java
    Set<String> friends
    = new HashSet<>(Arrays.asList("Raphael", "Olivia", "Thibaut"));
    ```
    
- 스트림 API 사용
    
    ```java
    Set<String> friends
    = Stream.of("Raphael", "Olivia", "Thibaut")
    				.collect(Collectors.toSet());
    ```
    

⇒ 두 방법 모두 불필요한 객체 할당 필요로 함

⇒ 자바9에서는 작은 리스트, 집합, 맵을 쉽게 만들 수 있는 팩토리 메서드 제공

## 8.1.1 리스트 팩토리

List.of() 메소드

- add() 불가
- set() 불가
- 컬렉션이 의도치 않게 변하는 것을 막음

1. 오버로딩
    
    ```java
    static <E> List<E> of(E e1, E e2, E e3, E e4)
    statuc <E> List<E> of(E e1, E e2, E e3, E e4, E e5)
    ```
    
    - 10개 미만 요소
2. 가변 인수
    
    ```java
    statuc <E> List<E> of(E... elements)
    ```
    
    - 내부적으로 추가 배열을 할당해서 리스트로 감쌈
    - 10개 이상의 요소는 가변 인수

### 8.1.2 집합 팩토리

Set.of() 메소드

- 중복된 요소 포함해 집합 생성: IllegalArgumentException 발생

### 8.1.3 맵 팩토리

Map.of() 메소드

- 10개 이하
- 키와 값 번걸아 제공
    
    ```java
    Map<String, Integer> ageOfFriends
    = Map.of("Raphael", 30, "Olivia", 25, "Thibaut", 26);
    ```
    

Map.ofEntries() 메소드

- 10개 이상
- 가변 인수로 구현
- Map.Entry<K, V> 객체를 인수로 받음
- 추가 객체 할당 필요
    
    ```java
    Map<String, Integer> ageOfFriends
    = Map.ofEntries(entry("Raphael", 30),
    								entry("Olivia", 25),
    								entry("Thibaut", 26));
    ```
    

# 8.2 리스트와 집합 처리

컬렉션을 바꾸는 동작: 에러 유발, 복잡함

⇒ 자바8: 호출한 컬렉션 자체를 바꾸는 메서드들 추가

- removeIf: List, Set에서 이용. 프리디케이트 만족하는 요소 제거
- replaceAll: List에서 이용. UnaryOperator 함수를 이용해 요소 변경
- sort: List에서 이용. 리스트 정렬

## 8.2.1 removeIf 메서드

```java
for (Transaction transaction : transactions) {
	if(Character, isDigit(transaction.getReferenceCode().charAt(0))) {
		transactions. remove(transaction);
	}
}
```

- 숫자로 시작되는 참조 코드를 가진 트랜잭션 삭제
- ConcurrentModificationException 발생
    - for-each는 내부적으로 Iterator 객체 사용
    
    ```java
    for (Iterator<Transaction> iterator = transactions,iterator();
    iterator hasNext);) {
    	Transaction transaction = iterator.next();
    	if(Character isDigit(transaction.getReferenceCode().charAt(0))) {
    		transactions.remove(transaction);
    	}
    }
    ```
    
    - Iterator 객체와 Collection 객체 둘 다 요소 관리/접근: 반복자의 상태와 컬렉션의 상태가 동기화되지 않음
    - Iterator를 명시적으로 사용하고 Iterator를 통해 remove하면 해결 가능 ⇒ 코드가 복잡해짐 ⇒ removeIf로 해결

## 8.2.2 replaceAll 메서드

요소를 새로운 요소로 변경

Stream API

```java
referenceCodes.stream()
	.map(code -> Character.toUpperCase(code.charAt(0)) + code.substring(1))
	.collect(Collectors.toList())
	.forEach(System.out::println);
```

- 그러나 스트림 API는 새 컬렉션 생성
- 우리가 원하는 건 기존 컬렉션의 변경

ListIterator 객체

```java
for (ListIterator<String> iterator = referenceCodes.listIterator();
iterator.hasNext();) {
	String code = iterator.next();
	iterator.set(Character.toUpperCase(code.charAt(0)) + code.substring(1));
```

- 코드 복잡함
- 컬렉션 객체를 Iterator와 혼용하면 문제가 쉽게 발생

replaceAll

```java
referenceCodes.replaceA11(code -> Character.toUpperCase(code.charAt(0)) + code.substring (1));
```

# 8.3 맵 처리

Map 디폴트 메서드: 기본적인 구현을 인터페이스에 제공하는 기능

## 8.3.1 forEach 메서드

```java
for(Map.Entry<String, Integer› entry : ageOfFriends.entrySet()) {
	String friend = entry.getKey();
	Integer age = entry.getValue();
	System.out println(friend + " is " + age + " years old");
}
```

자바 8 이후

```java
ageOfFriends.forEach((friend, age) -> System.out printin(friend + " is " + age + " years old"));
```

## 8.3.2 정렬 메서드

- Entry.comparingByValue: 값 기준 정렬 시 사용
- Entry.comparingByKey: 키 기준 정렬 시 사용

```java
Map<String, String> favouriteMovies
= Map.ofEntries(entry("Raphael", "Star Wars"),
								entry("Cristina", "Matrix"),
								entry("Olivia", "James Bond"));
```

```java
favouriteMovies
	.entrySet()
	.stream ()
	.sorted(Entry.comparingByKey())
	.forEachOrdered(System.out::println);
```

결과

```java
Cristina=Matrix
Olivia=James Bond
Raphael=Star Wars
```

## 8.3.3 getOrDefault 메서드

요청한 키가 맵에 존재하지 않을 때

- NPE 방지: 요청 결과가 널인지 확인
- getOrDefault: 기본값 반환

첫 번째 인수로 키, 두 번째 인수로 기본값 받음

⇒ 맵에 키가 존재하지 않으면 두 번째 인수로 바든 기본값 반환

```java
Map<String, String> favouriteMovies
= Map.ofEntries(entry ("Raphael", "Star Wars"), entry("Olivia", "James Bond"));

System.out.println(favouriteMovies.get0rDefault("Olivia", "Matrix"));
System.out.println(favouriteMovies.get0rDefault("Thibaut", "Matrix"));
```

- 키가 존재하고 값이 null이어도 getOrDefault는 null 반환

## 8.3.4 계산 패턴

맵에 키가 존재하는지 여부에 따라 어떤 동작을 실행하고 결과를 저장해야 하는 상황

- computeIfAbsent: 키에 해당하는 값이 없거나 널이면 키를 이용해 새 값 계산 + 맵에 추가
- computeIfPresent: 키가 존재하면 새 값 계산 + 맵 추가
- compute: 키로 새 값 계산하고 맵에 저장

## 8.3.5 삭제 패턴

remove 메서드: 제공된 키에 해당하는 맵 항목 제거

자바 8: 키가 특정한 값과 연관되었을 때만 항목 제거

- 오버로드 버전 메서드

```java
String key = "Raphael" ;
String value = "Jack Reacher 2";
if (favouriteMovies.containsKey(key) && Objects.equals(favouriteovies.get(key), value)) {
	favouriteMovies, remove(key);
	return true;
}
else {
	return false;
}
```

⇒

```java
favouriteMovies.remove(key, value);
```

## 8.3.6 교체 패턴

- replaceAll: BiFunction 적용 결과 → 각 항목의 값 교체
- Replace: 키가 존재하면 맵의 값 변경
    - 오버로드 버전: 키가 특정 값으로 매핑됐을 때만 교체

```java
Map<String, String> favouriteMovies = new HashMap<>();
favouriteMovies.put ("Raphael", "Star Wars");
tavouriteMovies.put ("Olivia", "james bond");
favouriteMovies.replaceAl1((friend, movie) -> movie, toUpperCase());
System.out println (favouriteMovies);
```

## 8.3.7 합침

putAll: 다른 맵의 모든 항목을 현재 맵에 복사

- 중복 키가 없으면 그대로 합쳐짐
- 중복 키가 있으면 기존 값 덮어써질 수 있음

```java
Map<String, String> family = Map.ofEntries(
    entry("Teo", "Star Wars"),
    entry("Cristina", "James Bond")
);

Map<String, String> friends = Map.ofEntries(
    entry("Raphael", "Star Wars")
);

Map<String, String> everyone = new HashMap<>(family);
everyone.putAll(friends);

System.out.println(everyone);
// {Cristina=James Bond, Raphael=Star Wars, Teo=Star Wars}
```

merge: 중복 키가 있을 때 값을 어떻게 합칠지 BiFunction으로 지정

- 키가 없거나 기존 값이 null이면 새 값을 그대로 추가
- 키가 이미 존재하면 기존 값과 새 값을 조합
- 병합 결과가 null이면 해당 항목 제거

```java
Map<String, String> family = Map.ofEntries(
    entry("Teo", "Star Wars"),
    entry("Cristina", "James Bond")
);

Map<String, String> friends = Map.ofEntries(
    entry("Raphael", "Star Wars"),
    entry("Cristina", "Matrix")
);

Map<String, String> everyone = new HashMap<>(family);

friends.forEach((k, v) ->
    everyone.merge(k, v, (movie1, movie2) -> movie1 + " & " + movie2)
);

System.out.println(everyone);
// {Raphael=Star Wars, Cristina=James Bond & Matrix, Teo=Star Wars}
```

- merge를 이용하면 초기화 검사 코드를 단순화할 수 있음
    - 기존 값이 없으면 두 번째 인수 값을 사용
    - 기존 값이 있으면 BiFunction을 적용해 값 갱신

```java
Map<String, Long> moviesToCount = new HashMap<>();
String movieName = "JamesBond";

moviesToCount.merge(movieName, 1L, (count, increment) -> count + 1L);
```

- 반복자를 이용한 삭제 코드는 removeIf로 단순화 가능
    - 맵의 entrySet()에 조건을 적용해 항목 삭제

```java
Map<String, Integer> movies = new HashMap<>();
movies.put("JamesBond", 20);
movies.put("Matrix", 15);
movies.put("Harry Potter", 5);

movies.entrySet().removeIf(entry -> entry.getValue() < 10);

System.out.println(movies);
// {Matrix=15, JamesBond=20}
```

# 8.4 개선된 ConcurrentHashMap

- 동시성 친화적, 최신 기술 반영한 HashMap
- 내부 자료구조의 특정 부분만 잠굼: 동시 추가, 갱신 작업 허용
- 동기화된 Hashtable 버전에 비해 읽기/쓰기 연산 성능이 월등함

## 8.4.1 리듀스와 검색

- forEach: 각 (키, 값) 쌍에 주어진 액션 실행
- reduce: 모든 (키, 값) 쌍을 리듀스 함수로 합쳐 결과 생성
- search: null이 아닌 값을 반환할 때까지 각 (키, 값) 쌍에 함수 적용

네 가지 연산 형태

- 키, 값으로 연산: forEach, reduce, search
- 키로 연산: forEachKey, reduceKeys, searchKeys
- 값으로 연산: forEachValue, reduceValues, searchValues
- Map, Entry 객체로 연산: forEachEntry, reduceEntries, searchEntries

ConcurrentHashMap의 상태를 잠그지 않고 연산 수행

- 계산이 진행되는 동안 바뀔 수 있는 객체, 값, 순서 등에 의존하지 않아야 함

연산에 병렬성 기준값 지정해야 함: 맵의 크기가 주어진 기준값보다 작으면 순차적으로 연산 실행

- 기준값 1: 공통 스레드 풀 이용 → 병렬성 극대화
- 기준값 Long.MAX_VALUE: 스레드 한 개

```java
ConcurrentHashMap<String, Long> map = new ConcurrentHashMap<>();
long parallelismThreshold = 1;
Optional<Integer> maxValue =
Optional.ofNullable(map, reduceValues(parallelismThreshold, Long::max));
```

- 기본값 전용 each reduce 연산: 박싱 작업 X
    - reduceValuesToInt
    - reduceKeysToLong

## 8.4.2 계수

mappingCount 메서드: 맵의 매핑 개수 반환

- size 메서드와 달리 long 반환: 매핑 개수가 int의 범위 넘어서는 상황 대처

## 8.4.3 집합뷰

keySet 메서드: ConcurrentHashMap → 집합 뷰로 변환

- 맵을 바꾸면 집합도 바뀌고, 집합을 바꾸면 맵도 영향을 받음

newKeySet 메서드: ConcurrentHashpMap으로 유지되는 집합 생성
