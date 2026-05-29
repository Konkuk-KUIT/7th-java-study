https://clover-shift-99f.notion.site/Chapter-9-36e0b82d4e9c801aad79d100123c2f6e?source=copy_link


## 9.1 가독성과 유연성을 개선하는 리팩터링

지금까지 배운 람다, 메서드 참조, 스트림 등의 기능을 이용해서 더 가독성이 좋고 유연한 코드로 리팩터링하는 방법을 소개한다.

### 9.1.1 코드 가독성 개선

코드 가독성을 개선한다는 것은 우리가 구현한 코드를 다른 사람이 쉽게 이해하고 유지보수할 수 있게 만드는 것을 의미한다.

자바 8의 새 기능을 통해 코드의 가독성을 높일 수 있다. 또한 메서드 참조와 스트림 API를 이용해 코드의 의도를 명확하게 보여줄 수 있다.

### 9.1.2 익명 클래스를 람다 표현식으로 리팩터링하기

하나의 추상 메서드를 구현하는 익명 클래스는 람다 표현식으로 리팩터링할 수 있다. 하지만 모든 익명 클래스를 람다 표현식으로 변환할 수 있는 것은 아니다. 

첫째, 익명 클래스에서 사용한 this와 super는 람다 표현식에서 다른 의미를 갖는다. 익명 클래스에서 this는 익명클래스 자신이지만 람다에서는 람다를 감싸는 클래스를 의미한다. 

둘째, 익명 클래스는 감싸고 있는 클래스의 변수를 가릴 수 있지만 람다로는 가릴 수 없다.

```java
// 익명 클래스 사용
int a = 10;
Runnable r1 = new Runnable() {
	     public void run(){
				     int a = 2; // 정상 작동
				     System.out.println(a);
			 }
};

// 람다 표현식 사용
Runnable r2 = () -> {
				int a = 2; // 컴파일 에러 (variable a is already defined)
				System.out.println(a);
}; 
```

### 9.1.3 람다 표현식을 메서드 참조로 리팩터링하기

람다 표현식 대신 메서드 참조를 이용하면 가독성을 높일 수 있다. 메서드명으로 코드의 의도를 명확하게 알릴 수 있기 때문이다.

6장에서 사용했던 예제 코드이다.

```java
Map<CaloricLevel, List<Dish>> dishesByCaloricLevel =
		menu.stream()
				.collect(
						groupingBy(dish -> {
							if (dish.getCalories() <= 400) return CaloricLevel.DIET;
							else if (dish.getCalories() <= 700) return CaloricLevel.NORMAL;
							else return CaloricLevel.FAT;
}));
```

람다 표현식을 별도의 메서드로 추출하고 groupingBy에 인수로 전달할 수 있다. 다음처럼 코드가 간결하고 의도도 명확해진다.

```java
Map<CaloricLevel, List<Dish>> dishesByCaloricLevel =
		menu.stream().collect(groupingBy(Dish::getCaloricLevel));
```

### 9.1.4 명령형 데이터 처리를 스트림으로 리팩터링하기

스트림 API는 데이터 처리 파이프라인의 의도를 더 명확하게 보여준다. 스트림은 쇼트서킷과 게으름이라는 강력한 최적화뿐 아니라 멀티코어 아키텍처를 활용할 수 있는 지름길을 제공한다. 

명령형 코드의 break, continue, return 등의 제어 흐름문을 모두 분석해서 같은 기능을 수행하는 스트림 연산으로 유추해야 하므로 명령형 코드를 스트림 API로 바꾸는 것은 쉬운 일이 아니지만 명령형 코드를 스트림 API로 바꾸도록 도움을 주는 도구들이 존재한다.

### 9.1.5 코드 유연성 개선

람다 표현식을 이용하면 동작 파라미터화를 쉽게 구현할 수 있고, 다양한 람다를 전달해서 다양한 동작을 표현할 수 있다. 따라서 변화하는 요구사항에 대응할 수 있는 코드를 구현할 수 있다.

#### 함수형 인터페이스 적용

람다 표현식을 이용하려면 함수형 인터페이스가 필요하다. 

조건부 연기 실행과 실행 어라운드, 즉 두 가지 자주 사용하는 패턴으로 람다 표현식 리팩터링을 살펴본다.

#### 조건부 연기 실행

실제 작업을 처리하는 코드 내부에 제어 흐름문이 복잡하게 얽힌 코드를 흔히 볼 수 있다.  

```java
if (logger.isLoggable(Log.FINER)) {
	logger.finer("Problem: " + generateDiagnostic());
}
```

위 코드는 다음과 같은 문제가 있다.

- logger의 상태가 isLoggable이라는 메서드에 의해 클라이언트 코드로 노출된다.
- 메시지를 로깅할 때마다 logger 객체의 상태를 매번 확인해야 한다.

람다를 이용하면 이 문제를 해결할 수 있다. 특정 조건에서만 메시지가 생성될 수 있도록 생성 과정을 연기할 수 있어야 한다. 자바 8에서는 Supplier를 인수로 갖는 오버로드된 log 메서드를 제공한다.

```java
public void log(Level level, Supplier<String> msgSupplier)
```

이 메서드는 logger의 수준이 적절하게 설정되어 있을 때만 람다를 실행한다.

만일 클라이언트 코드에서 객체 상태를 자주 확인하거나 객체의 일부 메서드를 호출하는 상황이라면 내부적으로 객체의 상태를 확인한 다음에 메서드를 호출하도록 새로운 메서드를 구현하는 것이 좋다.

#### 실행 어라운드

매번 같은 준비, 종료 과정을 반복적으로 수행하는 코드가 있다면 이를 람다로 변환할 수 있다. 

이 코드는 파일을 열고 닫을 때 같은 로직을 사용했지만 람다를 이용해서 다양한 방식으로 파일을 처리할 수 있도록 파라미터화되었다.

```java
String oneLine =
			processFile((BufferedReader b) -> b.readLine());
String twoLines =
			processFile((BufferedReader b) -> b.readLine() + b.readLine());
```

## 9.2 람다로 객체지향 디자인 패턴 리팩터링하기

디자인 패턴에 람다 표현식이 더해지면 색다른 기능을 발휘할 수 있다. 또한 람다 표현식으로 기존의 많은 객체지향 디자인 패턴을 제거하거나 간결하게 재구현할 수 있다.

다음 다섯 가지 패턴을 이 절에서 살펴본다.

- 전략
- 템플릿 메서드
- 옵저버
- 의무 체인
- 팩토리

### 9.2.1 전략

전략 패턴은 한 유형의 알고리즘을 보유한 상태에서 런타임에 적절한 알고리즘을 선택하는 기법이다. 다양한 기준을 갖는 입력값을 검증하거나, 다양한 파싱 방법을 사용하거나, 입력 형식을 설정하는 등 다양한 시나리오에 전략 패턴을 활용할 수 있다.

텍스트 입력이 다양한 조건에 맞게 포맷되어 있는지 검증한다고 가정하자.

먼저 인터페이스부터 구현한다.

```java
interface ValidationStrategy {
    boolean execute(String s);
  }
```

ValidationStrategy를 구현하는 클래스를 정의한다.

```java
static private class IsAllLowerCase implements ValidationStrategy {
    @Override
    public boolean execute(String s) {
      return s.matches("[a-z]+");
    }
  }

static private class IsNumeric implements ValidationStrategy {
    @Override
    public boolean execute(String s) {
      return s.matches("\\d+");
    }
  }
```

이를 다양한 검증 전략으로 활용할 수 있다.

```java
Validator v1 = new Validator(new IsNumeric());

Validator v2 = new Validator(new IsAllLowerCase());
```

#### 람다 표현식 사용

ValidationStrategy는 함수형 인터페이스이고 Predicate<String>과 같은 함수 디스크립터를 갖고 있다. 따라서 새로운 클래스를 구현할 필요 없이 람다 표현식을 직접 전달하면 코드가 간결해진다.

```java
 Validator v3 = new Validator((String s) -> s.matches("\\d+"));

 Validator v4 = new Validator((String s) -> s.matches("[a-z]+"));
```

### 9.2.2 템플릿 메서드

템플릿 메서드는 ‘이 알고리즘을 사용하고 싶은데 그대로는 안 되고 조금 고쳐야 하는’ 상황에 적합하다. 

예제를 살펴보자. 간단한 온라인 뱅킹 애플리케이션을 구현할 것이다. 사용자가 고객 ID를 애플리케이션에 입력하면 은행 DB에서 고객 정보를 가져오고 서비스를 제공한다. 은행마다 동작 방법이 다르다. 먼저 다음은 애플리케이션의 동작을 정의하는 추상 클래스이다.

```java
abstract class OnlineBanking {

  public void processCustomer(int id) {
    Customer c = Database.getCustomerWithId(id);
    makeCustomerHappy(c);
  }

  abstract void makeCustomerHappy(Customer c);
}
```

각각의 지점은 OnlineBanking 클래스를 상속받아 makeCustomerHappy 메서드가 원하는 동작을 수행하도록 구현할 수 있다.

#### 람다 표현식 사용

람다나 메서드 참조로 알고리즘에 추가할 다양한 컴포넌트를 구현할 수 있다. 

이전에 정의한 makeCustomerHappy의 메서드 시그니처와 일치하도록 **`Consumer<Customer>`** 형식을 갖는 두 번째 인수를 processCustomer에 추가한다.

```java
public void processCustomer(int id, Consumer<Customer> makeCustomerHappy) {
    Customer c = Database.getCustomerWithId(id);
    makeCustomerHappy.accept(c);
  }
```

이제 OnlineBanking 클래스를 상속받지 않고 직접 람다 표현식을 전달해서 다양한 동작을 추가할 수 있다.

```java
new OnlineBankingLambda().processCustomer(1337, (Customer c) -> 
System.out.println("Hello!"));
```

### 9.2.3 옵저버

어떤 이벤트가 발생했을 때 한 객체가 다른 객체 리스트에 자동으로 알림을 보내야 하는 상황에 옵저버 디자인 패턴을 사용한다.

다양한 신문 매체가 뉴스 트윗을 구독하고 있고, 특정 키워드를 포함하는 트윗이 등록되면 알림이 전송되는 예제를 보자.

우선 다양한 옵저버를 그룹화할 Observer 인터페이스가 필요하다.

```java
interface Observer {
    void notify(String tweet);
  }
```

이제 트윗에 포함된 다양한 키워드에 다른 동작을 수행할 수 있는 여러 옵저버를 정의할 수 있다.

```java
class NYTimes implements Observer {
    @Override
    public void notify(String tweet) {
      if (tweet != null && tweet.contains("money")) {
        System.out.println("Breaking news in NY!" + tweet);
      }
    }
  }

class Guardian implements Observer {
    @Override
    public void notify(String tweet) {
      if (tweet != null && tweet.contains("queen")) {
        System.out.println("Yet another news in London... " + tweet);
      }
    }
  }

class LeMonde implements Observer {
    @Override
    public void notify(String tweet) {
      if (tweet != null && tweet.contains("wine")) {
        System.out.println("Today cheese, wine and news! " + tweet);
      }
    }
  }
```

그리고 주제도 구현해야 한다.

```java
interface Subject {
    void registerObserver(Observer o);
    void notifyObservers(String tweet);
  }
```

주제는 registerObserver 메서드로 새로운 옵저버를 등록한 다음에notifyObservers 메서드로 트윗의 옵저버에 이를 알린다.

```java
class Feed implements Subject {

    private final List<Observer> observers = new ArrayList<>();

    @Override
    public void registerObserver(Observer o) {
      observers.add(o);
    }

    @Override
    public void notifyObservers(String tweet) {
      observers.forEach(o -> o.notify(tweet));
    }
  }
```

이제 주제와 옵저버를 연결하는 애플리케이션을 만들 수 있다.

```java
Feed f = new Feed();
f.registerObserver(new NYTimes());
f.registerObserver(new Guardian());
f.registerObserver(new LeMonde());
f.notifyObservers("The queen said her favourite book is Java 8 & 9 in Action!");
```

#### 람다 표현식 사용하기

Observer 클래스를 구현하는 모든 클래스는 하나의 메서드 notify를 구현한다. 람다를 통해 세 개의 옵저버를 명시적으로 인스턴스화하지 않고 람다 표현식을 직접 전달해서 실행할 동작을 지정할 수 있다.

```java
feedLambda.registerObserver((String tweet) -> {
      if (tweet != null && tweet.contains("money")) {
        System.out.println("Breaking news in NY! " + tweet);
      }
    });
feedLambda.registerObserver((String tweet) -> {
      if (tweet != null && tweet.contains("queen")) {
        System.out.println("Yet another news in London... " + tweet);
      }
    });
```

하지만 옵저버가 상태를 가지며, 여러 메서드를 정의하는 등 복잡하다면 람다 표현식보다 기존의 클래스 구현 방식을 고수하는 것이 바람직할 수도 있다.

### 9.2.4 의무 체인

작업 처리 객체의 체인을 만들 때는 의무 체인 패턴을 사용한다. 한 객체가 어떤 작업을 처리한 다음에 다른 객체로 결과를 전달하고, 다른 객체도 전달받아 작업하고 전달하는 식이다.

다음 예제는 텍스트를 처리하는 예제이다. 두 작업 처리 객체를 연결하여 작업 체인을 만들 수 있다.

```java
ProcessingObject<String> p1 = new HeaderTextProcessing();
ProcessingObject<String> p2 = new SpellCheckerProcessing();
p1.setSuccessor(p2);
String result1 = p1.handle("Aren't labdas really sexy?!!");
System.out.println(result1);
```

#### 람다 표현식 사용

작업 처리 객체를 Function<String, String>, UnaryOperator<String> 형식의 인스턴스로 표현할 수 있다. andThen 메서드로 이들 함수를 조합해서 체인을 만들 수 있다.

```java
UnaryOperator<String> headerProcessing = (String text) -> "From Raoul, Mario and Alan: " + text;
UnaryOperator<String> spellCheckerProcessing = (String text) -> text.replaceAll("labda", "lambda");
Function<String, String> pipeline = headerProcessing.andThen(spellCheckerProcessing);
String result2 = pipeline.apply("Aren't labdas really sexy?!!");
System.out.println(result2);
```

### 9.2.5 팩토리

인스턴스화 로직을 클라이언트에 노출하지 않고 객체를 만들 때 팩토리 디자인 패턴을 사용한다. 

다양한 상품을 만드는 Factory 클래스 예제를 보자.

```java
public class ProductFactory {

    public static Product createProduct(String name) {
      switch (name) {
        case "loan":
          return new Loan();
        case "stock":
          return new Stock();
        case "bond":
          return new Bond();
        default:
          throw new RuntimeException("No such product " + name);
      }
    }
}
```

#### 람다 표현식 사용

생성자도 메서드 참조처럼 접근할 수 있다.

다음처럼 Loan 생성자를 사용할 수 있다.

```java
 Supplier<Product> loanSupplier = Loan::new;
 Loan loan = loanSupllier.get();
```

이를 이용하여 상품명을 생성자로 연결하는 Map을 만들어서 코드를 재구현할 수 있다.

```java
final static private Map<String, Supplier<Product>> map = new HashMap<>();
  static {
    map.put("loan", Loan::new);
    map.put("stock", Stock::new);
    map.put("bond", Bond::new);
  }
  
public static Product createProductLambda(String name) {
      Supplier<Product> p = map.get(name);
      if (p != null) {
        return p.get();
      }
      throw new RuntimeException("No such product " + name);
}
```

하지만 팩토리 메서드가 상품 생성자로 여러 인수를 전달하는 상황에는 이 기법을 적용하기 어렵다. 인수 개수가 많아질수록 Map의 시그니처가 복잡해진다.

## 9.3 람다 테스팅

개발자는 프로그램이 의도대로 동작하는지 확인할 수 있는 단위 테스팅을 진행한다. 예를 들어 다음처럼 그래픽 애플리케이션의 일부인 Point 클래스가 있다고 가정하자.

```java
private static class Point {

    private int x;
    private int y;

    private Point(int x, int y) {
      this.x = x;
      this.y = y;
    }

    public int getX() {
      return x;
    }

    public void getY(int y) {
      return y;
    }
    
    public Point moveRightBy(int x) {
		   return new Point(this.x + x, this.y);
		}

  }
```

다음은 moveRIghtBy 메서드가 의도한대로 동작하는지 확인하는 단위 테스트다.

```java
@Test
public void testMoveRightBy() throws Exception{
		Point p1 = new Point(5, 5);
		Point p2 = p1.moveRightBy(10);
		assertEquals(15, p2.getX());
		assertEquals(5, p2.getY());
}
```

### 9.3.1 보이는 람다 표현식의 동작 테스팅

moveRightBy는 public이므로 테스트 케이스 내부에서 Point 클래스 코드를 테스트할 수 있다. 하지만 람다는 익명이므로 테스트 코드 이름을 호출할 수 없다.

따라서 필요하다면 람다를 필드에 저장해서 재사용할 수 있다.

```java
public final static Comparator<Point> compareByXAndThenY = 
		comparing(Point::getX).thenComparing(Point::getY);
		
		
// 단위 테스트
@Test
public void testComparingTwoPoints() throws Exception {
		Point p1 = new Point(10, 15);
		Point p1 = new Point(10, 20);
		int result = Point.compareByXAndThenY.compare(p1, p2);
		assertTrue(result < 0);
}
```

### 9.3.2 람다를 사용하는 메서드의 동작에 집중하라

람다의 목표는 정해진 동작을 다른 메서드에서 사용할 수 있도록 하나의 조각으로 캡슐화하는 것이다. 그러려면 세부 구현을 포함하는 람다 표현식을 공개하면 안 된다.

람다를 표현하는 메서드의 동작을 테스트함으로써 람다를 공개하지 않으면서도 람다 표현식을 검증할 수 있다.

다음 메서드를 살펴보자.

```java
public static List<Point> moveAllPointsRightBy(List<Point> points, int x) {
		return points.stream()
										.map(p -> new Point(p.getX() + x, p.getY()))
										.collect(toList());
}
```

이제 moveAllPointsRightBy 메서드의 동작을 확인할 수 있다.

```java
@Test
public void testMoveAllPointsRightBy() throws Exception {
		List<Point> points =
				Arrays.asList(new Point(5, 5), new Point(10, 5));
		List<Point> expectedPoints =
				Arrays.asList(new Point(15, 5), new Point(20, 5));
		List<Point> newPoints = Point.moveAllPointsRightBy(points, 10);
		assertEquals(expectedPoints, newpoints);
}
```

### 9.3.3 복잡한 람다를 개별 메서드로 분할하기

많은 로직을 포함하는 복잡한 람다 표현식을 접하는 경우가 있을 것이다. 이러한 람다 표현식은 람다 표현식을 메서드 참조로 바꾸면 일반 메서드를 테스트하듯이 람다 표현식을 테스트할 수 있다.

### 9.3.4 고차원 함수 테스팅

함수를 인수로 받거나 다른 함수를 반환하는 메서드는 사용하기 더 어렵다. 테스트해야 할 메서드가 다른 함수를 반환한다면 Comparator에서 살펴봤던 것처럼 함수형 인터페이스의 인스턴스로 간주하고 함수의 동작을 테스트한다.

## 9.4 디버깅

### 9.4.1 스택 트레이스 확인

프로그램이 멈췄다면 프로그램이 어떻게 멈추게 되었는지 프레임별로 보여주는 스택 트레이스를 얻을 수 있다. 

#### 람다와 스택 트레이스

람다 표현식은 이름이 없기 때문에 복잡한 스택 트레이스가 생성된다. 

```java
List<Point> points = Arrays.asList(new Point(12, 2), null);
points.stream().map(p -> p.getX()).forEach(System.out::println);
```

위 코드를 실행하면 다음과 같은 스택 트레이스가 출력된다.

```java
Exception in thread "main" java, lang. NullPointerException
	at Debugging.lambda$main$0(Debugging.java:6)
	at Debugging$$Lambda$5/284720968.apply(Unknown Source)
	at java.util.stream.ReferencePipeline$3$1.accept(ReferencePipeline
		java: 193)
	at java.util.Spliterators$ArraySpliterator.forEachRemaining(Spliterators
		java: 948)
```

```java
at Debugging.lambda$main$0(Debugging.java:6)
	at Debugging$$Lambda$5/284720968.apply(Unknown Source)
```

메서드 호출 리스트 중 이와 같은 문자는 람다 표현식 내부에서 에러가 발생했을을 가리킨다.

메서드 참조를 사용해도 스택 트레이스에는 메서드명이 나타나지 않는다.

그러나 메서드 참조를 사용하는 클래스와 같은 곳에 선언되어 있는 메서드를 참조할 때는 메서드 참조 이름이 스택 트레이스에 나타난다.

### 9.4.2 정보 로깅

스트림 파이프라인에 적용된 각각의 연산이 어떤 결과를 도출하는지 확인하려면 peek이라는 스트림 연산을 활용할 수 있다.

peek은 스트림 파이프라인의 각 동작 전후의 중간값을 출력한다.

