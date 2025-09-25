# Item 05 - 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라

## Intro

내부에 사전을 통해 다양한 기능을 제공하는 맞춤법 검사기를 생각해보자. 맞춤법 검사기는 사전에 의존하게 되는데 이런 맞춤법 검사기를 유틸리티 클래스나 싱글턴으로 구현할 경우 문제가 생길 수 있다.

먼저 유틸리티 클래스로 구현할 경우 맞춤법 검사기가 사전을 통한 부가 기능만 제공한다는 점과 맞춤법 검사기 객체를 매번 생성해서 사용하지 않아도 된다는 장점이 있지만 주어진 사전의 종류를 변경할 수 없다. 사전의 종류를 변경할 수 있게 구현할 경우 유틸리티 클래스가 상태를 가지게 되는데 이는 해당 유틸리티 클래스를 사용하는 다른 곳들에도 영향을 미칠 수 있어서 지양해야 한다.

싱글턴으로 구현한 경우도 마찬가지로 맞춤법 검사기의 사전을 변경할 수 없거나 변경 메서드를 제공하면 한번만 생성해서 공유하여 사용하는 싱글턴이 상태를 가지게 되어 이 역시 지양해야 한다.

---

## 상태를 가지는 유틸리티 클래스, 싱글톤 클래스

멀티스레드 환경에서 변경 가능한 공유 자원은 동기화 문제가 발생할 수 있다. 유틸리티 클래스의 경우 스레드마다 객체를 생성해서 사용하는 것이 아니라 클래스에 바로 접근해서 사용하는데 변경 가능한 필드가 존재하면 동기화 문제가 발생할 수 있다. 싱글턴 역시 하나의 싱글턴 객체를 모든 스레드가 공유해서 사용하는데 변경 가능한 필드가 존재하면 동기화 문제가 발생할 수 있다.

가상 시나리오로 사용자가 맞춤법 검사기에 활용할 사전의 종류와 단어들을 입력하면 사전을 기반으로 맞춤법 검사기를 준비하고(0.5초 소요) 올바른 단어인지 검색하고(0.5초 소요) 대체 단어를 추천하는 상황(0.5초 소요)이 있다고 했을 때 다음과 같이 영어 사전을 기반으로 대체 단어를 추천 받기 전에 다른 스레드가 맞춤법 검사기의 사전을 한국어로 변경하여 추천에 실패할 수 있다.

| 시간 (100ms) | 영어 스레드                            | 한국어 스레드      |
| ------------ | -------------------------------------- | ------------------ |
| 0            | 영어 사전 세팅                         |                    |
| 1            |                                        |                    |
| 2            |                                        |                    |
| 3            |                                        |                    |
| 4            |                                        |                    |
| 5            | 맞는 단어인지 검색                     |                    |
| 6            |                                        |                    |
| 7            | (영어 스레드가 사용하는 사전이 변경됨) | 한국어 사전 세팅   |
| 8            |                                        |                    |
| 9            |                                        |                    |
| 10           | 대체 단어 조회 (실패)                  |                    |
| 11           |                                        |                    |
| 12           |                                        | 맞는 단어인지 검색 |
| 13           |                                        |                    |
| 14           |                                        |                    |
| 15           | 작업 종료                              |                    |
| 16           |                                        |                    |
| 17           |                                        | 대체 단어 조회     |
| 18           |                                        |                    |
| 19           |                                        |                    |
| 20           |                                        |                    |
| 21           |                                        |                    |
| 22           |                                        | 작업 종료          |

---

아래와 같이 내부 사전을 변경할 수 있게 설계하고

```java
public class SpellCheckerUtil {

    private static Lexicon dictionary = new EnglishDictionary();

    private SpellCheckerUtil() {
    }

    public static boolean isValid(String word) {
        return dictionary.isValid(word);
    }

    public static List<String> suggestions(String typo) {
        return dictionary.getSuggestions(typo);
    }

    // 사전을 변경할 수 있는 메서드 제공으로 유연성을 높일 시 동기화 문제 발생 가능
    public static void setDictionary(Lexicon dictionary) {
        SpellCheckerUtil.dictionary = dictionary;
    }
}
```

```java
public class SpellCheckerSingleton {

    private Lexicon dictionary = new EnglishDictionary();

    // 싱글톤 파트
    private static final SpellCheckerSingleton INSTANCE = new SpellCheckerSingleton();

    private SpellCheckerSingleton() {
    }

    public static SpellCheckerSingleton getInstance() {
        return INSTANCE;
    }
    // 싱글톤 파트

    public boolean isValid(String word) {
        return dictionary.isValid(word);
    }

    public List<String> suggestions(String typo) {
        return dictionary.getSuggestions(typo);
    }

    // 사전을 변경할 수 있는 메서드 제공으로 유연성을 높일 시 동기화 문제 발생 가능
    public void setDictionary(Lexicon dictionary) {
        this.dictionary = dictionary;
    }
}
```

아래와 같이 시나리오에 해당하는 작업을 정의해주고

```java
// 맞춤법 검사기 작업으로 사용할 사전, 사전에 있는 단어인지 검색 작업, 대체 단어 검색 작업 수행
public class SpellCheckTaskV1 implements Runnable {

    private final Lexicon dictionary;
    private final String word;
    private final String typo;

    public SpellCheckTaskV1(Lexicon dictionary, String word, String typo) {
        this.dictionary = dictionary;
        this.word = word;
        this.typo = typo;
    }

    @Override
    public void run() {
        // 사전 세팅
        SpellCheckerUtil.setDictionary(dictionary);

        // 사전 세팅에 0.5초 걸린다고 가정
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        // 사전에 있는 단어인지 검색 작업
        boolean isValid = SpellCheckerUtil.isValid(word);
        log("word = " + word + ", isValid = " + isValid);

        // 사전에 있는 단어인지 검색 작업에 0.5초 걸린다고 가정
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        // 대체 단어 검색 작업
        List<String> suggestions = SpellCheckerUtil.suggestions(typo);
        log("typo = " + typo + ", suggestions = " + suggestions);

        // 대체 단어 검색 작업에 0.5초 걸린다고 가정
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        log("종료");
    }
}
```

두 스레드를 실행하면 영어 사전에 대체 단어가 존재하는데 추천에 실패하는 것을 볼 수 있었다. 싱글톤 역시 마찬가지 였다. 영어 스레드만 실행하면 정상적으로 동작한다.

```java
public class DictionaryMainV1 {

    public static void main(String[] args) throws InterruptedException {
        // 영어 사전으로 맞춤법 검사기를 사용하는 스레드와 한국어 사전으로 맞춤법 검사기를 사용하는 스레드 생성
        Thread thread1 = new Thread(new SpellCheckTaskV1(new EnglishDictionary(), "app", "ag"), "영어 스레드");
        Thread thread2 = new Thread(new SpellCheckTaskV1(new KoreanDictionary(), "기린", "기무"), "한국어 스레드");

        // 영어 스레드 실행
        thread1.start();

        // 0.7초 대기
        Thread.sleep(700);

        // 한국어 스레드 실행
        thread2.start();

        // 출력 결과
        // 00:03:30.934 [   영어 스레드] word = app, isValid = true
        // 00:03:31.442 [   영어 스레드] typo = ag, suggestions = [] <- 대체 단어 조회에서 한국어 사전으로 변경돼서 조회 실패
        // 00:03:31.629 [  한국어 스레드] word = 기린, isValid = true
        // 00:03:31.946 [   영어 스레드] 종료
        // 00:03:32.135 [  한국어 스레드] typo = 기무, suggestions = [기러기, 기린, 기차]
        // 00:03:32.641 [  한국어 스레드] 종료
    }
}
```

---

## 의존 객체 주입

맞춤법 검사기에서 다양한 사전을 활용하는 방법은 간단한데 맞춤법 검사기 객체를 생성할 때 외부에서 사용할 사전을 넣어주면 된다. 맞춤법 검사기의 생성자에 실제 사전 객체를 넣어서 생성하고 맞춤법 검사기는 해당 사전을 변경 불가능하게 하면 된다.

```java
public class SpellCheckerDI {

    private final Lexicon dictionary;

    public SpellCheckerDI(Lexicon dictionary) {
        this.dictionary = dictionary;  // NPE 방지까지 하면 굿
    }

    public boolean isValid(String word) {
        return dictionary.isValid(word);
    }

    public List<String> suggestions(String typo) {
        return dictionary.getSuggestions(typo);
    }
}
```

```java
// 맞춤법 검사기 작업으로 사용할 사전, 사전에 있는 단어인지 검색 작업, 대체 단어 검색 작업 수행
public class SpellCheckTaskV3 implements Runnable {

    private final Lexicon dictionary;
    private final String word;
    private final String typo;

    public SpellCheckTaskV3(Lexicon dictionary, String word, String typo) {
        this.dictionary = dictionary;
        this.word = word;
        this.typo = typo;
    }

    @Override
    public void run() {
        // 사전 세팅
        SpellCheckerDI spellChecker = new SpellCheckerDI(dictionary);

        // 사전 세팅에 0.5초 걸린다고 가정
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        // 사전에 있는 단어인지 검색 작업
        boolean isValid = spellChecker.isValid(word);
        log("word = " + word + ", isValid = " + isValid);

        // 사전에 있는 단어인지 검색 작업에 0.5초 걸린다고 가정
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        // 대체 단어 검색 작업
        List<String> suggestions = spellChecker.suggestions(typo);
        log("typo = " + typo + ", suggestions = " + suggestions);

        // 대체 단어 검색 작업에 0.5초 걸린다고 가정
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        log("종료");
    }
}
```

```java
public class DictionaryMainV3 {

    public static void main(String[] args) throws InterruptedException {
        // 영어 사전으로 맞춤법 검사기를 사용하는 스레드와 한국어 사전으로 맞춤법 검사기를 사용하는 스레드 생성
        Thread thread1 = new Thread(new SpellCheckTaskV3(new EnglishDictionary(), "app", "ag"), "영어 스레드");
        Thread thread2 = new Thread(new SpellCheckTaskV3(new KoreanDictionary(), "기린", "기무"), "한국어 스레드");

        // 영어 스레드 실행
        thread1.start();

        // 0.7초 대기
        Thread.sleep(700);

        // 한국어 스레드 실행
        thread2.start();

        // 출력 결과
        // 00:42:05.509 [   영어 스레드] word = app, isValid = true
        // 00:42:06.018 [   영어 스레드] typo = ag, suggestions = [app, apple, audio]
        // 00:42:06.210 [  한국어 스레드] word = 기린, isValid = true
        // 00:42:06.520 [   영어 스레드] 종료
        // 00:42:06.717 [  한국어 스레드] typo = 기무, suggestions = [기차, 기린, 기러기]
        // 00:42:07.222 [  한국어 스레드] 종료
    }
}
```

의존 객체 주입을 통해 새로운 맞춤법 검사기를 생성하는 방식으로 잘 동작하는 걸 볼 수 있었다.

---

## Etc.

아이템 끝 부분에 `Supplier`에 대한 언급이 잠깐 나와서 `Supplier`를 활용한 지연 연산을 추가해봤다.

```java
public interface Beverage {

    String getName();
}

public class Coffee implements Beverage {

    public Coffee() {
        System.out.println("커피 만들기(아주 무거운 작업)");
    }

    @Override
    public String getName() {
        return "커피";
    }
}

public class Milk implements Beverage {

    public Milk() {
        System.out.println("우유 만들기(아주 무거운 작업)");
    }

    @Override
    public String getName() {
        return "우유";
    }
}
```

위처럼 음료 인터페이스와 이를 구현한 커피, 우유 클래스를 만들었다. 각 음료 객체는 생성에 고비용이 드는 객체이다.

```java
public class HomeParty {

    public void prepare(boolean isReady, Beverage beverage) {
        System.out.println("HomeParty.prepare 즉시 연산");
        if (!isReady) return;

        System.out.println(beverage.getName() + " 준비 완료");
    }

    public void prepare(boolean isReady, Supplier<? extends Beverage> supplier) {
        System.out.println("HomeParty.prepare 지연 연산");
        if (!isReady) return;

        Beverage beverage = supplier.get();
        System.out.println(beverage.getName() + " 준비 완료");
    }
}
```

이건 홈파티 클래스로 음료 준비 여부와 음료를 받아 준비 안됐으면 그냥 반환, 준비됐으면 음료를 반환하는 메서드와 음료 대신 `Supplier`를 받아 동일하게 처리하는 메서드를 오버로딩했다.

```java
public class HomeParty {

    public void prepare(boolean isReady, Beverage beverage) {
        System.out.println("HomeParty.prepare 즉시 연산");
        if (!isReady) return;

        System.out.println(beverage.getName() + " 준비 완료");
    }

    public void prepare(boolean isReady, Supplier<? extends Beverage> supplier) {
        System.out.println("HomeParty.prepare 지연 연산");
        if (!isReady) return;

        Beverage beverage = supplier.get();
        System.out.println(beverage.getName() + " 준비 완료");
    }
}
```

main에서 돌려보면 객체를 파라미터로 넘기면 객체 생성 후 메서드가 호출되는 구조로 준비 여부와 상관없이 객체를 생성하지만 `Supplier`를 대신 넘기면 메서드가 먼저 호출되고 `supplier.get()`에서 객체가 생성된다. `Optional`에서 `orElse()`랑 `orElseGet()`의 차이가 이건데 객체 생성 말고 메서드 반환 값을 넘기거나 연산 결과를 넘기는 등에서도 지연 연산을 통한 리소스 절약이 가능하다.

```java
public class SupplierMain {

    public static void main(String[] args) {
        HomeParty homeParty = new HomeParty();
        boolean isReady = false;

        homeParty.prepare(isReady, new Coffee());
        homeParty.prepare(isReady, new Milk());
        homeParty.prepare(isReady, () -> new Coffee());
        homeParty.prepare(isReady, () -> new Milk());

        // 커피 만들기(아주 무거운 작업)
        // HomeParty.prepare 즉시 연산

        // 우유 만들기(아주 무거운 작업)
        // HomeParty.prepare 즉시 연산

        // HomeParty.prepare 지연 연산

        // HomeParty.prepare 지연 연산

        isReady = true;
        homeParty.prepare(isReady, new Coffee());
        homeParty.prepare(isReady, new Milk());
        homeParty.prepare(isReady, () -> new Coffee());
        homeParty.prepare(isReady, () -> new Milk());

        // 커피 만들기(아주 무거운 작업)
        // HomeParty.prepare 즉시 연산
        // 커피 준비 완료

        // 우유 만들기(아주 무거운 작업)
        // HomeParty.prepare 즉시 연산
        // 우유 준비 완료

        // HomeParty.prepare 지연 연산
        // 커피 만들기(아주 무거운 작업)
        // 커피 준비 완료

        // HomeParty.prepare 지연 연산
        // 우유 만들기(아주 무거운 작업)
        // 우유 준비 완료
    }
}
```

![](./assets/photo1.png)
![](./assets/photo2.png)

---
