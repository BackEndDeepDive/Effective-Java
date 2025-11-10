# Item 54 : null이 아닌, 빈 컬렉션이나 배열을 반환하라

**빈 컬렉션이나 빈 배열을 반환하는 것이 null을 반환하는 것보다 거의 항상 낫다.** 이 아이템에서는 왜 그런지, 그리고 어떻게 효율적으로 구현하는지 살펴본다.

<br/>

### null 반환의 문제점

먼저 null을 반환하는 전형적인 코드를 보자.

```java
public class Shop {
    private final List<Cheese> cheesesInStock = new ArrayList<>();
    
    /**
     * @return 재고가 있는 치즈 목록. 없으면 null 반환
     */
    public List<Cheese> getCheeses() {
        return cheesesInStock.isEmpty() ? null : new ArrayList<>(cheesesInStock);
    }
}
```

이 메서드를 사용하는 클라이언트는 **항상 null 체크를 해야 한다.**

```java
Shop shop = new Shop();
List<Cheese> cheeses = shop.getCheeses();

// 방어 코드 필수
if (cheeses != null && cheeses.contains(Cheese.STILTON)) {
    System.out.println("성공");
}
```

#### Q. 방어 코드를 빼먹으면 어떻게 될까?

```java
Shop shop = new Shop();
List<Cheese> cheeses = shop.getCheeses();

// null 체크를 깜빡함
for (Cheese cheese : cheeses) {  // NullPointerException
    System.out.println(cheese);
}
```

재고가 없을 때 `getCheeses()`가 `null`을 반환하면 위 코드는 `NullPointerException`을 던진다. 

문제는 **이런 오류가 런타임에만 발견된다**는 점이다. 테스트 커버리지가 충분하지 않으면 프로덕션 환경에서 발견될 수 있다.

#### null 반환은 API 사용을 어렵게 만든다

null을 반환하는 API는 **사용하기 어렵고 오류를 유발하기 쉽다.** 클라이언트는 항상 다음을 기억해야 한다:

- 이 메서드가 null을 반환할 수 있는가?
- null 체크를 해야 하는가?
- null 체크를 어디서 해야 하는가?

반면 빈 컬렉션을 반환하면 이런 고민이 사라진다.

<br/>

### 빈 컬렉션 반환의 장점

빈 컬렉션을 반환하도록 수정해보자.

```java
public class Shop {
    private final List<Cheese> cheesesInStock = new ArrayList<>();
    
    /**
     * @return 재고가 있는 치즈 목록. 항상 null이 아닌 리스트 반환
     */
    public List<Cheese> getCheeses() {
        return new ArrayList<>(cheesesInStock);
    }
}
```

클라이언트 코드가 훨씬 간결해진다.

```java
Shop shop = new Shop();
List<Cheese> cheeses = shop.getCheeses();

// null 체크 불필요
for (Cheese cheese : cheeses) {  // 재고가 없으면 루프를 실행하지 않음
    System.out.println(cheese);
}
```

빈 컬렉션은 **정상적인 반복문의 시작 조건을 만족하지 않으므로** 자연스럽게 아무것도 실행하지 않는다. **즉, null 체크가 필요 없다.**

<br/>

### "빈 컬렉션 할당은 비용이 든다"는 오해

#### 1. 성능 차이는 무시할 수 있는 수준이다

빈 컬렉션을 할당하는 비용은 대부분의 경우 측정조차 어려울 정도로 작다. **성능 최적화는 측정 가능한 성능 문제가 있을 때만** 해야 한다.

```java
// 빈 ArrayList 할당 비용은 매우 작다
public List<Cheese> getCheeses() {
    return new ArrayList<>();
}
```

#### 2. 불변 빈 컬렉션을 재사용하면 할당도 없다

빈 컬렉션을 매번 할당하는 것이 정말 문제라면, **불변 빈 컬렉션을 재사용**하면 된다.

```java
// 최적화된 버전: 불변 빈 컬렉션 재사용
public List<Cheese> getCheeses() {
    return cheesesInStock.isEmpty() 
        ? Collections.emptyList()  // 할당 없음, 항상 같은 인스턴스
        : new ArrayList<>(cheesesInStock);
}
```

`Collections.emptyList()`는 **싱글턴 인스턴스**를 반환한다. 즉, 어디서 호출하든 같은 객체를 반환하므로 메모리 할당이 일어나지 않는다.
![](https://velog.velcdn.com/images/kguswo/post/2fb9c946-b147-4aa5-9f7c-7df7cdd158b3/image.png)

#### 3. 잘못된 최적화는 코드 품질을 해친다

null 반환으로 얻는 미세한 성능 이득(있다면)보다 **클라이언트 코드의 복잡도 증가와 오류 가능성**이 훨씬 큰 손실이다. Donald Knuth의 유명한 말을 기억하라: "성급한 최적화는 모든 악의 근원이다."

<br/>

### 빈 배열 반환

배열을 반환하는 메서드도 null 대신 **길이가 0인 배열**을 반환해야 한다.

```java
// null 반환
public Cheese[] getCheeses() {
    return cheesesInStock.isEmpty() ? null : 
        cheesesInStock.toArray(new Cheese[0]);
}

// 빈 배열 반환
public Cheese[] getCheeses() {
    return cheesesInStock.toArray(new Cheese[0]);
}
```

빈 배열도 매번 새로 할당하지 않고 재사용할 수 있다.

```java
// 빈 배열 재사용
private static final Cheese[] EMPTY_CHEESE_ARRAY = new Cheese[0];

public Cheese[] getCheeses() {
    return cheesesInStock.toArray(EMPTY_CHEESE_ARRAY);
}
```

`toArray` 메서드는 **입력 배열이 충분히 크면 그 배열에 채워 반환하고, 충분히 크지 않으면 새 배열을 할당해 반환한다.**

#### 잘못된 최적화 패턴 주의

다음과 같은 코드를 작성하지 말라.

```java
// 나쁜 예: 잘못된 최적화
public Cheese[] getCheeses() {
    return cheesesInStock.toArray(new Cheese[cheesesInStock.size()]);
}
```

이 코드는 `toArray`에 미리 크기를 맞춘 배열을 넘긴다. 얼핏 보면 효율적일 것 같지만, 실제로는 **성능을 해칠 수 있다.** 최신 JVM은 길이 0인 배열을 전달받으면 내부적으로 최적화된 경로를 사용한다.

```java
// 권장: 길이 0인 배열 전달
return cheesesInStock.toArray(new Cheese[0]);
```

<br/>

### 성능 측정: null vs 빈 컬렉션

실제로 성능 차이가 얼마나 될까? 간단한 벤치마크를 작성해보자.

```java
import java.util.*;

public class PerformanceTest {
    private static final int ITERATIONS = 10_000_000;
    private static final List<String> EMPTY = Collections.emptyList();
    
    public static void main(String[] args) {
        // 테스트 1: null 반환
        long start1 = System.nanoTime();
        for (int i = 0; i < ITERATIONS; i++) {
            List<String> result = returnsNull();
            if (result != null) {
                result.size();
            }
        }
        long time1 = System.nanoTime() - start1;
        
        // 테스트 2: 빈 컬렉션 반환 (매번 생성)
        long start2 = System.nanoTime();
        for (int i = 0; i < ITERATIONS; i++) {
            List<String> result = returnsNewEmpty();
            result.size();
        }
        long time2 = System.nanoTime() - start2;
        
        // 테스트 3: 빈 컬렉션 반환 (재사용)
        long start3 = System.nanoTime();
        for (int i = 0; i < ITERATIONS; i++) {
            List<String> result = returnsSharedEmpty();
            result.size();
        }
        long time3 = System.nanoTime() - start3;
        
        System.out.println("null 반환: " + time1 / 1_000_000 + "ms");
        System.out.println("빈 리스트 매번 생성: " + time2 / 1_000_000 + "ms");
        System.out.println("빈 리스트 재사용: " + time3 / 1_000_000 + "ms");
    }
    
    private static List<String> returnsNull() {
        return null;
    }
    
    private static List<String> returnsNewEmpty() {
        return new ArrayList<>();
    }
    
    private static List<String> returnsSharedEmpty() {
        return EMPTY;
    }
}
```

실행 결과 (환경에 따라 다를 수 있음):
```
null 반환: 3ms
빈 리스트 매번 생성: 62ms
빈 리스트 재사용: 4ms
```

**빈 컬렉션을 재사용하면 null 반환과 성능이 거의 동일하다.** null 반환과 빈 컬렉션 재사용의 차이는 1ms에 불과하다. 반면 빈 리스트를 매번 생성하더라도 1천만 번 반복에 62ms로, 한 번 호출당 6.2 나노초에 불과하다. 이 정도 차이는 실제 애플리케이션에서는 무시할 수 있는 수준이다.

<br/>

---

### References
- 이펙티브 자바 3/E