# Item 17 - 변경 가능성을 최소화하라

## 불변 클래스

불변 클래스란 간단히 말해 그 인스턴스의 내부 값을 수정할 수 없는 클래스다. 불변 인스턴스에 간직된 정보는 고정되어 객체가 파괴되는 순간까지 절대 달라지지 않는다. 자바 플랫폼 라이브러리에도 다양한 불변 클래스가 있다. String, 기본 타입의 박싱된 클래스들, BigInteger, BigDecimal이 여기 속한다. 이 클래스들을 불변으로 설계한 데는 그럴만한 이유가 있다. 불변 클래스는 가변 클래스보다 설계하고 구현하고 사용하기 쉬우며, 오류가 생길 여지가 적고 훨씬 안전하다.

불변 클래스를 만드는 방법은 다음 다섯 가지 규칙을 따르면 된다.

1. 객체의 상태를 변경하는 메서드를 제공하지 않는다.
2. 클래스를 확장할 수 없도록 한다.(하위 클래스에서 객체의 상태를 변하게 만드는 사태를 막아준다)
3. 모든 필드를 `final`로 선언한다.
4. 모든 필드를 `private`으로 선언한다.(기술적으로는 `public final`로 선언해도 불변 객체가 되지만 이는 공개 API가 된다)
5. 자신 외에는 내부의 가변 컴포넌트에 접근할 수 없도록 한다.

아래는 p.106의 코드 일부를 가져와 테스트한 것으로 복소수 클래스를 불변 클래스로 설계되었다. 연산 메서드는 새로운 `Complex` 인스턴스를 반환하여 자신의 인스턴스 상태가 바뀌지 않으며 메서드 체이닝을 통한 함수형 프로그래밍 패턴까지 적용되었다.

```java
public final class Complex {
    private final double re;
    private final double im;

    public Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    public Complex plus(Complex c) {
        return new Complex(re + c.re, im + c.im);
    }

    public Complex minus(Complex c) {
        return new Complex(re - c.re, im - c.im);
    }

    @Override
    public String toString() {
        return "Complex{" +
                "re=" + re +
                ", im=" + im +
                '}';
    }
}

public class ComplexMain {

    public static void main(String[] args) {
        Complex c1 = new Complex(1.0, 1.0);
        Complex c2 = new Complex(2.0, -2.0);

        Complex ans1 = c1.plus(c2);
        System.out.println("ans1 = " + ans1);  // ans1 = Complex{re=3.0, im=-1.0}

        Complex ans2 = c1.plus(c1).plus(c1).plus(c1);  // ans2 = Complex{re=4.0, im=4.0}
        System.out.println("ans2 = " + ans2);
    }
}
```

`Date` 클래스는 가변 클래스로 설계되어서 `Period` 클래스처럼 바로 참조를 저장하면 외부에서 변경할 여지가 있다. `Period2`처럼 주입된 값을 통해 새로운 객체를 생성해 참조를 저장하거나 아예 `LocalDate` 같은 불변 클래스를 활용해야 한다.

```java
public final class Period {
    private final Date start;
    private final Date end;

    public Period(Date start, Date end) {
        this.start = start;
        this.end = end;
    }

    public long getDateDiff() {
        return end.getTime() - start.getTime();
    }
}

public final class Period2 {
    private final Date start;
    private final Date end;

    public Period2(Date start, Date end) {
        this.start = new Date(start.getTime());
        this.end = new Date(end.getTime());
    }

    public long getDateDiff() {
        return end.getTime() - start.getTime();
    }
}

public class PeriodMain {

    public static void main(String[] args) {
        Date start = new Date();
        Date end = new Date();

        Period p = new Period(start, end);
        Period2 p2 = new Period2(start, end);

        System.out.println("p.getDateDiff() = " + p.getDateDiff());  // p.getDateDiff() = 0
        System.out.println("p2.getDateDiff() = " + p2.getDateDiff());  // p2.getDateDiff() = 0

        end.setYear(2026);
        System.out.println("p.getDateDiff() = " + p.getDateDiff());  // p.getDateDiff() = 59989680000000
        System.out.println("p2.getDateDiff() = " + p2.getDateDiff());  // p2.getDateDiff() = 0
    }
}
```

---

## 불변 클래스의 장단점

불변 클래스로 설계하면 객체가 생성된 시점의 상태를 파괴될 때까지 그대로 간직한다. 이런 불변 객체는 근본적으로 스레드 안전하여 따로 동기화를 할 필요가 없어 클래스를 스레드 안전하게 만드는 가장 쉬운 방법 중 하나이다.

또한 불변 클래스는 자주 사용되는 객체를 캐싱하여 같은 객체를 중복 생성하는 것을 방지하게 설계하여 메모리 사용량과 가비지 컬렉션 비용을 줄이게 나아갈 수 있다. 불변 클래스인 `String`의 경우 String 상수 풀을 활용하고 기본 타입의 박싱 클래스들도 일부 값을 캐싱하여 재사용한다. 객체에서 `==` 비교가 동일하게 나오는 것도 캐싱으로 재사용한 같은 객체이기 때문이다.

![](./assets/photo1.png)

불변 객체는 `Map`의 키, `Set`의 원소에 활용하기도 적합한데 객체의 값이 변하는 문제가 없기 때문이다.

아래처럼 가변 객체를 `Set`의 원소로 넣을 경우 예상치 못한 에러가 발생할 수 있다.

```java
public class User {

    public String name;

    public User(String name) {
        this.name = name;
    }

    @Override
    public boolean equals(Object object) {
        if (object == null || getClass() != object.getClass()) return false;
        User user = (User) object;
        return Objects.equals(name, user.name);
    }

    @Override
    public int hashCode() {
        return Objects.hashCode(name);
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                '}';
    }
}

public class UserMain {

    public static void main(String[] args) {
        Set<User> users = new HashSet<>();

        User user1 = new User("bobo");
        users.add(user1);
        System.out.println("users.size() = " + users.size());  // users.size() = 1

        user1.name = "bobo2";
        User user2 = new User("bobo2");
        users.add(user2);
        System.out.println("users.size() = " + users.size());  // users.size() = 2

        User user3 = new User("bobo2");
        users.add(user3);
        System.out.println("users.size() = " + users.size());  // users.size() = 2

        users.forEach(System.out::println);
        // User{name='bobo2'}
        // User{name='bobo2'}
    }
}
```

장점이 많은 불변 클래스이지만 단점도 있는데 일단 값이 다르면 반드시 독립된 새로운 객체로 만들어야 한다는 점이다. 다양한 최적화를 통해 개선해나가야 한다.

---
