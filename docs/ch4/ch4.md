스트림의 필요성

- 컬렉션으로 데이터베이스 질의처럼 ‘선언형’으로 처리되는 기능을 만들 수 있지 않을까?
- 컬렉션의 병렬 처리 코드는 복잡하고 어려움

# 4.1 스트림이란 무엇인가?

스트림

- 자바 8 API에 새로 추가
- 선언형으로 컬렉션 데이터 처리
- ‘투명한’ 병렬성 제공

기존 자바7 코드

```java
List<Dish> lowCaloricDishes = new ArrayList<>();
for(Dish dish: menu) {
	if(dish.getcalories () < 400) {
		lowCaloricDishes.add (dish);
	}
}
Collections.sort(loCaloricDishes, new Comparator Dish>() {
	public int compare(Dish dish1, Dish dish2) {
		return Integer.compare(dish1.getCalories(), dish2.getCalories());
	}
});
List<String> lowCaloricDishesName = new ArrayList<>();
for (Dish dish: lowCaloricDishes) {
	lowCaloricDishesName.add(dish.getName());
}
```

1. 요소 필터링: for문
2. 요소 정렬: 익명 클래스
3. 요소 속성 선택: for문
- 가비지 변수: lowCaloricDishes (처리 결과 저장을 위해 필요)

자바8 코드

```java
import static java.util. Comparator.comparing;
import static java.util.stream.Collectors.toList;
List<String> lowCaloricDishesName =
						menu.stream()
								.filter(d -> d.getCalories() < 400)
								.sorted(comparing(Dish::get(alories))
								.map(Dish::getname)
								.collect(toList())
```

1. 요소 필터링: filter
2. 요소 정렬: sorted
3. 요소 속성 선택: map
- 가비지 변수로 중간 처리 결과를 저장하는 대신, collect로 바로 결과 반환
- parallelStream(): 7장에서 설명

### 스트림의 장점

- 선언형 코드 구현
    - 제어 블록으로 동작을 직접 구현하기보다, 어떤 동작을 수행할지 ‘지정’해줄 수 있음
- 조립 가능
    - 빌딩 블록 연산(filter, sorted, map, collect) 연결 → 데이터 처리 파이프라인
    - 각 연산의 결과는 여전히 스트림으로, 다음 연산으로 전달됨
    - 고수준 빌딩 블록
    
    ![image.png](attachment:e41001c3-bbc7-4acd-8272-1aea2a54fd96:image.png)
    
- 병렬화: 스레드, 락 걱정 없이 성능 향상

# 4.2 스트림 시작하기

### 컬렉션 스트림

**컬렉션 인터페이스의 stream():** 스트림 반환

```java
default Stream<E> stream() {
    return StreamSupport.stream(spliterator(), false);
}
```

- **spliterator()**
    - 현재 컬렉션의 요소들을 순회할 수 있는 Spliterator<E> 객체를 가져옴
    - 컬렉션 원소를 직접 스트림으로 바꾸는 게 아니라, 먼저 컬렉션을 순회할 수 있는 표준 인터페이스인 Spliterator를 얻고, 그걸 스트림의 소스로 사용
    - Spliterator: 요소를 순회하고, 필요하면 분할할 수도 있는 순회 인터페이스
        - Iterator보다 더 많은 정보와 기능을 가짐
        - 특히 분할(split) 기능이 있어서 병렬 처리에 유리함
- **StreamSupport.stream(...)**
    - 이 컬렉션의 spliterator를 이용해서 순차 스트림을 하나 만들어 반환
    - Spliterator를 스트림 파이프라인의 “소스”로 등록해서, 스트림의 맨 앞단(Head) 객체를 만들어 반환하는 메서드
    
    ```java
    public static <T> Stream<T> stream(Spliterator<T> spliterator, boolean parallel) {
        Objects.requireNonNull(spliterator);
        return new ReferencePipeline.Head<>(
            spliterator,
            StreamOpFlag.fromCharacteristics(spliterator),
            parallel
        );
    }
    ```
    
    1. spliterator가 null인지 검사
    2. 그 spliterator의 특성(characteristics)으로 스트림 플래그를 생성 (ORDERED, SIZED, DISTINCT, SORTED)
    3. ReferencePipeline.Head라는 스트림 파이프라인의 시작점 객체를 생성해서 반환

```java
List<Integer> list = List.of(1, 2, 3, 4);
Stream<Integer> s = list.stream();
```

1. list.spliterator()가 호출
2. StreamSupport.stream(...)이 그 spliterator를 감싸서 스트림 객체 생성
    1. 내 데이터 소스는 list의 spliterator
    2. 아직 실제 순회는 시작 안 함
    3. 나중에 최종 연산이 오면 그때 원소를 꺼내 쓰자
3. 여기까진
    - 1, 2, 3, 4를 전부 복사하지도 않고
    - 필터링도 안 하고
    - 변환도 안 함
4. 

### 스트림이란?

데이터 처리 연산을 지원하도록 소스에서 추출된 연속된 요소

1. 연속된 요소: 
2. 소스: 컬렉션, 배열, I/O 자원 등의 데이터 제공 소스로부터 데이터 소비
3. 데이터 처리 연산: 데이터베이스와 비슷한 연산 지원

### 스트림 특징

1. 파이프라이닝: 스트림 연산은 스트림 자신을 반환 → 파이프라인 구성
    1. 게으름
    2. 쇼트서킷
2. 내부 반복

```java
import static java.util.stream.Collectors.toList;
List<String> threeHighCaloricDishNames =
				menu.stream()
						.filter(dish -> dish.get(alories()> 300);
						.map(Dish::getName)
						.limit(3)
						.collect(toList());
System.out.printin(threenighcaloricDishNames
```

- 데이터 소스: menu
    - 연속된 요소를 스트림에 제공
- 데이터 처리 연산: filter, map, limit
    - 스트림 반환 → 파이프라인 형성
- 결과 반환: collect
    - collect 호출 전까진 아무 일도 일어나지 않음 → 최적화 가능

# 4.3 스트림과 컬렉션

스트림과 컬렉션 모두 연속된 요소 형식 값 인터페이스 제공

“연속된”

- 순서를 무시하고 아무 값에나 접근하는 것이 아닌, 순차적응로 값에 접근

### 컬렉션

- DVD: 사용자가 시청하기 전에 모든 데이터를 미리 저장
- 이미 계산되었거나 이미 존재하는 값을 저장
- “결과” 저장

### 스트림

- 인터넷 스트리밍: 사용자가 시청하기 시작하면 데이터 불러오기 시작
- ‘어떻게 계산할지’ 미리 들고 있다가 ‘소비’하는 시점에 계산
- “과정” 저장
- 게으르게 만들어지는 컬렉션: 사용자가 데이터 요청할 때만 값 계산

⇒ 둘의 차이는 ‘데이터를 언제 계산하느냐’

## 4.3.1 딱 한 번만 탐색할 수 있다

탐색된 스트림의 요소는 소비됨

- 한 번 탐색한 요소를 다시 탐색하려면 초기 데이터 소스에서 새로운 스트림 만들어야 함

## 4.3.2 외부 반복과 내부 반복

![image.png](attachment:f3285a71-70ad-4c2f-8a0b-ba8e7fb1481e:image.png)

### 외부 반복

- 컬렉션 반복 방식
- 사용자가 직접 요소 반복

### 내부 반복

- 스트림 반복 방식
- 반복을 알아서 처리하고 결과 스트림값을 어딘가에 저장해줌

외부 반복은 각 요소 하나하나마다 동작을 지정해줘야 하는데,

내부 반복은 동작 하나를 지정해주면 알아서 모든 요소에 적용

- 내부 반복은 그 ‘동작’을 나타내는 연산이 제공되어야 함
- ‘동작 파라미터화’

# 4.4 스트림 연산

![image.png](attachment:268ff57a-7f91-4b16-bc84-bdc1ae297a71:image.png)

## 4.4.1 중간 연산

게으름: 아무 연산도 수행하지 않음

중간 연산을 합친 다음에, 합쳐진 중간 연산을 최종 연산으로 한 번에 처리

### 예시

```java
List<String> names =
		menu.stream()
				.filter(dish -> {
						System.out.printIn("filtering:" + dish.getName());
						return dish.getCalories() > 300;
				})
				.map (dish -> {
						System.out.println("mapping:" + dish.getName()); return dish.getName();
				})
				.limit(3)
				.collect(toList());
System.out.println(names) ;
```

### 처리 결과

```java
filtering:pork
mapping:pork
filtering:beef
mapping:beef
filtering:chicken
mapping:chicken
[pork, beef, chicken]
```

- 쇼트서킷: limit 연산에 의해 처음 3개만 선택
- 루프 퓨전: filter와 map이 한 과정으로 병합

## 4.4.2 최종 연산

결과 도출

스트림 이외의 결과 반환: List, Integer, void 등

```java
menu.stream().forEach(System.out::println);
```

## 4.4.3 스트림 이용하기

### 이용 과정

1. 질의를 수행할 데이터 소스
2. 스트림 파이프라인을 구성할 중간 연산 연결
3. 스트림 파이프라인을 실행하고 결과를 만들 최종 연산

### 빌더 패턴

- 스트림 파이프라인: 데이터를 어떻게 흘려보내고 변환할지 표현
- 빌더 패턴: 객체를 어떻게 조립해서 만들지 표현

```java
User user = User.builder()
    .name("Kim")
    .age(30)
    .email("kim@example.com")
    .build();
```
